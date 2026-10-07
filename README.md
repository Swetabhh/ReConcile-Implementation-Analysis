# ReConcile — Implementation Study & Independent Analysis

**Paper:** *ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs*  
**Venue:** ACL 2024  
**Original authors:** Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal

**Official paper:** [arXiv:2309.13007](https://arxiv.org/abs/2309.13007)  
**Official repository:** [dinobby/ReConcile](https://github.com/dinobby/ReConcile)

![ReConcile framework](https://i.imgur.com/mREgiI7.png)

## Overview

ReConcile is a multi-agent reasoning framework in which multiple language models independently attempt a problem and then interact across multiple rounds. Instead of relying on a single model response, the framework uses discussion, confidence information, and consensus to produce the final answer.

I worked through the authors' official codebase to understand how the system is implemented, with particular attention to:

- how different language-model backends are integrated;
- how demonstrations are selected and inserted into prompts;
- how initial independent responses are generated;
- how disagreement between agents is detected;
- how follow-up discussion rounds are constructed;
- how confidence values are converted into weighted votes;
- how raw model outputs are parsed, normalized, and evaluated.

![Multi-round discussion](https://i.imgur.com/4UmumgD.png)

## What This Repository Is (and Is Not)

This repository presents an **implementation-focused study** of ReConcile. It is not a from-scratch reimplementation of the method.

The study covers:

- the end-to-end path from dataset loading to final evaluation;
- how `run.py`, `generation.py`, `data_utils.py`, `utils.py`, and `claude.py` interact;
- how prompts are constructed for different model backends;
- how multi-round debate and consensus are implemented;
- how answers are normalized and confidence information is handled;
- how parsing and evaluation vary across datasets.

The research idea, original implementation, datasets, and reported results belong to the original authors.

## Repository Structure

```text
.
├── claude.py
├── data_utils.py
├── generation.py
├── run.py
├── utils.py
├── convincing/
│   ├── Aqua/
│   ├── ECQA/
│   ├── GSM8k/
│   └── SQA/
├── dataset/
│   ├── Aqua/
│   ├── ECQA/
│   ├── GSM8k/
│   └── SQA/
├── requirements.txt
├── LICENSE
└── README.md
```

## How the Pipeline Flows

```text
Dataset
   │
   ▼
Sample preparation
   │
   ▼
Initial responses
   ├───────────────┐
   ▼               ▼
 Claude           GPT
   └───────┬───────┘
           ▼
          Bard
           │
           ▼
    Output parsing
           │
           ▼
Consensus / weighted voting
           │
           ▼
      Debate prompt
           │
           ▼
    Additional rounds
           │
           ▼
       Evaluation
```

## Implementation-Level Observations

### 1. Model-specific interfaces

The released implementation does not expose one fully uniform interaction layer across all model providers. Claude, GPT, and Bard use different interaction patterns, so generation and debate logic are adapted for each model family.

This is visible in `generation.py`, where model-specific generation and debate functions are maintained separately.

### 2. Disagreement triggers additional discussion

After the initial round, `parse_output()` in `utils.py` converts model responses into a structured representation and checks whether the agents agree.

When disagreement remains, the agents' answers and explanations are incorporated into a debate prompt for a subsequent round.

This makes disagreement an explicit control signal in the multi-agent reasoning process.

### 3. Confidence contributes to consensus

`trans_confidence()` maps confidence values to discrete weights. The implementation then aggregates those weights across predicted answers, providing a weighted alternative to plain majority voting.

The mechanism therefore uses both the predicted answer and the model's reported confidence when computing consensus.

### 4. Output processing is a substantial part of the implementation

The prompting logic expects structured JSON responses, but the implementation also accounts for responses that are malformed or incomplete.

Parsing, normalization, fallback handling, and confidence conversion are therefore integral parts of the pipeline rather than a separate cosmetic post-processing stage.

## Datasets

The repository contains data for:

- StrategyQA (`SQA`)
- GSM8K (`GSM8k`)
- ECQA (`ECQA`)
- AQuA (`Aqua`)

The dataset files are stored under `dataset/`.

The `convincing/` directory contains precomputed examples used as demonstrations when constructing prompts.

## Environment and Reproducibility

The original repository specifies:

```text
Python 3.10.11
```

My local inspection environment was:

```text
Python 3.11.15
Intel Core i7-13620H
15.7 GB RAM
NVIDIA RTX 3050 6 GB
```

ReConcile relies primarily on external model APIs rather than local GPU inference for the model backends used by the released implementation. Running the original pipeline at full scale therefore depends on access to the relevant provider APIs and credentials.

### Local Validation

I syntax-checked the main Python modules in the repository with Python 3.11.15:

```text
run.py
generation.py
data_utils.py
utils.py
claude.py
```

All five compiled successfully without syntax errors.

This repository does **not** claim a full end-to-end reproduction of the paper's experiments.

## Running the Original Pipeline

The original implementation expects environment variables for its API integrations, including Azure OpenAI, PaLM, and Claude.

The original execution pattern is:

```powershell
python run.py --num_samples 100 --dataset SQA
```

Supported dataset identifiers include:

```text
SQA
GSM8k
ECQA
Aqua
```

The released implementation depends on older API interfaces and external credentials. The command above therefore documents the original execution path; it is not presented as evidence of a current paper-scale reproduction.

## Evidence and Scope of Validation

I separate three levels of evidence:

**Implementation inspection**  
I traced the source code and documented the main execution paths and design choices.

**Lightweight local validation**  
The main Python modules compile successfully in my local environment.

**Paper reproduction**  
I have not regenerated the paper's full experimental results in this repository, and I do not claim to have done so.

Keeping these levels separate avoids conflating understanding of a research implementation with independent reproduction of its reported experiments.

## Limitations

The released implementation reflects the API and model ecosystem available when the project was developed:

- the generation stack relies on older provider/API interfaces;
- running the models requires external credentials;
- the Claude path uses a third-party API wrapper;
- the prompting pipeline relies on structured model outputs and includes fallback parsing when those outputs are imperfect;
- the setup is not a modern provider-agnostic LLM serving framework.

As a result, reproducing the original environment today can require additional compatibility work.

## Research Questions Motivated by the Implementation

Tracing the implementation raises several questions about multi-agent LLM reasoning:

1. How should disagreement be represented so that debate focuses on consequential conflicts rather than superficial differences?
2. Under what conditions does confidence-weighted consensus outperform simple majority voting?
3. How many discussion rounds are useful before additional interaction stops providing meaningful benefit?
4. How should heterogeneous agents be selected or assigned complementary roles?
5. Can communication cost be reduced without losing the benefits of multi-agent reasoning?
6. How robust is consensus when participating agents share similar failure modes?

These are open research questions motivated by the implementation. They are not contributions claimed by this repository.

## Attribution and Scope

This repository builds on the authors' official implementation of:

**ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs**

**Original authors:** Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal  
**Venue:** ACL 2024  
**Official paper:** [arXiv:2309.13007](https://arxiv.org/abs/2309.13007)  
**Official repository:** [dinobby/ReConcile](https://github.com/dinobby/ReConcile)

The original research, implementation, datasets, and reported results belong to the original authors. This repository presents an implementation study with independent documentation and technical observations; it does not claim authorship of the ReConcile method or the authors' original code.

The original license and copyright notice are retained in [`LICENSE`](LICENSE).

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

Detailed code-level walkthrough:

[`docs/IMPLEMENTATION_NOTES.md`](docs/IMPLEMENTATION_NOTES.md)
