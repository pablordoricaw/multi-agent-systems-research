# Internals: What Frameworks Are Actually Doing

Every major multi-agent framework — LangChain's `AgentExecutor`, CrewAI's `Crew.kickoff()`, LangGraph's graph traversal, and the OpenAI Agents SDK's `Runner.run()` — wraps the same fundamental mechanics. This page strips the abstractions away to show you exactly what runs on the wire, how state moves between turns, and what the real costs of orchestration are.

---

## 1. The Agent Loop

### The Core While-Loop

[Every agent framework is a thin wrapper](https://dev.to/klement_gunndu/build-an-ai-agent-loop-in-50-lines-of-python-59jk) around the same structure:

```
LLM call → check for tool calls → execute tools → inject results → repeat
```

The loop terminates when: (a) the model produces a response with no tool calls, (b) a designated "finish" sentinel is triggered, or (c) a maximum iteration count is exceeded.

Here is the canonical minimal implementation using the raw OpenAI API — no framework, no magic:

```python title="agent_loop.py"
import json
from openai import OpenAI

client = OpenAI()  # uses OPENAI_API_KEY env var

def agent_loop(user_message: str, max_iterations: int = 10) -> str:
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user",   "content": user_message},
    ]
    for _ in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o-mini", messages=messages, tools=tools
        )
        msg = response.choices[0].message
        messages.append(msg)                        # (1) record assistant turn

        if not msg.tool_calls:                      # (2) no tools → done
            return msg.content

        for tc in msg.tool_calls:                   # (3) execute every call
            fn   = available_functions[tc.function.name]
            args = json.loads(tc.function.arguments)
            result = fn(**args)
            messages.append({                       # (4) inject result
                "role": "tool",
                "tool_call_id": tc.id,
                "content": result,
            })
    return "Max iterations reached."
```

Source: [DEV Community — Build an AI Agent Loop in 50 Lines of Python](https://dev.to/klement_gunndu/build-an-ai-agent-loop-in-50-lines-of-python-59jk), [Victor Dibia — The Agent Execution Loop](https://victordibia.com/blog/agent-execution-loop/)

**What each step does:**

1. **LLM call** — send the full `messages` list to the API and receive a response.
2. **Tool detection** — inspect `response.choices[0].message.tool_calls`; if empty, the model has finished reasoning and produced a final answer.
3. **Tool execution** — dispatch each tool call to the corresponding Python function, parsing the JSON-encoded argument string.
4. **Result injection** — append a `role: "tool"` message back into `messages` so the model can see what the tool returned.
5. **Repeat** — the loop starts over from step 1 with the extended `messages` list.

!!! note "The Key Insight"
    The entire conversation — system prompt, user query, all assistant messages with tool calls, all tool result messages — **is replayed from scratch on every single API call.** There is no server-side memory by default. Each call to `client.chat.completions.create()` is stateless; the framework feeds back the accumulating `messages` list every time. Source: [Peter Roelants — ReAct + OpenAI Function Calling](https://peterroelants.github.io/posts/react-openai-function-calling/)

### What Frameworks Hide

The [OpenAI Agents SDK `Runner.run()`](https://openai.github.io/openai-agents-python/ref/run/) adds session memory management (`SQLiteSession`), guardrail tripwires, max-turn enforcement, and lifecycle hooks — but the core loop is identical. Here is the full table of what each framework wraps:

| Framework | What Wraps the Loop | Termination Signal |
|---|---|---|
| LangChain | `AgentExecutor` | No tool calls in response |
| CrewAI | `Crew.kickoff()` | Task `expected_output` produced |
| OpenAI Agents SDK | `Runner.run()` | `output_type` matched or no tool calls |
| LangGraph | Graph traversal + node execution | `END` node reached |

Sources: [DEV Community](https://dev.to/klement_gunndu/build-an-ai-agent-loop-in-50-lines-of-python-59jk), [Victor Dibia](https://victordibia.com/blog/agent-execution-loop/), [Peter Roelants](https://peterroelants.github.io/posts/react-openai-function-calling/)

---

## 2. Tool Calling Mechanics

### What Actually Gets Sent Over the Wire

#### Request: the `tools` array entry

Every API call that involves tools includes a `tools` array. Each entry looks like this ([OpenAI Function Calling Documentation](https://platform.openai.com/docs/guides/function-calling)):

```json title="Tool definition in request"
{
  "type": "function",
  "name": "get_weather",
  "description": "Retrieves current weather for the given location.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City and country e.g. Bogotá, Colombia"
      },
      "units": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Units the temperature will be returned in."
      }
    },
    "required": ["location", "units"],
    "additionalProperties": false
  },
  "strict": true
}
```

The `strict: true` flag (Structured Outputs mode) enforces that the model's output exactly matches the schema with no extra keys.

#### Response: the `function_call` object in `output`

When the model decides to call a tool, the response's `output` array contains ([OpenAI Function Calling Documentation](https://platform.openai.com/docs/guides/function-calling)):

```json title="Function call in response output"
[
  {
    "id": "fc_12345xyz",
    "call_id": "call_12345xyz",
    "type": "function_call",
    "name": "get_weather",
    "arguments": "{\"location\":\"Paris, France\",\"units\":\"celsius\"}"
  }
]
```

!!! warning "Common Bug"
    `arguments` is a **JSON-encoded string**, not a nested object. You must call `json.loads(tool_call.function.arguments)` before you can access the values. Forgetting this is one of the most common errors when working with the raw API.

#### Result injection: `function_call_output`

After executing the tool, the result is injected back as ([OpenAI Function Calling Documentation](https://platform.openai.com/docs/guides/function-calling)):

```json title="Tool result injection payload"
{
  "type": "function_call_output",
  "call_id": "call_12345xyz",
  "output": "{\"temperature\": 18, \"unit\": \"celsius\"}"
}
```

The `call_id` must match the `call_id` from the function call response, creating a paired request/response that the model uses to track which results correspond to which calls. For the older Chat Completions API (`/v1/chat/completions`), the equivalent is `{"role": "tool", "tool_call_id": "call_123", "content": "..."}`.

### Python Function → JSON Schema Conversion

Frameworks use three main approaches to auto-generate JSON schemas from Python code, saving you from writing the tool definition JSON by hand.

**Type hints + docstrings** is the lightest approach. Libraries like [`annotated-docs`](https://peterroelants.github.io/posts/react-openai-function-calling/) use `typing.Annotated` for field descriptions and `typing.Literal` for enums. An `as_json_schema(func)` utility introspects the type annotations and the function's docstring to produce the schema. This works well for simple tools but requires discipline in keeping docstrings accurate.

**Pydantic `model_json_schema()`** is the most common production approach. Define your tool's parameters as a Pydantic `BaseModel` and call `.model_json_schema()` to get the full schema automatically ([Pydantic JSON Schema Documentation](https://docs.pydantic.dev/latest/concepts/json_schema/)):

```python title="Pydantic model → JSON schema"
from pydantic import BaseModel
from typing import Literal

class GetCurrentWeather(BaseModel):
    """Get the current weather in a given location."""
    location: str
    unit: Literal["celsius", "fahrenheit"] | None = None

# Produces the full JSON schema automatically
schema = GetCurrentWeather.model_json_schema()

# OpenAI convenience wrapper:
tool_def = openai.pydantic_function_tool(GetCurrentWeather)
```

Pydantic handles nested objects, optional fields, enums, and field-level `description` annotations via `Field(description="...")`. The [OpenAI Community](https://community.openai.com/t/tool-calls-does-the-schema-matter/859354) recommends this approach for any tool with more than a couple of parameters.

**LangChain `@tool` decorator** uses the function's docstring as the `description` and inspects type hints to build the `parameters` schema automatically. The schema generation is compatible with OpenAI's expected format and the decorator also handles result serialization.

### The `tool_choice` Parameter

Controls model behavior when tools are available ([OpenAI Function Calling Documentation](https://platform.openai.com/docs/guides/function-calling)):

```json title="tool_choice options"
"tool_choice": "auto"      // model decides whether to call a tool
"tool_choice": "required"  // must call at least one tool
"tool_choice": {"type": "function", "name": "get_weather"}  // force a specific tool
```

`"required"` is useful for structured extraction workflows where you always want the model to populate a schema. Forcing a specific tool is useful for single-purpose agents.

---

## 3. State and Memory Serialization

The three dominant patterns reflect fundamentally different answers to: *"What does an agent need to remember between turns?"*

### (a) Full Message History Replay — OpenAI API / Agents SDK

**Mechanism:** Every API call replays the entire conversation history from scratch. The `messages` list *is* the state. There is no server-side memory ([OpenAI Agents SDK Sessions Documentation](https://openai.github.io/openai-agents-python/sessions/)).

```python title="Message history as state"
messages = [
    {"role": "system",    "content": "..."},
    {"role": "user",      "content": "..."},                       # Turn 1
    {"role": "assistant", "content": "...", "tool_calls": [...]},  # Turn 1 response
    {"role": "tool",      "tool_call_id": "...", "content": "..."}, # Tool result
    {"role": "assistant", "content": "..."},                       # Turn 2 response
    {"role": "user",      "content": "..."},                       # Turn 2 follow-up
]
response = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=tools
)
```

The OpenAI Agents SDK builds on this with `SQLiteSession` and `InMemorySession` objects (both implementing `SessionABC`). When `session=session` is passed to `Runner.run()`, the SDK loads existing history, appends new turn items, and handles context trimming automatically via `TrimmingSession` ([OpenAI Cookbook — Context Engineering with Session Memory](https://developers.openai.com/cookbook/examples/agents_sdk/session_memory/)).

**Tradeoffs:**

| Factor | Assessment |
|---|---|
| Token cost | Grows linearly with conversation length — each turn pays for all prior tokens |
| Consistency | Perfect — model sees the entire history, no information loss |
| Implementation complexity | Low — just manage a Python list |
| Context window pressure | Hits limit after ~50–200 turns depending on tool output sizes |
| Cross-session persistence | Requires external storage (SQLite, Redis) |

### (b) Typed State Object — LangGraph `StateGraph`

**Mechanism:** State is a `TypedDict` (or Pydantic `BaseModel`) that flows through a directed graph. Each node receives the current state as input and returns a partial state update. Reducer functions (e.g., `add_messages`) merge updates with the existing state ([LangGraph Persistence Documentation](https://docs.langchain.com/oss/python/langgraph/persistence)):

```python title="LangGraph typed state with reducer"
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]  # reducer appends, never replaces
    data_to_save: dict                        # arbitrary non-message state

builder = StateGraph(State)
# Compile with a checkpointer for persistence
graph = builder.compile(checkpointer=AsyncPostgresSaver(conn))
config = {"configurable": {"thread_id": "user_123_session_456"}}
result = await graph.ainvoke(input_state, config=config)
```

LangGraph serializes state at every node transition using `JsonPlusSerializer` (which uses `ormsgpack`). Checkpoint backends include `InMemorySaver`, `SqliteSaver`, and `AsyncPostgresSaver`. Every checkpoint stores the full state object, the parent checkpoint ID (forming a tree that enables branching and time-travel), and the thread ID for conversation isolation.

!!! warning "Serialization Constraint"
    **All values in the `State` TypedDict must be JSON-serializable** (or handled by custom serializer extensions). Pydantic objects, LangChain `AIMessage` objects, and custom classes require special handling. This is a common source of cryptic errors in production — see [LangGraph GitHub Issue #3441](https://github.com/langchain-ai/langgraph/issues/3441).

**Tradeoffs:**

| Factor | Assessment |
|---|---|
| Token cost | Controllable — non-message data can be stored out-of-band |
| Consistency | Strong — state transitions are atomic via checkpointer |
| Implementation complexity | Higher — requires graph design, reducer functions, serializer configuration |
| Cross-session persistence | Built-in via checkpointer backends |
| Branching / replay | Supported via checkpoint tree ("time travel" debugging) |
| Failure recovery | Resume from last checkpoint on crash |

### (c) Role-Scoped Memory — CrewAI Unified Memory

**Mechanism:** CrewAI uses a unified `Memory` class backed by a LanceDB vector database (stored under `./.crewai/memory` or `$CREWAI_STORAGE_DIR/memory`). Memory is LLM-driven on both save and recall ([CrewAI Memory Documentation](https://docs.crewai.com/en/concepts/memory)):

- **On save:** The LLM analyzes content to infer scope (a hierarchical path like `/agent/researcher`), categories, and an importance score.
- **On recall:** A composite scoring formula ranks candidates:

```
composite = semantic_weight × similarity
          + recency_weight  × decay
          + importance_weight × importance

where:
  similarity = 1 / (1 + vector_distance)      # 0–1
  decay      = 0.5^(age_days / half_life_days)  # exponential decay
  importance = record's importance score at encoding (0–1)
```

Before each task, the agent recalls relevant context from memory and injects it into the task prompt. After each task, the crew auto-extracts discrete facts from the task output via `extract_memories()` and stores them.

Two recall depths are available:

| Depth | Latency | LLM Calls | Description |
|---|---|---|---|
| `shallow` | ~200ms | None | Direct vector search + composite scoring |
| `deep` (default) | 1–3s+ | Yes | Multi-step: query analysis, scope selection, parallel search, recursive exploration |

**Tradeoffs:**

| Factor | Assessment |
|---|---|
| Token cost | Selective retrieval — only relevant memories injected; much lower per-turn cost vs. full replay |
| Consistency | Risk of recall failures — relevant context may not score high enough to surface |
| Implementation complexity | High — LLM used for both save and recall; LanceDB infrastructure required |
| Cross-session persistence | Native — LanceDB persists across runs |
| Scope isolation | Per-agent private scopes possible via `/agent/<name>` hierarchy |

### Comparison Table

| Framework | Storage | Replay Strategy | Token Cost Growth | Consistency Risk |
|---|---|---|---|---|
| OpenAI API (raw) | In-memory list | Full replay every call | O(n) — linear | Low (full context) |
| OpenAI Agents SDK | SQLite / Memory session | Trimming + summarization | Bounded with trimming | Medium (trimmed context) |
| LangGraph | Postgres / SQLite checkpoints | State delta per node | Controlled (non-message state offloaded) | Low (atomic checkpoints) |
| CrewAI | LanceDB vector store | Semantic recall per task | O(1) per task (fixed recall limit) | Medium (recall may miss context) |

Sources: [OpenAI Agents SDK Sessions](https://openai.github.io/openai-agents-python/sessions/), [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence), [CrewAI Memory](https://docs.crewai.com/en/concepts/memory)

---

## 4. Handoffs Under the Hood

### OpenAI Agents SDK

A handoff in the Agents SDK is **a special tool call**. The tool name defaults to `transfer_to_<agent_name>`, generated by `Handoff.default_tool_name()`. When the model calls this tool, the `Runner` transfers control to the specified agent ([OpenAI Agents SDK Handoffs Documentation](https://openai.github.io/openai-agents-python/handoffs/)).

**What crosses the boundary — `HandoffInputData`:**

| Field | Content |
|---|---|
| `input_history` | Full input history before `Runner.run()` started |
| `pre_handoff_items` | Items generated before the agent turn where handoff was invoked |
| `new_items` | Items from the current turn, including the handoff call and its output item |
| `input_items` | Optional override: items forwarded to next agent (for filtering) |
| `run_context` | Active `RunContextWrapper` at handoff time |

By default, the receiving agent sees the **entire conversation history** including all prior tool calls, tool results, and assistant messages, unless an `input_filter` is provided.

```python title="Typed handoff with on_handoff callback"
from pydantic import BaseModel
from agents import Agent, handoff, RunContextWrapper

class EscalationData(BaseModel):
    reason: str

async def on_handoff(ctx: RunContextWrapper[None], input_data: EscalationData):
    print(f"Escalation reason: {input_data.reason}")

handoff_obj = handoff(
    agent=escalation_agent,
    on_handoff=on_handoff,
    input_type=EscalationData,
)
```

The model generates `{"reason": "duplicate_charge"}` as the tool call arguments. The SDK validates this JSON, parses it into an `EscalationData` instance, and passes it to `on_handoff`. The `input_type` parameters are metadata attached to the handoff decision — they do not replace the next agent's main input.

!!! tip "Reducing Token Costs in Long Chains"
    Setting `RunConfig.nest_handoff_history=True` (opt-in beta) collapses the prior transcript into a single assistant summary message wrapped in a `<CONVERSATION HISTORY>` block, rather than passing the full verbatim history. This meaningfully reduces token costs in long multi-agent chains. Source: [OpenAI Agents SDK Handoffs](https://openai.github.io/openai-agents-python/handoffs/)

### LangGraph

LangGraph has two handoff mechanisms ([LangGraph Handoffs How-To](https://langchain-ai.github.io/langgraph/how-tos/agent-handoffs/)):

**Conditional edges** are used when routing logic depends only on the current graph state:

```python title="Static routing via conditional edges"
builder.add_conditional_edges(
    "supervisor",
    should_continue,   # routing function → returns node name string
    path_map={
        "multiplication_expert": "multiplication_expert",
        "end": END
    }
)
```

**`Command` objects** combine routing and state mutation atomically — useful when the routing decision itself produces state changes:

```python title="Dynamic routing via Command"
from langgraph.types import Command
from typing import Literal

def addition_expert(state: MessagesState) -> Command[Literal["multiplication_expert", "__end__"]]:
    ai_msg = model.bind_tools([transfer_to_multiplication_expert]).invoke(state["messages"])
    if ai_msg.tool_calls:
        tool_call_id = ai_msg.tool_calls[-1]["id"]
        tool_msg = {
            "role": "tool",
            "content": "Successfully transferred",
            "tool_call_id": tool_call_id,
        }
        return Command(
            goto="multiplication_expert",      # routing
            update={"messages": [ai_msg, tool_msg]}  # state mutation
        )
    return {"messages": [ai_msg]}
```

**What crosses the handoff boundary:** The shared `State` TypedDict. The receiving node receives the complete, mutated state as input — all messages accumulated up to the handoff point, merged by reducer functions. Use `graph=Command.PARENT` when a handoff originates inside a subgraph and must navigate to a sibling node in the parent graph.

### CrewAI

CrewAI uses a different paradigm: agents don't hand off dynamically. The Crew's `Process` (sequential or hierarchical) defines the task execution order at configuration time. Task outputs are passed via `TaskOutput` objects ([CrewAI Tasks Documentation](https://docs.crewai.com/en/concepts/tasks)):

```python title="TaskOutput chaining in sequential process"
print(f"Task:   {task1.output.description}")
print(f"Output: {task1.output.raw}")
```

In **sequential** mode, each task's output is injected into the next task's context. In **hierarchical** mode, a manager agent delegates tasks to workers and aggregates results. The raw output text becomes part of the next agent's context window for its task.

### Comparison Table

| Framework | Handoff Trigger | Payload to Receiver | History Handling |
|---|---|---|---|
| OpenAI Agents SDK | `transfer_to_<agent>` tool call | Full `HandoffInputData` (history + new items) | Full replay by default; summarized with `nest_handoff_history=True` |
| LangGraph | `Command(goto=...)` return from node | Full shared `State` TypedDict | All messages merged via reducers |
| CrewAI | Task completion in sequential / hierarchical Process | `TaskOutput.raw` string | Context window injection per task |

Sources: [OpenAI Agents SDK Handoffs](https://openai.github.io/openai-agents-python/handoffs/), [LangGraph Handoffs](https://langchain-ai.github.io/langgraph/how-tos/agent-handoffs/), [CrewAI Tasks](https://docs.crewai.com/en/concepts/tasks)

---

## 5. Why Different Philosophies Exist

These frameworks are not interchangeable with different UX skins. They encode fundamentally different engineering tradeoffs.

### Graph-Based: LangGraph

**Core tradeoff:** Setup complexity in exchange for runtime predictability. Every agent action is a node; every routing decision is an explicit edge. State is a typed object that flows through the graph — nothing implicit, nothing emergent.

**Why it exists:** Complex production workflows require determinism, auditability, and failure recovery. A graph structure makes control flow visible, replayable, and debuggable. LangGraph adds native LangSmith integration, `interrupt()` for human-in-the-loop approvals at any node, checkpoint-based resume on crash, and visual graph rendering. The `Command` object allows combining routing and state mutation atomically — no equivalent exists in conversation-based frameworks ([Langfuse — Comparing Open-Source AI Agent Frameworks](https://langfuse.com/blog/2025-03-19-ai-agent-comparison)).

**Best for:** Compliance-regulated workflows, long-running processes (days/weeks) that must survive crashes, document processing pipelines, multi-step research synthesis, any workflow requiring human approval at specific steps.

**Not ideal for:** Simple linear tasks (the graph setup overhead is pure waste), teams unfamiliar with directed graph concepts.

### Conversational: AutoGen

**Core tradeoff:** Maximum flexibility and rapid prototyping speed, in exchange for less predictable control flow. Agents are participants in a structured conversation; flow emerges from agent interactions rather than explicit edges.

**Why it exists:** Iterative refinement maps naturally to a dialogue model — one agent writes code, another executes and critiques it, the first revises. [AutoGen originated from Microsoft Research in 2023](https://www.leanware.co/insights/auto-gen-vs-langgraph-comparison) specifically for code generation benchmarks. AutoGen v0.4 (2025) rebuilt the runtime on an actor model, enabling horizontal scaling across machines ([Galileo — AutoGen vs CrewAI vs LangGraph vs OpenAI](https://galileo.ai/blog/autogen-vs-crewai-vs-langgraph-vs-openai-agents-framework)).

**Best for:** Code generation and execution loops, mathematical reasoning with iterative critique, rapid prototyping, customer-facing conversational applications.

**Not ideal for:** Structured workflows requiring deterministic branching, production systems requiring auditability of control flow.

### Role-Based: CrewAI

**Core tradeoff:** Intuitive cognitive model and fast setup, in exchange for less fine-grained control. Agents are employees with defined roles, goals, and backstories; tasks are assigned based on role. The mental model maps naturally to team structures ([DataCamp — CrewAI vs LangGraph vs AutoGen](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)).

**Why it exists:** Many real-world workflows are team-shaped: a researcher gathers information, a writer drafts, an editor reviews. This intuitive model lowers the conceptual barrier for non-engineering stakeholders and produces prototypes ~40% faster than LangGraph by some benchmarks. The LanceDB-backed semantic memory system allows agents to accumulate knowledge across runs — something no other framework provides natively ([Let's Data Science — AI Agent Frameworks 2026](https://letsdatascience.com/blog/ai-agent-frameworks-compared)).

**Best for:** Content creation pipelines, customer support triage with specialist routing, structured team-like workflows with clear role boundaries.

**Not ideal for:** Complex conditional branching that doesn't fit a crew metaphor, workflows requiring deterministic replay or fine-grained execution control.

### Lightweight Handoff: OpenAI Agents SDK

**Core tradeoff:** Minimal opinions and minimal setup, in exchange for building orchestration patterns yourself. The SDK provides `Agent`, `handoff()`, `Runner`, `Session`, and guardrails — but does not impose graph structure or conversation history management patterns. It is deliberately a set of building blocks ([Langfuse — Comparing Open-Source AI Agent Frameworks](https://langfuse.com/blog/2025-03-19-ai-agent-comparison)).

**Best for:** Simple routing workflows, OpenAI-native apps leveraging built-in tools (web search, code interpreter, file search), teams that want to stay close to the raw API.

### Framework Selection Matrix

| Requirement | Best Framework | Reason |
|---|---|---|
| Complex branching workflows | LangGraph | Explicit edges, conditional routing, `Command` for atomic routing+mutation |
| Compliance / auditability | LangGraph | State transition logs, checkpoint replay, visual graph |
| Long-running workflows (days/weeks) | LangGraph | Checkpoint-based persistence and crash resume |
| Code generation / iterative refinement | AutoGen | Conversation-based write/execute/critique loops |
| Rapid prototyping | CrewAI or AutoGen | Low setup overhead |
| Content pipelines (researcher → writer → editor) | CrewAI | Role metaphor fits naturally; semantic memory |
| Cross-run persistent memory | CrewAI | Native LanceDB persistence |
| Customer support triage | LangGraph or OpenAI Agents SDK | Predictable routing, reliable handoffs |
| OpenAI-native integration | OpenAI Agents SDK | Official support, built-in tools |

Sources: [DataCamp](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen), [DEV Community 2026](https://dev.to/pockit_tools/langgraph-vs-crewai-vs-autogen-the-complete-multi-agent-ai-orchestration-guide-for-2026-2d63), [Galileo AI](https://galileo.ai/blog/autogen-vs-crewai-vs-langgraph-vs-openai-agents-framework)

---

## 6. The Orchestration Tax

The "orchestration tax" is the cumulative penalty paid in latency, cost, and quality degradation when adding more agents to a pipeline. It has four components.

### Latency Accumulation

Sequential agent chains accumulate latency multiplicatively. A single LLM call completes in roughly **800 milliseconds** and achieves 60–70% accuracy on complex tasks. An orchestrator-worker flow with reflection loops can reach 95%+ accuracy, but extends latency to **10–30 seconds** — a 12–37× increase ([Parloa — Why Bad Agentic AI Latency Costs You Customers and Revenue](https://www.parloa.com/knowledge-hub/agentic-ai-latency-cost/)).

Each agent invocation in a chain adds: one LLM call (800ms–3s), tool execution time (variable), and context assembly overhead. For user-facing applications, this is often the deciding factor against multi-agent architectures.

### Error Propagation

The most important empirical finding on orchestration tax comes from [arXiv:2603.04474 — Modeling and Mitigating Error Cascades in LLM-Based Multi-Agent Systems](https://arxiv.org/html/2603.04474v1) (March 2026). The paper seeded a single error into a 4-agent network and measured propagation over 5 rounds:

!!! warning "100% Final Infection Rate"
    In **5 out of 6** tested frameworks, a single seeded error reaches **100% final infection rate** across the entire agent network. Minor inaccuracies "solidify into system-level false consensus through iteration and context reuse."

| Framework | Topology | Agents | Final Error Rate |
|---|---|---|---|
| MetaGPT | Chain | 4 | 100.0% |
| LangGraph | Star | 4 | 100.0% |
| CrewAI | Star | 4 | 100.0% |
| AutoGen | Mesh | 3 | 100.0% |
| Camel | Mesh | 3 | 100.0% |
| LangChain | Chain | 4 | 89.2% |

**Hub impact factor:** In LangGraph's star topology, hub infection = 100% while leaf infection = 9.7%, producing an **impact factor of 10.31×** — a hub node is 10× more likely to cause system-wide error than a leaf node. CrewAI's hub impact factor is 6.29× ([arXiv:2603.04474](https://arxiv.org/html/2603.04474v1)).

**Consensus inertia:** An error introduced at t=6 requires **3.9× more rounds to correct** than an error introduced at t=2, because prior agents have already built consensus around the false information.

**Defense:** A genealogy-based governance layer raises the defense success rate (BICR) from 0.32 (baseline reflection) to **0.89–0.94**, at a cost of ~49% more tokens and ~2.1× longer latency in strict mode.

### The 45% Rule: Diminishing Returns

[Research reported in Towards Data Science](https://towardsdatascience.com/why-your-multi-agent-system-is-failing-escaping-the-17x-error-trap-of-the-bag-of-agents/) establishes the saturation threshold:

- Multi-agent coordination provides its highest returns when **baseline single-agent performance is below 45%**
- If the base model already achieves **80%**, adding agents may generate more noise than value
- Performance increases initially but plateaus — **often around 4 agents** — after which additional agents contribute marginally

Poorly coordinated "bag of agents" networks can experience up to **17.2× error amplification**. Centralized coordination limits amplification to ~**4.4×**.

Benchmark results vary dramatically by task type:

| Benchmark | Task Type | Key Finding |
|---|---|---|
| Finance-Agent (2025) | Structured analyst reasoning | Centralized MAS outperforms single agent by **+80.8%** |
| BrowseComp-Plus (2025) | Web retrieval, multi-site | Decentralized +9.2%; Centralized +0.2%; Independent −35% |
| PlanCraft (2024) | Long-horizon Minecraft planning | All MAS variants: **−39% to −70%** |
| WorkBench (2024) | Business tool selection | Minimal movement: −11% to +6% |

The key variable is **task decomposability**: MAS works when tasks can be genuinely parallelized. Sequential tasks see no benefit, only coordination overhead.

### Cost Compounding

Total MAS cost = **Work cost + Coordination cost** ([Towards Data Science — Why Your Multi-Agent System is Failing](https://towardsdatascience.com/why-your-multi-agent-system-is-failing-escaping-the-17x-error-trap-of-the-bag-of-agents/)):

```
Work cost        = n × k × (input_tokens + output_tokens per step)
Coordination cost = (r × n + d × n² + p × n² × m) × token_cost
```

Where `n` = agents, `k` = max iterations per agent, `r` = orchestrator rounds, `d` = debate rounds, `p` = peer-communication rounds, `m` = average peer requests per round.

The **n² term** in decentralized topologies means coordination costs grow quadratically with agent count. A 4-agent decentralized debate can easily cost 8–16× more per query than a single agent.

In practice, multi-agent at **~$0.08/request** vs. single-agent at **~$0.03/request** — a **2.7× cost multiplier** — has been documented even for relatively simple orchestration setups ([YouTube — Stop Building Multi-Agent Systems](https://www.youtube.com/watch?v=4e5SGwiF0Xs)).

### Context Window Pressure

In full-history-replay architectures, every agent in a chain sees not just its own task context but the accumulated history from all prior agents. A 4-agent sequential chain where each agent produces 2,000 tokens of reasoning and tool results will have the final agent processing ~8,000 tokens of prior context before it starts its own task.

For `gpt-4o` at $5/1M input tokens, this is a material cost. It is also a real accuracy risk: LLMs degrade in long-context retrieval — the **"lost in the middle" problem**, where information in the middle of a long context window is retrieved less reliably than information at the beginning or end.

!!! tip "When Multi-Agent Is Worth It"
    Use multi-agent architectures when: (1) tasks are genuinely decomposable into parallel subtasks, (2) baseline single-agent accuracy is below ~45%, and (3) you can afford the 2–3× cost and 10–30× latency premium. For linear tasks, a single well-prompted agent with good tools almost always wins on cost, speed, and reliability.
