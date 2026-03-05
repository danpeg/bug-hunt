<img width="1280" height="480" alt="image" src="https://github.com/user-attachments/assets/8432d699-b4bb-4049-afcc-d2430ac1b58e" />

# /bug-hunt

Adversarial bug finding skill for [Claude Code](https://claude.com/claude-code). Uses 3 isolated AI agents to find and verify real bugs with high fidelity.

## How it works

Inspired by [@systematicls's article]([https://x.com/systematicls](https://x.com/systematicls/status/2028814227004395561)) on exploiting LLM sycophancy for better code review:

1. **Hunter** - Scans your code and reports every possible bug (biased to over-report)
2. **Skeptic** - Tries to disprove each bug (biased to dismiss false positives)
3. **Referee** - Reads the code independently and makes final verdicts

Each agent runs in a **completely isolated context** — they can't see each other's reasoning, only structured findings. This prevents anchoring bias and produces high-fidelity results.

## Install

```bash
git clone https://github.com/danpeg/bug-hunt.git ~/.claude/skills/bug-hunt
```

Claude Code auto-discovers skills in `~/.claude/skills/`.

## Usage

```
/bug-hunt              # Scan entire project
/bug-hunt src/         # Scan specific directory
/bug-hunt lib/auth.ts  # Scan specific file
```

### Dynamic Model Assignment

Assign different AI providers to each role using CLI flags. Defaults to all-Claude when no flags are given.

**Presets:**

```
/bug-hunt --preset=claude src/     # All Claude (default)
/bug-hunt --preset=codex src/      # All Codex
/bug-hunt --preset=gemini src/     # All Gemini
/bug-hunt --preset=mixed src/      # Hunter=Codex, Skeptic=Claude, Referee=Gemini
```

**Individual role overrides:**

```
/bug-hunt --hunter=codex --skeptic=claude --referee=gemini src/
/bug-hunt --hunter=codex src/      # Only Hunter uses Codex, rest default to Claude
```

Individual flags override preset values:

```
/bug-hunt --preset=codex --referee=claude src/   # Codex for Hunter+Skeptic, Claude for Referee
```

**Supported providers:**

| Provider | CLI Required | Install |
|----------|-------------|---------|
| `claude` | None (built-in) | Included with Claude Code |
| `codex` | [Codex CLI](https://github.com/openai/codex) | `npm install -g @openai/codex` |
| `gemini` | [Gemini CLI](https://github.com/google-gemini/gemini-cli) | `npm install -g @google/gemini-cli` |

Claude roles run as isolated Claude Code subagents. Codex and Gemini roles shell out to their respective CLI tools.

## Update

```bash
cd ~/.claude/skills/bug-hunt && git pull
```

## Uninstall

```bash
rm -rf ~/.claude/skills/bug-hunt
```

## How the scoring works

The scoring incentives are load-bearing — they exploit each agent's desire to maximize its score:

- **Hunter**: +1/+5/+10 for low/medium/critical bugs. Motivates thoroughness.
- **Skeptic**: Earns points for disproving false positives, but pays **2x penalty** for wrongly dismissing real bugs. Creates calibrated caution.
- **Referee**: Symmetric +1/-1 scoring with "ground truth" framing. Makes it precise rather than biased.

## Attribution

Based on the adversarial bug hunting technique described by [@systematicls](https://x.com/systematicls) in "How To Be A World-Class Agentic Engineer."

Brand by [Kitt](https://x.com/Kitt_Curious) at [Curious Endeavor](https://curiousendeavor.com/)

## Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

MIT
