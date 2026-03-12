# health

A Claude Code skill that audits your Claude Code configuration health across all layers.

## What it does

Systematically reviews your project's Claude Code setup using the six-layer framework:
`CLAUDE.md → rules → skills → hooks → subagents → verifiers`

- Detects project tier (Simple / Standard / Complex) and calibrates checks accordingly
- Runs three parallel diagnostic agents: Context Layer, Control + Verification Layer, Behavior Pattern
- Outputs a prioritized report: 🔴 Critical / 🟡 Structural / 🟢 Incremental

## Install

```bash
npx skills add hiclaude/health
```

## Usage

In any Claude Code session:

```
/health
```

Or describe what you want:

> "Run a health check on my Claude Code config"

## What gets checked

| Layer | Checks |
|-------|--------|
| CLAUDE.md | Signal-to-noise ratio, missing Verification/Compact Instructions sections, prose bloat |
| rules/ | Language-specific rules placement, coverage gaps |
| skills/ | Description token count, trigger clarity, auto-invoke appropriateness |
| hooks | Pattern field presence, file-type coverage, stale entries |
| allowedTools | Dangerous or stale one-time commands |
| Behavior | Rules violated in practice, repeated corrections, missing patterns |

## License

MIT
