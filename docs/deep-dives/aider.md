# Aider

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [Aider-AI/aider](https://github.com/Aider-AI/aider) |
| **Stars** | 42,286 |
| **Language** | Python |
| **License** | Apache-2.0 |
| **Last Push** | 2026-03-17 |
| **Maturity** | Production-ready |
| **Use Case Fit** | AI pair programming, code generation, refactoring |

[Aider](https://aider.chat) is an AI pair programming tool that runs in your terminal. Unlike multi-agent orchestration frameworks, Aider focuses on the human-AI collaboration loop — you describe what you want, Aider edits your code, and changes are automatically committed to git. It is the most widely adopted terminal-based coding assistant with 42k stars.

## Architecture

Aider operates as a single intelligent agent with deep codebase awareness:

```
┌─────────────────────────────────────┐
│              Aider                   │
│                                     │
│  ┌──────────┐   ┌───────────────┐  │
│  │ Repo Map │   │   LLM API     │  │
│  │ (Tree-   │──>│   (Claude,    │  │
│  │  sitter) │   │    GPT, etc.) │  │
│  └──────────┘   └───────┬───────┘  │
│                         │          │
│  ┌──────────┐   ┌───────▼───────┐  │
│  │   Git    │<──│  Code Editor  │  │
│  │ (Auto-   │   │  (Search/     │  │
│  │  commit) │   │   Replace)    │  │
│  └──────────┘   └───────────────┘  │
└─────────────────────────────────────┘
```

### Core Components

- **Repository Map**: Uses tree-sitter to create a map of your entire codebase — functions, classes, imports, dependencies. This map is included in the LLM context so the model understands your project structure.
- **Edit Formats**: Supports multiple edit formats (diff, whole file, search/replace) optimized for different LLMs and tasks.
- **Git Integration**: Automatically commits each change with descriptive commit messages. Easy to diff, review, and undo.
- **Model Flexibility**: Works with Claude 3.7 Sonnet, DeepSeek R1/Chat V3, OpenAI o1/o3-mini/GPT-4o, and nearly any LLM including local models via Ollama.

## Execution Support

| Mode | Support |
|------|---------|
| **Local** | Full support. Terminal-based. |
| **IDE Integration** | Works within VS Code, Neovim, Emacs, and other editors. |
| **Voice** | Voice-to-code support for hands-free coding. |
| **Linting** | Auto-runs linters after each edit; fixes issues automatically. |
| **Testing** | Auto-runs tests after changes; can fix failing tests. |
| **Web Chat** | Copy/paste mode for using LLMs via web interfaces. |

## Code Example: Usage Patterns

```bash
# Install
pip install aider-install && aider-install

# Start with Claude
cd /your/project
aider --model sonnet --api-key anthropic=YOUR_KEY

# Start with a local model
aider --model ollama/llama3.2

# Start with specific files in context
aider src/auth.py src/models/user.py tests/test_auth.py

# Non-interactive: run a single command
aider --message "Add input validation to the login endpoint" --yes
```

### In-Session Commands

```
/add src/new_file.py      # Add file to context
/drop src/old_file.py     # Remove file from context
/run pytest tests/         # Run a command and share output
/diff                      # Show uncommitted changes
/undo                      # Undo the last commit
/voice                     # Switch to voice input
```

## Benchmarks

Aider is used as a coding agent in the [SWE-MERA benchmark](https://arxiv.org/abs/2507.11059) — a dynamic, continuously updated benchmark for evaluating LLMs on software engineering tasks. Its performance tracks closely with the underlying model quality, demonstrating that Aider's codebase mapping provides strong context for any capable LLM.

## Strengths and Limitations

**Strengths:**

- Best-in-class developer experience for terminal users
- Deep codebase understanding via repository mapping
- Automatic git integration with sensible commits
- 100+ language support
- Voice-to-code capability
- Lint and test integration (auto-fix)
- Model-agnostic (cloud and local models)
- Large, active community (42k stars)
- Apache-2.0 license

**Limitations:**

- Single-agent tool — not designed for multi-agent orchestration
- Performance depends heavily on the underlying LLM
- Context window limits can be a constraint on very large codebases
- No built-in sandboxing (executes in your local environment)
- Terminal-based interface may not suit all workflows
- No autonomous long-running execution (requires human interaction)

## Aider vs. Multi-Agent Frameworks

Aider occupies a different niche than frameworks like OpenHands or SWE-agent:

| Aspect | Aider | Multi-Agent Frameworks |
|--------|-------|----------------------|
| **Interaction** | Human-in-the-loop, conversational | Autonomous or semi-autonomous |
| **Scope** | Single files or small sets of files | Entire repositories |
| **Execution** | Local, in your terminal | Sandboxed (Docker, cloud VMs) |
| **Git** | Auto-commit every change | Typically creates PRs |
| **Best For** | Day-to-day coding with AI assistance | Large-scale automated tasks |

## When to Use Aider

Choose Aider when you want AI-assisted coding that fits naturally into your existing terminal and git workflow. It is the best tool for individual developers who want a pair programmer, not an autonomous agent. Ideal for feature development, refactoring, bug fixes, and exploration where human judgment guides the process.
