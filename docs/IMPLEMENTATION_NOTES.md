\# ReConcile: Implementation Notes



These are my notes from reading through the authors' released code for \[ReConcile: Round-Table Conference Improves Reasoning via Consensus among Diverse LLMs](https://aclanthology.org/2024.acl-long.381/) (ACL 2024) by Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal. Official repo: https://github.com/dinobby/ReConcile



I wanted to see how the paper's multi-agent idea turns into a running pipeline, so I worked through the main code paths myself: data prep, generation, parsing, confidence handling, consensus, discussion rounds, and evaluation. The research and the original implementation are the authors' work. These notes are just my account of how it fits together.



\---



\## 1. The pieces



The repo is a handful of Python modules:



| File | Role |

|---|---|

| `run.py` | Entry point. Picks the dataset, runs inference and discussion rounds, evaluates, saves results |

| `generation.py` | Separate generation and discussion paths for Claude, GPT, and Bard |

| `data\_utils.py` | Loads benchmark data into the structures the pipeline expects |

| `utils.py` | Prompts, parsing, normalization, confidence handling, consensus, discussion helpers, evaluation |

| `claude.py` | Claude-specific client |

| `convincing/` | Demonstration examples used in prompts |

| `dataset/` | The benchmark files |



\## 2. How a run goes



`run.py` picks a dataset, loads the samples and the convincing examples, and asks all three models for an initial answer. The outputs are cleaned and parsed, the answers are aggregated, and the current round is evaluated. If the agents disagree, it builds a discussion context, generates another round, and evaluates again. Results are saved at the end.



The command-line controls are `--dataset`, `--num\_samples`, and `--round`, so you can do small test runs without touching the source. That doesn't remove the external dependency problem, though, since the model calls still need their services and credentials.



\## 3. Datasets



Four benchmarks are supported, and the adapters in `data\_utils.py` turn each into a common sample format carrying the question, answer, explanation, and options where they exist.



| Dataset | File | Answer style |

|---|---|---|

| SQA | `dev.json` | yes / no |

| GSM8k | `test.jsonl` | number |

| ECQA | `cqa\_data\_test.csv` | option 1 to 5 |

| Aqua | `test.json` | option A to E |



\## 4. Prompts



Prompt construction lives mostly in `utils.py` and changes per benchmark, since each one expects a different answer format. Every prompt asks the model to reply with a structured response:



```json

{

&#x20; "reasoning": "",

&#x20; "answer": "",

&#x20; "confidence\_level": ""

}

```



What a model actually receives is more than the raw question. It can include instructions, demonstrations from `convincing/`, the question itself, answer-format constraints, and, in later rounds, what the other agents said.



\## 5. Why there are three separate model paths



There's no generic model interface. `generation.py` has separate logic for Claude, GPT, and Bard. They all do the same basic job of sending a context, getting raw text back, and cleaning it into a normalized result, but each backend works differently enough that they were written separately.



\*\*Claude.\*\* It goes through `claude.py`, which uses cookie-based access to `claude.ai`, creates a conversation, and sends prompts on the historical `claude-2` path. The generation layer wraps the request in retry and reconnect handling. The practical upshot is that this isn't local inference. It depends on an external service and the exact access method the original repo expected. The file also credits the external Claude-API project the authors built on.



\*\*GPT.\*\* It uses the legacy OpenAI Python interface with an Azure-style setup. The code builds the messages, sends the request, pulls out the content, and passes it into the same parsing and normalization as the others. The configuration comes from environment variables, not from values hard-coded in the script.



\*\*Bard.\*\* Its interaction path differs from the other two. The code prepares the main prompt, the convincing examples, and some extra context the prompting code needs. Since the structured reply isn't guaranteed, there's extra handling for responses that don't match the expected format. That says something about the whole codebase: formatting problems are treated as a normal operational issue, not something that won't happen.



\## 6. The structured-output boundary



Reasoning, answer, and confidence are the interface between generation and everything after it. Consensus, discussion, and evaluation don't work on raw model text. They work on whatever structured fields could be pulled out of it.



\## 7. `parse\_json()` isn't JSON parsing



Despite the name, it never calls `json.loads()`. It finds a dictionary-like substring in the raw reply, does some string cleanup, and runs `ast.literal\_eval()` on the result.



That makes it part of the model pipeline in a real sense. When a model returns something malformed, the parser decides whether anything usable gets recovered.



\## 8. Normalization



After extraction, answers are normalized per dataset so that surface differences don't affect comparisons:



| Dataset | Normalization |

|---|---|

| SQA | lowercase |

| Aqua | uppercase |

| GSM8k | string |

| ECQA | string |



\## 9. Confidence and consensus



`trans\_confidence()` turns a model's confidence into a discrete weight:



```text

confidence <= 0.6        -> 0.1

0.6 < confidence < 0.8   -> 0.3

0.8 <= confidence < 0.9  -> 0.5

0.9 <= confidence < 1    -> 0.8

confidence == 1          -> 1.0

```



So confidence actually drives the outcome. It isn't a field stored just for reporting.



The consensus step adds weights up per candidate answer. Say Claude answers A with confidence c1, GPT answers B with c2, and Bard answers A with c3. Then A scores weight(c1) + weight(c3), B scores weight(c2), and the highest total wins. That's different from just taking the answer of the single most confident agent.



Two team-level decisions are exposed: a majority-style decision (the most frequent answer) and a confidence-weighted one (the highest accumulated weight). The paper also discusses a Max Conf baseline, but the released `evaluate\_all()` doesn't report it as a separate field the way it does for `majority\_ans` and `weighted\_max`. Worth keeping in mind when you map the paper's terms onto the code's variable names.



\## 10. Disagreement and discussion



Once the first responses are parsed, the code checks whether the agents agree. If they all gave the same answer, that sample needs no discussion. If there are several distinct answers, it builds a discussion context from the competing responses.



That context can include the other agents' answers and reasoning, the current question, the convincing examples, and the formatting instructions. Each model is asked to reconsider and return a new structured response, which becomes the next round. So information flows explicitly from one round to the next, and later rounds don't start from scratch.



\## 11. Convincing examples



`convincing/` has one folder per benchmark (Aqua, ECQA, GSM8k, SQA) holding question and explanation examples that can be inserted into prompts. They play a different role from the test data. The benchmark supplies the problem to solve, and the convincing examples add demonstration context around it.



\## 12. Failure handling



The code deals with failure in several ways: retries on certain external request errors, reconnect handling for Claude, handling of malformed or incomplete outputs, fallback structured results, and normalization after cleaning.



The part I found most notable is that invalid outputs aren't always discarded. Depending on the dataset and the failure path, a fallback answer can be substituted, and for some datasets that fallback is randomized. So an invalid generation can still show up in evaluation as a real prediction.



Two related gaps: the confidence transform assumes values are in the intended range and there's no thorough validation for every possible bad value, and malformed outputs can land in fallback handling instead of being recorded as a clean "generation failed" state. Both matter when reading results, because output processing can change the effective prediction regardless of what the model reasoned.



\## 13. Evaluation



The system evaluates after every round, not just at the end: initial generation, round 0 evaluation, a discussion round, round 1 evaluation, and so on. Final outputs are written to result files, which keeps the intermediate rounds available instead of only the last prediction.



\## 14. The historical environment



The repo targets an older software and model landscape. It depends on older API and library versions, and the model-access layer expects services that have changed a lot since release. The Claude path is the most tightly coupled to the original access method.



Installing the requirements is not the same as reproducing the experiment. Exact reproduction would also depend on model availability, API behavior, credentials, model versions, and how the services and prompts behave today.



\## 15. Resources



```text

OS       Windows 11

CPU      Intel Core i7

GPU      NVIDIA RTX 3050 6GB

RAM      16GB

Storage  512GB SSD

```



The GPU isn't the limiting factor, because generation goes to external services and nothing is trained locally. What actually limits things is the legacy interfaces, the credentials, service availability, and the volume of repeated API calls across discussion rounds. Code tracing, reconstruction, parsing analysis, and reasoning about the consensus mechanism need very little compute.



\## 16. What you can study without running full inference



Quite a lot can be understood straight from the code: prompt construction, expected output structure, parsing behavior, the confidence transform, answer-level aggregation, disagreement detection, discussion-context construction, normalization, fallback behavior, and the round-by-round evaluation logic. That's why I scoped this as an implementation study and not an attempt at full reproduction.



\## 17. What I took away



1\. \*\*Model diversity is explicit.\*\* Each model family has its own generation function, with no shared abstraction.

2\. \*\*Parsing is a critical boundary.\*\* Everything downstream relies on structured fields being extracted successfully.

3\. \*\*Confidence shapes the team decision.\*\* It's discretized and then aggregated by answer.

4\. \*\*Discussion is conditional.\*\* Extra reasoning only happens when the agents disagree.

5\. \*\*Later rounds build on earlier ones.\*\* They're conditioned on what came before.

6\. \*\*Demonstrations affect prompting.\*\* Convincing examples can be part of the model's context.

7\. \*\*Failure handling can change predictions.\*\* Malformed generations may become fallback predictions rather than being removed.

8\. \*\*The code is historically coupled.\*\* Its model interfaces reflect the systems available when it was written.



\## 18. What I actually did



I worked through the implementation without modifying the authors' core research code. I followed the path from the paper description through `run.py`, `data\_utils.py`, `generation.py`, and `utils.py`, and on into discussion, consensus, and evaluation. That let me understand how a benchmark becomes a prompt, how each model receives it, how outputs get structured, how confidence becomes a vote, how disagreement becomes a discussion state, how later rounds reuse earlier reasoning, and how the final answer is scored.



\## 19. Reproducibility



This work isn't a reproduction of the ACL 2024 benchmark results. Three things are worth keeping apart:



\- results reported by the authors,

\- results independently reproduced,

\- observations about the implementation.



My notes fall into the third category, along with my own reconstruction of the code paths.



\## 20. Limitations I found in the code



1\. The model interfaces are tied to historical services.

2\. The parser expects a tightly constrained response format.

3\. Dataset-specific answer handling adds special cases.

4\. Invalid generations can go through fallback behavior.

5\. The confidence transform assumes values in an expected range.

6\. Running the main path needs external credentials and services.

7\. Because of all this, recreating the original environment exactly is hard today.



These are observations about the released implementation, not changes I made to it.



\## 21. Scope and attribution



This repo is meant to show that I can read research code, reconstruct how it works, connect code to concepts, and reason about multi-agent coordination, confidence-aware consensus, and reproducibility limits, rather than leaning only on a paper summary.



It doesn't claim ownership of the ReConcile method, authorship of the original code, a new consensus algorithm, a full reproduction of the ACL 2024 results, or state-of-the-art performance.



All credit for the research and the implementation goes to Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal. The original license and upstream attribution are kept in `LICENSE`.
