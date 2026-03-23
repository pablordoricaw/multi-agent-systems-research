# Patchwork

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [patched-codes/patchwork](https://github.com/patched-codes/patchwork) |
| **Stars** | 1,548 |
| **Language** | Python |
| **License** | AGPL-3.0 |
| **Last Push** | 2025-04-18 |
| **Maturity** | Growing / Enterprise-focused |
| **Use Case Fit** | Automated PR reviews, security patching, CI/CD automation |

[Patchwork](https://www.patched.codes) by Patched.codes is an agentic AI framework designed for the "outer loop" of software development — the tasks that happen after code is written. While tools like Aider and Devin help write code, Patchwork automates PR reviews, security vulnerability fixes, dependency upgrades, and documentation generation. It is self-hosted, LLM-agnostic, and integrates directly into CI/CD pipelines.

## Architecture

Patchwork is built around three composable primitives:

```
┌──────────────────────────────────────┐
│            Patchflow                  │
│  (Workflow: e.g., AutoFix)           │
│                                      │
│  ┌──────┐  ┌──────┐  ┌──────┐      │
│  │Step 1│─>│Step 2│─>│Step 3│      │
│  │Scan  │  │LLM   │  │Create│      │
│  │Code  │  │Fix   │  │PR    │      │
│  └──────┘  └──────┘  └──────┘      │
│                                      │
│  ┌──────────────────────────────┐   │
│  │    Prompt Templates          │   │
│  │  (Customizable per task)     │   │
│  └──────────────────────────────┘   │
└──────────────────────────────────────┘
```

### Core Components

- **Steps**: Reusable atomic actions — create PR, commit changes, call an LLM, run a scanner.
- **Prompt Templates**: Customizable LLM prompts optimized for specific tasks (security fixes, code review, documentation).
- **Patchflows**: End-to-end workflows that chain steps and prompts. Can be defined via Python or a visual no-code builder.

### Built-in Patchflows

| Patchflow | Description |
|-----------|-------------|
| **AutoFix** | Scans code with Semgrep/depscan, generates and applies vulnerability fixes |
| **PRReview** | Extracts diff on PR creation, summarizes changes, comments on PR |
| **GenerateDocstring** | Generates docstrings for methods across a codebase |
| **DependencyUpgrade** | Updates vulnerable dependencies to fixed versions |
| **ResolveIssue** | Identifies files needing changes to fix an issue, creates a PR |
| **GenerateREADME** | Creates README documentation for repository folders |

## Execution Support

| Mode | Support |
|------|---------|
| **Local** | Self-hosted CLI agent. |
| **CI/CD** | Official GitHub Action (`patched-codes/patchwork-action@v1`). |
| **LLMs** | Any OpenAI-compatible endpoint (GPT, Claude, Gemini, Llama via Ollama). |
| **Security Tools** | Semgrep and depscan integration for vulnerability scanning. |
| **RAG** | ChromaDB integration for issue resolution with codebase context. |

## Code Example: CI/CD Integration

```yaml
# .github/workflows/autofix.yml
name: Patchwork AutoFix
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  autofix:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: patched-codes/patchwork-action@v1
        with:
          patchflow: 'AutoFix'
          github_token: ${{ secrets.GITHUB_TOKEN }}
          openai_api_key: ${{ secrets.OPENAI_API_KEY }}
```

```bash
# CLI usage
pip install 'patchwork-cli[security]'
export OPENAI_API_KEY="your-key"

# Run AutoFix on current repo
patchwork AutoFix

# Run with a different model
patchwork AutoFix --model gemini-1.5-pro-latest --google_api_key "your-key"

# Review a PR
patchwork PRReview --pr_url https://github.com/user/repo/pull/42
```

## Strengths and Limitations

**Strengths:**

- Focuses on the underserved "outer loop" of SDLC (reviews, patching, docs)
- Self-hosted and LLM-agnostic — no vendor lock-in
- Direct CI/CD integration via GitHub Actions
- Code-native workflows (Python) with no-code visual builder option
- Human-in-the-loop via PR-based workflow (AI proposes, human approves)
- Enterprise focus with AGPL-3.0 license (free self-hosted, paid managed)

**Limitations:**

- Smaller community (1.5k stars) compared to major frameworks
- AGPL-3.0 license may be restrictive for some enterprise use cases
- Last push was April 2025 — development pace has slowed
- No autonomous long-running execution
- Limited to post-coding tasks (not a general coding agent)

## When to Use Patchwork

Choose Patchwork when you want to automate the repetitive maintenance tasks in your SDLC — security patching, PR reviews, dependency management, and documentation. It is the best choice for teams that want CI/CD-integrated automation with human oversight (PR-based approval flow) and need to self-host with LLM flexibility.
