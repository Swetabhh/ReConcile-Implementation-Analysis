ReConcile: Implementation Study and Independent Analysis
Paper: ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs (ACL 2024)
Original Authors: Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal.

arXiv:2309.13007 · dinobby/ReConcile

ReConcile Framework
Overview
ReConcile is a multi-agent reasoning framework. We have several language models make their own initial attempt to solve a problem, and then they discuss the solution across a couple of rounds of conversation. The final answer is derived from the discussion between models, their confidence in their answer, and whether they reached a consensus.

By reviewing the official codebase of the authors, I was able to understand how this system works. The areas I focused on included:
- Model backend integration
- Selecting and inserting demonstration prompts
- Generating the initial answers
- Identifying when models disagree
- Creating the follow-up discussion prompts
- Confidence values to weighted votes
- Extracting, parsing, and evaluating model outputs
Multi-Round Discussion

What this repo contains
This is an analysis of an existing implementation, I didn’t create ReConcile from scratch.

I walked through:
- The overall flow of the code from loading the dataset, through to evaluation.
- How run.py, generation.py, data_utils.py, utils.py and claude.py work together.
- How prompts vary depending on the model backend.
- How multi-round debating and consensus is implemented.
- How answers are normalised, and confidence is handled.
- How parsing/evaluation varies depending on the dataset.
All research, code, datasets, and results are the original authors’.
Directory Structure
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
Pipeline Flow

Interesting points identified in code
Each model has its own interface
As this is a released implementation, there is no unified interface for each model provider. Each model (Claude, GPT and Bard) are handled slightly differently. This means that there are variations in the way the models are generated and debated depending on the model family, as seen in generation.py.
Disagreement triggers extra discussion
Once the initial responses have been created, parse_output() will extract the responses in a structured format. This will identify if the models disagreed, and if so, insert the answers and explanations into a debate prompt. This means disagreement triggers extra discussion.
Confidence converted to weighted votes
trans_confidence() will convert the confidence values to a limited number of weighted values. These weights will be added for each candidate answer. The weighted values are used to provide a weighted version of majority voting. This means the consensus is based on the answer given, and the confidence.
Parsing of model outputs
The prompts ask for structured JSON outputs. However, models aren’t always consistent with this. This means that there is some parsing of the outputs.
Datasets
There are 4 datasets provided within this repo:
- StrategyQA (SQA)
- GSM8K (GSM8k)
- ECQA (ECQA)
- AQuA (Aqua)
Precomputed examples are stored in the convincing/ directory.
Environment
The original repo specifies that it requires:
- Python 3.10.11
However, my environment is:
- Python 3.11.15
- Intel Core i7-13620H
- 15.7 GB RAM
- NVIDIA RTX 3050 6 GB
The released implementation uses external model APIs. This means that the GPU isn’t really a factor. To run the pipeline fully, you require access to the respective providers.
Validation
I validated the main Python files for syntax issues using Python 3.11.15:
- run.py
- generation.py
- data_utils.py
- utils.py
- claude.py
All five files had no syntax errors. However, I haven’t reproduced the experiments in the paper. This repo makes no claims to doing so.
Original Pipeline
The original code requires environment variables for the API integrations (Azure OpenAI, PaLM, Claude). The original way to run this would be:
python run.py --num_samples 100 --dataset SQA
The dataset names that can be passed are:
- SQA
- GSM8k
- ECQA
- Aqua
Due to the use of older API’s and external credentials, this is just showing how the authors ran the code and does not indicate that I have reproduced their results today.
What I verified
I’d like to clarify what I have validated:
- Implementation Review: I have followed the source code and noted the main execution paths and design decisions.
- Minimal Local Validation: Main modules compile in my environment.
- No Full Reproduction: I have not reproduced the full results in the paper.
It isn’t the same to understand how a research codebase operates as it is to reproduce the experiments. I’d rather not confuse the two.
Limitations
There are limitations with the code due to the state of the APIs and models during the creation of this project:
- Older model interfaces are used for generation.
- External credentials are required to run the models.
- Claude uses a third party API wrapper.
- Prompts require structured outputs and parsing fallbacks.
- Not using a modern, model agnostic LLM serving setup.
As a result, setting up the original environment can be a bit of a struggle.
Points of Interest
Whilst going through the implementation I was left thinking about the following points regarding multi-agent LLM reasoning:
- How should we represent disagreement to ensure we debate meaningful disagreements?
- When is confidence weighted consensus better than majority voting?
- How many rounds of discussion should we do?
- How should we select different agents, and give them complementary roles?
- How can we reduce communication overhead whilst still benefiting from using multiple agents?
- How robust is consensus when models fail similarly?
These are just points I thought about, and are not contributions from this repo.
Attribution
This repo builds upon the official implementation from the authors of:

ReConcile: Round-Table Conference Improves Reasoning via Consensus Among Diverse LLMs
Original Authors: Justin Chih-Yao Chen, Swarnadeep Saha, and Mohit Bansal
Venue: ACL 2024
Paper: arXiv:2309.13007
Repo: dinobby/ReConcile.

All research, code, datasets and results are the original authors. This is my own documentation and observations.

The original license and copyright information can be found in LICENSE.
Citation
@inproceedings{chen-etal-2024-reconcile,
title = "{R}e{C}oncile: Round-Table Conference Improves Reasoning via Consensus Among Diverse {LLM}s",
author = "Chen, Justin Chih-Yao and Saha, Swarnadeep and Bansal, Mohit",
booktitle = "Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)",
year = "2024"
}
Implementation Notes
See docs/IMPLEMENTATION_NOTES.md for more detailed notes.
