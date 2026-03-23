# OpenAI Swarm

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [openai/swarm](https://github.com/openai/swarm) |
| **Stars** | 21,211 |
| **Language** | Python |
| **License** | MIT |
| **Last Push** | 2025-03-11 |
| **Maturity** | Experimental / Archived |
| **Use Case Fit** | Education, learning multi-agent concepts, prototyping |

!!! warning "Deprecated"
    OpenAI Swarm was officially shelved in March 2025. OpenAI now points developers to the **OpenAI Agents SDK** as the production-ready successor. Swarm remains useful as a learning tool and reference implementation.

OpenAI Swarm is a lightweight, educational framework that explores ergonomic multi-agent orchestration. Built on the Chat Completions API, its core design fits in [under 100 lines of code](https://campustechnology.com/articles/2024/10/29/new-openai-swarm-framework-offers-experimental-tool-for-multi-agent-ai-networks.aspx). The framework demonstrates the handoff pattern — where specialized agents transfer control to each other based on task requirements.

## Architecture

Swarm's architecture is deliberately minimal:

```
User Request
     │
     ▼
┌──────────┐   handoff()   ┌──────────┐   handoff()   ┌──────────┐
│ Triage   │──────────────>│ Sales    │──────────────>│ Support  │
│ Agent    │               │ Agent    │               │ Agent    │
└──────────┘               └──────────┘               └──────────┘
```

### Core Components

- **Agents**: Defined by instructions and a list of functions (tools). Stateless — no persistent memory between calls.
- **Handoffs**: Functions that return another agent, transferring control. The key abstraction.
- **Functions**: Tools that agents can call, including handoff functions to other agents.
- **Client**: Thin wrapper around the Chat Completions API that manages agent execution.

### Key Design Decisions

- **Stateless**: No built-in memory, state management, or persistence. Each call is independent.
- **Lightweight**: Minimal abstraction over the API. Easy to understand and modify.
- **Client-side**: All orchestration happens on the client. No server component.

## Code Example

```python
from swarm import Swarm, Agent

client = Swarm()

def transfer_to_sales():
    """Transfer the conversation to the sales agent."""
    return sales_agent

def transfer_to_support():
    """Transfer the conversation to the support agent."""
    return support_agent

triage_agent = Agent(
    name="Triage",
    instructions="Route the user to the appropriate department.",
    functions=[transfer_to_sales, transfer_to_support],
)

sales_agent = Agent(
    name="Sales",
    instructions="Help the user with purchasing decisions.",
)

support_agent = Agent(
    name="Support",
    instructions="Help the user resolve technical issues.",
)

response = client.run(
    agent=triage_agent,
    messages=[{"role": "user", "content": "I need help with billing"}],
)
```

## Why It Matters (Despite Being Archived)

Swarm's value is educational. It demonstrates:

1. **Agent handoffs**: The simplest possible implementation of inter-agent communication.
2. **Function-based routing**: Agents decide which agent to call next using regular function calls.
3. **Minimal orchestration**: Proves that multi-agent systems don't require complex frameworks.

The concepts pioneered by Swarm — particularly the handoff pattern — were carried forward into the OpenAI Agents SDK and influenced other frameworks like [Agency Swarm](agency-swarm.md).

## Successor: OpenAI Agents SDK

The Agents SDK (released March 2025) provides everything Swarm lacked for production use:

- Built-in tracing and guardrails
- Managed runtime
- Memory and state management
- Streaming support
- The "cleanest handoff model" among frameworks ([source](https://gurusup.com/blog/best-multi-agent-frameworks-2026))

## When to Use Swarm

Use Swarm only for **learning and experimentation**. It is the best resource for understanding the fundamentals of multi-agent handoffs with zero framework overhead. For anything production-bound, use the OpenAI Agents SDK or a framework like LangGraph or CrewAI.
