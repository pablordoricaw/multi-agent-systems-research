# Comparison Table

A structured comparison of the major multi-agent frameworks and software development agents covered in this research.

## Research & Orchestration Frameworks

| Framework | Stars | Orchestration Model | Use Case Fit | Local | Remote | Language | Maturity | Learning Curve |
|-----------|-------|-------------------|--------------|-------|--------|----------|----------|----------------|
| [LangGraph](deep-dives/langgraph.md) | 27.2k | Graph state machine | Complex branching workflows, pipelines | Yes | LangGraph Cloud | Python, TS | Production | Moderate |
| [AutoGen](deep-dives/autogen.md) | 56.1k | Conversational collaboration | Dynamic dialogues, human-in-the-loop | Yes | Distributed | Python | Production | Moderate |
| [CrewAI](deep-dives/crewai.md) | 47.0k | Role-based teams | Business workflows, rapid prototyping | Yes | CrewAI AMP | Python | Production (growing) | Low |
| [OpenAI Swarm](deep-dives/openai-swarm.md) | 21.2k | Agent handoffs | Learning, prototyping only | Yes | No | Python | Experimental (archived) | Low |
| [Agency Swarm](deep-dives/agency-swarm.md) | 4.1k | Hierarchical agencies | Business automation, API workflows | Yes | No | Python | Growing | Low-Moderate |

## Software Development Agents

| Tool | Stars | Agent Type | Autonomous | Sandbox | Open Source | Key Strength |
|------|-------|-----------|------------|---------|-------------|--------------|
| [OpenHands](deep-dives/openhands.md) | 69.6k | Full-stack coding platform | Yes | Docker | Yes (custom license) | Complete SDK + enterprise integrations |
| [Aider](deep-dives/aider.md) | 42.3k | AI pair programmer | No (human-in-loop) | No | Yes (Apache-2.0) | Best terminal DX, 100+ languages |
| [SWE-agent](deep-dives/swe-agent.md) | 18.8k | Issue resolution agent | Yes | Docker | Yes (MIT) | ACI design, research ecosystem |
| [Devin](deep-dives/devin.md) | N/A | AI software engineer | Yes | Cloud | No (proprietary) | Enterprise scale, 67% PR merge rate |
| [Patchwork](deep-dives/patchwork.md) | 1.5k | SDLC automation | Semi (CI/CD triggered) | No | Yes (AGPL-3.0) | Outer-loop automation, self-hosted |

## Head-to-Head: Orchestration Philosophies

| Dimension | LangGraph | AutoGen | CrewAI |
|-----------|-----------|---------|--------|
| **Core Metaphor** | Directed graph | Conversation | Team with roles |
| **State Management** | Centralized typed state | Message-based history | Role-based memory + RAG |
| **Control Flow** | Explicit nodes & edges | Dynamic turn-taking | Sequential / hierarchical |
| **Parallel Execution** | Scatter-gather, pipeline | Limited | Task-level parallelism |
| **Human-in-the-Loop** | Interrupt at any node | UserProxy agent | Task checkpoint hooks |
| **Streaming** | Per-node token streaming | Limited | Limited |
| **Observability** | LangSmith integration | OpenTelemetry | Crew Control Plane |
| **Checkpointing** | Built-in, time-travel | No | Limited |
| **Config Format** | Python code | Python code | YAML or Python |
| **Prototyping Speed** | Slow (graph design) | Medium | Fast |
| **Production Readiness** | High | Medium-High | Medium |

## Maturity Assessment

```
Production-Ready ─────────────────────────────────────── Experimental
│                                                        │
│  LangGraph    Aider    OpenHands    Devin              │
│      AutoGen    SWE-agent                              │
│          CrewAI                                         │
│              Agency Swarm                              │
│                  Patchwork                              │
│                                     OpenAI Swarm       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Decision Matrix

**Choose based on your primary need:**

| If you need... | Use... | Why |
|----------------|--------|-----|
| Complex branching workflows | LangGraph | Explicit graph control, checkpointing, streaming |
| Quick multi-agent prototype | CrewAI | Fastest setup, intuitive role metaphor |
| Dynamic agent conversations | AutoGen | Conversational paradigm, Microsoft ecosystem |
| AI pair programming | Aider | Best terminal DX, auto-git, 100+ languages |
| Autonomous coding platform | OpenHands | Full SDK, Docker sandbox, enterprise integrations |
| Research-grade SE agent | SWE-agent | ACI design, training pipeline (SWE-smith) |
| Enterprise code at scale | Devin | Proven at Goldman Sachs, migrations, security |
| CI/CD automation | Patchwork | Self-hosted, PR reviews, security patching |
| Learning multi-agent basics | OpenAI Swarm | Simplest possible implementation |
| OpenAI ecosystem + structure | Agency Swarm | Extends Agents SDK with practical features |

## Cross-Domain Presence

Some frameworks appear in both academia and industry:

| Framework | Academic Publications | Industry Adoption | Crossover |
|-----------|----------------------|-------------------|-----------|
| LangGraph | Referenced in healthcare, orchestration papers | LangChain ecosystem, enterprise | High |
| AutoGen | Microsoft Research papers, WEF 2025 | Microsoft Agent Framework, enterprises | High |
| CrewAI | Simulation testing research | 1.7B workflows, AMP platform | Medium |
| SWE-agent | NeurIPS 2024, NeurIPS 2025, multiple papers | Growing industry adoption | High |
| OpenHands | arXiv 2025 SDK paper, SWE-Dev benchmark | 69k stars, enterprise partnerships | High |
