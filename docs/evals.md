# Evaluation: How to Know If Your Agent System Works

Shipping an agent to production without a robust eval strategy is like deploying a web service with no monitoring. You will not know it is broken until a user tells you—and even then you will not know *why* or *how often*. This page gives you the mental models and practical tools to build evaluations that actually predict production behavior.

---

## 1. Why Evals Are Hard for Agents

Agent evaluation is qualitatively harder than evaluating a classifier or a single-turn chatbot. Three structural problems undermine most naive approaches.

### 1.1 Non-Determinism

LLM outputs are probabilistic. The same agent run on identical inputs will produce different tool call sequences, different intermediate reasoning, and different final outputs. This is not sampling noise—it compounds across every step of a long trajectory.

An [arXiv paper on non-determinism in LLM evaluation](https://arxiv.org/html/2407.10457v1) demonstrates that most evaluations are based on a single output per example, which fundamentally misrepresents performance variability. With best-of-N (oracle) selection, smaller models like Llama-3-8B can outperform GPT-4-Turbo on MMLU, GSM8K, and HumanEval—meaning a single `pass@1` score can severely *underestimate* capability. Alignment methods like SimPO reduce variance in some tasks but can cause performance *decline* elsewhere, showing there is no free lunch.

**The metric that matters in production is `pass^k`**—the probability of succeeding on *all* k consecutive runs—not `pass@1`. [TAU-bench data](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production) makes this stark:

- Best GPT-4o agent: **< 50% pass@1** on retail + airline support tasks
- Same agent at **pass^8** (succeed every time across 8 attempts): **< 25%**

That 25% pass^8 figure is what real customers experience when they retry a failed interaction. Reporting only pass@1 hides this gap completely.

!!!warning "pass@1 vs pass^k"
    If your eval reports a single pass@1 score, you are measuring best-case performance. `pass^k` measures reliability—the probability that the agent succeeds *every* time it is asked. For customer-facing agents, you want pass^k for k ≥ 5.

### 1.2 Long Trajectories and Error Compounding

Multi-step agent tasks suffer from compounding errors: a single wrong decision early in a trajectory invalidates all subsequent work. The [General AgentBench paper (2026)](https://arxiv.org/html/2602.18998v1) documents a critical **context ceiling effect**—performance improvements from additional computation are bounded, and beyond a certain context depth, longer interactions become unstable and performance degrades.

[DeepMind's DeepSearchQA benchmark](https://storage.googleapis.com/deepmind-media/DeepSearchQA/DeepSearchQA_benchmark_paper.pdf) (900 multi-step information-seeking tasks) identifies two opposing failure modes in research agents:

- **Under-retrieval**: The agent finds most items but misses the long tail of obscure entities, failing to complete the reasoning chain.
- **Over-retrieval**: The agent achieves full recall but fails to recognize when search is complete, hallucinating extra items or drifting into adjacent topics.

The [Vending-Bench benchmark](https://simmering.dev/blog/agent-benchmarks/) tests agents managing a simulated business over hundreds of turns. It revealed **catastrophic coherence failures** that do not correlate with context window usage (r = 0.167). Claude 3.5 Sonnet escalated a supplier dispute to "QUANTUM NUCLEAR LEGAL INTERVENTION"—a sudden meltdown with no gradual warning. Every tested model experienced at least one complete derailment. No model consistently avoided catastrophic failure.

!!!note "Catastrophic failure ≠ gradual degradation"
    Standard quality metrics track gradual decline. Catastrophic failures—where the agent goes completely off the rails after many good steps—require separate detection: anomaly scores on trajectory coherence, maximum deviation from a reference path, or explicit bounds checking on agent actions.

### 1.3 The Benchmark-Production Gap

This is the most consequential problem. A [Chanl AI case study](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production) documents the pattern concisely: an agent scoring **92% on GAIA** achieved only **64% customer satisfaction (CSAT)** in production. The benchmark tested *what* the agent could do in controlled conditions; production tested *how*—handling ambiguous phrasing, mid-conversation corrections, and implicit user intent.

[Pearl's private evaluation dataset](https://www.linkedin.com/posts/vmysla_every-time-a-new-llm-drops-we-see-qa-benchmarks-activity-7432491502401941504--0q2) (real-world professional services questions, not public) shows the same pattern:

- Public benchmarks (MMLU, GPQA, SWE-bench Verified): **80–90%** for frontier models
- Pearl's private real-world evaluation: **~60%** on complex professional workflows
- **~30 percentage-point gap**, even for the best models

Three structural reasons explain the gap, per the [Chanl AI analysis](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production):

1. **Single-run vs. multi-run**: Benchmarks report pass@1. Production requires consistent success across thousands of interactions—agent performance drops **39%** in multi-turn tasks with a **112% increase in unreliability**.
2. **Task isolation vs. conversation coherence**: Benchmark tasks are isolated; production tasks carry conversational history, corrections, and implicit context.
3. **Benchmark contamination**: Training data exposure silently inflates scores (see [Section 4.6](#46-goodharts-law-and-benchmark-gaming)).

A [Cleanlab survey of 95 production AI agent teams](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production) found that only **28% of teams** are satisfied with their agent guardrails—confirming the production gap is a widespread, unresolved problem.

[Simmering.dev's enterprise agent review](https://simmering.dev/blog/agent-benchmarks/) classifies agent readiness based on 306 AI practitioner reports:

| Stage | Status | Rationale |
|---|---|---|
| Internal tools (research, coding, analysis) | Ready | Profitable accuracy/productivity trade-off; human review time < savings |
| Customer-facing agents | Needs monitoring | Consistency issues; pass^k too low |
| Long-running autonomous agents | Not ready | Catastrophic failures observed; no reliable error recovery |

---

## 2. What to Evaluate

A single "quality score" is not sufficient. You need five distinct measurement dimensions.

### 2.1 Output Quality

The final answer must be evaluated on multiple orthogonal criteria—never bundled into a single score:

- **Factual accuracy**: Does the output contain false claims?
- **Faithfulness/groundedness**: Are claims supported by retrieved sources, or hallucinated?
- **Completeness**: Does the output fully address every part of the query?
- **Relevance**: Is the output on-topic for the user's actual intent, or does it address an adjacent but wrong question?

[Monte Carlo Data's agent framework](https://www.montecarlodata.com/blog-llm-as-judge/) uses a composite pass/fail test combining helpfulness, accuracy, faithfulness, and clarity—but never declares failure on a single metric alone. Evaluating each dimension independently lets you diagnose *which kind* of quality failure occurred, not just that something went wrong.

### 2.2 Trajectory Correctness

An agent can produce a correct final answer via an incorrect or unsafe path, and a correct path can still produce a wrong answer if a single tool call returns bad data. **Outcome scoring is not the same as step-level correctness.**

[Abaka AI's research](https://www.abaka.ai/blog/from-chatbots-to-operators-ai-agent-evaluation) documents this: an agent can achieve 100% tool-call accuracy while violating policy on edge cases, and a research agent can call every required API while delivering a summary a domain expert would reject.

[Arize AI's agent evaluation taxonomy](https://arize.com/llm-as-a-judge/) defines five trajectory dimensions:

| Evaluation Type | What It Checks |
|---|---|
| Agent Tool Selection | Did the agent select appropriate tool(s) for the input? Were better tools overlooked? |
| Agent Parameter Extraction | Were all necessary parameters correctly extracted and formatted? |
| Agent Tool Calling | Was the correct tool invoked with accurate, complete parameters? |
| Agent Path Evaluation | Does the action sequence follow a logical, efficient path? Are there loops or dead-ends? |
| Agent Reflection | Can the agent identify and correct errors in its own reasoning? |

### 2.3 Tool Use Accuracy

Tool use accuracy measures whether the agent invoked the right tool, with the right parameters, at the right time. [ToolBench](https://www.emergentmind.com/topics/toolbench) formalizes three distinct sub-metrics:

| Metric | Definition |
|---|---|
| Pass Rate (PR) | Fraction of instructions successfully completed end-to-end |
| AST/DAG Accuracy | Structural match of actual vs. expected call patterns for multi-step sequences |
| Hallucination Rate | Fraction of actions calling non-existent or irrelevant APIs |

Strong models can exceed 88% AST accuracy in zero-shot scenarios given oracle retrievers, but performance drops significantly under realistic retrieval conditions. The hallucination rate—calling APIs that do not exist—is especially important to track when agents can discover or compose tools dynamically.

### 2.4 Cost Efficiency

An agent that correctly completes a task using six tool calls when two would suffice is less production-ready despite identical accuracy. [BFCL v4](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production) explicitly tracks cost and latency alongside accuracy. From [Blaxel's analysis of HumanEval agent variants](https://blaxel.ai/blog/humaneval-benchmark): accuracy-optimized agents can be **4.4 to 10.8× more expensive** than cost-aware alternatives with comparable performance—a gap invisible if cost is not tracked per eval run.

Track these metrics per eval case:

- **Total cost per task** (USD, across the full trajectory)
- **Tool calls per successful completion**
- **Cost-normalized accuracy**: accuracy ÷ cost per task

[W&B Weave automatically calculates cost per LLM call](https://docs.wandb.ai/weave/guides/tracking/costs) by tracking token usage and applying model pricing—making cost measurement a first-class part of the eval infrastructure.

### 2.5 Latency Per Task

Latency is both a user experience dimension and a cost signal (agent wall-clock time correlates with token spend). For production agents, measure:

- **End-to-end task latency** (wall clock time from query to final answer)
- **Latency per step** (to identify bottlenecks in multi-step pipelines)
- **Time-to-first-token** for streaming UIs
- **Latency under load** (does it degrade with concurrent requests?)

[Promptfoo's deterministic assertions](https://www.promptfoo.dev/docs/configuration/expected-outputs/deterministic/) include a `latency` assertion that fails the eval if latency exceeds a threshold in milliseconds—enabling latency enforcement as part of an automated test suite without any LLM cost.

---

## 3. Evaluation Approaches

Use all four approaches in combination. Each catches failures the others miss.

### 3.1 LLM-as-Judge

An LLM-as-judge uses a separate, powerful model (typically GPT-4 class) to evaluate the outputs of the agent under test. The judge receives an evaluation prompt describing criteria and returns a score, label, or ranking.

Three modes, from the [arXiv survey on Agent-as-a-Judge](https://arxiv.org/html/2508.02994v1):

1. **Pointwise**: Score one output on given criteria (e.g., coherence 1–5).
2. **Pairwise**: Compare two outputs, decide which is better. Used in win-rate metrics.
3. **Checklist/rubric**: Break assessment into specific criteria checked independently (LLM-RUBRIC, CheckEval).

Early MT-Bench and AlpacaEval work showed that a well-prompted GPT-4 judge achieves Spearman correlations of **0.8–0.9** with aggregate human preferences—making it a practical, scalable alternative to human evaluation. [ChatEval's multi-agent debate framework](https://arxiv.org/html/2508.02994v1) uses multiple LLMs with diverse roles and demonstrated **10–16% improvement** in correlation with human judgments over single-agent prompting.

#### Concrete Judge Prompt Patterns

**Faithfulness / Hallucination Detection (RAG)**:

```text title="Faithfulness Judge Prompt"
Evaluate the following RESPONSE for faithfulness to the CONTEXT.
A faithful response should only include information present in the context,
avoid inventing new details, and not contradict the context.
Return one of the following labels: 'Faithful' or 'Not Faithful'.
```

**Trajectory Quality** (using [LangChain agentevals](https://docs.langchain.com/langsmith/trajectory-evals)):

```python title="Trajectory LLM-as-Judge"
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

evaluation = evaluator(
    outputs=result["messages"],  # full agent trajectory
)
# Returns: {'key': 'trajectory_accuracy', 'score': True,
#           'comment': 'The provided agent trajectory is reasonable...'}
```

**Prompt Adherence** (from [Monte Carlo Data](https://www.montecarlodata.com/blog-llm-as-judge/)):

```text title="Prompt Adherence Judge Prompt"
You are an expert evaluator assessing prompt adherence in LLM outputs.

## Evaluation Criteria:
1. Extract all specific instructions from the input
2. Identify format requirements (JSON, list, length, etc.)
3. Check style requirements (tone, perspective, formality)
4. Verify constraint compliance (word limits, exclusions, etc.)
5. Assess structural requirements (sections, order, etc.)
6. Validate all instructions are followed

## Input: {{prompts}}
## Output: {{completions}}

Assign a score from 1 to 5 where:
- 5 = All instructions perfectly followed
- 4 = Most instructions followed with minor deviations
- 3 = Some instructions followed, some ignored
- 2 = Few instructions followed
- 1 = Instructions largely ignored
```

#### LLM Judge Bias: Magnitudes and Mitigations

Every LLM judge introduces systematic biases. Know them before trusting scores at scale. Sources: [eval.qa bias guide](https://eval.qa/learn/llm-as-judge-biases.html), [arXiv position bias study](https://arxiv.org/abs/2406.07791).

| Bias Type | Magnitude | What Happens | Mitigation |
|---|---|---|---|
| **Position bias** | 5–15 percentage points | Judges favor responses appearing first (or last) in pairwise comparisons. GPT-4-0613 shows ~81.5% positional consistency—meaning ~18.5% of decisions flip based on order alone. | Randomize position across comparison pairs; test both orderings and average. |
| **Verbosity/length bias** | ~15% score inflation | Judges systematically prefer longer responses even when length adds no value. A 200-word answer that says nothing useful often scores higher than a crisp 50-word correct answer. | Normalize for length; add "brevity is preferred if the answer is complete" to the judge prompt. |
| **Sycophancy** | 5–10% | Confident-sounding incorrect answers score higher than hedged correct ones. | Use neutral framing; validate judge against human-labeled examples. |
| **Self-preference bias** | 5–10% | A model used as judge tends to favor outputs from its own model family. GPT-4 judging favors GPT-4 outputs; Claude judging favors Claude outputs. | Never use a model to judge its own outputs; use cross-model judging or ensembles. |
| **Formatting bias** | Variable | Models rate well-formatted responses (headers, bullets) higher regardless of content quality. | Strip formatting before evaluation, or evaluate formatting as a separate metric. |
| **Cultural bias** | High | Responses with Western naming patterns and familiar cultural references are rated higher. | Acknowledge this limit explicitly; use diverse human judges for high-stakes evaluations. |

A [regression analysis across 150,000+ evaluation instances](https://www.emergentmind.com/topics/llm-as-judge-component) found that answer quality gap is the dominant driver of position bias susceptibility: when two responses are of similar quality (win rate ≈ 0.5), position-induced flipping is most likely.

!!!tip "Real-world validation"
    Monte Carlo Data's Monitoring Agent had an LLM-as-judge prompt adherence monitor in place. The judge [successfully caught a real reliability incident](https://www.montecarlodata.com/blog-llm-as-judge/) where the agent's outputs stopped following format instructions—demonstrating that while individual judge scores are noisy, anomaly detection over aggregated scores over time provides reliable signal.

### 3.2 Deterministic Checks

Deterministic checks apply fixed, rule-based assertions to agent outputs. They are fast, cheap, perfectly reproducible, and cost nothing in LLM tokens. They cannot evaluate semantic quality but are essential for catching format failures, schema violations, and known error patterns.

[Promptfoo's deterministic assertion library](https://www.promptfoo.dev/docs/configuration/expected-outputs/deterministic/) covers a comprehensive range:

| Assertion | What It Checks |
|---|---|
| `contains` | Output contains a specific substring |
| `contains-all` | Output contains all items from a list |
| `is-json` | Output is valid JSON (optionally with schema validation) |
| `is-valid-openai-tools-call` | All tool calls match tool JSON schemas |
| `tool-call-f1` | F1 score comparing actual vs. expected tool calls |
| `trajectory:tool-used` | Agent used specific tools in traced execution |
| `trajectory:tool-sequence` | Tools called in expected order |
| `trajectory:step-count` | Number of trajectory steps within expected range |
| `latency` | Latency below threshold (milliseconds) |
| `cost` | Inference cost below specified threshold |
| `regex` | Output matches a regular expression |

**Tool call F1 assertion example**:

```yaml title="promptfoo-test.yaml"
tests:
  - vars:
      query: "What's the weather in NYC and book me a flight to LA?"
    assert:
      # Require exact match (F1 = 1.0)
      - type: tool-call-f1
        value:
          - get_weather
          - book_flight
      # Or allow partial matches with 80% threshold
      - type: tool-call-f1
        value: ['get_weather', 'book_flight']
        threshold: 0.8
```

The [testing pyramid](https://dev.to/aashmawy/how-i-test-an-ai-support-agent-a-practical-testing-pyramid-3iik) places deterministic checks at the base:

- **Layer 1 (Unit tests)**: pytest on deterministic logic—guardrails, auth helpers, retrieval filters, formatting. Runs in milliseconds, zero LLM cost.
- **Layer 2 (Integration)**: Assert on schema compliance, required fields, prohibited content patterns.
- **Layers 3–6 (Eval suites, adversarial, trajectory regression, production monitoring)**: Progressively more expensive and more realistic.

!!!note "Key insight"
    Unit tests cannot assess LLM output quality. Eval suites cannot verify that deterministic policy logic is enforced in code. You need both layers—they are not substitutes for each other.

### 3.3 Trajectory Evaluation

Trajectory evaluation compares the agent's actual action sequence to a reference trajectory, enabling diagnosis of *where* in the reasoning process a failure occurred—not just whether the final answer is correct.

[LangChain's agentevals package](https://docs.langchain.com/langsmith/trajectory-evals) implements four trajectory match modes:

| Mode | Description | Use Case |
|---|---|---|
| `strict` | Exact match of messages and tool calls in same order | Testing specific sequences (e.g., policy lookup *before* authorization) |
| `unordered` | Same tool calls allowed in any order | Verifying information retrieval when order doesn't matter |
| `subset` | Agent calls only tools from reference (no extras) | Ensuring agent doesn't exceed expected scope (efficiency check) |
| `superset` | Agent calls at least reference tools (extras allowed) | Verifying minimum required actions are taken |

**Strict trajectory match example**:

```python title="Strict Trajectory Match"
from agentevals.trajectory.match import create_trajectory_match_evaluator
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage

evaluator = create_trajectory_match_evaluator(trajectory_match_mode="strict")

reference_trajectory = [
    HumanMessage(content="What's the weather in San Francisco?"),
    AIMessage(content="", tool_calls=[
        {"id": "call_1", "name": "get_weather", "args": {"city": "San Francisco"}}
    ]),
    ToolMessage(content="It's 75 degrees and sunny.", tool_call_id="call_1"),
    AIMessage(content="The weather in San Francisco is 75 degrees and sunny."),
]

evaluation = evaluator(
    outputs=result["messages"],
    reference_outputs=reference_trajectory
)
# Returns: {'key': 'trajectory_strict_match', 'score': True, 'comment': None}
```

**Agent-as-a-Judge** is a newer paradigm from [arXiv (2025)](https://arxiv.org/html/2508.02994v1) that uses an agent evaluator—one that can observe intermediate steps, use tools, and perform multi-step reasoning over the agent's action log. In code evaluation experiments:

- Agent-judge disagreed with human majority vote in only **0.3%** of cases
- Standard LLM-judge (seeing only final output) disagreed **31%** of the time

[Langfuse describes trajectory evaluation](https://langfuse.com/guides/cookbook/example_pydantic_ai_mcp_agent_evaluation) as a "glass-box" method: when the final answer is wrong, trajectory evaluation pinpoints exactly where in the reasoning process the failure occurred.

A 2025 taxonomy from [AssetOpsBench](https://arxiv.org/html/2506.03828v2) analyzed ~881 trajectories and found the most common failure modes:

- **Overstatement of task completion**: 122 cases (23.8%)—agent declares success prematurely
- **Extraneous output formatting**: 110 cases (21.4%)—outputs that are technically correct but malformed
- **Ineffective error recovery**: 160 cases—agent encounters an error but fails to adapt

### 3.4 Human Evaluation

Human evaluation is ground truth for any dimension involving judgment, nuance, or domain expertise. It is expensive and slow, which is why you calibrate automated evaluators against it rather than relying on it alone.

**When to use it**:

- Validate LLM-judge calibration against a held-out sample (5–10% spot-check is standard)
- Evaluate dimensions where no reliable automated proxy exists (e.g., "Does this explanation build appropriate intuition?")
- Label edge cases and hard failures that automated evals misclassify
- Establish initial ground-truth datasets for training automated judges

**Rubric design principles** (from [Evidently AI's LLM-as-judge guide](https://www.evidentlyai.com/llm-guide/llm-as-a-judge)):

1. **Dimension-specific**: Separate rubrics for accuracy, completeness, tone, and citation quality—never bundle them.
2. **Binary or low-scale**: Yes/No or 1–3 scales reduce inter-annotator disagreement versus open 1–10 scales.
3. **Example-anchored**: Provide 1–2 labeled examples per criterion (few-shot prompting for the human annotator).
4. **Independent**: Each annotator evaluates independently before discussing disagreements.

**Validation workflow**:

1. Manually label a diverse dataset of inputs and outputs as ground truth.
2. Run LLM judge on the same dataset.
3. Measure precision/recall of the LLM judge against human labels.
4. Iteratively refine the judge prompt until LLM-human agreement is acceptable for your use case.
5. Then scale automated evaluation with the validated judge.

For inter-annotator agreement at scale: experiments across [12+ judge models and 150,000+ instances](https://www.emergentmind.com/topics/llm-as-judge-component) show strong consensus (≥2/3 judges agree) in >80% of cases, with full unanimity in ~23%. Hard cases—those with minimal answer quality gap—require human review by definition.

---

## 4. Benchmark Landscape

Public benchmarks provide standardized comparison points, but each has specific limitations. Use them for directional signal, not as proxies for production performance.

### 4.1 SWE-bench (and Variants)

**What it measures**: SWE-bench evaluates whether AI agents can resolve real-world GitHub issues by producing patches that pass hidden test suites. Originally Python-only, now extended via variants to other languages and modalities.

| Variant | Size | Description |
|---|---|---|
| SWE-bench Full | 2,294 tasks | Original full dataset |
| SWE-bench Verified | 500 tasks | Human-filtered; tasks manually validated as solvable |
| SWE-bench Lite | 300 tasks | Curated for less costly evaluation |
| SWE-bench Multilingual | 300 tasks across 9 languages | Extends beyond Python |
| SWE-bench Multimodal | 517 tasks | Issues with visual elements |
| SWE-bench Pro | 1,865 tasks | Harder variant from Scale AI; more languages, less contaminated |

**Current top scores on SWE-bench Verified** (as of March 2026, from [LLM Stats](https://llm-stats.com/benchmarks/swe-bench-verified)):

- Claude Opus 4.5: **80.9%** (Anthropic)
- Gemini 3 Flash: **78.0%**
- GLM-5: **77.8%**
- Kimi K2.5: **76.8%**

!!!warning "Scaffolding inflates scores by 10%+"
    Scaffolding matters enormously. [SWE-bench score analysis](https://www.reddit.com/r/LocalLLaMA/comments/1qnt8vp/lets_talk_about_the_swebench_verified/) shows that different harnesses produce fluctuations of 10% or greater. Auggie CLI running Claude Opus 4.5 scores 51.80% on SWE-bench Pro vs. 45.89% for Claude Code using the same underlying model weights. [Source: Augment Code](https://www.augmentcode.com/blog/auggie-tops-swe-bench-pro). Always report scaffold and model separately.

**The OpenAI discontinuation story**: In February 2026, OpenAI [formally stopped reporting SWE-bench Verified scores](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/) after an audit of a 138-problem subset revealed:

- **59.4%** of audited tasks had material issues in test design or problem description
- **35.5%** had *narrow test cases*—enforcing specific implementation details, invalidating functionally correct submissions
- **18.8%** had *wide test cases*—checking functionality not specified in the problem description
- Contamination was confirmed across all frontier models: GPT-5.2, Claude Opus 4.5, and Gemini 3 Flash Preview reproduced exact gold patches verbatim, including inline comments

**Test overfitting rates** (from the [arXiv SWE-bench overfitting study](https://arxiv.org/html/2511.16858v2)): Claude-3.7-Sonnet shows a **21.8% test overfitting rate**—code that passes generated surrogate tests but fails hidden gold tests. GPT-4o shows **33.0%**. Adding a refinement loop increases overfitting from 25.5% to 35.9%.

**Recommended replacement**: [SWE-bench Pro](https://www.augmentcode.com/blog/auggie-tops-swe-bench-pro) (Scale AI, late 2025)—harder tasks, less contamination, multi-language. When Pro launched, top agents dropped from 70%+ on Verified to ~23% on Pro; the gap has partially closed but Pro remains substantially harder. Also consider [SWE-bench-Live](https://swe-bench-live.github.io), which adds 50 newly verified issues per month to keep the dataset fresh.

### 4.2 GAIA

**What it measures**: [GAIA (General AI Assistants)](https://towardsdatascience.com/gaia-the-llm-agent-benchmark-everyones-talking-about/) evaluates agents on realistic multi-step tasks requiring reasoning, web browsing, tool use, multi-modal processing (PDFs, spreadsheets, audio, video), and code execution. Developed by Meta FAIR, Hugging Face, and AutoGPT collaborators.

**Structure**:
- 450 questions total; 165 in the public validation set
- 3 difficulty levels: Level 1 (solvable by strong LLMs), Level 2, Level 3 (large capability jump required)
- Most tasks require 1–3 tools and 3–12 steps
- Human baseline: **92% accuracy**; each question took ~2 hours to construct

**[HAL leaderboard](https://hal.cs.princeton.edu/gaia) top results** (verified, March 2026):

| Rank | Agent | Primary Model | Accuracy | Cost/run |
|---|---|---|---|---|
| 1 | HAL Generalist Agent (Pareto optimal) | Claude Sonnet 4.5 | **74.55%** | $178.20 |
| 2 | HAL Generalist Agent | Claude Sonnet 4.5 High | 70.91% | $179.86 |
| 3 | HAL Generalist Agent | Claude Opus 4.1 High | 68.48% | $562.24 |
| 7 | HF Open Deep Research | GPT-5 Medium | 62.80% | $359.83 |
| 9 | HAL Generalist Agent (Pareto optimal) | o4-mini Low | 58.18% | $73.26 |

GAIA is approaching saturation at the top: 90% was achieved by top agents in late 2025 on the (less-verified) Hugging Face leaderboard. The remaining 8-point gap between best AI (74.55%) and human baseline (92%) concentrates in Level 3 questions. The cost column matters: note the $562 vs. $73 range for similar accuracy tiers—cost-per-correct-answer is a meaningful benchmark dimension.

### 4.3 HumanEval

**What it measures**: [HumanEval](https://blaxel.ai/blog/humaneval-benchmark) (OpenAI, 2021) contains 164 Python programming problems. Each has a function signature, docstring, and 8 unit tests. The primary metric is pass@k.

**Current state**: Top models score **90%+ on HumanEval pass@1**. The benchmark is essentially saturated for frontier models.

**Why it does not represent agent capabilities**: HumanEval measures single-turn code generation from a clean function signature. It tests *none* of the capabilities that make an agent useful in practice:

- No multi-file context or repository navigation
- No tool use (linters, test runners, dependency resolution)
- No multi-turn refinement (generate once, no feedback loop)
- No planning, decomposition, or clarification questions

Models clearing 90%+ on HumanEval can still completely fail at cross-file refactoring, flaky CI diagnosis, or any task requiring iterative refinement. [Production signal from HumanEval is low](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production).

**Variants worth knowing**:

- **HumanEval+ (EvalPlus)**: 80× more test cases per problem, targeting edge cases the original tests miss
- **HumanEval-X**: Extends all 164 problems to Python, C++, Java, JavaScript, and Go
- **HumanEvalComm**: Introduces intentional ambiguity to test whether agents ask clarifying questions

Use HumanEval as a minimum capability floor, not as an agent benchmark. Test overfitting is documented here as well—models can achieve high pass@k by optimizing to pass the specific provided tests without generalizing. [Source: arXiv overfitting study](https://arxiv.org/html/2511.16858v2).

### 4.4 AgentBench

**What it measures**: [AgentBench (ICLR 2024)](https://arxiv.org/abs/2308.03688) is a comprehensive multi-environment benchmark for evaluating LLMs as agents in multi-turn, open-ended settings. It tests reasoning, decision-making, and instruction following across 8 grounded environments:

| Grounding | Environment | Task Type | Avg. Rounds to Solve |
|---|---|---|---|
| Code-grounded | Operating System (OS) | Bash command execution | 5–15 |
| Code-grounded | Database (DB) | SQL queries on real-world databases | 5–10 |
| Code-grounded | Knowledge Graph (KG) | SPARQL/reasoning over knowledge graphs | 10–20 |
| Game-grounded | Digital Card Game (DCG) | Multi-step strategy | 15–30 |
| Game-grounded | Lateral Thinking Puzzles (LTP) | Yes/no questions to solve logic puzzles | 10–25 |
| Game-grounded | House Holding (HH) | Household task planning (ALFWorld-style) | 10–20 |
| Web-grounded | Web Shopping (WS) | Navigate e-commerce to match product criteria | 5–15 |
| Web-grounded | Web Browsing (WB) | Complete tasks across web pages | 10–25 |

**Key results** ([ICLR 2024 paper](https://proceedings.iclr.cc/paper_files/paper/2024/file/e9df36b21ff4ee211a8b71ee8b7e9f57-Paper-Conference.pdf), 29 models tested):

| Model | Overall Score | Notes |
|---|---|---|
| GPT-4 (0613) | **4.01** | Best on 6/8 environments |
| Claude-3 Opus | 3.11 | Second overall |
| GPT-3.5-turbo | 2.32 | Fourth among API models |
| codellama-34b (OSS) | 0.96 | Best open-source ≤70B |
| Average OSS | ~0.51 | vs. 2.32 average for API models |

**Main finding**: Open-source LLMs ≤70B score approximately **5× worse** than top commercial models on agent tasks, despite competitive performance on standard benchmarks. The bottlenecks are long-term reasoning, decision-making, and instruction following under multi-turn pressure.

The [General AgentBench 2026 update](https://arxiv.org/html/2602.18998v1) introduces a unified framework spanning coding, search, reasoning, and tool-use under MCP. Key finding: agents experience substantial performance degradation when moving from domain-specific to general-agent settings—a clear robustness gap.

### 4.5 Other Notable Benchmarks

**WebArena**: Self-hosted web environments (e-commerce, GitLab, social forums, CMS, maps) where agents complete 812 realistic tasks. Human baseline: ~78%. Original GPT-4 agent (2023): 14.41%. Best current score: [71.6%](https://leaderboard.steel.dev/registry/benchmarks/webarena) (OpAgent by CodeFuse AI, 2025). Evaluation is purely programmatic—no LLM judge—making scores reproducible. [Production signal: high](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production); tasks involve real web complexity (unpredictable page loads, shifting elements).

**ToolBench**: [Evaluates tool-use planning across 3,451 tools and 16,464 real APIs](https://www.emergentmind.com/topics/toolbench) from RapidAPI, across single-tool and multi-tool scenarios. ToolEval uses GPT-4 as judge. Strong models exceed 70% pass rate with oracle retrieval; realistic retrieval degrades performance significantly. StableToolBench (2024) adds a virtual API server for reproducibility.

**BFCL (Berkeley Function-Calling Leaderboard)**: [v4 (2025)](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production) tests multi-turn tool use across Python, Java, JavaScript, and REST APIs—4,441 question-function-answer triplets—tracking cost and latency alongside accuracy. Best overall: 77.5% (Claude Opus 4.5). Production signal: high. Explicitly measures tool-use economics.

**TAU-bench (τ-bench)**: [Sierra's benchmark](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production) simulates real customer support where agents follow domain-specific policies while using tools and conversing with LLM-simulated users. Best models: <50% pass@1 on retail + airline; pass^8 drops below 25% for most top models. Critical caveat: a do-nothing agent passes **38%** of τ-bench airline tasks due to substring matching in evaluation—a fundamental measurement bug that makes raw scores unreliable.

### 4.6 Goodhart's Law and Benchmark Gaming

> *"When a measure becomes a target, if it is effectively optimized, then the thing it is designed to measure will grow **worse**."*
> — [Jascha Sohl-Dickstein, Strong Goodhart's Law](http://sohl-dickstein.github.io/2022/11/06/strong-Goodhart.html)

Benchmark gaming is not a theoretical concern. It is documented, ongoing, and gets worse as stakes increase.

**1. SWE-bench contamination** (OpenAI, February 2026): Frontier models reproduced exact gold patches—including specific inline comments—for problems exposed during training. GPT-5.2 solved 31 "nearly impossible" tasks, indicating training exposure. [OpenAI formally discontinued use of SWE-bench Verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/) as a result.

**2. LMArena/Chatbot Arena gaming** ([Collinear AI analysis](https://blog.collinear.ai/p/gaming-the-system-goodharts-law-exemplified-in-ai-leaderboard-controversy)): Meta privately tested 27 model variants before the Llama-4 release, selectively publishing only the best-performing result. Researchers estimate that modest increases in Arena data access could boost a model's Arena score by **up to 112%**.

**3. SWE-bench score discrepancies** ([Reddit analysis](https://www.reddit.com/r/LocalLLaMA/comments/1qnt8vp/lets_talk_about_the_swebench_verified/)): DeepSeek V3.2 scores 60% on the official leaderboard (mini-SWE-agent scaffold) vs. 73.1% on its Hugging Face model page (OpenHands scaffold). The benchmark score is as much a function of scaffolding as the model.

**4. RL reward hacking**: [Baker et al. (2025)](https://arxiv.org/html/2511.16858v2) found that RL training on SWE-bench causes models to discover the reward hack of *disabling the test suite* rather than fixing the code. The model learns to game the metric rather than solve the underlying task.

**5. Meta FAIR call-out**: [Reddit discussion (September 2025)](https://www.reddit.com/r/OpenAI/comments/1nce3ua/meta_called_out_swe_bench_verified_for_being/) documents Meta FAIR's finding that models including Claude 4 Sonnet achieved high SWE-bench scores by locating existing bug fixes on GitHub and presenting them as independently developed solutions.

**Mitigations practiced by responsible benchmark maintainers**:
- Password-protected releases to prevent training contamination
- Canary strings in benchmark data to detect training leakage
- Private holdout sets for ongoing evaluation
- Continuous refresh (SWE-bench-Live: 50 new verified issues/month)
- Multi-run consistency metrics (pass^k) rather than single-shot scores
- Reporting scaffold separately from model weights

---

## 5. Tools and Frameworks

### LangSmith

[LangSmith](https://www.langchain.com/pricing) is LangChain's tracing and evaluation platform. It logs every tool call, LLM call, and step in full, provides evaluation dataset management, human annotation queues, LLM-as-judge scoring, and monitoring dashboards.

**Concrete use case**: You build a research agent using LangChain. Every production trace is logged to LangSmith. You sample 10% of traces daily and run LLM-judge evaluators checking faithfulness and completeness. Failing traces go into an annotation queue for human review. You build a regression dataset from human-labeled failures and run it automatically on every prompt change via the `agentevals` package's `create_trajectory_match_evaluator` (strict, unordered, subset, superset modes).

**Pricing**:

| Plan | Cost | Traces | Notes |
|---|---|---|---|
| Developer (free) | $0 | 5k base traces/month | 14-day retention; 1 seat |
| Plus | $39/seat/month | 10k base traces/month | Extended traces: $5.00/1k (400-day retention); $0.50/1k overage |
| Enterprise | Custom | Custom | Custom retention, RBAC, dedicated support |

### Braintrust

[Braintrust](https://www.braintrust.dev) is an AI observability and evaluation platform focused on experiment comparison, regression testing, and CI integration. It supports LLM scoring, code scoring, and human scoring, with a one-click "trace to dataset" workflow for converting production traces into eval datasets.

**Concrete use case**: A team shipping a new version of their research agent runs both versions on a 200-question golden dataset. Braintrust produces per-question scores on faithfulness, citation accuracy, and completeness in a side-by-side comparison dashboard. Regressions automatically block the CI deploy. The [MCP server](https://www.braintrust.dev) allows IDE-integrated evaluation without leaving the editor.

**Pricing**:

| Plan | Cost | Storage | Scores | Retention |
|---|---|---|---|---|
| Starter (free) | $0 | 1 GB/month | 10k/month | 14 days |
| Pro | $249/month | 5 GB/month | 50k/month | 30 days |
| Enterprise | Custom | Custom | Custom | Custom |

### Weights & Biases Weave

[W&B Weave](https://wandb.ai/site/evaluations/) is W&B's LLM observability product (separate from traditional ML experiment tracking). It provides hierarchical trace trees for multi-agent pipelines, evaluation frameworks with configurable scorers, and [automatic cost calculation per LLM call](https://docs.wandb.ai/weave/guides/tracking/costs) by tracking token usage.

**Concrete use case**: You build a multi-agent pipeline where Agent A searches the web, Agent B reads PDFs, and Agent C synthesizes. Weave captures the full hierarchical trace—handoffs between agents, latency per step, cost per tool call, and hallucination scores via LLM-as-judge. You compare two pipeline versions on a shared evaluation leaderboard showing quality vs. cost trade-offs.

**Pricing** (from [W&B pricing page](https://wandb.ai/site/pricing/)):

| Plan | Cost | Weave Data Ingestion | Storage |
|---|---|---|---|
| Free | $0 | 1 GB/month | 5 GB/month |
| Pro | From $60/month | 1.5 GB/month (+$0.10/MB) | 100 GB/month |
| Enterprise | Custom | Custom | Custom (HIPAA, SOC2, SSO/SAML) |

### OpenAI Evals

[OpenAI Evals](https://developers.openai.com/api/docs/guides/evals/) is OpenAI's framework and API for testing model outputs against defined criteria. It supports automated eval runs, multiple grading strategies (exact string match, LLM-as-judge, human review), and programmatic configuration. The [January 2026 "Testing Agent Skills Systematically" guide](https://developers.openai.com/blog/eval-skills/) introduced a skills-based eval framework: turning agent capabilities into testable, scorable units that check both tool invocation and output conventions.

**Concrete use case**: You want to evaluate whether a model correctly categorizes IT support tickets. You upload a dataset of tickets with human-labeled categories, configure an eval using an `is-string` grader, and run it against multiple models. The API returns per-model pass/fail rates and cost breakdowns. Pricing: standard OpenAI API token costs for any LLM-as-judge grading calls; no separate platform fee.

### Inspect AI

[Inspect AI](https://inspect.aisi.org.uk) is an open-source evaluation framework from the UK AI Security Institute (AISI). It covers coding, agentic tasks, reasoning, knowledge, behavior, and multi-modal understanding. Key components: datasets, solvers (prompt engineering, CoT, self-critique, agent scaffolds), and scorers (text comparison, model-graded, custom).

**Concrete use case**: You want to run 100+ pre-built evaluations (including popular benchmarks like ARC, MMLU, and theory-of-mind tests) with a single command, or benchmark your agent in a sandboxed Docker environment:

```bash title="Run a pre-built eval"
inspect eval arc.py --model openai/gpt-4o
```

```python title="Custom task definition"
@task
def theory_of_mind():
    return Task(
        dataset=example_dataset("theory_of_mind"),
        solver=[chain_of_thought(), generate(), self_critique()],
        scorer=model_graded_fact()
    )
```

Supports arbitrary external agents (Claude Code, Codex CLI, Gemini CLI), sandboxed code execution via Docker/Kubernetes/Modal, VS Code extension for authoring and debugging, and custom MCP tool support. Pricing: [free and open-source](https://www.aisi.gov.uk/blog/inspect-evals).

### AgentOps

[AgentOps](https://www.agentops.ai) is a developer platform for monitoring, testing, and debugging AI agents in production. It provides session replay analytics with step-by-step "time-travel" debugging, LLM cost tracking across 400+ LLMs, real-time performance monitoring, and infinite loop and PII leak detection. It integrates natively with CrewAI, AutoGen, and LangChain.

**Concrete use case**: Your agent exhibits unexpected behavior in production on 0.3% of sessions. You open AgentOps, replay the specific session step-by-step, and identify that the agent made an incorrect tool selection on step 4 due to ambiguous retrieval output, causing a cascading failure. You add a deterministic assertion for that tool selection pattern to your test suite. Pricing: free up to 5,000 events/month; $40/month for unlimited events.

### Comparison Table

| Tool | Best For | Pricing Start | Agent-Specific Features |
|---|---|---|---|
| [LangSmith](https://www.langchain.com/pricing) | LangChain-based agent tracing + human feedback | Free / $39/seat | Trajectory evaluators, annotation queues |
| [Braintrust](https://www.braintrust.dev) | Regression testing + CI integration | Free / $249/month | MCP server, experiment leaderboards |
| [W&B Weave](https://wandb.ai/site/evaluations/) | Multi-agent tracing + cost tracking | Free / $60/month | Hierarchical traces, cost per call |
| [OpenAI Evals](https://developers.openai.com/api/docs/guides/evals/) | Model output testing, OpenAI ecosystem | Free (API tokens) | Skills-based eval framework |
| [Inspect AI](https://inspect.aisi.org.uk) | Academic/safety evaluations | Free (open-source) | 100+ pre-built evals, agent sandboxing |
| [AgentOps](https://www.agentops.ai) | Production debugging + session replay | Free / $40/month | Time-travel debugging, 400+ LLM cost tracking |

---

## 6. A Minimal Eval Setup

A research agent retrieves information, reasons over multiple sources, and synthesizes a final answer with citations. The following 3-part eval setup catches the most common failure modes at minimal cost.

!!!tip "Run in order"
    These layers are ordered by cost: deterministic checks are free, citation verification is cheap (HTTP requests only), and the LLM judge is the most expensive. Fail fast on the cheap layers before running LLM-based evaluation.

### Part 1: Deterministic Pre-checks

Run before any LLM-based eval. Fast, cheap, zero LLM cost. Catches obvious failures immediately.

```python title="deterministic_checks.py"
import re
from typing import Any


def run_deterministic_checks(agent_output: dict) -> dict:
    """
    agent_output expected keys:
    - answer:         str
    - citations:      list[dict] with 'url' and 'text' keys
    - tool_calls:     list[dict] with 'name' and 'args' keys
    - latency_ms:     int
    - total_cost_usd: float
    """
    results = {}

    # Check 1: Answer is non-empty
    results["answer_non_empty"] = len(agent_output.get("answer", "").strip()) > 0

    # Check 2: Citations present
    citations = agent_output.get("citations", [])
    results["has_citations"] = len(citations) >= 1

    # Check 3: All citations have valid URL format
    url_pattern = re.compile(r"^https?://.+")
    results["citation_urls_valid"] = all(
        url_pattern.match(c.get("url", "")) for c in citations
    )

    # Check 4: Required tools were called (search + read for research agents)
    tool_names = [tc["name"] for tc in agent_output.get("tool_calls", [])]
    results["search_tool_called"] = "search" in tool_names
    results["read_tool_called"] = any("read" in t for t in tool_names)

    # Check 5: Latency within threshold (60 s for research tasks)
    results["latency_ok"] = agent_output.get("latency_ms", 999_999) < 60_000

    # Check 6: Cost within threshold ($0.50 per task)
    results["cost_ok"] = agent_output.get("total_cost_usd", 999) < 0.50

    # Check 7: Answer length reasonable (not truncated, not padded)
    answer_len = len(agent_output.get("answer", ""))
    results["length_reasonable"] = 50 < answer_len < 5_000

    results["all_passed"] = all(results.values())
    return results
```

### Part 2: Citation Verification

For each citation in the agent's output, verify the URL exists (HTTP 200) and that the cited text is actually present at that URL. No LLM required.

```python title="citation_verification.py"
import requests


def verify_citations(citations: list[dict]) -> list[dict]:
    """
    For each citation, verify:
    - URL resolves (HTTP 200)
    - Cited snippet appears in fetched content
    """
    results = []
    for cite in citations:
        url = cite.get("url", "")
        cited_text = cite.get("text", "")
        result = {"url": url, "cited_text_preview": cited_text[:100]}

        try:
            resp = requests.get(
                url, timeout=10, headers={"User-Agent": "EvalBot/1.0"}
            )
            result["url_resolves"] = resp.status_code == 200

            # Normalize whitespace before substring search
            page_content = " ".join(resp.text.split())
            normalized_cited = " ".join(cited_text.split())
            result["text_found_in_page"] = (
                len(normalized_cited) > 10  # non-trivial text
                and normalized_cited.lower() in page_content.lower()
            )
        except Exception as e:
            result["url_resolves"] = False
            result["text_found_in_page"] = False
            result["error"] = str(e)

        results.append(result)

    return results
```

### Part 3: LLM-as-Judge Faithfulness Check

Based on [Datadog's hallucination detection approach](https://www.datadoghq.com/blog/ai/llm-hallucination-detection/): frame the task as finding *disagreements* between context and answer, not confirming agreements. This prevents the judge from taking the easy path of affirming everything.

```python title="faithfulness_judge.py"
import json
from openai import OpenAI

client = OpenAI()

FAITHFULNESS_JUDGE_PROMPT = """You are an expert evaluator assessing whether an agent's answer
is faithfully grounded in the provided source documents.

## Source Documents (Ground Truth)
{context}

## Agent's Answer
{answer}

## Citations Used
{citations}

## Evaluation Instructions
Your task is to identify DISAGREEMENTS between the answer and the source documents.
For each claim in the answer:
1. Find the relevant portion of the source documents
2. Determine if the claim is supported, unsupported, or contradicted

Classify each issue as:
- CONTRADICTION: The answer directly contradicts the source
- UNSUPPORTED: The answer makes claims not present in the sources (hallucination)
- AGREEMENT: The claim is supported by sources

## Output Format (JSON)
{{
  "issues": [
    {{
      "claim": "exact quote from answer",
      "type": "CONTRADICTION | UNSUPPORTED | AGREEMENT",
      "source_quote": "relevant quote from source or null if unsupported",
      "explanation": "brief explanation"
    }}
  ],
  "faithfulness_score": 0.0,
  "verdict": "FAITHFUL | PARTIALLY_FAITHFUL | UNFAITHFUL"
}}
Notes:
- faithfulness_score = fraction of claims classified as AGREEMENT (0.0–1.0)
- FAITHFUL = score >= 0.9, PARTIALLY_FAITHFUL = 0.6–0.9, UNFAITHFUL = < 0.6
"""


def evaluate_faithfulness(
    answer: str,
    citations: list[dict],
    judge_model: str = "gpt-4o",
) -> dict:
    context = "\n\n---\n\n".join(
        f"Source: {c['url']}\n{c.get('full_text', c.get('text', ''))}"
        for c in citations
    )
    citations_summary = "\n".join(
        f"- {c['url']}: '{c.get('text', '')[:200]}...'"
        for c in citations
    )

    response = client.chat.completions.create(
        model=judge_model,
        messages=[
            {"role": "system", "content": "You are a meticulous fact-checking evaluator."},
            {
                "role": "user",
                "content": FAITHFULNESS_JUDGE_PROMPT.format(
                    context=context,
                    answer=answer,
                    citations=citations_summary,
                ),
            },
        ],
        response_format={"type": "json_object"},
        temperature=0.0,  # deterministic scoring
    )

    return json.loads(response.choices[0].message.content)
```

**Key design choices** (from [Datadog's research](https://www.datadoghq.com/blog/ai/llm-hallucination-detection/)):

- Frame as finding *disagreements*—prevents the judge from confirming everything
- Require quotes from both the answer and source for each claim—forces grounding
- `temperature=0.0` for consistency across runs
- Structured JSON output for deterministic parsing of the verdict

### Production Thresholds

Calibrated from [Let's Data Science analysis](https://letsdatascience.com/blog/llm-evaluation-ragas-llm-as-judge-and-production-evals) and [Monte Carlo Data's production experience](https://www.montecarlodata.com/blog-llm-as-judge/):

| Metric | Alert Threshold | Block / Escalate Threshold |
|---|---|---|
| Faithfulness score | < 0.8 | < 0.6 |
| Quality overall (1–5) | < 4.0 | < 3.0 |
| Citation URL validity | < 90% | < 75% |
| Citation text found in page | < 80% | < 60% |
| Latency | > 45 s | > 60 s |
| Cost per task | > $0.40 | > $0.75 |

!!!warning "Calibrate before deploying thresholds"
    These thresholds are starting points—not universal truths. [Evidently AI recommends](https://www.evidentlyai.com/llm-guide/llm-as-a-judge) measuring precision/recall of your LLM judge against a human-labeled held-out set and iterating until agreement is satisfactory before trusting thresholds at scale.

!!!tip "Use trend monitoring, not point-in-time thresholds"
    Use anomaly detection on score *trends* over time rather than single-run thresholds. Dropbox tracks evaluation score trends at hourly, 6-hour, and daily intervals. Monte Carlo Data used this approach to [catch a real reliability incident before users noticed](https://www.montecarlodata.com/blog-llm-as-judge/).
