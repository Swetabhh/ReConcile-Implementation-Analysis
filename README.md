# ReConcile: Implementation Study & Independent Analysis

**Paper:** *ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs* (ACL 2024)
**Original authors:** Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal

**Paper:** [arXiv:2309.13007](https://arxiv.org/abs/2309.13007)
**Official repo:** [dinobby/ReConcile](https://github.com/dinobby/ReConcile)

![ReConcile framework](https://i.imgur.com/mREgiI7.png)

## Overview

ReConcile is a multi-agent reasoning framework. Several language models each take a first, independent shot at a problem, then talk it over across a few rounds. Rather than trusting one model's answer, the final result comes from their discussion, how confident each one is, and whether they end up agreeing.

I read through the authors' official codebase to figure out how the system actually works. I paid the most attention to:

- how the different model backends are plugged in;
- how demonstrations get picked and dropped into prompts;
- how the first round of independent answers is generated;
- how the code notices that agents disagree;
- how the follow-up discussion prompts are built;
- how confidence scores turn into weighted votes;
- how raw model output is parsed, cleaned up, and evaluated.

![Multi-round discussion](https://camo.githubusercontent.com/aad0afbd7e71ccb2b33ca6f01fe498ccb9fff8ed6a22d3a1d016cdde142d178e/68747470733a2f2f692e696d6775722e636f6d2f34754d756d67442e706e67)

## What this repo is (and isn't)

This is a study of the existing implementation. I did not rebuild ReConcile from scratch.

Here's what I walked through:

- the full path from loading a dataset to producing evaluation results;
- how `run.py`, `generation.py`, `data_utils.py`, `utils.py`, and `claude.py` fit together;
- how prompts differ from one model backend to the next;
- how the multi-round debate and consensus steps work in code;
- how answers are normalized and confidence is handled;
- how parsing and evaluation change from dataset to dataset.

The research idea, the original code, the datasets, and the reported results all belong to the original authors.

## Repository structure

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

## How the pipeline flows

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

## What I noticed in the code

### 1. Each model has its own interface

The released code doesn't give you one clean, uniform layer for talking to every provider. Claude, GPT, and Bard are each called a little differently, so the generation and debate logic is adapted per model family. You can see this in `generation.py`, where the model-specific functions live side by side but are kept separate.

### 2. Disagreement is what triggers more discussion

After the first round, `parse_output()` in `utils.py` turns each model's response into a structured form and checks whether the agents agree. If they don't, their answers and explanations get folded into a debate prompt for the next round. In practice, disagreement acts as the signal that says "keep talking."

### 3. Confidence feeds into the vote

`trans_confidence()` maps confidence values onto a small set of discrete weights, and those weights are summed across each candidate answer. The result is a weighted alternative to plain majority voting, so the consensus depends on both what a model answered and how sure it said it was.

### 4. Parsing is a bigger deal than it looks

The prompts ask for structured JSON, but models don't always comply, and the code has to cope with malformed or incomplete responses. Parsing, normalization, fallbacks, and confidence conversion end up being a real part of the pipeline, not just tidying up at the end.

## Datasets

The repo includes data for four datasets, all under `dataset/`:

- StrategyQA (`SQA`)
- GSM8K (`GSM8k`)
- ECQA (`ECQA`)
- AQuA (`Aqua`)

The `convincing/` folder holds precomputed examples that get used as demonstrations when prompts are built.

## Environment and reproducibility

The original repo specifies:

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

The released implementation leans on external model APIs rather than local GPU inference, so the GPU isn't really a factor. Running the pipeline at full scale depends on having access to the relevant providers and credentials.

### Local validation

I syntax-checked the main Python modules with Python 3.11.15:

```text
run.py
generation.py
data_utils.py
utils.py
claude.py
```

All five compiled with no syntax errors. That's the extent of it: **I haven't reproduced the paper's experiments end to end, and this repo doesn't claim to.**

## Running the original pipeline

The original code expects environment variables for its API integrations (Azure OpenAI, PaLM, and Claude). The original way to run it is:

```powershell
python run.py --num_samples 100 --dataset SQA
```

The dataset identifiers you can pass are:

```text
SQA
GSM8k
ECQA
Aqua
```

The code depends on older API interfaces and external credentials, so this command just documents how the authors ran things. It isn't evidence that I've reproduced their results today.

## What I did and didn't verify

I want to be clear about three different levels of evidence:

- **Implementation inspection.** I traced the source and wrote down the main execution paths and design choices.
- **Light local validation.** The main modules compile in my environment.
- **Paper reproduction.** I haven't regenerated the paper's full results, and I'm not claiming I have.

Understanding how a research codebase works is not the same thing as independently reproducing its experiments, and I'd rather not blur the two.

## Limitations

The code reflects the API and model landscape at the time the project was built:

- generation relies on older provider interfaces;
- running the models needs external credentials;
- the Claude path goes through a third-party API wrapper;
- the prompts depend on structured outputs, with fallback parsing for when those come back imperfect;
- it's not a modern, provider-agnostic LLM serving setup.

Because of all this, getting the original environment working today can take some extra compatibility effort.

## Questions this raised for me

Going through the implementation left me wondering about a few things in multi-agent LLM reasoning:

1. How should disagreement be represented so debate focuses on conflicts that matter, not surface-level differences?
2. When does confidence-weighted consensus actually beat simple majority voting?
3. How many discussion rounds are worth it before extra interaction stops helping?
4. How should different agents be chosen, or given complementary roles?
5. Can communication costs come down without losing the benefit of multiple agents?
6. How well does consensus hold up when the agents share similar failure modes?

These are open questions the code made me think about. I'm not claiming them as contributions of this repo.

## Attribution

This repo builds on the authors' official implementation of:

**ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs**

**Original authors:** Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal
**Venue:** ACL 2024
**Paper:** [arXiv:2309.13007](https://arxiv.org/abs/2309.13007)
**Official repo:** [dinobby/ReConcile](https://github.com/dinobby/ReConcile)

The research, the original code, the datasets, and the reported results are the original authors' work. What's here is my own documentation and technical observations. I'm not claiming authorship of the ReConcile method or the authors' code.

The original license and copyright notice are kept in [`LICENSE`](LICENSE).

## Citation

```bibtex
@inproceedings{chen-etal-2024-reconcile,
    title = "{R}e{C}oncile: Round-Table Conference Improves Reasoning via Consensus Among Diverse {LLM}s",
    author = "Chen, Justin Chih-Yao and Saha, Swarnadeep and Bansal, Mohit",
    booktitle = "Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)",
    year = "2024"
}
```

## Implementation notes

For a more detailed, code-level walkthrough, see [`docs/IMPLEMENTATION_NOTES.md`](docs/IMPLEMENTATION_NOTES.md).
