# Agency Swarm

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [VRSEN/agency-swarm](https://github.com/VRSEN/agency-swarm) |
| **Stars** | 4,108 |
| **Language** | Python |
| **License** | MIT |
| **Last Push** | 2026-03-23 |
| **Maturity** | Growing / Community-driven |
| **Use Case Fit** | Business automation, custom agent teams, API-driven workflows |

Agency Swarm is a community-driven framework built by [VRSEN](https://agents.vrsen.ai/) that extends the OpenAI Agents SDK with specialized features for building reliable multi-agent applications. It takes the handoff pattern from OpenAI Swarm and adds the production capabilities that Swarm lacked — async execution, parallel tool calling, validation, and open-source model support.

## Architecture

Agency Swarm organizes agents into hierarchical agencies:

```
┌────────────────────────────────┐
│         Agency                 │
│  ┌──────┐    ┌──────────────┐ │
│  │ CEO  │───>│ Communication│ │
│  │ Agent│    │ Flows        │ │
│  └──┬───┘    └──────────────┘ │
│     │                         │
│  ┌──┴───┐  ┌──────────┐      │
│  │Dev   │  │Marketing │      │
│  │Agent │  │Agent     │      │
│  └──────┘  └──────────┘      │
└────────────────────────────────┘
```

### Core Components

- **Agents**: Built on OpenAI Agents SDK with extended capabilities — few-shot learning, response validators, fine-tuned model support.
- **Agencies**: Collections of agents with defined communication flows. CEO agent typically orchestrates.
- **Tools**: First-class support for parallel tool calling (up to 4x speedup). OpenAPI schema auto-conversion.
- **Async Modes**: Threading and async execution for concurrent agent operations.

## Execution Support

| Mode | Support |
|------|---------|
| **Local** | Full support. Python-based. |
| **Async** | Threading mode (agents in separate threads) and async mode. |
| **Parallel Tools** | Up to 4x speedup with parallel tool execution. |
| **Open-Source Models** | Support via Astra Assistants API (3 lines of code). |
| **OpenAPI Integration** | Auto-convert OpenAPI schemas into agent tools. |

## Code Example

```python
from agency_swarm import Agency, Agent, set_openai_key

set_openai_key("your-key")

ceo = Agent(
    name="CEO",
    description="Manages the agency and delegates tasks",
    instructions="You oversee all operations and delegate to specialists.",
)

developer = Agent(
    name="Developer",
    description="Writes and reviews code",
    instructions="You write clean, tested Python code.",
    tools=[CodeExecutionTool],
)

researcher = Agent(
    name="Researcher",
    description="Conducts research and analysis",
    instructions="You find and synthesize information from multiple sources.",
    tools=[WebSearchTool],
)

agency = Agency(
    [
        ceo,                    # CEO is the entry point
        [ceo, developer],      # CEO can communicate with Developer
        [ceo, researcher],     # CEO can communicate with Researcher
        [developer, researcher] # Developer can ask Researcher for info
    ],
    async_mode="threading",     # Enable async execution
)

agency.run_demo()  # Interactive Gradio UI
```

## Key Features

- **Response Validators**: Custom validation logic to ensure agent outputs meet quality criteria before proceeding.
- **Few-Shot Learning**: Provide example interactions to guide agent behavior without fine-tuning.
- **Gradio UI**: Built-in demo interface for testing agencies interactively.
- **Communication Flows**: Explicit definition of which agents can talk to each other, preventing chaotic message passing.

## Strengths and Limitations

**Strengths:**

- Extends OpenAI Agents SDK with practical production features
- Parallel tool calling for significant speedup
- Explicit communication flow definitions
- Built-in Gradio demo interface
- Open-source model support
- Active community development

**Limitations:**

- Smaller community compared to major frameworks (4k stars)
- Tightly coupled to OpenAI's API ecosystem
- Less documentation than LangGraph or CrewAI
- Not as battle-tested in large enterprise deployments

## When to Use Agency Swarm

Choose Agency Swarm when you're building on the OpenAI ecosystem and want more structure than the Agents SDK alone provides. Good for teams that want explicit agent communication flows, parallel tool execution, and a quick demo UI without heavy framework overhead.
