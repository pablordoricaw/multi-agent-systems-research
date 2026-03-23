# OpenHands

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands) |
| **Stars** | 69,606 |
| **Language** | Python |
| **License** | Custom (OpenHands License) |
| **Last Push** | 2026-03-23 |
| **Maturity** | Production-ready |
| **Use Case Fit** | Full-stack software development, autonomous coding, enterprise engineering |

OpenHands is the most-starred open platform for cloud coding agents, with 69k+ GitHub stars. It provides a complete ecosystem for building, deploying, and managing software development agents — from local prototyping to production-scale remote execution. The [OpenHands Software Agent SDK](https://arxiv.org/html/2511.03690v1) (November 2025) represents a full architectural redesign built on event-sourced state management.

## Architecture

The SDK is organized into four composable Python packages:

```
┌──────────────────────────────────────────┐
│              openhands.agent_server       │
│         (REST/WebSocket APIs)            │
├──────────────────────────────────────────┤
│              openhands.workspace          │
│    (Docker, Hosted API, Local, Remote)   │
├──────────────────────────────────────────┤
│              openhands.tools              │
│        (Concrete tool implementations)   │
├──────────────────────────────────────────┤
│              openhands.sdk                │
│   (Agent, Conversation, LLM, Tool, MCP) │
└──────────────────────────────────────────┘
```

### Core Design Principles

- **Event-sourced state**: All agent actions and observations are stored as an immutable event stream. Enables deterministic replay, fault recovery, and debugging.
- **Immutable configuration**: Prevents configuration drift during long-running agent sessions.
- **Typed tool system**: Tools are strongly typed with MCP (Model Context Protocol) integration.
- **Workspace abstraction**: Same agent code runs locally for prototyping or remotely in secure containers.

### CodeAct Agent

OpenHands' primary agent uses the CodeAct paradigm — agents interact by generating and executing Python code rather than natural language directives. According to the [CodeAct framework research](https://www.emergentmind.com/topics/codeact-agent-framework), this approach achieves:

- 55–87% reduction in input tokens
- 41–70% reduction in output tokens
- Significant gains on VirtualHome, GAIA, and HotpotQA benchmarks

## Execution Support

| Mode | Support |
|------|---------|
| **Local** | Full support via CLI and GUI applications. |
| **Remote** | Containerized execution with REST/WebSocket APIs. |
| **Sandboxing** | Docker containers with security analyzer for agent actions. |
| **Interfaces** | Browser-based VS Code IDE, VNC desktop, persistent Chromium browser. |
| **Sub-agents** | Independent conversations inheriting parent's config and workspace. |
| **MCP** | Full MCP integration with OAuth for local and remote servers. |

## Benchmarks

| Benchmark | Score | Notes |
|-----------|-------|-------|
| SWE-Bench Verified | State-of-the-art | Across multiple LLM backends |
| GAIA | State-of-the-art | General AI assistant tasks |
| SWE-Dev (hard split) | 56.44% | Feature-driven development ([SWE-Dev paper](https://arxiv.org/abs/2505.16975)) |

## Integrations

OpenHands integrates natively with the tools engineers already use:

- **Version Control**: GitHub, GitLab
- **CI/CD**: Pipeline integration
- **Communication**: Slack
- **Ticketing**: Issue tracking systems
- **Models**: Model-agnostic — works with any LLM provider
- **AMD Partnership**: Local coding agents on Ryzen AI PCs for privacy and cost efficiency

## Code Example: Using the SDK

```python
from openhands.sdk import Agent, Conversation
from openhands.workspace import DockerWorkspace

# Configure workspace
workspace = DockerWorkspace(
    image="python:3.12-slim",
    mount_path="/workspace",
)

# Create agent
agent = Agent(
    model="claude-sonnet-4-6",
    workspace=workspace,
    tools=["bash", "file_edit", "browser"],
)

# Start conversation
conversation = Conversation(agent=agent)
result = conversation.run(
    "Fix the authentication bug in src/auth.py. "
    "The issue is that expired tokens are not being rejected."
)
```

## Key Papers

1. **The OpenHands Software Agent SDK** ([arXiv 2511.03690](https://arxiv.org/html/2511.03690v1), November 2025). Complete architectural redesign. Event-sourced, composable, four-package design.
2. **SWE-Dev: Evaluating and Training Autonomous Feature-Driven Software Development** ([Du et al., 2025](https://arxiv.org/abs/2505.16975)). OpenHands CodeActAgent evaluated at 56.44% on hard feature development tasks.

## Strengths and Limitations

**Strengths:**

- Highest star count of any coding agent platform (69k+)
- Complete ecosystem: SDK, CLI, GUI, server
- Event-sourced architecture enables replay, recovery, debugging
- Docker sandboxing with security analyzer
- Full MCP integration
- Model-agnostic with strong performance across providers
- Native enterprise integrations (GitHub, GitLab, Slack, CI/CD)
- Sub-agent delegation for structured parallelism

**Limitations:**

- Complex setup compared to simpler tools like Aider
- Docker dependency for sandboxed execution
- Custom license (not standard OSS like MIT/Apache)
- Resource-intensive for local development (2–4GB per agent instance)
- Not designed for general-purpose multi-agent orchestration (focused on software development)

## When to Use OpenHands

Choose OpenHands when you need a production-grade platform for autonomous software development at scale. It is the strongest choice for teams that need secure remote execution, enterprise integrations, and the flexibility to work with any LLM provider. The SDK architecture makes it ideal for building custom coding agents and workflows.
