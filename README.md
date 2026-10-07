# ReConcile: Implementation Study & Independent Analysis

**Paper:** *ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs*
**Venue:** ACL 2024
**Original authors:** Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal

**Official paper:** [arXiv:2309.13007](https://arxiv.org/abs/2309.13007)
**Official repository:** [dinobby/ReConcile](https://github.com/dinobby/ReConcile)

![ReConcile framework](https://i.imgur.com/mREgiI7.png)

## Overview

ReConcile is a multi-agent reasoning framework. Several different language models each try a problem on their own, then talk it over across multiple rounds. Instead of trusting one model's answer, the system uses the discussion to reach a consensus.

I went through the authors' official codebase to understand how it actually works under the hood, in particular:

- how the different language-model backends are plugged in;
- how demonstrations end up in the prompts;
- how the first, independent answers are generated;
- how the system notices that agents disagree;
- how the follow-up discussion rounds are put together;
- how confidence values turn into weighted votes;
- how raw model outputs get cleaned, parsed, and evaluated.

![Multi-round discussion](https://i.imgur.com/4uMumgD.png)

## What this repo is (and isn't)

This is an **implementation study**. It is not a from-scratch reimplementation of ReConcile.

What I did was read and document the existing research code, including:

- the full path from loading a dataset to final evaluation;
- how `run.py`, `generation.py`, `data_utils.py`, `utils.py`, and `claude.py` work together;
- how prompts are built for each model backend;
- how the multi-round debate and consensus step works;
- how answers are normalized and confidence is handled;
- how parsing and evaluation differ from dataset to dataset.

The research idea, the implementation, the datasets, and the reported results all belong to the original authors.

## Repository structure

```text
.
â”œâ”€â”€ claude.py
â”œâ”€â”€ data_utils.py
â”œâ”€â”€ generation.py
â”œâ”€â”€ run.py
â”œâ”€â”€ utils.py
â”œâ”€â”€ convincing/
â”‚   â”œâ”€â”€ Aqua/
â”‚   â”œâ”€â”€ ECQA/
â”‚   â”œâ”€â”€ GSM8k/
â”‚   â””â”€â”€ SQA/
â”œâ”€â”€ dataset/
â”‚   â”œâ”€â”€ Aqua/
â”‚   â”œâ”€â”€ ECQA/
â”‚   â”œâ”€â”€ GSM8k/
â”‚   â””â”€â”€ SQA/
â”œâ”€â”€ requirements.txt
â”œâ”€â”€ LICENSE
â””â”€â”€ README.md
```

### How the pipeline flows

```text
Dataset
   â”‚
   â–¼
Sample preparation
   â”‚
   â–¼
Initial responses
 â”Œâ”€â”´â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
 â–¼                 â–¼
Claude            GPT
 â””â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
         â–¼
        Bard
         â”‚
         â–¼
   Output parsing
         â”‚
         â–¼
 Consensus / weighted voting
         â”‚
         â–¼
   Debate prompt
         â”‚
         â–¼
   Additional rounds
         â”‚
         â–¼
      Evaluation
```

## Things I noticed in the code

### 1. Each model has its own interface

There's no single, unified model abstraction here. Claude, GPT, and Bard each expect a different interaction format, so the generation logic is adapted separately for each one.

You can see this most clearly in `generation.py`, which keeps separate generation and debate functions for each model family.

### 2. Disagreement is what starts a debate

After the first round, `parse_output()` in `utils.py` turns the model predictions into a structured form and checks whether the agents agree.

If they don't all agree, their answers and explanations get pulled into a debate prompt for the next round.

### 3. Confidence counts toward the final answer

`trans_confidence()` maps confidence values onto discrete weights, and the code adds those weights up for each predicted answer. The result is a weighted alternative to plain majority voting.

### 4. A lot of the work is in cleaning up outputs

The code asks models to return structured JSON, but it also expects that outputs will sometimes be malformed or incomplete. Parsing, normalization, fallback predictions, and confidence conversion are all handled explicitly.

That makes the output-handling layer a real part of the research implementation, not just post-processing tacked on at the end.

## Datasets

The repo includes the data the original implementation uses for:

- StrategyQA (`SQA`)
- GSM8K (`GSM8k`)
- ECQA (`ECQA`)
- AQuA (`Aqua`)

The dataset files are already in `dataset/`.

There are also precomputed "convincing" examples in `convincing/`. The code uses these as demonstrations when it builds prompts.

## Environment & reproducibility

The original repo says it was tested with:

```text
Python 3.10.11
```

My local setup for inspecting it:

```text
Python 3.11.15
Intel Core i7-13620H
15.7 GB RAM
NVIDIA RTX 3050 6 GB
```

ReConcile mostly runs on **API calls, not GPU power**, so my RTX 3050 isn't a bottleneck for reading or inspecting the code. Running it at full scale depends on having access to the external model APIs and the credentials the original code expects.

### Local validation

I syntax-checked the repo's Python files with Python 3.11.15:

```text
run.py
generation.py
data_utils.py
utils.py
claude.py
```

All five compiled without any syntax errors.

I have **not** done a full end-to-end reproduction of the paper's experiments, and I'm not claiming to.

## Running the original pipeline

The original code expects environment variables for its API integrations, which include Azure OpenAI, PaLM, and Claude.

The original way to run it is:

```bash
python run.py --num_samples 100 --dataset SQA
```

The dataset identifiers it supports are:

```text
SQA
GSM8k
ECQA
Aqua
```

The code depends on older API interfaces and on credentials you supply yourself, so please read that command as **how the original authors run it**. It doesn't mean you can reproduce the paper-scale experiments today.

## How much I can actually vouch for

I want to be clear about three different levels of evidence:

**Implementation inspection**
I read through the source code and syntax-checked it locally.

**Lightweight local validation**
The Python files compile fine in my environment.

**Paper reproduction**
I have not regenerated the paper's full experimental results in this repo.

I keep these separate on purpose. Understanding an implementation and reproducing a paper are related, but they aren't the same claim.

## Limitations

A few parts of the original implementation reflect the model ecosystem that existed when the project was built:

- the generation stack relies on older API interfaces;
- you need external credentials to run the models;
- the Claude integration goes through a third-party API wrapper;
- models are expected to follow JSON formatting instructions, with fallback logic for when they don't;
- the setup isn't a modern, provider-agnostic LLM serving stack.

Because of all this, recreating the original environment is quite different from just running the repo on a current machine.

## Questions this made me curious about

Working through the code left me with a few research questions about multi-agent LLM reasoning:

- How should disagreement be represented so that debates focus on conflicts that matter, not superficial differences?
- When does confidence-weighted consensus beat simple majority voting?
- How many discussion rounds are worth it before extra interaction stops helping?
- How should different agents be chosen, or given complementary roles?
- Can communication cost come down without losing the benefits of multi-agent reasoning?
- What happens to consensus when the agents share similar failure modes?

These are questions the study raised for me. They aren't contributions this repo is claiming.

## Attribution and scope

This repo builds on the authors' official implementation of **ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs**.

The original research, implementation, datasets, and reported results belong to the original authors:

**Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal**

What I'm adding is an **implementation study with my own documentation and analysis**. I'm not claiming authorship of the ReConcile method or of the authors' original code.

See:

- [Official paper](https://arxiv.org/abs/2309.13007)
- [Official repository](https://github.com/dinobby/ReConcile)
- [`LICENSE`](./LICENSE)

## Citation

```bibtex
@inproceedings{chen-etal-2024-reconcile,
    title = "{R}e{C}oncile: Round-Table Conference Improves Reasoning via Consensus Among Diverse {LLM}s",
    author = "Chen, Justin Chih-Yao and Saha, Swarnadeep and Bansal, Mohit",
    booktitle = "Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)",
    year = "2024"
}
```


## Implementation Notes
Detailed code-level walkthrough: [docs/IMPLEMENTATION_NOTES.md](docs/IMPLEMENTATION_NOTES.md)
