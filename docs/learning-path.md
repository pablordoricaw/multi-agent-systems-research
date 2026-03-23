# Recommended Learning Path

A sequenced, opinionated 4-week learning path for an engineer going from zero to running multi-agent pipelines. Assumes Python proficiency and basic LLM familiarity.

---

## Week 1: Foundations — Single Agent with Tools

**Goal**: Understand how a single LLM agent uses tools, maintains context, and interacts with external systems.

### Day 1–2: Core Concepts

- Read the [Getting Started](getting-started.md) page for key terminology (memory, tool use, orchestration patterns).
- Understand the difference between chat completion, function calling, and agentic loops.
- Explore the [OpenAI Agents SDK](deep-dives/openai-agents-sdk.md) and its predecessor Swarm's source code — Swarm is under 100 lines and demonstrates the handoff pattern in its simplest form, while the Agents SDK shows how those patterns scale to production.

### Day 3–4: Build a Single-Agent Tool User

- Install Aider and use it on a small project to experience AI pair programming:

```bash
pip install aider-install && aider-install
cd /your/project
aider --model sonnet --api-key anthropic=YOUR_KEY
```

- Build a simple agent that can call tools (web search, file read/write) using the OpenAI function calling API directly. No framework yet.

### Day 5–7: Add Memory and Context

- Set up a local vector database (ChromaDB) and build a basic RAG pipeline:

```bash
pip install chromadb
```

- Create an agent that retrieves relevant context from a document collection before answering questions.
- Understand the difference between short-term (context window), long-term (vector DB), and procedural memory.

**Milestone**: A single agent that can search the web, read/write files, and retrieve context from a vector store.

---

## Week 2: Two-Agent Pipelines — CrewAI

**Goal**: Build multi-agent systems with clear role separation using the most accessible framework.

### Day 1–2: CrewAI Fundamentals

- Install CrewAI and complete the quickstart:

```bash
pip install crewai crewai-tools
```

- Build a research crew with a Researcher agent and a Writer agent (see [CrewAI deep dive](deep-dives/crewai.md) for full example).
- Understand roles, goals, backstories, tasks, and process types (sequential vs. hierarchical).

### Day 3–4: Add Critique and Iteration

- Add a Reviewer agent that evaluates the Writer's output and sends feedback.
- Implement a simple feedback loop where the Writer revises based on the Reviewer's comments.
- Experiment with hierarchical process mode where a Manager agent delegates tasks.

### Day 5–7: Real-World Project

- Build a multi-agent content pipeline:
    1. Agent 1 (Researcher): Searches the web for information on a topic.
    2. Agent 2 (Analyst): Synthesizes findings into key insights.
    3. Agent 3 (Writer): Produces a structured report.
    4. Agent 4 (Reviewer): Checks for accuracy and completeness.
- Experiment with parallel task execution for the research phase.

**Milestone**: A 3–4 agent pipeline that takes a topic and produces a reviewed research report.

---

## Week 3: Graph-Based Orchestration — LangGraph

**Goal**: Master the most flexible orchestration framework for production-grade workflows.

### Day 1–2: Graph Fundamentals

- Install LangGraph and understand the core abstractions:

```bash
pip install langgraph
```

- Build a simple graph with 3 nodes (research → synthesis → output).
- Understand StateGraph, typed state, conditional edges, and the compilation step.

### Day 3–4: Advanced Patterns

- Implement a **conditional routing** graph where the output of one agent determines the next agent.
- Build a **critique loop** using conditional edges (if review score < threshold → loop back to research).
- Add **parallel execution** with scatter-gather: multiple researcher agents working simultaneously.

### Day 5–7: Production Features

- Add **checkpointing** for fault recovery. Simulate a failure and resume from checkpoint.
- Implement **human-in-the-loop** by adding an interrupt node where the graph pauses for human review.
- Connect to **LangSmith** (free tier) for tracing and debugging agent interactions.
- If time permits, explore **subgraphs** for modular, reusable agent components.

**Milestone**: A LangGraph pipeline with conditional routing, parallel execution, checkpointing, and human-in-the-loop.

---

## Week 4: Software Engineering Agents & Advanced Topics

**Goal**: Apply multi-agent patterns to real software engineering tasks and explore the frontier.

### Day 1–2: Coding Agents

- Set up [OpenHands](deep-dives/openhands.md) locally with Docker:
    - Try the CLI to autonomously fix a bug in a test repository.
    - Explore the browser-based VS Code IDE interface.
- Review the [SWE-agent](deep-dives/swe-agent.md) architecture and understand the ACI (Agent-Computer Interface) concept.

### Day 3–4: AutoGen for Conversations

- Build an AutoGen group chat with specialized agents (coder, reviewer, tester):

```bash
pip install autogen-agentchat autogen-ext
```

- Experiment with different turn-taking strategies and observe how conversational dynamics affect output quality.
- Add a UserProxy agent for human-in-the-loop interaction.

### Day 5–6: Multi-Model Tiering

- Build a system that uses different models for different agents:
    - Fast/cheap model (GPT-4o-mini, Claude Haiku) for triage and routing.
    - Capable model (GPT-4o, Claude Sonnet) for complex reasoning.
    - Local model (Llama 3 via Ollama) for sensitive data processing.
- Measure cost and latency differences.

### Day 7: Explore the Frontier — Agentic Security Research

Multi-agent systems are rapidly expanding into security research, a domain with natural parallels to the plan-act-observe loop that agents excel at. Explore these three frontiers:

**Vulnerability Discovery with SWE-agent**

- SWE-agent's [Agent-Computer Interface](deep-dives/swe-agent.md) was designed for bug fixing, but the same architecture applies to offensive security. The repo explicitly lists cybersecurity as a supported use case. Experiment with pointing SWE-agent at a deliberately vulnerable codebase (e.g., OWASP Juice Shop or DVWA) and observe how the ACI navigates code to locate weaknesses.
- [SWE-smith](https://swesmith.com) (NeurIPS 2025) provides a pipeline for synthesizing bugs in real codebases — the inverse of this process is automated vulnerability generation.

**Multi-Agent Security Operations**

- Read [Multi-Agent LLM Orchestration Achieves Deterministic, High-Quality Decision Support for Incident Response](https://arxiv.org/abs/2511.15755) (Drammeh, 2025). In 348 controlled trials, multi-agent orchestration achieved a 100% actionable recommendation rate vs. 1.7% for single-agent approaches — an 80x improvement in action specificity with zero quality variance. This paper makes the case that multi-agent architecture is a production-readiness requirement, not a performance optimization, for LLM-based incident response.
- Security Data Pipeline Platforms are evolving into agentic systems where autonomous agents act as data engineers — generating parsing rules for unseen log formats, executing Sigma detection rules within the pipeline layer, and orchestrating extraction-transformation-loading alongside threat hunting.

**Hardware and Firmware Analysis Agents — Emerging Frontier**

- Multi-agent LLM frameworks are beginning to appear in hardware design space exploration. A [2025 paper](https://arxiv.org/abs/2512.08476) (Shih et al.) demonstrates specialized LLM agents for autonomous driving system DSE — agents handle user input interpretation, design point generation, execution orchestration, and analysis of visual/textual outputs from 3D simulations.
- The same multi-agent patterns apply to firmware analysis: one agent for binary disassembly and function identification, another for control flow analysis, a third for vulnerability pattern matching, and an orchestrator to synthesize findings. This remains largely unexplored territory with high potential for engineers who understand both hardware internals and agentic architectures.
- Cross-domain network orchestration using multi-agent workflows has been demonstrated across IP, optical, and robotic domains ([Xu et al., 2025](https://arxiv.org/abs/2410.10831)), suggesting the pattern generalizes to any domain requiring coordinated analysis of heterogeneous systems.

**Recommended reading:**

- [Live-SWE-agent](https://arxiv.org/abs/2511.13646) — self-evolving agents that improve their own scaffold at runtime
- [LEGOMem](https://arxiv.org/abs/2510.04851) — procedural memory for multi-agent workflow automation
- [SagaLLM](https://dl.acm.org/doi/10.14778/3750601.3750611) — transactional guarantees for multi-agent planning

**Milestone**: Hands-on experience with 4+ frameworks, understanding of production patterns, and a clear direction for further specialization — particularly at the intersection of multi-agent systems and security/hardware research.

---

## Summary Timeline

| Week | Focus | Framework | Output |
|------|-------|-----------|--------|
| 1 | Single agent + tools + memory | None (raw API) + Aider | Tool-using agent with RAG |
| 2 | Multi-agent role-based pipelines | CrewAI | 4-agent research pipeline |
| 3 | Graph-based orchestration + production | LangGraph | Production-ready graph with checkpoints |
| 4 | SE agents + advanced patterns | OpenHands, AutoGen | Multi-model tiered system |

## Recommended Resources

- **Courses**: [Coursera AI Specializations](https://www.coursera.org/), [Hugging Face Learn](https://huggingface.co/learn)
- **Roadmaps**: [The Ultimate AI Agents Roadmap 2025](https://www.the-ai-corner.com/p/the-ultimate-ai-agents-roadmap-2025) — curated training sources covering agent design, multi-agent systems, RAG, evaluation, memory, and deployment.
- **Community**: r/AI_Agents, r/LLMDevs, LangChain Discord, CrewAI Discord
- **Papers**: See [Changelog & Sources](meta.md) for the full list of academic references.
