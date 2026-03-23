# Learning Path

An opinionated 4-week curriculum for engineers who want to go from zero to production-ready multi-agent systems. Each week has a concrete milestone. Do the work — don't just read.

---

## Resources Consulted

The following learning resources were reviewed to identify what the field covers well and where gaps remain. This curriculum is designed to fill those gaps.

| Resource | What It Covers | Strengths | Gaps |
|----------|---------------|-----------|------|
| **DeepLearning.AI — Multi AI Agent Systems with crewAI** (2h41m, taught by crewAI creator João Moura) | Role-based agents, task delegation, hierarchical process, CrewAI internals | Authoritative source; covers CrewAI deeply | Framework-specific only; no framework-agnostic thinking |
| **DeepLearning.AI — AI Agents in LangGraph** (1h32m, taught by Harrison Chase) | StateGraph, typed state, conditional routing, checkpointing | Concise and practical; taught by the LangGraph author | Minimal coverage of production observability |
| **DeepLearning.AI — AI Agentic Design Patterns with AutoGen** (1h25m, taught by AutoGen creators) | Conversational patterns, group chat, turn-taking strategies | Covers AutoGen patterns not documented elsewhere | Outdated post-AutoGen 0.4 merge with Semantic Kernel |
| **DeepLearning.AI — Building Agentic RAG with LlamaIndex** (44m) | RAG pipelines, document agents, query routing | Fast path to RAG agents | Very short; no multi-agent coordination |
| **HuggingFace Learn AI Agents Course** (free) | smolagents, LangGraph, LlamaIndex, competitive leaderboard | Free; covers three frameworks; community competition keeps it current | Lighter on production patterns |
| **Microsoft AI Agents for Beginners** (54K GitHub stars, 18 lessons) | Agent fundamentals, MCP protocol, A2A protocol, multi-framework overview | Broadest protocol coverage; MCP/A2A sections are unique | Beginner-oriented; thin on architecture depth |
| **UC Berkeley LLM Agents MOOC** (Fall 2024 + Fall 2025 + Spring 2025 advanced; guest speakers from OpenAI, DeepMind, Meta) | Theoretical foundations, reasoning, planning, emerging research | Best academic depth; frontier research from practitioner speakers | Not hands-on; requires strong ML background |
| **Maven Agentic AI Engineering Bootcamp** ($1,200, 6 weeks) | End-to-end production agent pipelines, LangGraph, AutoGen, LlamaIndex, deployment | Most production-complete paid course; includes 3 projects and live Q&A | Cost; cohort-only format |
| **Anthropic — "Building Effective Agents"** (guide) | Workflows vs. autonomous agents, failure modes, prompt engineering for agents | The canonical practitioner guide; concise and opinionated | No code examples |
| **Anthropic — "How We Built Our Multi-Agent Research System"** (engineering post) | Real production architecture, orchestration decisions, cost management | Rare production case study from a frontier lab | Single system; may not generalize |
| **Lilian Weng — "LLM Powered Autonomous Agents"** (blog post) | ReAct loop, tool use, memory taxonomy, planning taxonomy | Canonical conceptual reference; well-cited in the literature | Pre-dates most modern frameworks |

**Key gaps across all reviewed resources:**

- No framework-agnostic architecture thinking — almost every course teaches one framework in isolation
- No practical evaluation curriculum — how to actually measure whether your agents are working
- Minimal production observability — tracing, cost attribution, failure diagnosis
- No cost optimization content — multi-agent systems can be expensive; nobody teaches you how to manage this

This curriculum addresses all four gaps explicitly.

---

## Week 1: Foundations — The Raw Mechanics

**Goal:** Understand what's happening under the hood before touching any framework.

If you start with CrewAI or LangGraph on day one, you'll use magic you don't understand. When it breaks in production, you won't know why. Spend this week building the primitives by hand.

### Day 1–2: The Agent Loop

Read the [Internals](internals.md) page. Every framework — CrewAI, LangGraph, AutoGen — is a wrapper around the same 20-line loop:

```python
def agent_loop(system_prompt, tools, max_iterations=10):
    messages = [{"role": "system", "content": system_prompt}]
    for _ in range(max_iterations):
        response = llm.chat(messages, tools=tools)
        if response.finish_reason == "stop":
            return response.content
        # Execute tool call, append result to message history
        tool_result = execute_tool(response.tool_calls[0])
        messages.append({"role": "tool", "content": tool_result})
    raise MaxIterationsError()
```

Build this yourself. Understand that the entire conversation history is replayed on every LLM call — this is why context window management matters. Read [Lilian Weng's "LLM Powered Autonomous Agents"](https://lilianweng.github.io/posts/2023-06-23-agent/) for the foundational taxonomy of agent components: planning, memory, and tool use.

### Day 3–4: Tool Calling Deep Dive

Build a single agent with web search + file read/write using the raw OpenAI function calling API — no wrappers. This forces you to understand how tool schemas work, how the model decides when to call a tool, and how to handle multi-step tool chains.

```bash
pip install openai
```

Reference DeepLearning.AI's "Functions, Tools and Agents with LangChain" for the function calling mechanics, then implement the same pattern without LangChain to see what the framework is hiding.

Key things to understand:
- Tool schemas are JSON Schema — the model reads your docstrings and parameter types
- Parallel tool calling: the model can emit multiple tool calls in a single response
- Tool errors must be handled gracefully — return structured error messages, not exceptions

### Day 5–6: Memory and State

Set up ChromaDB and build a basic RAG pipeline from scratch:

```bash
pip install chromadb openai
```

Understand the three memory types you'll use in production:

| Memory Type | Storage | Use Case |
|-------------|---------|----------|
| Short-term | Context window | Current task context |
| Long-term | Vector DB (ChromaDB, Pinecone) | Facts, past research, user preferences |
| Procedural | Structured store | Reusable task execution patterns ([LEGOMem](https://arxiv.org/abs/2510.04851)) |

Reference the [Internals](internals.md) page section on state serialization patterns. The key insight: when an agent "remembers" something, it's either in the context window (ephemeral) or written to an external store (persistent). There is no magic.

### Day 7: When NOT to Use Agents

Read Anthropic's ["Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) guide. The most important thing you'll learn this week is when a simple workflow beats an autonomous agent. Anthropic's framing: prefer deterministic code over agent autonomy when the task is well-defined. Reserve autonomy for tasks where the solution space is too large to enumerate.

**Milestone:** A single agent with tools, memory, and RAG — built without any framework.

---

## Week 2: Multi-Agent Pipelines — CrewAI

**Goal:** Build multi-agent systems with role separation.

CrewAI is the right first multi-agent framework because its abstractions map to how you naturally think about teams. Agents have roles, goals, and backstories. Tasks have expected outputs. The framework handles the orchestration.

### Day 1–2: CrewAI Fundamentals

```bash
pip install crewai crewai-tools
```

Reference the [DeepLearning.AI "Multi AI Agent Systems with crewAI" course](https://learn.deeplearning.ai/courses/multi-ai-agent-systems-with-crewai) and the [CrewAI deep dive](deep-dives/crewai.md). Build a researcher + writer crew:

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="Senior Research Analyst",
    goal="Find accurate, up-to-date information on the given topic",
    backstory="Expert at synthesizing technical papers and web sources",
    verbose=True,
)

writer = Agent(
    role="Technical Writer",
    goal="Produce clear, well-structured documentation",
    backstory="Skilled at making complex topics accessible to engineers",
)

research_task = Task(
    description="Research the top multi-agent frameworks in 2025-2026",
    expected_output="Structured summary with pros/cons for each framework",
    agent=researcher,
)

writing_task = Task(
    description="Write a getting-started guide from the research",
    expected_output="Markdown document with code examples, 500-800 words",
    agent=writer,
    context=[research_task],
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential,
    verbose=True,
)
result = crew.kickoff()
```

Understand: roles, goals, backstories, sequential vs. hierarchical process types.

### Day 3–4: Critique Loops and Hierarchical Process

Add a reviewer agent that evaluates the writer's output against defined criteria. Use hierarchical process mode with a Manager agent that delegates and reviews:

```python
manager = Agent(
    role="Editorial Manager",
    goal="Ensure all output meets quality and accuracy standards",
    backstory="Experienced editor with deep technical knowledge",
)

crew = Crew(
    agents=[researcher, writer, reviewer],
    tasks=[...],
    process=Process.hierarchical,
    manager_agent=manager,
)
```

The critique loop is one of the most valuable patterns in multi-agent systems — it dramatically improves output quality without increasing task complexity.

### Day 5–6: Real-World Pipeline

Build a 4-agent content pipeline with parallel task execution:

```
Researcher → Analyst → Writer → Reviewer
```

Reference the [Full Workflow page](workflow.md) for the research workflow pattern. Add `async_execution=True` to tasks that can run in parallel. Measure wall-clock time improvement vs. sequential execution.

### Day 7: Evaluation

Read the [Evaluation page](evals.md). Build a basic eval pipeline for your research crew:

1. **Deterministic checks**: Does the output contain required sections? Is it within the expected length range? Does it cite sources?
2. **LLM-as-judge faithfulness check**: Does the final document contain claims that aren't supported by the researcher's output?

```python
def faithfulness_check(research: str, final_output: str) -> float:
    prompt = f"""Rate 0-1 whether every claim in the output is supported by the research.
    Research: {research}
    Output: {final_output}
    Score:"""
    return float(llm.complete(prompt))
```

**Milestone:** A 4-agent pipeline with a basic eval suite.

---

## Week 3: Graph-Based Orchestration — LangGraph

**Goal:** Master the most flexible orchestration framework.

LangGraph is harder to learn than CrewAI but more powerful. It's the right choice for production systems with complex branching logic, human-in-the-loop requirements, or fault recovery needs. See the [LangGraph deep dive](deep-dives/langgraph.md) for full architectural details.

### Day 1–2: LangGraph Fundamentals

```bash
pip install langgraph langchain-openai
```

Reference the [DeepLearning.AI "AI Agents in LangGraph" course](https://learn.deeplearning.ai/courses/ai-agents-in-langgraph). Core concepts: `StateGraph`, typed state with `TypedDict`, conditional edges, graph compilation.

```python
from langgraph.graph import StateGraph, MessagesState
from typing import TypedDict, Annotated
import operator

class ResearchState(TypedDict):
    query: str
    research: str
    draft: str
    feedback: str
    iteration: int

graph = StateGraph(ResearchState)
graph.add_node("researcher", researcher_node)
graph.add_node("writer", writer_node)
graph.add_node("reviewer", reviewer_node)
graph.add_edge("researcher", "writer")
graph.add_conditional_edges(
    "reviewer",
    lambda state: "writer" if state["iteration"] < 3 else "end",
    {"writer": "writer", "end": "__end__"},
)
graph.set_entry_point("researcher")
app = graph.compile()
```

Build a simple 3-node graph. Understand that the graph is compiled — this is what enables checkpointing and replay.

### Day 3–4: Advanced Patterns

Implement the patterns that make LangGraph worth learning:

- **Conditional routing**: Branch based on output content (e.g., route to different analysts based on topic classification)
- **Critique loops**: Writer → Reviewer → Writer with an iteration counter to prevent infinite loops
- **Parallel scatter-gather**: Fan out to multiple researcher nodes, merge results with a reducer

```python
# Parallel execution with reducers
class State(TypedDict):
    results: Annotated[list, operator.add]  # Reducer: append, don't overwrite
```

### Day 5–6: Production Features

Three features that separate LangGraph prototypes from production systems:

**Checkpointing** — fault recovery without rerunning the entire graph:
```python
from langgraph.checkpoint.memory import MemorySaver
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)
# Resume from any node after failure
result = app.invoke(input, config={"configurable": {"thread_id": "run-123"}})
```

**Human-in-the-loop** — pause at decision points for human review:
```python
app = graph.compile(interrupt_before=["publish_node"])
# Graph pauses; human reviews state; then resume
app.invoke(None, config={"configurable": {"thread_id": "run-123"}})
```

**LangSmith tracing** — full execution visibility:
```bash
export LANGCHAIN_API_KEY=your_key
export LANGCHAIN_TRACING_V2=true
```

Reference [workflow.md](workflow.md) for the human review gate pattern used in the full research workflow.

### Day 7: Framework Comparison

Take the 4-agent pipeline you built in Week 2 and implement the same workflow in LangGraph. Document the tradeoffs:

| Dimension | CrewAI | LangGraph |
|-----------|--------|-----------|
| Setup time | Low | Medium |
| Flexibility | Medium | High |
| Debugging | Role-based logs | Full graph traces |
| Human-in-the-loop | Limited | First-class |
| Checkpointing | No | Yes |
| Learning curve | Low | Medium |

Reference the [Internals page](internals.md) section on framework philosophy tradeoffs.

**Milestone:** A LangGraph pipeline with conditional routing, checkpointing, and human-in-the-loop.

---

## Week 4: Software Engineering Agents & Advanced Topics

**Goal:** Apply multi-agent patterns to software engineering and explore the frontier.

### Day 1–2: Coding Agents with OpenHands

Set up [OpenHands](https://github.com/All-Hands-AI/OpenHands) locally — the most capable open-source software engineering agent:

```bash
docker pull docker.all-hands.dev/all-hands-ai/runtime:0.20-nikolaik
docker run -it --rm \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.20-nikolaik \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 3000:3000 \
  ghcr.io/all-hands-ai/openhands:0.20
```

Review the [SWE-agent ACI pattern](deep-dives/swe-agent.md) — the Agent-Computer Interface is the key architectural innovation that makes software agents effective. Reference [workflow.md](workflow.md) for the full SWE workflow. Try fixing a bug in a test repository and observe how the agent navigates the codebase.

Also see the [OpenHands deep dive](deep-dives/openhands.md) for a comparison of its CodeAct architecture against SWE-agent.

### Day 3–4: AutoGen for Conversational Multi-Agent Systems

```bash
pip install autogen-agentchat autogen-ext[openai]
```

Reference the [DeepLearning.AI "AI Agentic Design Patterns with AutoGen" course](https://learn.deeplearning.ai/courses/ai-agentic-design-patterns-with-autogen) and the [AutoGen deep dive](deep-dives/autogen.md). Build a group chat with coder + reviewer + tester agents. Experiment with different turn-taking strategies: round-robin, selector (LLM-based routing), and custom speaker selection.

AutoGen's strength is conversational workflows where agents need to debate or negotiate — code review, architecture decisions, multi-perspective analysis.

### Day 5–6: Multi-Model Tiering

One of the highest-leverage production techniques: route different subtasks to models matched to their complexity and cost profile.

| Tier | Model | Use Case | Typical Cost |
|------|-------|----------|-------------|
| Fast/cheap | GPT-4o-mini, Haiku | Triage, classification, simple extraction | ~$0.15/M tokens |
| Capable | GPT-4o, Sonnet | Reasoning, drafting, code generation | ~$3/M tokens |
| Local | Llama 3.2 via Ollama | Sensitive data, high-volume tasks | $0 |

Implement tiering in LangGraph using conditional routing based on task classification:

```python
def route_by_complexity(state):
    if state["task_type"] == "classification":
        return "cheap_model_node"
    elif state["task_type"] == "reasoning":
        return "capable_model_node"
    else:
        return "local_model_node"
```

Measure cost and latency differences across tiers. Reference the [Internals page](internals.md) orchestration tax section — every agent hop adds latency and cost.

### Day 7: Explore the Frontier — Agentic Security Research

Multi-agent systems are rapidly expanding into security research, a domain with natural parallels to the plan-act-observe loop that agents excel at. Explore these three frontiers:

**Vulnerability Discovery with SWE-agent**

SWE-agent's Agent-Computer Interface ([deep-dives/swe-agent.md](deep-dives/swe-agent.md)) was designed for bug fixing, but the same architecture applies to offensive security. The repo explicitly lists cybersecurity as a supported use case. Experiment with pointing SWE-agent at a deliberately vulnerable codebase (e.g., OWASP Juice Shop or DVWA) and observe how the ACI navigates code to locate weaknesses.

SWE-smith (NeurIPS 2025) provides a pipeline for synthesizing bugs in real codebases — the inverse of this process is automated vulnerability generation.

**Multi-Agent Security Operations**

Read ["Multi-Agent LLM Orchestration Achieves Deterministic, High-Quality Decision Support for Incident Response"](https://arxiv.org/abs/2511.15755) (Drammeh, 2025). In 348 controlled trials, multi-agent orchestration achieved a 100% actionable recommendation rate vs. 1.7% for single-agent approaches — an 80x improvement in action specificity with zero quality variance.

Security Data Pipeline Platforms are evolving into agentic systems where autonomous agents act as data engineers — generating parsing rules for unseen log formats, executing Sigma detection rules within the pipeline layer, and orchestrating extraction-transformation-loading alongside threat hunting.

**Hardware and Firmware Analysis Agents — Emerging Frontier**

Multi-agent LLM frameworks are beginning to appear in hardware design space exploration. A [2025 paper](https://arxiv.org/abs/2512.08476) demonstrates specialized LLM agents for autonomous driving system DSE.

The same multi-agent patterns apply to firmware analysis: one agent for binary disassembly and function identification, another for control flow analysis, a third for vulnerability pattern matching, and an orchestrator to synthesize findings.

Cross-domain network orchestration using multi-agent workflows has been demonstrated across IP, optical, and robotic domains ([arXiv:2410.10831](https://arxiv.org/abs/2410.10831)).

**Recommended reading:**

- [Live-SWE-agent](https://arxiv.org/abs/2511.13646) — self-evolving agents that modify their own scaffold at runtime
- [LEGOMem](https://arxiv.org/abs/2510.04851) — procedural memory for multi-agent systems
- [SagaLLM](https://dl.acm.org/doi/10.14778/3750601.3750611) — transactional guarantees for multi-agent workflows

**Milestone:** Hands-on experience with 4+ frameworks, understanding of production patterns, and a clear direction for further specialization.

---

## Summary Timeline

| Week | Focus | Framework | Key Deliverable |
|------|-------|-----------|----------------|
| 1 | Foundations — raw mechanics | None (raw API) | Single agent with tools, memory, RAG |
| 2 | Multi-agent pipelines | CrewAI | 4-agent pipeline with eval suite |
| 3 | Graph-based orchestration | LangGraph | Pipeline with routing, checkpointing, HITL |
| 4 | SWE agents + advanced topics | OpenHands, AutoGen | Hands-on with 4+ frameworks |

---

## Recommended Resources

### Courses

| Course | Provider | Length | Cost | Link |
|--------|----------|--------|------|------|
| Multi AI Agent Systems with crewAI | DeepLearning.AI | 2h41m | Free | [learn.deeplearning.ai](https://learn.deeplearning.ai/courses/multi-ai-agent-systems-with-crewai) |
| AI Agents in LangGraph | DeepLearning.AI | 1h32m | Free | [learn.deeplearning.ai](https://learn.deeplearning.ai/courses/ai-agents-in-langgraph) |
| AI Agentic Design Patterns with AutoGen | DeepLearning.AI | 1h25m | Free | [learn.deeplearning.ai](https://learn.deeplearning.ai/courses/ai-agentic-design-patterns-with-autogen) |
| Building Agentic RAG with LlamaIndex | DeepLearning.AI | 44m | Free | [learn.deeplearning.ai](https://learn.deeplearning.ai/courses/building-agentic-rag-with-llamaindex) |
| AI Agents Course | HuggingFace Learn | Self-paced | Free | [huggingface.co/learn/agents-course](https://huggingface.co/learn/agents-course) |
| AI Agents for Beginners | Microsoft (GitHub) | 18 lessons | Free | [github.com/microsoft/ai-agents-for-beginners](https://github.com/microsoft/ai-agents-for-beginners) |
| LLM Agents MOOC (Fall 2024) | UC Berkeley | ~12 lectures | Free | [llmagents-learning.org](https://llmagents-learning.org/f24) |
| LLM Agents MOOC (Fall 2025) | UC Berkeley | ~12 lectures | Free | [llmagents-learning.org](https://llmagents-learning.org/f25) |
| Agentic AI Engineering Bootcamp | Maven | 6 weeks | $1,200 | [maven.com/stemplicity/become-an-agentic-ai-engineer](https://maven.com/stemplicity/become-an-agentic-ai-engineer) |

### Guides

- [Lilian Weng — "LLM Powered Autonomous Agents"](https://lilianweng.github.io/posts/2023-06-23-agent/) — canonical conceptual reference; read this first
- [Anthropic — "Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) — the practitioner's guide to when and how to use agents
- [Anthropic — "How We Built Our Multi-Agent Research System"](https://www.anthropic.com/engineering/built-multi-agent-research-system) — rare production case study from a frontier lab
- [OpenAI — "A Practical Guide to Building Agents"](https://platform.openai.com/docs/guides/agents) — covers the Agents SDK and orchestration patterns

### Community

- [r/AI_Agents](https://reddit.com/r/AI_Agents) — active community, good for finding new frameworks and production war stories
- [r/LLMDevs](https://reddit.com/r/LLMDevs) — broader LLM engineering discussions
- [LangChain Discord](https://discord.gg/langchain) — LangGraph support and announcements
- [CrewAI Discord](https://discord.gg/crewai) — CrewAI community and framework updates

### Papers

See [meta.md](meta.md) for the full list of 20+ academic papers referenced in this research, with DOIs and links to code repositories.
