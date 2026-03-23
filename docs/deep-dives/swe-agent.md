# SWE-agent

## Overview

| Attribute | Detail |
|-----------|--------|
| **Repository** | [SWE-agent/SWE-agent](https://github.com/SWE-agent/SWE-agent) |
| **Stars** | 18,821 |
| **Language** | Python |
| **License** | MIT |
| **Last Push** | 2026-03-23 |
| **Maturity** | Production-ready (research-originated) |
| **Use Case Fit** | Automated bug fixing, issue resolution, cybersecurity |

SWE-agent is a system that enables LLM agents to autonomously solve software engineering tasks by interacting with codebases through a custom **Agent-Computer Interface (ACI)**. Published at [NeurIPS 2024](https://arxiv.org/abs/2405.15793), it pioneered the idea that LLM agents are a new category of end users that benefit from specially designed interfaces to the software they use.

## Architecture

```
┌─────────────────────────────────────┐
│           SWE-agent                 │
│                                     │
│  ┌──────────┐    ┌───────────────┐ │
│  │   LLM    │───>│  Agent-Comp.  │ │
│  │  (Brain) │    │  Interface    │ │
│  └──────────┘    │  (ACI)        │ │
│                  └───────┬───────┘ │
│                          │         │
│         ┌────────────────┼─────┐   │
│         ▼                ▼     ▼   │
│  ┌──────────┐  ┌──────┐ ┌──────┐  │
│  │File Edit │  │Repo  │ │Test  │  │
│  │Commands  │  │Nav   │ │Exec  │  │
│  └──────────┘  └──────┘ └──────┘  │
└─────────────────────────────────────┘
```

### Agent-Computer Interface (ACI)

The ACI is SWE-agent's key contribution. Rather than giving the LLM raw bash access, it provides a curated set of commands optimized for how LLMs interact with code:

- **File operations**: Create, edit, view files with LLM-friendly output formatting
- **Repository navigation**: Search across files, jump to definitions, explore directory structures
- **Test execution**: Run tests and parse results in a structured format
- **Contextual feedback**: Error messages and outputs formatted for LLM comprehension

The ACI design significantly impacts performance — the paper shows that interface design choices can swing benchmark scores by 20+ percentage points.

## Benchmarks

| Benchmark | Score | Notes |
|-----------|-------|-------|
| SWE-bench Lite | Strong baseline | First major autonomous SE agent |
| SWE-bench Verified | Competitive | Via Live-SWE-agent variant (77.4%) |
| HumanEvalFix | Evaluated | Code repair tasks |

### SWE-agent Variants

- **SWE-agent (original)**: The NeurIPS 2024 system. Custom ACI for code interaction.
- **SWE-agent-LM-32B**: Trained on [SWE-smith](https://neurips.cc/virtual/2025/poster/121828) data (50k instances from 128 repos). Achieves 40.2% on SWE-bench Verified — SOTA among open-source models.
- **[Live-SWE-agent](https://arxiv.org/abs/2511.13646)**: Self-evolving variant that autonomously improves its own scaffold at runtime. 77.4% on SWE-bench Verified — outperforms all existing agents including proprietary solutions.

## Execution Support

| Mode | Support |
|------|---------|
| **Local** | Full support. Runs locally with Docker. |
| **Remote** | Docker-based sandboxed execution. |
| **Models** | Model-agnostic — works with any LLM via API. |
| **Security** | Also used for offensive cybersecurity research. |

## Code Example

```bash
# Install SWE-agent
pip install swe-agent

# Run on a GitHub issue
swe-agent run \
    --model gpt-4 \
    --data-path https://github.com/user/repo/issues/42 \
    --config-file config/default.yaml

# Run on a local repository
swe-agent run \
    --model claude-3-sonnet \
    --repo-path /path/to/repo \
    --problem-statement "Fix the authentication bypass in login.py"
```

## Key Papers

1. **SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering** ([Yang et al., NeurIPS 2024](https://arxiv.org/abs/2405.15793)). The foundational paper introducing ACI design.
2. **SWE-smith: Scaling Data for Software Engineering Agents** ([Yang et al., NeurIPS 2025](https://neurips.cc/virtual/2025/poster/121828)). Data generation pipeline; SWE-agent-LM-32B achieves 40.2% SOTA.
3. **Live-SWE-agent: Can Software Engineering Agents Self-Evolve on the Fly?** ([Xia et al., 2025](https://arxiv.org/abs/2511.13646)). Self-evolving agent achieving 77.4% on SWE-bench Verified.
4. **SWE-Debate: Competitive Multi-Agent Debate for Software Issue Resolution** ([Li et al., 2025](https://arxiv.org/abs/2507.23348)). Multi-agent debate + MCTS for patch generation.

## Related: Agentless

[Agentless](https://arxiv.org/abs/2407.01489) (Xia et al., 2024) is a counterpoint to SWE-agent's approach. It uses a simple three-phase process — localize, repair, validate — without autonomous agent behavior. Despite its simplicity, Agentless achieved competitive results at $0.70 per issue and was adopted by OpenAI and DeepSeek for model evaluation.

## Strengths and Limitations

**Strengths:**

- Pioneered ACI design — purpose-built interfaces for LLM agents
- Strong academic grounding (NeurIPS publications)
- Active research ecosystem (SWE-smith, Live-SWE-agent, SWE-Debate)
- Model-agnostic
- Also applicable to cybersecurity and competitive coding
- Open-source training pipeline (SWE-smith)

**Limitations:**

- Primarily research-focused — less polished developer experience than Aider or OpenHands
- Performance heavily dependent on ACI design choices
- Docker required for safe execution
- Not designed for general-purpose multi-agent orchestration (focused on SE tasks)

## When to Use SWE-agent

Choose SWE-agent when you need a research-grade tool for automated software engineering tasks, especially bug fixing and issue resolution. It is the go-to choice for academic research on SE agents and for teams wanting to train custom coding models (via SWE-smith).
