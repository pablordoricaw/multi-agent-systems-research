# Multi-Agent Systems Research

## Executive Overview

Multi-agent systems (MAS) powered by large language models have moved from academic curiosity to production infrastructure in under two years. The core insight driving adoption is that orchestrating multiple specialized agents — each with a focused role, constrained context, and dedicated toolset — consistently outperforms monolithic single-agent approaches on complex tasks. A [2025 study on incident response](https://arxiv.org/abs/2511.15755) demonstrated this starkly: multi-agent orchestration achieved a 100% actionable recommendation rate versus 1.7% for single-agent systems, with zero quality variance across 348 controlled trials.

The landscape is dominated by a handful of open-source frameworks spanning two primary use cases. For **research and orchestration**, [LangGraph](https://github.com/langchain-ai/langgraph) (27k stars), [AutoGen](https://github.com/microsoft/autogen) (56k stars), and [CrewAI](https://github.com/crewAIInc/crewAI) (47k stars) lead with distinct philosophies — graph-based state machines, conversational collaboration, and role-based teams, respectively. For **software development**, [OpenHands](https://github.com/All-Hands-AI/OpenHands) (70k stars), [Aider](https://github.com/Aider-AI/aider) (42k stars), and [SWE-agent](https://github.com/SWE-agent/SWE-agent) (19k stars) dominate, with proprietary entrants like [Devin](https://devin.ai) (Cognition Labs) reaching enterprise adoption at Goldman Sachs and major banks. On the benchmark front, [Live-SWE-agent](https://arxiv.org/abs/2511.13646) achieved 77.4% on SWE-bench Verified, while the [Agentless](https://arxiv.org/abs/2407.01489) approach demonstrated that simpler three-phase pipelines can rival complex agent scaffolds at a fraction of the cost.

**Key trends shaping the field in 2025–2026:**

- **Convergence of frameworks**: Microsoft merged AutoGen with Semantic Kernel into a unified Agent Framework (October 2025). OpenAI shelved Swarm in favor of the Agents SDK. The industry is consolidating around fewer, more capable platforms.
- **Code-as-action paradigm**: The [CodeAct framework](https://www.emergentmind.com/topics/codeact-agent-framework) demonstrates 55–87% input token reduction by having agents emit executable Python rather than natural language directives.
- **Memory and state**: Procedural memory ([LEGOMem](https://arxiv.org/abs/2510.04851)), transactional guarantees ([SagaLLM](https://dl.acm.org/doi/10.14778/3750601.3750611)), and cognitive alignment ([OSC](https://arxiv.org/abs/2509.04876)) are active research frontiers addressing the brittleness of stateless agent interactions.
- **Self-evolving agents**: Live-SWE-agent autonomously evolves its own scaffold at runtime, pointing toward agents that improve themselves without human-designed architecture.
- **Emerging domains**: Multi-agent systems are spreading into security operations (agentic SOC pipelines), autonomous driving (design space exploration), healthcare administration, and cross-domain network orchestration.

**Open problems** include reliable long-horizon planning, inter-agent trust and coordination at scale, cost management for multi-model tiering, and robust evaluation beyond narrow benchmarks. For the raw mechanics behind every framework — agent loops, tool calling, state serialization, and the orchestration tax — see [Internals](internals.md). To see how these mechanics manifest in closed-source systems you use every day, see [Production Systems](production-systems/index.md), which reverse-engineers Claude Code and Perplexity Computer using the vocabulary from the rest of this site. For a practical guide to measuring whether your agent system actually works, see [Evaluation](evals.md).

---

*This research was compiled on March 23, 2026. Navigate the sections above for deep dives into each framework, comparisons, and a recommended learning path.*
