# Production Systems

This section reverse-engineers two production agentic systems you can use today — [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) (Anthropic) and [Perplexity Computer](https://www.perplexity.ai/) — using the vocabulary and mental models from the rest of this documentation site. The goal is to bridge the gap between the frameworks, internals, and patterns covered in earlier pages and the closed-source products that millions of engineers interact with daily.

---

## Methodology: The Five-Layer Stack

Every production system page follows the same five-layer analysis, moving from what you can directly observe down to what you could build yourself:

| Layer | Name | What It Answers | Evidence Standard |
|-------|------|-----------------|-------------------|
| 0 | **Observable Behavior** | What does the user see? What tools are exposed? What patterns are consistent? | Direct interaction — anyone can verify |
| 1 | **Inferred Architecture** | Given the observable behavior, what is the system *probably* doing internally? | Reasoned inference from behavior + published clues — labeled as **INFERRED** |
| 2 | **Published Information** | What has the company officially confirmed about the architecture? | Engineering blogs, papers, leaked system prompts, conference talks — labeled as **CONFIRMED** |
| 3 | **OSS Analog Mapping** | Which open-source frameworks from our deep dives implement similar patterns? | Direct architectural comparison to [Internals](../internals.md) and [Deep Dives](../deep-dives/langgraph.md) |
| 4 | **DIY Replication Path** | How would you build a simplified version using open-source models and tools? | Concrete component lists, model recommendations, framework choices |

!!! note "Why This Ordering Matters"
    Most documentation starts with published architecture and works outward. We start with **what you can observe yourself** because (a) these are closed-source systems where published details are incomplete, and (b) building the habit of inferring architecture from behavior is one of the most valuable engineering skills you can develop. When you encounter the next production system — one we haven't documented — you'll know how to analyze it yourself.

---

## Systems Covered

| System | Type | Architecture Pattern | Primary OSS Analogs |
|--------|------|---------------------|---------------------|
| [Claude Code](claude-code.md) | Agentic coding assistant | Single-agent + tool loop | [OpenHands](../deep-dives/openhands.md), [SWE-agent](../deep-dives/swe-agent.md), [Aider](../deep-dives/aider.md) |
| [Perplexity Computer](perplexity-computer.md) | Multi-agent research & task assistant | Orchestrator + specialized subagents | [AutoGen](../deep-dives/autogen.md), [LangGraph](../deep-dives/langgraph.md) |

These two systems were chosen because they represent the two dominant production architectures:

1. **Single-agent with sophisticated tooling** (Claude Code) — one powerful model with a rich tool set, extended thinking, and a well-crafted system prompt. This maps to the agent loop pattern from [Internals § 1](../internals.md#1-the-agent-loop).

2. **Multi-agent orchestration** (Perplexity Computer) — an orchestrator that delegates to specialized subagents running in parallel, each with their own tools and context. This maps to the handoff and orchestration patterns from [Internals § 4](../internals.md#4-handoffs-under-the-hood).

---

## How to Read These Pages

If you've already read the rest of this site, you have all the vocabulary you need. Each production system page will reference specific sections of:

- **[Internals](../internals.md)** — for the raw mechanics (agent loop, tool calling, state serialization)
- **[Deep Dives](../deep-dives/langgraph.md)** — for the OSS framework comparisons
- **[Evaluation](../evals.md)** — for how these systems are benchmarked
- **[Full Workflow](../workflow.md)** — for the end-to-end workflow patterns

If you haven't read those pages yet, start with [Internals](../internals.md) — it provides the foundation that makes everything here make sense.

---

*Added in [v1.3.0](../meta.md).*
