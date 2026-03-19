<div align="center">
  <h1>Claude Health</h1>
  <p><em>Audit your Claude Code configuration health across all layers.</em></p>
  <a href="https://github.com/hiclaude/health/stargazers"><img src="https://img.shields.io/github/stars/hiclaude/health?style=flat-square" alt="Stars"></a>
  <a href="https://github.com/hiclaude/health/releases"><img src="https://img.shields.io/github/v/tag/hiclaude/health?label=version&style=flat-square" alt="Version"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square" alt="License"></a>
</div>

<br/>

A Claude Code skill that systematically reviews your project's setup using the six-layer framework: `CLAUDE.md → rules → skills → hooks → subagents → verifiers`. It detects project complexity, runs two parallel diagnostic agents, and outputs a prioritized report telling you what to fix first.

## Install

Recommended, install globally for Claude Code:

```bash
npx skills add hiclaude/health -a claude-code -s health -g -y
```

Install only in the current project:

```bash
npx skills add hiclaude/health -a claude-code -s health -y
```

**Claude Plugin:**

```bash
claude plugin marketplace add hiclaude/health
claude plugin install health
```

## Usage

Restart Claude Code after installation. Then in any Claude Code session, run `/health` or just say:

> "Run a health check on my Claude Code config"

The skill automatically detects your project tier (Simple / Standard / Complex) and calibrates checks accordingly. It won't flag missing layers that aren't needed for your project size.

## Troubleshooting

- `Unknown skill: health`: install to Claude Code explicitly with `-a claude-code`. Use `-g` if you want the skill available in every project. Restart Claude Code after installation.
- Installer summary shows `./.agents/skills/health`: that summary is generic. For Claude Code, the installed path should end up at `./.claude/skills/health` or `~/.claude/skills/health`.
- Scope: this repository targets Claude Code only. Codex and OpenClaw are not supported here.

## What Gets Checked

| Layer | Checks |
| :--- | :--- |
| **CLAUDE.md** | Signal-to-noise ratio, missing Verification/Compact Instructions, prose bloat |
| **rules/** | Language-specific rules placement, coverage gaps |
| **skills/** | Description token count, trigger clarity, auto-invoke strategy, frequency-based optimization |
| **skill security** | Prompt injection, data exfiltration, destructive commands, hardcoded credentials, obfuscation, safety overrides |
| **hooks** | Pattern field presence, file-type coverage, stale entries |
| **MCP** | Server count, token cost estimation, context pressure detection, filesystem allowlist failures |
| **allowedTools** | Dangerous or stale one-time commands |
| **Prompt Cache** | Dynamic timestamps, tool reordering, mid-session model switching |
| **Three-Layer Defense** | Critical rules covered by CLAUDE.md + Skill + Hook together |
| **Behavior** | Rules violated in practice, repeated corrections, context hygiene habits |

## Output

Results are grouped into three priority levels:

- **Critical**: Fix now: rule violations, dangerous permissions, cache-breaking patterns, MCP overhead >12.5%, skill security issues (prompt injection, data exfiltration, etc.)
- **Structural**: Fix soon: misplaced content, missing hooks, single-layer critical rules, skill quality issues (bloated content, broken references)
- **Incremental**: Nice to have: context hygiene, HANDOFF.md adoption, skill frequency tuning, skill provenance (symlink sources, version tracking)

## Background

Built around the six-layer framework. Run `/health` to quickly see how your configuration measures up.

## License

MIT License, feel free to enjoy and participate in open source.
