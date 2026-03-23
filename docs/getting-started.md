# Getting Started

A practical onboarding guide for engineers new to multi-agent systems. This covers the key concepts, recommended entry points, and a minimal viable local-first stack.

## Key Concepts

### What Is a Multi-Agent System?

A multi-agent system (MAS) is an architecture where multiple LLM-powered agents — each with a defined role, instructions, and tools — collaborate to solve tasks that a single agent handles poorly. The agents communicate through messages, shared state, or structured handoffs rather than a single monolithic prompt.

### Agent Memory

Agents need memory to maintain context across interactions:

- **Short-term memory**: The current conversation or task context held in the LLM's context window.
- **Long-term memory**: Persistent storage (vector databases, structured stores) for facts, preferences, and past trajectories.
- **Procedural memory**: Reusable task execution patterns. [LEGOMem](https://arxiv.org/abs/2510.04851) decomposes past trajectories into modular, reusable memory units that can be allocated across orchestrators and task agents.
- **Episodic memory**: Records of specific past interactions, useful for learning from successes and failures.

### Tool Use

Agents extend LLM capabilities by calling external tools — APIs, code interpreters, file systems, web browsers, databases. The [Model Context Protocol (MCP)](https://docs.openhands.dev/sdk/guides/mcp) is emerging as a standard for how agents interact with external tools and data sources.

### Orchestration Patterns

| Pattern | Description | Best For |
|---------|-------------|----------|
| **Sequential** | Agents execute one after another in a pipeline | Simple, linear workflows |
| **Parallel (Scatter-Gather)** | Tasks distributed to multiple agents, results consolidated | Research, data collection |
| **Hierarchical** | Manager agent delegates to specialist sub-agents | Complex projects with clear decomposition |
| **Conversational** | Agents debate/discuss to reach consensus | Creative tasks, decision-making |
| **Graph-Based** | Agents as nodes in a directed graph with conditional edges | Complex branching logic |

### Orchestration Models by Framework

Each major framework embodies a different orchestration philosophy:

- **LangGraph**: Graph state machine — you define nodes (agents/functions) and edges (transitions) explicitly. Most flexible but requires graph design knowledge.
- **AutoGen**: Conversational collaboration — agents chat with each other, optionally including human participants.
- **CrewAI**: Role-based teams — agents have defined roles (Manager, Worker, Researcher) with hierarchical delegation.

## Recommended Entry Points

### If You Want the Fastest Prototype

Start with **CrewAI**. Its role-based abstraction is the most intuitive — define agents with roles and goals, define tasks, and let the framework handle orchestration.

```python
# pip install crewai
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Senior Researcher",
    goal="Find the latest trends in multi-agent AI",
    backstory="Expert at synthesizing research papers",
)

writer = Agent(
    role="Technical Writer",
    goal="Create clear documentation from research",
    backstory="Skilled at making complex topics accessible",
)

research_task = Task(
    description="Research the top 5 multi-agent frameworks in 2025",
    expected_output="A structured summary with pros/cons",
    agent=researcher,
)

writing_task = Task(
    description="Write a getting-started guide based on the research",
    expected_output="A markdown document with code examples",
    agent=writer,
)

crew = Crew(agents=[researcher, writer], tasks=[research_task, writing_task])
result = crew.kickoff()
```

### If You Need Production-Grade Workflows

Start with **LangGraph**. Its graph-based architecture gives you explicit control over state, branching, and recovery.

```python
# pip install langgraph
from langgraph.graph import StateGraph, MessagesState

def researcher(state: MessagesState):
    # Agent logic here
    return {"messages": [("assistant", "Research findings...")]}

def writer(state: MessagesState):
    # Agent logic here
    return {"messages": [("assistant", "Document draft...")]}

graph = StateGraph(MessagesState)
graph.add_node("researcher", researcher)
graph.add_node("writer", writer)
graph.add_edge("researcher", "writer")
graph.set_entry_point("researcher")

app = graph.compile()
result = app.invoke({"messages": [("user", "Research multi-agent systems")]})
```

### If You Want AI Pair Programming Now

Install **Aider** and point it at your codebase:

```bash
pip install aider-install && aider-install
cd /your/project
aider --model sonnet --api-key anthropic=YOUR_KEY
```

Aider maps your entire codebase, auto-commits changes with sensible messages, and supports 100+ languages.

## Minimal Viable Local-First Stack

For an engineer wanting to run multi-agent pipelines locally with minimal cloud dependency:

| Component | Recommended Tool | Purpose |
|-----------|-----------------|---------|
| **Framework** | LangGraph or CrewAI | Agent orchestration |
| **LLM** | Ollama + Llama 3 / Qwen 2.5 | Local model inference |
| **Vector DB** | ChromaDB | Local embedding storage |
| **Code Execution** | Docker sandbox | Isolated agent code execution |
| **Observability** | LangSmith (free tier) or OpenTelemetry | Tracing agent interactions |

### Setup

```bash
# Install core dependencies
pip install langgraph crewai chromadb

# Install Ollama for local LLM inference
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2

# For sandboxed code execution
docker pull python:3.12-slim
```

## What to Build First

1. **Single-agent tool user**: One agent that can search the web, read files, and write summaries. Understand tool calling.
2. **Two-agent pipeline**: A researcher agent that gathers information and a writer agent that produces output. Understand handoffs.
3. **Critique loop**: Add a reviewer agent that evaluates the writer's output and sends feedback. Understand iterative refinement.
4. **Parallel research**: Multiple researcher agents working on different subtopics simultaneously. Understand scatter-gather.

Once these patterns feel natural, move to the [Deep Dives](deep-dives/langgraph.md) for framework-specific architecture details, or follow the [Learning Path](learning-path.md) for a structured 4-week curriculum.
