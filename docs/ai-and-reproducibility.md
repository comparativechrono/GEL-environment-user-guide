# AI and reproducibility

[Back to the guide](../README.md)

AI tools are now part of research: in Wiley’s 2025 survey of 2,430 researchers, 84% had used them, up from 57% in 2024. They are good at explaining error messages, writing and translating code, and drafting text. The R workbench and r-training practicals have an AI tab for exactly this: an assistant that explains errors, and an agent that carries out a task in the terminal and keeps a record of every step (`agent-record.md`).

**There is no AI in the Research Environment, and no way to reach one.** You need to be able to do the analysis yourself.

## Why not inside a trusted research environment

- **No internet, so no AI service.** Genomics England: “There are no coding models available within the Research Environment, we also do not allow connections from the Research Environment to external AI models.”
- **Nothing can be copied out to ask about**, and the Airlock lets out results, not data.
- **An agent has to read your data to work**, and everything it reads is sent to the AI company. In a TRE, that would mean sending participants’ data outside.
- **Trained models can’t leave either**: they can memorise their training data.

The simulator’s RStudio has no AI tab, on purpose.

## Four problems with AI, wherever you use it

1. **It can be confidently wrong.** In 576,000 code samples from 16 models, 19.7% of the 2.23 million packages the code referred to did not exist (Spracklen et al., USENIX Security 2025). Models also invent functions, arguments and references, in the same confident tone.
2. **It isn’t reproducible.** The same prompt can give different answers: one prompt run 1,000 times at temperature 0 gave 80 different completions (He and Thinking Machines Lab, 2025). And models are retired: a request to a retired model fails.
3. **It sees what you send.** Your prompts go to a company, and consumer chatbots may keep them, or train on them unless you opt out. **Never paste personal or patient data into an AI tool.**
4. **It may refuse biology.** Frontier models restrict biology to prevent misuse: Anthropic launched Claude Fable 5 in 2026 “with almost all biology queries blocked”, and OpenAI says its safeguards “will sometimes accidentally prevent safe uses”. Fuller access is offered to verified organisations, not to individuals.

## Learn to do it yourself

Inside the RE you have:

- **The error message**: read it slowly; it usually says what is wrong.
- **R’s and Python’s own help**: `?mean`, `help(mean)`, vignettes, and the RE documentation, which is whitelisted.
- **Small steps**: run a little at a time, and check each result.
- **Scripts, not console history**: they are your record.
- **People**: a colleague, your statistician, the course team.

Outside the RE, AI is a good way to learn. Practise with it on public or synthetic data, then do the analysis yourself inside. If you can’t explain a line of your code, you can’t defend it to a reviewer, or to the Airlock team.

## Using AI reproducibly (outside the RE)

- Record the **tool, the model and its version, and the date**.
- Keep the **prompts and the answers**, as the workbench agent does in `agent-record.md`.
- Keep the **code, not the chat**, as the record of the analysis, and **test** it on data where you know the answer, such as simulated data.
- **Disclose it.** The ICMJE recommendations ask journals to require disclosure of AI use; Springer Nature journals want it described in the methods; an AI tool cannot be an author (COPE). Genomics England’s publication policy asks for the same disclosure. For studies of language models themselves there is a reporting guideline, TRIPOD-LLM (Gallifant et al., *Nature Medicine* 2025).
- **You are responsible for every line, whoever wrote it.**

## Pre-registration matters more with AI

Every analysis involves choices: which participants, which covariates, how to code the outcome, which test. An agent can try a hundred combinations in minutes and report the one that “works”. A registered plan says in advance which analysis counts; everything else is reported as exploratory. So pre-registration matters more with AI, not less.

## Sources

- Anthropic (2026) [Improving Fable 5’s biology safeguards](https://anthropic.com/news/improving-fable-5-s-biology-safeguards)
- COPE (2023) [Authorship and AI tools](https://publicationethics.org/guidance/cope-position/authorship-and-ai-tools)
- Gallifant J et al. (2025) The TRIPOD-LLM reporting guideline for studies using large language models. *Nat Med* 31, 60–69
- Genomics England, [RE documentation](https://re-docs.genomicsengland.co.uk/): the whitelist, the Airlock rules, and the Airlock and machine learning
- GRAIMATTER (2022) [Recommendations for disclosure control of trained machine learning models from trusted research environments](https://doi.org/10.5281/zenodo.7089491)
- He H, Thinking Machines Lab (2025) [Defeating nondeterminism in LLM inference](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference)
- ICMJE (2026) [Recommendations](https://www.icmje.org/icmje-recommendations.pdf)
- OpenAI (2025) [ChatGPT agent system card](https://openai.com/index/chatgpt-agent-system-card)
- Spracklen J et al. (2025) We have a package for you! A comprehensive analysis of package hallucinations by code generating LLMs. USENIX Security 2025 ([arXiv:2406.10279](https://arxiv.org/abs/2406.10279))
- Springer Nature, [AI guidance for researchers](https://group.springernature.com/gp/group/ai/ai-guidance-for-our-researchers-and-communities)
- Wiley (2025) [ExplanAItions: AI adoption jumps to 84% among researchers](https://newsroom.wiley.com/press-releases/press-release-details/2025/AI-Adoption-Jumps-to-84-Among-Researchers-as-Expectations-Undergo-Significant-Reality-Check/default.aspx)
