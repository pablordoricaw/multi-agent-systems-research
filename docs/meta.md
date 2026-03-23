# Changelog & Sources

## Changelog

### v1.3.0 — March 23, 2026

- New section: **Production Systems** — reverse-engineers two closed-source agentic systems using a five-layer methodology (Observable Behavior → Inferred Architecture → Published Information → OSS Analog Mapping → DIY Replication Path).
- New page: **Claude Code** (`production-systems/claude-code.md`) — deep dive into Anthropic's agentic coding assistant. Covers the single-agent + tool loop architecture, 17 exposed tools, extended thinking, context compaction, CLAUDE.md convention, SWE-bench benchmarks, and DIY replication with open-source models (Qwen2.5-Coder, DeepSeek, Devstral).
- New page: **Perplexity Computer** (`production-systems/perplexity-computer.md`) — deep dive into Perplexity's multi-agent research and task assistant. Covers the orchestrator + subagent architecture, Firecracker VM isolation, integrated search pipeline (200B+ URLs, Vespa AI), skill system, connector ecosystem, and DIY replication with LangGraph + open-source search.
- New page: **Production Systems Overview** (`production-systems/index.md`) — methodology explanation and system comparison table.
- Updated **Learning Path** (`learning-path.md`) — added Week 0 ("Production Systems — What You're Actually Using") before existing weeks. Curriculum is now 5 weeks. Existing Weeks 1–4 unchanged.
- Updated navigation: added Production Systems section between Internals and Getting Started.
- Updated Executive Overview in `index.md` with reference to Production Systems.
- Sources: 50+ new web sources, engineering blogs, system prompt analyses, and benchmark data referenced.

### v1.0.0 — March 23, 2026

- Initial research compilation covering 10 frameworks and tools across research/orchestration and software development agents.
- Deep dives for: LangGraph, AutoGen, CrewAI, OpenAI Agents SDK, Agency Swarm, SWE-agent, OpenHands, Aider, Patchwork, Devin.
- Comparison tables, decision matrix, and maturity assessment.
- 4-week learning path for engineers.
- Sources: 15+ academic papers, 10+ GitHub repositories, industry reports, and framework documentation.

### v1.1.0 — March 23, 2026

- Replaced OpenAI Swarm deep dive with OpenAI Agents SDK deep dive (Swarm's production successor).
- Added `site_url` to `mkdocs.yml` for canonical URLs.
- Created root `README.md` with project description and live-docs link.
- Expanded Day 7 of the learning path with agentic security research use cases: vulnerability discovery (SWE-agent), multi-agent SOC operations (arXiv:2511.15755), and hardware/firmware analysis agents.
- Updated comparison tables and decision matrix to reflect Agents SDK.

### v1.2.0 — March 23, 2026

- New page: **Internals** (`internals.md`) — raw agent loop, tool calling wire format, state serialization patterns (full replay vs. typed state vs. role-scoped memory), handoff mechanics, framework philosophy tradeoffs, and the orchestration tax with empirical data.
- New page: **Evaluation** (`evals.md`) — why evals are hard, what to evaluate (5 dimensions), evaluation approaches (LLM-as-judge, deterministic checks, trajectory eval, human eval), benchmark landscape (SWE-bench, GAIA, HumanEval, AgentBench, WebArena, ToolBench, BFCL, TAU-bench), eval tools comparison, and a minimal 3-layer eval setup with code.
- New page: **Full Workflow** (`workflow.md`) — end-to-end annotated walkthroughs for research and software development workflows with concrete data payloads, failure tables, and Mermaid comparison diagram.
- Rewrote **Learning Path** (`learning-path.md`) — surveyed 11 external resources (DeepLearning.AI, HuggingFace, Microsoft GitHub, UC Berkeley, Maven, Anthropic, Lilian Weng), restructured Week 1 to start with raw internals before frameworks, added eval day (Week 2 Day 7), added framework comparison day (Week 3 Day 7), preserved v1.1.0 Day 7 security frontier content.
- Updated navigation: added Internals, Full Workflow, and Evaluation to `mkdocs.yml` nav.
- Updated Executive Overview in `index.md` with references to new Internals and Evaluation pages.
- Sources: 40+ new web sources, 5+ new academic papers referenced.

---

## Academic Sources

| # | Paper | Authors | Venue / Year | DOI / Link | Code Available |
|---|-------|---------|--------------|------------|----------------|
| 1 | SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering | Yang et al. | NeurIPS 2024 | [arXiv:2405.15793](https://arxiv.org/abs/2405.15793) | Yes — [GitHub](https://github.com/SWE-agent/SWE-agent) |
| 2 | Agentless: Demystifying LLM-based Software Engineering Agents | Xia et al. | ACM 2025 | [DOI:10.1145/3715754](https://dl.acm.org/doi/10.1145/3715754) | Yes — [GitHub](https://github.com/OpenAutoCoder/Agentless) |
| 3 | Live-SWE-agent: Can Software Engineering Agents Self-Evolve on the Fly? | Xia et al. | arXiv 2025 | [arXiv:2511.13646](https://arxiv.org/abs/2511.13646) | Yes — [GitHub](https://github.com/OpenAutoCoder/live-swe-agent) |
| 4 | HALO: Hierarchical Autonomous Logic-Oriented Orchestration | Hou et al. | arXiv 2025 | [arXiv:2505.13516](https://arxiv.org/abs/2505.13516) | Yes — [GitHub](https://github.com/23japhone/HALO) |
| 5 | SagaLLM: Context Management, Validation, and Transaction Guarantees | Chang & Geng | VLDB 2025 | [DOI:10.14778/3750601.3750611](https://dl.acm.org/doi/10.14778/3750601.3750611) | No |
| 6 | LEGOMem: Modular Procedural Memory for Multi-agent LLM Systems | Han et al. | arXiv 2025 | [arXiv:2510.04851](https://arxiv.org/abs/2510.04851) | No |
| 7 | OSC: Cognitive Orchestration through Dynamic Knowledge Alignment | Zhang et al. | arXiv 2025 | [arXiv:2509.04876](https://arxiv.org/abs/2509.04876) | No |
| 8 | MASAI: Modular Architecture for Software-engineering AI Agents | Arora et al. | arXiv 2024 | [arXiv:2406.11638](https://arxiv.org/abs/2406.11638) | No |
| 9 | ChatDev: Communicative Agents for Software Development | Qian et al. | arXiv 2024 | [arXiv:2307.07924](https://arxiv.org/abs/2307.07924) | Yes — [GitHub](https://github.com/OpenBMB/ChatDev) |
| 10 | HyperAgent: Generalist Software Engineering Agents | Phan et al. | arXiv 2024 | [arXiv:2409.16299](https://arxiv.org/abs/2409.16299) | No |
| 11 | The OpenHands Software Agent SDK | OpenHands Team | arXiv 2025 | [arXiv:2511.03690](https://arxiv.org/html/2511.03690v1) | Yes — [GitHub](https://github.com/All-Hands-AI/OpenHands) |
| 12 | SWE-smith: Scaling Data for Software Engineering Agents | Yang et al. | NeurIPS 2025 | [NeurIPS Poster](https://neurips.cc/virtual/2025/poster/121828) | Yes — [swesmith.com](https://swesmith.com) |
| 13 | SWE-Debate: Competitive Multi-Agent Debate for Software Issue Resolution | Li et al. | arXiv 2025 | [arXiv:2507.23348](https://arxiv.org/abs/2507.23348) | No |
| 14 | SWE-Dev: Evaluating and Training Autonomous Feature-Driven Software Development | Du et al. | arXiv 2025 | [arXiv:2505.16975](https://arxiv.org/abs/2505.16975) | Yes — [GitHub](https://github.com/DorothyDUUU/SWE-Dev) |
| 15 | Large Language Model-Based Agents for Software Engineering: A Survey | Liu et al. | arXiv 2024 | [arXiv:2409.02977](https://arxiv.org/abs/2409.02977) | No |
| 16 | Multi-Agent Real-time Chat Orchestration (MARCO) | Shrimal et al. | arXiv 2024 | [arXiv:2410.21784](https://arxiv.org/abs/2410.21784) | No |
| 17 | Multi-Agent LLM Orchestration Achieves Deterministic Decision Support for Incident Response | Drammeh | arXiv 2025 | [arXiv:2511.15755](https://arxiv.org/abs/2511.15755) | Yes |
| 18 | OmniNova: A General Multimodal Agent Framework | Du | arXiv 2025 | [arXiv:2503.20028](https://arxiv.org/abs/2503.20028) | No |
| 19 | A Multi-Agent LLM Framework for Design Space Exploration in Autonomous Driving | Shih et al. | arXiv 2025 | [arXiv:2512.08476](https://arxiv.org/abs/2512.08476) | No |
| 20 | VoCare AI: A Multi-Agent LLM Workflow for Improved Clinic Operational Efficiency | Han et al. | IEEE TENCON 2025 | [IEEE:11375163](https://ieeexplore.ieee.org/document/11375163/) | Yes — [GitHub](https://github.com/DerrickLJH2000/langgraph-voice-assistant) |

## GitHub Repositories Referenced

| Repository | Stars | Language | License | Last Active |
|------------|-------|----------|---------|-------------|
| [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) | 27,241 | Python | MIT | 2026-03-23 |
| [microsoft/autogen](https://github.com/microsoft/autogen) | 56,069 | Python | CC-BY-4.0 | 2026-03-21 |
| [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) | 46,965 | Python | MIT | 2026-03-23 |
| [openai/swarm](https://github.com/openai/swarm) | 21,211 | Python | MIT | 2025-03-11 |
| [VRSEN/agency-swarm](https://github.com/VRSEN/agency-swarm) | 4,108 | Python | MIT | 2026-03-23 |
| [SWE-agent/SWE-agent](https://github.com/SWE-agent/SWE-agent) | 18,821 | Python | MIT | 2026-03-23 |
| [All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands) | 69,606 | Python | Custom | 2026-03-23 |
| [Aider-AI/aider](https://github.com/Aider-AI/aider) | 42,286 | Python | Apache-2.0 | 2026-03-17 |
| [openai/openai-agents-python](https://github.com/openai/openai-agents-python) | 20,220 | Python | MIT | 2026-03-23 |
| [patched-codes/patchwork](https://github.com/patched-codes/patchwork) | 1,548 | Python | AGPL-3.0 | 2025-04-18 |
| [OpenAutoCoder/Agentless](https://github.com/OpenAutoCoder/Agentless) | 2,022 | Python | MIT | 2024-12-22 |
| [OpenAutoCoder/live-swe-agent](https://github.com/OpenAutoCoder/live-swe-agent) | 339 | Python | MIT | 2026-01-19 |

## Web Sources

- [AutoGen — Microsoft Research](https://www.microsoft.com/en-us/research/project/autogen/)
- [LangGraph: Agent Orchestration Framework](https://www.langchain.com/langgraph)
- [CrewAI: The Leading Multi-Agent Platform](https://crewai.com)
- [Devin's 2025 Performance Review — Cognition Labs](https://cognition.ai/blog/devin-annual-performance-review-2025)
- [Goldman Sachs Deploys Devin — IBM Think](https://www.ibm.com/think/news/goldman-sachs-first-ai-employee-devin)
- [OpenHands: The Open Platform for Cloud Coding Agents](https://openhands.dev)
- [Aider: AI Pair Programming in Your Terminal](https://aider.chat)
- [Patched.codes: Production-ready Agentic Workflows](https://www.patched.codes)
- [CrewAI vs LangGraph vs AutoGen — DataCamp](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)
- [Best Multi-Agent Frameworks in 2026 — GuruSup](https://gurusup.com/blog/best-multi-agent-frameworks-2026)
- [Best AI Agent Frameworks 2025 — Maxim AI](https://www.getmaxim.ai/articles/top-5-ai-agent-frameworks-in-2025-a-practical-guide-for-ai-builders/)
- [CodeAct Agent Framework — Emergent Mind](https://www.emergentmind.com/topics/codeact-agent-framework)
- [OpenAI Agents SDK Documentation](https://openai.github.io/openai-agents-python/)
- [OpenAI Agents SDK — Mem0 Review](https://www.mem0.ai/blog/openai-agents-sdk-review/)
- [OpenAI Agents SDK — 0DeepResearch Analysis](https://0deepresearch.com/blog/post/openai-agents-sdk)
- [OpenAI Agents SDK — Fast.io Comprehensive Guide](https://fast.io/blog/a-comprehensive-guide-to-the-openai-agents-sdk)
- [OpenAI Swarm Announcement — Campus Technology](https://campustechnology.com/articles/2024/10/29/new-openai-swarm-framework-offers-experimental-tool-for-multi-agent-ai-networks.aspx)
- [How to Learn Agentic AI in 2025 — The Hustling Engineer](https://thehustlingengineer.substack.com/p/how-to-learn-agentic-ai-in-2025-a)
- [The Ultimate AI Agents Roadmap 2025 — The AI Corner](https://www.the-ai-corner.com/p/the-ultimate-ai-agents-roadmap-2025)
- [OpenAI Function Calling Documentation](https://developers.openai.com/api/docs/guides/function-calling/)
- [Build an AI Agent Loop in 50 Lines of Python — DEV Community](https://dev.to/klement_gunndu/build-an-ai-agent-loop-in-50-lines-of-python-59jk)
- [The Agent Execution Loop — Victor Dibia](https://victordibia.com/blog/agent-execution-loop/)
- [ReAct Agent with OpenAI Function Calling — Peter Roelants](https://peterroelants.github.io/posts/react-openai-function-calling/)
- [Modeling and Mitigating Error Cascades in LLM-Based Multi-Agent Systems — arXiv:2603.04474](https://arxiv.org/html/2603.04474v1)
- [Why Your Multi-Agent System is Failing — Towards Data Science](https://towardsdatascience.com/why-your-multi-agent-system-is-failing-escaping-the-17x-error-trap-of-the-bag-of-agents/)
- [Why Bad Agentic AI Latency Costs You — Parloa](https://www.parloa.com/knowledge-hub/agentic-ai-latency-cost/)
- [OpenAI Agents SDK Handoffs Documentation](https://openai.github.io/openai-agents-python/handoffs/)
- [OpenAI Agents SDK Sessions Documentation](https://openai.github.io/openai-agents-python/sessions/)
- [CrewAI Memory Documentation](https://docs.crewai.com/en/concepts/memory)
- [LangGraph Persistence Documentation](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Comparing Open-Source AI Agent Frameworks — Langfuse](https://langfuse.com/blog/2025-03-19-ai-agent-comparison)
- [Why We No Longer Evaluate SWE-bench Verified — OpenAI](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)
- [AI Agent Evaluation Benchmarks vs Production — Chanl AI](https://www.chanl.ai/blog/ai-agent-evaluation-benchmarks-predict-production)
- [Benchmark Reliability for Enterprise Agents — simmering.dev](https://simmering.dev/blog/agent-benchmarks/)
- [GAIA Leaderboard — HAL Princeton](https://hal.cs.princeton.edu/gaia)
- [LLM-as-Judge Best Practices — Monte Carlo Data](https://www.montecarlodata.com/blog-llm-as-judge/)
- [Deterministic Assertion Library — Promptfoo](https://www.promptfoo.dev/docs/configuration/expected-outputs/deterministic/)
- [Trajectory Evaluation — LangChain](https://docs.langchain.com/langsmith/trajectory-evals)
- [Hallucination Detection with LLM Judge — Datadog](https://www.datadoghq.com/blog/ai/llm-hallucination-detection/)
- [Inspect AI — UK AI Security Institute](https://inspect.aisi.org.uk)
- [Inside the Architecture of a Deep Research Agent — Egnyte](https://www.egnyte.com/blog/post/inside-the-architecture-of-a-deep-research-agent/)
- [VMAO: Verified Multi-Agent Orchestration — arXiv:2603.11445](https://arxiv.org/html/2603.11445v2)
- [SWE-agent ACI Documentation](https://swe-agent.com/0.7/background/aci/)
- [AI Agent Design Patterns — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Human-in-the-Loop Agents with interrupt — LangChain Blog](https://blog.langchain.com/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt/)
- [LLM Powered Autonomous Agents — Lilian Weng](https://lilianweng.github.io/posts/2023-06-23-agent/)
- [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
- [How We Built Our Multi-Agent Research System — Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Multi AI Agent Systems with crewAI — DeepLearning.AI](https://www.deeplearning.ai/short-courses/multi-ai-agent-systems-with-crewai/)
- [AI Agents in LangGraph — DeepLearning.AI](https://www.deeplearning.ai/short-courses/ai-agents-in-langgraph/)
- [AI Agentic Design Patterns with AutoGen — DeepLearning.AI](https://www.deeplearning.ai/short-courses/ai-agentic-design-patterns-with-autogen/)
- [Building Agentic RAG with LlamaIndex — DeepLearning.AI](https://www.deeplearning.ai/short-courses/building-agentic-rag-with-llamaindex/)
- [HuggingFace AI Agents Course](https://huggingface.co/learn/agents-course/)
- [Microsoft AI Agents for Beginners — GitHub](https://github.com/microsoft/ai-agents-for-beginners)
- [UC Berkeley LLM Agents MOOC (Fall 2024)](https://llmagents-learning.org/f24)
- [UC Berkeley Agentic AI (Fall 2025)](https://rdi.berkeley.edu/agentic-ai/f25)
- [Agentic AI Engineering Bootcamp — Maven](https://maven.com/stemplicity/become-an-agentic-ai-engineer)

---

*All star counts and last-active dates were collected on March 23, 2026 via the GitHub API.*
