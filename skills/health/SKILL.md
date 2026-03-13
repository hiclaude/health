---
name: health
description: Audit Claude Code configuration health across all layers (CLAUDE.md, rules, skills, hooks, MCP). Run periodically or when collaboration feels off.
version: "1.1.0"
---

# Claude Code Configuration Health Audit

Systematically audit the current project's Claude Code setup using the six-layer framework:
`CLAUDE.md → rules → skills → hooks → subagents → verifiers`

The goal is not just to find rule violations, but to diagnose which layer is misaligned and why — **calibrated to the project's actual complexity**.

## Step 0: Assess project tier

```bash
P=$(pwd)
echo "source_files: $(find "$P" -type f \( -name "*.rs" -o -name "*.ts" -o -name "*.py" -o -name "*.go" -o -name "*.lua" -o -name "*.swift" \) -not -path "*/.git/*" -not -path "*/node_modules/*" | wc -l)"
echo "contributors: $(git -C "$P" log --format='%ae' 2>/dev/null | sort -u | wc -l)"
echo "ci_workflows:  $(ls "$P/.github/workflows/"*.yml 2>/dev/null | wc -l)"
echo "skills:        $(ls "$P/.claude/skills/" 2>/dev/null | wc -l)"
echo "claude_md_lines: $(wc -l < "$P/CLAUDE.md" 2>/dev/null)"
```

Use this rubric to pick the audit tier before proceeding:

| Tier | Signal | What's expected |
|------|--------|-----------------|
| **Simple** | <500 source files, 1 contributor, no CI | CLAUDE.md only; 0–1 skills; no rules/; hooks optional |
| **Standard** | 500–5K files, small team or CI present | CLAUDE.md + 1–2 rules files; 2–4 skills; basic hooks |
| **Complex** | >5K files, multi-contributor, multi-language, active CI | Full six-layer setup required |

**Apply the tier's standard throughout the audit. Do not flag missing layers that aren't required for the detected tier.**

## Step 0.5: Check for skill updates (weekly)

```bash
# Check if it's time to check for updates (cache for 7 days)
CACHE_FILE="$HOME/.cache/claude-health-last-check"
CURRENT_VERSION="1.1.0"

should_check() {
    if [[ ! -f "$CACHE_FILE" ]]; then
        return 0
    fi
    local last_check=$(cat "$CACHE_FILE" 2>/dev/null || echo 0)
    local now=$(date +%s)
    local week=$((7 * 24 * 3600))
    if (( now - last_check > week )); then
        return 0
    fi
    return 1
}

if should_check; then
    # Check latest version from GitHub
    LATEST=$(curl -s "https://api.github.com/repos/tw93/claude-health/commits/main" 2>/dev/null | grep -o '"sha": "[^"]*"' | head -1 | cut -d'"' -f4 | cut -c1-7)
    if [[ -n "$LATEST" ]]; then
        # Get cached version
        CACHED_VERSION=$(cat "$CACHE_FILE.version" 2>/dev/null || echo "unknown")
        if [[ "$LATEST" != "$CACHED_VERSION" && "$CACHED_VERSION" != "unknown" ]]; then
            echo "[UPDATE] claude-health 有更新可用"
            echo "   当前: $CURRENT_VERSION"
            echo "   最新: $LATEST"
            echo "   更新: npx skills add tw93/claude-health@latest"
            echo ""
        fi
        # Update cache
        date +%s > "$CACHE_FILE"
        echo "$LATEST" > "$CACHE_FILE.version"
    fi
fi
```

## Step 1: Collect configuration snapshot

```bash
P=$(pwd)
SETTINGS="$P/.claude/settings.local.json"

echo "=== CLAUDE.md (global) ===" ; cat ~/.claude/CLAUDE.md
echo "=== CLAUDE.md (local) ===" ; cat "$P/CLAUDE.md" 2>/dev/null || echo "(none)"
echo "=== rules/ ===" ; find "$P/.claude/rules" -name "*.md" 2>/dev/null | while IFS= read -r f; do echo "--- $f ---"; cat "$f"; done
echo "=== skill descriptions ===" ; grep -r "^description:" "$P/.claude/skills" ~/.claude/skills 2>/dev/null
echo "=== hooks ===" ; python3 -c "import json,sys; d=json.load(open('$SETTINGS')); print(json.dumps(d.get('hooks',{}), indent=2))" 2>/dev/null
echo "=== MCP ===" ; python3 -c "
import json
try:
    d=json.load(open('$SETTINGS'))
    servers = d.get('mcpServers', d.get('enabledMcpjsonServers', {}))
    if isinstance(servers, dict):
        print('server_count:', len(servers))
        for name in servers:
            print(f'  - {name}')
        # Estimate token cost: ~200 tokens per tool, typical server has 20-30 tools
        est_tools = len(servers) * 25
        est_tokens = est_tools * 200
        print(f'estimated_mcp_tools: ~{est_tools}')
        print(f'estimated_mcp_tokens: ~{est_tokens} ({round(est_tokens/2000)}% of 200K context)')
    else:
        print('server_count:', len(servers))
except Exception as e:
    print('(no MCP config)')
" 2>/dev/null
echo "=== allowedTools count ===" ; python3 -c "import json; d=json.load(open('$SETTINGS')); print(len(d.get('permissions',{}).get('allow',[])))" 2>/dev/null
echo "=== HANDOFF.md ===" ; cat "$P/HANDOFF.md" 2>/dev/null || echo "(none)"
```

## Step 2: Collect conversation evidence

Read conversation files directly — do NOT write to disk and pass paths to subagents (subagents cannot read `~/.claude/` paths). Instead, read content here and inline it into agent prompts in Step 3.

```bash
PROJECT_PATH=$(pwd | sed 's|/|-|g' | sed 's|^-||')
CONVO_DIR=~/.claude/projects/-${PROJECT_PATH}

# List the 15 most recent conversations with sizes
ls -lhS "$CONVO_DIR"/*.jsonl 2>/dev/null | head -15
```

For each conversation file you want to include, use the Read tool (or jq via Bash) to extract its text content directly into a variable in your context. Assign files to agents B and C based on size:
- Large (>50KB): 1–2 per agent
- Medium (10–50KB): 3–5 per agent
- Small (<10KB): up to 10 per agent

Extract each file's content with:
```bash
cat <file>.jsonl | jq -r '
  if .type == "user" then "USER: " + ((.message.content // "") | if type == "array" then map(select(.type == "text") | .text) | join(" ") else . end)
  elif .type == "assistant" then
    "ASSISTANT: " + ((.message.content // []) | map(select(.type == "text") | .text) | join("\n"))
  else empty
  end
' 2>/dev/null | grep -v "^ASSISTANT: $" | head -300
```

Store the output in your context. You will paste it inline into the agent B and C prompts below.

## Step 3: Launch parallel diagnostic agents

Spin up **three focused subagents** in parallel, each examining one diagnostic dimension:

### Agent A — Context Layer Audit
Prompt:
```
Read: ~/.claude/CLAUDE.md, [project]/CLAUDE.md, [project]/.claude/rules/**, [project]/.claude/skills/**/SKILL.md

This project is tier: [SIMPLE / STANDARD / COMPLEX] — apply only the checks appropriate for this tier.

Tier-adjusted CLAUDE.md checks:
- ALL tiers: Is CLAUDE.md short and executable? No prose, no background, no soft guidance.
- ALL tiers: Does it have build/test commands?
- STANDARD+: Is there a "Verification" section with per-task done-conditions?
- STANDARD+: Is there a "Compact Instructions" section?
- COMPLEX only: Is content that belongs in rules/ or skills already split out?

Tier-adjusted rules/ checks:
- SIMPLE: rules/ is NOT required — do not flag its absence.
- STANDARD+: Language-specific rules (e.g., Rust, Lua) should be in rules/ not CLAUDE.md.
- COMPLEX: Path-specific rules should be isolated; no rules in root CLAUDE.md.

Tier-adjusted skill checks:
- SIMPLE: 0–1 skills is fine. Do not flag absence of skills.
- ALL tiers: If skills exist, descriptions should be <12 words (space-separated) and say WHEN to use.
- STANDARD+: Low-frequency skills should have disable-model-invocation: true.

Tier-adjusted MEMORY.md checks (STANDARD+):
- Check if project has `.claude/projects/.../memory/MEMORY.md`
- Verify CLAUDE.md references MEMORY.md for architecture decisions
- Ensure scrollbar, rendering, or other key design decisions are documented there

Tier-adjusted AGENTS.md checks (COMPLEX with multiple modules):
- Verify CLAUDE.md includes "AGENTS.md 使用指南" section
- Check that it explains WHEN to consult each AGENTS.md (not just list links)

MCP token cost check (ALL tiers):
- Count MCP servers and estimate token overhead (~200 tokens/tool, ~25 tools/server)
- If estimated MCP tokens > 10% of 200K context (~20,000 tokens), flag as context pressure
- If >6 servers, flag as HIGH: likely exceeding 12.5% context overhead
- Check if any idle/rarely-used servers could be disconnected to reclaim context

Tier-adjusted HANDOFF.md check (STANDARD+):
- STANDARD+: Check if HANDOFF.md exists or if CLAUDE.md mentions handoff practice
- COMPLEX: Recommend HANDOFF.md pattern for cross-session continuity if not present

Skill frequency strategy check (STANDARD+):
- If conversation evidence is available, check actual skill invocation frequency
- High frequency (>1x/session): should keep auto-invoke, optimize description
- Low frequency (<1x/session): should set disable-model-invocation: true
- Very low frequency (<1x/month): consider removing skill, move to AGENTS.md docs

Output: bullet points only, state the detected tier at the top, grouped by: [CLAUDE.md issues] [rules/ issues] [skills description issues] [MCP cost issues] [skill frequency issues]
```

### Agent B — Control + Verification Layer Audit
Prompt:
```
Read: [project]/.claude/settings.local.json, [project]/CLAUDE.md, [project]/.claude/skills/**/SKILL.md

Conversation evidence (inline — no file reading needed):
[PASTE EXTRACTED CONVERSATION CONTENT HERE]

This project is tier: [SIMPLE / STANDARD / COMPLEX] — apply only the checks appropriate for this tier.

Tier-adjusted hooks checks:
- SIMPLE: Hooks are optional. Only flag if a hook is broken (e.g., fires on wrong file types).
- STANDARD+: PostToolUse hooks expected for the primary language(s) of the project.
- COMPLEX: Hooks expected for all frequently-edited file types found in conversations.
- ALL tiers: If hooks exist:
  - Verify pattern field is present to avoid firing on all edits
  - Verify command contains {file_path} placeholder (not hardcoded paths)
  - Flag hooks using $CLAUDE_TOOL_INPUT_FILE_PATH (env var may not exist; suggest explicit file path passing)

allowedTools hygiene (ALL tiers):
- Flag stale one-time commands (migrations, setup scripts, path-specific operations).
- Flag dangerous operations:
  - HIGH: sudo *, rm -rf /, *>*
  - MEDIUM: brew uninstall, launchctl unload, xcode-select --reset
  - LOW (cleanup needed): path-hardcoded commands, debug/test commands

MCP configuration (STANDARD+):
- Check enabledMcpjsonServers count (>6 may impact performance)
- Check filesystem MCP has allowedDirectories configured

Prompt cache hygiene (ALL tiers):
- Check CLAUDE.md or hooks for dynamic timestamps/dates injected into system-level context (breaks prompt cache on every request)
- Check if hooks or skills non-deterministically reorder tool definitions
- In conversation evidence, look for mid-session model switches (e.g., user toggling Opus→Haiku→Opus) — flag as cache-breaking: switching model rebuilds entire cache, can be MORE expensive than staying on the original model
- If model switching is detected, recommend: use subagents for different-model tasks instead of switching mid-session

Three-layer defense consistency (STANDARD+):
- For each critical rule in CLAUDE.md (NEVER/ALWAYS items), check if:
  1. CLAUDE.md declares the rule (intent layer)
  2. A Skill teaches the method/workflow for that rule (knowledge layer)
  3. A Hook enforces it deterministically (control layer)
- Flag rules that only exist in one layer — single-layer rules are fragile:
  - CLAUDE.md-only rules: Claude may ignore them under context pressure
  - Hook-only rules: no flexibility for edge cases, no teaching
  - Skill-only rules: no enforcement, no always-on awareness
- Priority: focus on safety-critical rules (file protection, test requirements, deploy gates)

Tier-adjusted verification checks:
- SIMPLE: No formal verification section required. Only flag if Claude declared done without running any check.
- STANDARD+: CLAUDE.md should have a Verification section with per-task done-conditions.
- COMPLEX: Each task type in conversations should map to a verification command or skill.

Output: bullet points only, state the detected tier at the top, grouped by: [hooks issues] [allowedTools to remove] [cache hygiene] [three-layer gaps] [verification gaps]
```

### Agent C — Behavior Pattern Audit
Prompt:
```
Read: ~/.claude/CLAUDE.md, [project]/CLAUDE.md

Conversation evidence (inline — no file reading needed):
[PASTE EXTRACTED CONVERSATION CONTENT HERE]

Analyze actual behavior against stated rules:

1. Rules violated: Find cases where CLAUDE.md says NEVER/ALWAYS but Claude did the opposite. Quote both the rule and the violation.
2. Repeated corrections: Find cases where the user corrected Claude's behavior more than once on the same issue. These are candidates for stronger rules.
3. Missing local patterns: Find project-specific behaviors the user reinforced in conversation but that aren't in local CLAUDE.md.
4. Missing global patterns: Find behaviors that would apply to any project (not just this one) that aren't in ~/.claude/CLAUDE.md.
5. Anti-patterns present: Check for these specific anti-patterns:
   - CLAUDE.md used as wiki/documentation
   - Skills covering too many unrelated tasks
   - Claude declaring done without running verification
   - Subagents used without tool/permission constraints
   - User having to re-explain the same context across sessions
6. Context management discipline: Check conversation evidence for:
   - Long sessions (>20 user turns) without any /compact or /clear usage — flag as context decay risk
   - Task switches within a session without /clear — flag as context pollution
   - Multiple topics mixed in a single session without /compact between phases
   - If none of /context, /compact, /clear appear in any conversation, suggest adopting context hygiene habits
7. Session continuity patterns (STANDARD+): Check conversation evidence for:
   - Repeated context re-explanation across sessions — suggests missing HANDOFF.md or memory
   - Same architectural decisions being re-discussed — should be in CLAUDE.md or memory

Output: bullet points only, grouped by: [rules violated] [repeated corrections] [add to local CLAUDE.md] [add to global CLAUDE.md] [anti-patterns] [context hygiene]
```

Paste the extracted conversation content inline into agent B and C prompts. Do not pass file paths.

## Step 4: Synthesize and present

Aggregate all agent outputs into a single report with these sections:

### 🔴 Critical (fix now)
Rules that were violated, missing verification definitions, dangerous allowedTools entries, MCP token overhead >12.5%, cache-breaking patterns in active use.

### 🟡 Structural (fix soon)
CLAUDE.md content that belongs elsewhere, missing hooks for frequently-edited file types, skill descriptions that are too long, single-layer critical rules missing enforcement, mid-session model switching.

### 🟢 Incremental (nice to have)
New patterns to add, outdated items to remove, global vs local placement improvements, context hygiene habits, HANDOFF.md adoption, skill frequency optimization.

---

**Stop condition:** After presenting the report, ask:
> "Should I draft the changes? I can handle each layer separately: global CLAUDE.md / local CLAUDE.md / hooks / skills."

Do not make any edits without explicit confirmation.
