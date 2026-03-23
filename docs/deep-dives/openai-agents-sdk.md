# OpenAI Agents SDK

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [openai/openai-agents-python](https://github.com/openai/openai-agents-python) |
| **Stars** | 20,220 |
| **Language** | Python, TypeScript |
| **License** | MIT |
| **Last Push** | 2026-03-23 |
| **Maturity** | Production-ready |
| **Use Case Fit** | Multi-agent workflows, agent handoffs, production agent systems |

The OpenAI Agents SDK is a lightweight, open-source framework for building multi-agent workflows. [Launched in March 2025](https://mem0.ai/blog/openai-agents-sdk-review) as the production-ready successor to the experimental [Swarm](#from-swarm-to-agents-sdk) project, it provides four core primitives — Agents, Handoffs, Guardrails, and Tools — that handle the key building blocks of agent systems without the overhead of heavier orchestration frameworks. While optimized for OpenAI models, it works with [100+ LLMs](https://mem0.ai/blog/openai-agents-sdk-review) via the Chat Completions API.

## Architecture

The SDK's architecture is intentionally minimal — a small set of composable primitives rather than a sprawling framework:

```
┌─────────────────────────────────────────────────┐
│                    Runner                        │
│         (Manages the agent execution loop)       │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────┐  handoff()  ┌──────────────────┐  │
│  │ Triage   │────────────>│ Specialist       │  │
│  │ Agent    │             │ Agent            │  │
│  │          │  handoff()  │                  │  │
│  │          │<────────────│                  │  │
│  └──────────┘             └──────────────────┘  │
│       │                          │               │
│       ▼                          ▼               │
│  ┌──────────┐             ┌──────────────────┐  │
│  │ Tools    │             │ Guardrails       │  │
│  │ (func,   │             │ (input, output,  │  │
│  │  MCP,    │             │  tool)           │  │
│  │  hosted) │             │                  │  │
│  └──────────┘             └──────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │              Tracing                      │   │
│  │  (Automatic spans, custom spans, export)  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Core Primitives

| Primitive | Description |
|-----------|-------------|
| **Agents** | Instruction-driven entities with access to tools and models. Defined by a system prompt, a model, and a set of tools. |
| **Handoffs** | Native mechanism for delegating tasks between agents. One agent transfers control (and conversation context) to another specialized agent. Stays within a single run. |
| **Guardrails** | Input, output, and tool-level validation. Can run in parallel with agent execution (for latency) or blocking (for cost/safety). |
| **Tools** | Python/TypeScript functions auto-converted to tool schemas. Also supports hosted tools (WebSearch, FileSearch, CodeInterpreter, ImageGeneration) and MCP servers. |

### The Agent Loop

The [Runner](https://0deepresearch.com/posts/2025-06-11-a-deep-analysis-of-the-openai-agents-sdk-architecture-capabilities-and-ecosystem/) manages the iterative agent execution loop: the LLM receives input, decides on an action, invokes a tool, receives the output, and feeds it back for the next reasoning step — continuing until the task is complete or a stopping condition is met. This automates the core orchestration that developers would otherwise build manually.

### Multi-Agent Patterns

The SDK supports two complementary patterns for multi-agent collaboration, as described in the [official documentation](https://openai.github.io/openai-agents-python/handoffs/):

1. **Handoffs** — Peer-to-peer delegation. Agent A transfers the conversation to Agent B. Flexible for open-ended or conversational workflows, but can make it harder to maintain a global view.

2. **Agents as Tools** — Hierarchical orchestration. A manager agent calls specialist agents as tools, retaining overall control. Keeps a single thread of control and tends to simplify coordination.

## Execution Support

| Mode | Support |
|------|---------|
| **Local** | Full support. Runs on your own infrastructure. |
| **Remote** | Client-side framework; deploy to containers, edge, or cloud. |
| **Languages** | Python and TypeScript with feature parity. |
| **Streaming** | Full token streaming support. |
| **Tracing** | Automatic tracing of agent runs without custom instrumentation. Custom spans supported. OpenTelemetry compatible. |
| **MCP** | Full Model Context Protocol integration via `openai-agents-mcp` package. |
| **Models** | Optimized for OpenAI (GPT-4o-class), works with 100+ LLMs via Chat Completions API. |

## Code Example: Multi-Agent with Handoffs and Guardrails

```python
from agents import Agent, Runner, InputGuardrail, GuardrailFunctionOutput

# Define a guardrail to block off-topic requests
async def topic_guardrail(ctx, agent, input) -> GuardrailFunctionOutput:
    # Use a fast/cheap model to check if the input is on-topic
    result = await Runner.run(
        Agent(
            name="Topic Checker",
            instructions="Return 'off_topic' if the user is asking about unrelated subjects.",
            model="gpt-4o-mini",
        ),
        input,
    )
    is_off_topic = "off_topic" in result.final_output.lower()
    return GuardrailFunctionOutput(
        output_info={"decision": result.final_output},
        tripwire_triggered=is_off_topic,
    )

# Define specialist agents
billing_agent = Agent(
    name="Billing Specialist",
    instructions="You help users with billing questions, refunds, and payment issues.",
    tools=[lookup_invoice, process_refund],
)

technical_agent = Agent(
    name="Technical Support",
    instructions="You help users debug technical issues with the product.",
    tools=[search_docs, check_status],
)

# Define triage agent with handoffs and guardrails
triage_agent = Agent(
    name="Triage",
    instructions="Route users to the appropriate specialist based on their request.",
    handoffs=[billing_agent, technical_agent],
    input_guardrails=[
        InputGuardrail(guardrail_function=topic_guardrail),
    ],
)

# Run the agent
result = await Runner.run(triage_agent, "I was charged twice for my subscription")
print(result.final_output)
```

## Guardrails in Depth

Guardrails are a [key differentiator](https://openai.github.io/openai-agents-python/guardrails/) of the Agents SDK. They operate at three levels:

| Level | When It Runs | Scope |
|-------|-------------|-------|
| **Input Guardrails** | Before the first agent processes input | First agent in the chain only |
| **Output Guardrails** | After the final agent produces output | Last agent in the chain only |
| **Tool Guardrails** | Before and after each tool invocation | Every custom function-tool call |

Guardrails can run in **parallel** (default — best latency, but agent may consume tokens before guardrail fails) or **blocking** (guardrail completes before agent starts — prevents wasted tokens and side effects).

```python
from agents import function_tool, tool_input_guardrail, ToolGuardrailFunctionOutput

@tool_input_guardrail
def block_secrets(data):
    args = json.loads(data.context.tool_arguments or "{}")
    if "sk-" in json.dumps(args):
        return ToolGuardrailFunctionOutput.reject_content(
            "Remove secrets before calling this tool."
        )
    return ToolGuardrailFunctionOutput.allow()

@function_tool(tool_input_guardrails=[block_secrets])
def classify_text(text: str) -> str:
    """Classify text for internal routing."""
    return f"length:{len(text)}"
```

## From Swarm to Agents SDK

The Agents SDK is the direct successor to [OpenAI Swarm](https://github.com/openai/swarm) (21k stars, archived March 2025). Swarm was an educational, experimental framework that demonstrated the handoff pattern in under 100 lines of code, but was explicitly [not intended for production use](https://campustechnology.com/articles/2024/10/29/new-openai-swarm-framework-offers-experimental-tool-for-multi-agent-ai-networks.aspx) — no built-in memory, no tracing, no guardrails.

The Agents SDK carries forward Swarm's core handoff abstraction while adding everything needed for production:

| Capability | Swarm | Agents SDK |
|-----------|-------|------------|
| Agent handoffs | Yes (basic) | Yes (with input filters, context management) |
| Guardrails | No | Yes (input, output, tool-level) |
| Tracing | No | Yes (automatic, OpenTelemetry-compatible) |
| Streaming | No | Yes (full token streaming) |
| MCP support | No | Yes (via `openai-agents-mcp`) |
| Hosted tools | No | Yes (WebSearch, FileSearch, CodeInterpreter, etc.) |
| TypeScript | No | Yes (full feature parity) |
| Provider support | OpenAI only | 100+ LLMs via Chat Completions API |
| Maintenance | Archived | Active development (pushed daily) |

## Ecosystem Context

In late 2025, OpenAI expanded the agent platform further:

- **[AgentKit](https://developers.openai.com/blog/openai-for-developers-2025/)** — Higher-level building blocks for orchestrating agents, complementing the Agents SDK.
- **Conversations API** — Durable threads with replayable state for persistent agent conversations.
- **Connectors and MCP Servers** — Standardized external context and tool access.
- **Apps SDK** — Extends MCP to let developers build UIs alongside MCP servers.

## Strengths and Limitations

**Strengths:**

- Minimalist design — four primitives cover most agent patterns
- Production-ready tracing out of the box (strongest among lightweight frameworks)
- Cleanest handoff model of any framework ([source](https://gurusup.com/blog/best-multi-agent-frameworks-2026))
- Dual Python/TypeScript support with feature parity
- Provider-agnostic (100+ LLMs) despite OpenAI optimization
- Active daily development (20k stars, growing)
- Strong guardrail system with input/output/tool-level validation
- MCP integration for standardized tool access

**Limitations:**

- No built-in persistent memory — developers must implement their own state management
- Handoffs stay within a single run, limiting long-running multi-session workflows
- Best performance requires OpenAI models; other providers may have degraded experience
- Less flexible than LangGraph for complex graph-based workflows with conditional routing
- No built-in human-in-the-loop patterns (must be implemented manually)
- Newer framework — smaller ecosystem than LangGraph or CrewAI

## When to Use the OpenAI Agents SDK

Choose the Agents SDK when you want a lightweight, production-ready framework for multi-agent systems with minimal abstraction overhead. It is the best choice for teams already in the OpenAI ecosystem, projects that need strong tracing and guardrails out of the box, and scenarios where the handoff pattern (triage → specialist delegation) maps naturally to the problem. For complex graph-based workflows with conditional branching and checkpointing, consider LangGraph instead.
