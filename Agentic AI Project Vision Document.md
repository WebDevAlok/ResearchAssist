This decument describes a project which is much more interesting than a simple **“chat with PDFs” RAG application**.

We would design it as a **Research Copilot / Autonomous Research Agent** whose job is to move through the scientific workflow:

> **Papers → Understanding → Literature Map → Research Gap → Hypothesis → Experimental Design → Colab Execution → Analysis → Iteration → Paper**

The important distinction is that the agent should **not invent a research contribution and then write a plausible-looking paper**. It should maintain an evidence trail from literature → hypothesis → code → results → claims.

MiniMax is a reasonable core model for this. Its current API exposes models such as M2.7, and MiniMax describes its newer models as supporting long-context and agentic/code workflows. ([MiniMax][1])

## 1. The system to be built

Think of the architecture like this:

```text
                    ┌──────────────────────┐
                    │     Researcher       │
                    │  You + your papers   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Research Manager   │
                    │      AI Agent        │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼─────────────────┐
              ▼                ▼                 ▼
       ┌─────────────┐ ┌──────────────┐ ┌──────────────┐
       │ Paper Reader│ │ Literature   │ │ Experiment   │
       │             │ │ Researcher   │ │ Scientist    │
       └─────────────┘ └──────────────┘ └──────────────┘
              │                │                 │
              ▼                ▼                 ▼
       PDFs / tables      OpenAlex / S2      Colab
       figures / refs    citations / papers  GPU / CPU
              │                │                 │
              └────────────────┼─────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │ Evidence / Knowledge │
                    │       Store          │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Research Analyst   │
                    │ stats / plots /      │
                    │ error analysis       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Paper Generator    │
                    │ IMRaD + references   │
                    └──────────────────────┘
```

But I would make one architectural principle absolutely central:

### **The agent must distinguish facts, hypotheses, results and interpretations.**

For example:

```text
LITERATURE FACT
"Paper X reports 91.3% accuracy."

         ↓

OBSERVATION
"Paper X does not evaluate on Dataset Y."

         ↓

HYPOTHESIS
"Method X may generalize differently to Dataset Y."

         ↓

EXPERIMENT
"Compare X against A/B/C under identical preprocessing."

         ↓

RESULT
"X achieved 87.2 ± 0.8%."

         ↓

INTERPRETATION
"Performance decreased relative to the reported benchmark."

         ↓

CLAIM
"Under our cross-dataset protocol, X does not retain
its original benchmark performance."
```

That separation will make the eventual paper **much more scientifically defensible**.

---

# 2. Give the agent several specialized roles

Don't make one enormous prompt saying:

> "You are a research scientist. Read these papers and write a paper."

Instead, create specialized agents/modules.

### Agent 1 — Paper Reader

Input:

```text
paper.pdf
```

Output structured information:

```json
{
  "problem": "...",
  "research_question": "...",
  "hypothesis": "...",
  "method": "...",
  "dataset": "...",
  "preprocessing": "...",
  "baselines": [],
  "metrics": [],
  "results": [],
  "limitations": [],
  "future_work": [],
  "important_equations": [],
  "important_figures": [],
  "claims": [],
  "evidence": []
}
```

Most importantly, every extracted claim should have a location:

```text
Claim:
"Method X improves robustness."

Evidence:
page 7, paragraph 3
```

That gives you **citation-grounded reasoning**.

---

# 3. Build a Research Knowledge Graph

This is where your project becomes substantially more interesting.

Instead of simply putting papers into a vector database, represent research as relationships:

```text
Paper A
 ├── proposes → Method X
 ├── uses → Dataset D
 ├── compares → Method Y
 ├── cites → Paper B
 ├── reports → Accuracy 91.3%
 └── identifies limitation → Poor cross-domain generalization

Paper B
 └── proposes → Method Y

Paper C
 └── challenges → assumption in Paper A
```

Eventually you could ask:

> "What assumptions are common across these papers?"

or:

> "Which limitations have been repeatedly mentioned but not experimentally investigated?"

or:

> "Which datasets are being overused?"

or:

> "Which two methods have never been directly compared?"

Those are much closer to **research discovery** than ordinary RAG.

OpenAlex is particularly useful here because its scholarly graph connects works, authors, sources, institutions, topics and citations; its Works API also exposes citation relationships and related works. ([OpenAlex Help Center][2])

Semantic Scholar is another useful source because its API exposes papers, citations, references, authors and SPECTER2 embeddings. ([Semantic Scholar][3])

---

# 4. Literature-review agent

Suppose you give it:

```text
10 papers
```

The agent shouldn't merely summarize them individually.

It should construct something like:

| Question   | Paper A    | Paper B     | Paper C      | Paper D     |
| ---------- | ---------- | ----------- | ------------ | ----------- |
| Problem    | ✓          | ✓           | ✓            | ✓           |
| Dataset    | D1         | D1          | D2           | D1          |
| Method     | CNN        | Transformer | CNN          | Hybrid      |
| Baseline   | ResNet     | ViT         | ResNet       | ViT         |
| Evaluation | Accuracy   | F1          | Accuracy     | F1          |
| Limitation | Small data | Compute     | Domain shift | Label noise |

Then identify patterns.

For example:

```text
Finding 1:
8/10 papers evaluate only on Dataset D1.

Finding 2:
7/10 papers compare against the same baseline.

Finding 3:
Only 2 papers investigate domain shift.

Finding 4:
No paper combines Method A with augmentation strategy B.
```

**Now you have candidates for research questions.**

---

# 5. Research-gap generator

This component should be deliberately adversarial.

Instead of:

> "Here is a great research idea."

make it ask:

```text
Could this already have been done?

Is the proposed novelty merely an implementation variation?

Would the experiment actually distinguish the hypothesis?

Is the dataset large enough?

Is there leakage?

Is the baseline appropriate?

Is the proposed improvement statistically meaningful?

Could the observed improvement simply result from
hyperparameter tuning?

Is this contribution significant enough for the intended venue?
```

This is critical.

A lot of AI-generated "research" is essentially:

> combine two existing methods + obtain 0.7% improvement = novel paper.

A serious research agent should actively try to **kill its own idea** before you spend compute on it.

---

# 6. Research hypothesis board

I would give the agent an internal structure like:

```text
Research Question
       ↓
Hypothesis
       ↓
Mechanism
       ↓
Prediction
       ↓
Experiment
       ↓
Expected observations
       ↓
Alternative explanations
```

Example:

### RQ

> Does method X improve robustness under domain shift?

### H1

> X should degrade less than baseline Y when training and testing distributions differ.

### Prediction

```text
Domain shift ↑
        ↓
Performance(X) decreases less
than Performance(Y)
```

### Experiment

```text
Train:
Dataset A

Test:
Dataset B

Models:
Baseline
X
X + augmentation

Metrics:
Accuracy
F1
AUROC
Calibration
```

### Alternative explanation

> X might simply have more parameters.

So the agent proposes:

```text
parameter-matched baseline
```

That's the kind of reasoning you want.

---

# 7. The Colab agent is especially important

I would **not** allow the LLM to arbitrarily execute code.

Give it a controlled tool:

```text
run_colab_cell()
read_colab_output()
inspect_dataframe()
plot_results()
save_artifact()
```

Then the agent can perform:

```text
Agent
 ↓
Generate experiment
 ↓
Send cell to Colab
 ↓
Execute
 ↓
Read stdout
 ↓
Inspect metrics
 ↓
Inspect plots
 ↓
Decide next experiment
```

For example:

```text
Experiment 1
     ↓
Result: 82.1%

Agent notices:
class imbalance
     ↓
Experiment 2
weighted loss
     ↓
Result: 85.7%
     ↓
Agent proposes
Experiment 3
     ↓
cross-validation
```

That becomes an actual **closed-loop research system**.

---

# 8. Keep every experiment reproducible

This is non-negotiable.

Every experiment should automatically record:

```yaml
experiment_id: EXP_017

git_commit: 4f81a2

dataset:
  name: UCI_HAR
  version: 1

seed: 42

model:
  architecture: Transformer
  parameters: 1.8M

hyperparameters:
  learning_rate: 0.0001
  batch_size: 64

hardware:
  accelerator: T4

code:
  notebook: experiment_017.ipynb

results:
  accuracy: 0.921
  f1: 0.917

artifacts:
  - confusion_matrix.png
  - results.json
  - model.pt
```

Then your paper becomes connected directly to experimental evidence.

---

# 9. Give the agent an experiment ledger

Something like:

```text
RESEARCH LEDGER
─────────────────────────────────────

EXP-001
Question:
Does X outperform baseline?

Result:
No

Conclusion:
Hypothesis unsupported

─────────────────────────────────────

EXP-002
Question:
Does X outperform baseline after
domain-specific preprocessing?

Result:
Yes

─────────────────────────────────────

EXP-003
Question:
Is improvement statistically significant?

Result:
p = ...

─────────────────────────────────────
```

This is incredibly valuable.

It prevents the classic problem:

> "Let's only report the experiments that worked."

Your agent should retain **failed experiments** too.

---

# 10. Statistical analysis agent

Don't let the LLM simply look at:

```text
91.2%
vs
92.1%
```

and say:

> "Method B significantly outperforms A."

Instead it should ask:

```text
How many runs?

Same random seeds?

Same dataset split?

Confidence interval?

Variance?

Statistical test?

Effect size?

Multiple comparisons?

Practical significance?
```

For example:

```text
Model A
91.2 ± 0.7

Model B
92.1 ± 0.4
```

could lead to:

```text
difference = 0.9 percentage points
95% CI = ...
effect size = ...
```

Only then should the paper make the appropriate claim.

---

# 11. The paper-writing agent comes last

The paper generator should **not be allowed to invent content**.

Its inputs should be:

```text
Literature database
+
Research question
+
Hypotheses
+
Experimental protocol
+
Experiment ledger
+
Raw results
+
Statistical analysis
+
Figures
+
Tables
```

Then:

```text
Introduction
      ↓
Related Work
      ↓
Research Gap
      ↓
Research Questions
      ↓
Methodology
      ↓
Experimental Setup
      ↓
Results
      ↓
Discussion
      ↓
Limitations
      ↓
Conclusion
```

And every important statement should have provenance.

For example:

```text
[CLAIM-034]

"The proposed model improves F1 by 3.2 percentage points."

SOURCE:
EXP-017/results.json

CALCULATION:
0.921 - 0.889

STATUS:
VERIFIED
```

---

# 12. Add a "Reviewer 2" agent

This could be one of the most useful components.

After generating a manuscript:

```text
             Manuscript
                 ↓
        ┌─────────────────┐
        │ Reviewer Agents │
        └────────┬────────┘
                 │
     ┌───────────┼───────────┐
     ↓           ↓           ↓
 Methodology   Novelty     Statistics
 reviewer      reviewer     reviewer
     ↓           ↓           ↓
     └───────────┼───────────┘
                 ↓
          Revision Agent
```

Reviewer prompts:

### Reviewer A — Methodology

> Find confounders, leakage, inappropriate baselines and reproducibility problems.

### Reviewer B — Novelty

> Search the literature and determine whether each claimed contribution already exists.

### Reviewer C — Statistics

> Look for unsupported statistical conclusions.

### Reviewer D — Writing

> Find ambiguity, overclaiming and unclear explanations.

### Reviewer E — Reproducibility

> Determine whether another researcher could reproduce the experiments.

Then the author agent revises the paper.

---

# 13. A very useful feature: claim verification

Before submission:

```text
          PAPER
            │
            ▼
      Extract claims
            │
            ▼
 ┌───────────────────────┐
 │ Evidence available?   │
 └───────────┬───────────┘
             │
       ┌─────┴─────┐
       │           │
      YES          NO
       │           │
       ▼           ▼
 verify       flag/remove
       │
       ▼
 citation check
       │
       ▼
 experiment check
```

The final manuscript might contain an internal audit:

```text
Claims:             47
Literature-backed:  19
Experiment-backed:  21
Derived analysis:    5
Unsupported:         2
```

The two unsupported claims must be fixed before submission.

---

# 14. MiniMax's role

I would use MiniMax as the **reasoning/orchestration engine**, rather than making it the entire system.

For example:

```text
                 MiniMax
                    │
       ┌────────────┼─────────────┐
       ↓            ↓             ↓
    Reasoning     Coding       Writing
       │            │             │
       ↓            ↓             ↓
 literature      Colab        manuscript
 analysis        agent        generation
```

The current MiniMax API supports standard programmatic text calls, and the current M2.7 documentation shows the `/v1/text/chatcompletion_v2` interface. ([MiniMax][1])

Its current token-plan material also advertises very large context windows, which is particularly interesting for research workflows involving long papers and codebases. ([MiniMax API Docs][4])

But I would **not rely on context length alone**.

Even with a huge context window, you want:

```text
LLM
+
retrieval
+
structured knowledge
+
citations
+
persistent research state
```

rather than dumping 50 PDFs into one prompt.

---

# 15. Proposed technology stack

Given your background, I would keep the first version Python-heavy.

```text
Python
│
├── FastAPI
│
├── MiniMax API
│
├── PyMuPDF / pdfplumber
│
├── scientific-paper parser
│
├── Chroma / Qdrant
│
├── PostgreSQL / SQLite
│
├── OpenAlex API
│
├── Semantic Scholar API
│
├── GitHub
│
└── Google Colab
```

For the scholarly layer:

```text
OpenAlex
    +
Semantic Scholar
```

is particularly attractive.

OpenAlex gives you a large scholarly graph and citation relationships. ([OpenAlex Help Center][2])

Semantic Scholar provides citation/reference data and paper embeddings, which can help with semantic retrieval. ([Semantic Scholar][3])

---

# 16. Don't start with LangChain/LangGraph everywhere

You *can* use an agent framework, but I'd initially build the core yourself.

Something as simple as:

```python
class ResearchAgent:

    def read_paper(self, pdf):
        ...

    def analyze_literature(self, papers):
        ...

    def identify_gaps(self, literature):
        ...

    def formulate_hypothesis(self, gap):
        ...

    def design_experiment(self, hypothesis):
        ...

    def run_experiment(self, experiment):
        ...

    def analyze_results(self, results):
        ...

    def propose_next_experiment(self, results):
        ...

    def write_paper(self, research_state):
        ...

    def review_paper(self, manuscript):
        ...
```

Then add tools:

```python
TOOLS = {
    "search_openalex": search_openalex,
    "search_semantic_scholar": search_semantic_scholar,
    "read_paper": read_paper,
    "search_literature": search_literature,
    "write_colab_cell": write_colab_cell,
    "run_colab": run_colab,
    "read_results": read_results,
    "plot_results": plot_results,
    "git_commit": git_commit,
}
```

This will teach you far more about **agent architecture** than hiding everything behind a framework.

---

# 17. The really ambitious version

Eventually, I would make the agent operate as a **Research Operating System**.

You open it and see:

```text
╔══════════════════════════════════════════════╗
║             RESEARCH PROJECT                 ║
║                                              ║
║ Topic: Transformer-based anomaly detection   ║
╚══════════════════════════════════════════════╝

Literature
──────────
127 papers
18 key papers
6 research clusters

Research Questions
──────────────────
RQ1 ✓
RQ2 ?
RQ3 ?

Hypotheses
──────────
H1  Testing
H2  Supported
H3  Rejected

Experiments
───────────
23 completed
4 running
7 proposed

Evidence
────────
156 claims
143 verified
9 pending
4 contradicted

Manuscript
──────────
Introduction       ✓
Related Work      ✓
Methods            ✓
Experiments        ✓
Results            ✓
Discussion         ⚠
Reviewer audit     ⚠
```

And you interact conversationally:

> **You:** I think domain adaptation might explain the difference.

> **Agent:** Three papers support this possibility, but none directly test the mechanism under your dataset split. I propose Experiment 24.

> **You:** Run it.

> **Agent:** Experiment 24 requires approximately 38 minutes on a T4. Shall I proceed?

> **You:** Yes.

> **Agent:** Completed. The hypothesis is supported under split A/B but not under split C. I've added the result to the research ledger and updated Figure 4.

That would be a **genuinely impressive AI-agent project**.

---

# 18. An important change to the design goal

We should slightly change:

> "Help me write a new research paper worthy of being published in reputed research journals."

to:

> **"Help me conduct research whose methodology, evidence and contribution can withstand peer review."**

That's a much better engineering objective.

The agent cannot guarantee that a journal will publish the result. What it *can* do is systematically improve:

**novelty + rigor + reproducibility + evidence + clarity.**

And crucially, it should be willing to tell you:

> **"This idea isn't sufficiently novel. Let's abandon it."**

That may ultimately save more time than an agent that generates endless papers.

---

# 19. I would build this in four phases

### Phase 1 — Paper Scientist

```text
PDF
 ↓
Parser
 ↓
MiniMax
 ↓
Structured paper representation
 ↓
Citation-grounded Q&A
 ↓
Literature comparison
```

### Phase 2 — Research Scientist

```text
Paper collection
 ↓
Literature graph
 ↓
Gap detection
 ↓
Hypothesis generation
 ↓
Experiment proposal
```

### Phase 3 — Experimental Scientist

```text
Hypothesis
 ↓
Experiment specification
 ↓
Colab
 ↓
Results
 ↓
Statistics
 ↓
Research ledger
 ↓
Next experiment
```

### Phase 4 — Author + Reviewer

```text
Evidence
 ↓
Manuscript
 ↓
Reviewer 1
Reviewer 2
Reviewer 3
 ↓
Revision
 ↓
Citation verification
 ↓
Reproducibility audit
 ↓
Submission-ready manuscript
```

**Phase 1 alone is already a substantial project.** Phase 3 is where it becomes genuinely agentic.

---

## And I think we should build it around one real research domain

Given your interests, we could use one of your areas such as **bioinformatics, computer vision, NLP, cybersecurity, or GPU/parallel computing** as the pilot.

For example, if we choose **bioinformatics**, we could give the agent 10–20 papers on a narrowly defined question. It would build the literature map, identify disagreements/gaps, formulate 2–3 hypotheses, design an experiment, generate a **real Google Colab notebook**, run it, analyze the results, and maintain the research ledger.

That would give you both **a useful research tool and a serious AI-agent engineering project** rather than a generic PDF chatbot.

[1]: https://www.minimax.io/models/text/m27?utm_source=chatgpt.com "MiniMax M2.7 - Model Self-Improvement, Driving Productivity Innovation Through Technological Breakthroughs | MiniMax"
[2]: https://help.openalex.org/api/endpoints/?utm_source=chatgpt.com "Endpoints Overview | OpenAlex Help Center"
[3]: https://api.semanticscholar.org/api-docs/snippets?utm_source=chatgpt.com "Semantic Scholar - Academic Graph API"
[4]: https://platform.minimax.io/subscribe/token-plan?tab=individual__monthly&utm_source=chatgpt.com "Token Plan - MiniMax API Platform"
