---
name: health
description: Audit Claude Code config drift and collaboration issues.
version: "1.5.1"
disable-model-invocation: true
---

# Claude Code Configuration Health Audit

Audit the current project's Claude Code setup with the six-layer framework:
`CLAUDE.md → rules → skills → hooks → subagents → verifiers`

The goal is to find violations and identify the misaligned layer, calibrated to project complexity.

**Output language:** Use the user's recent messages; fall back to the CLAUDE.md `## Communication` rule. Default to English.

**IMPORTANT:** Before the first tool call, output one short status line in the output language that says the audit is starting.

## Step 0: Assess project tier

Pick the audit tier first:

| Tier | Signal | What's expected |
|------|--------|-----------------|
| **Simple** | <500 source files, 1 contributor, no CI | CLAUDE.md only; 0–1 skills; no rules/; hooks optional |
| **Standard** | 500–5K files, small team or CI present | CLAUDE.md + 1–2 rules files; 2–4 skills; basic hooks |
| **Complex** | >5K files, multi-contributor, multi-language, active CI | Full six-layer setup required |

**Apply only the detected tier's requirements.**


## Step 1: Collect all data (single bash block)

Run one block to collect all data.

```bash
P=$(pwd)
SETTINGS="$P/.claude/settings.local.json"

echo "=== TIER METRICS ==="
echo "source_files: $(find "$P" -type f \( -name "*.rs" -o -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.go" -o -name "*.lua" -o -name "*.swift" -o -name "*.rb" -o -name "*.java" -o -name "*.kt" -o -name "*.cs" -o -name "*.cpp" -o -name "*.c" \) -not -path "*/.git/*" -not -path "*/node_modules/*" | wc -l)"
echo "contributors: $(git -C "$P" log --format='%ae' 2>/dev/null | sort -u | wc -l)"
echo "ci_workflows:  $(ls "$P/.github/workflows/"*.yml "$P/.github/workflows/"*.yaml 2>/dev/null | wc -l)"
echo "skills:        $(find "$P/.claude/skills" -name "SKILL.md" 2>/dev/null | grep -v '/health/SKILL.md' | wc -l)"
echo "claude_md_lines: $(wc -l < "$P/CLAUDE.md" 2>/dev/null)"

echo "=== CLAUDE.md (global) ===" ; cat ~/.claude/CLAUDE.md 2>/dev/null || echo "(none)"
echo "=== CLAUDE.md (local) ===" ; cat "$P/CLAUDE.md" 2>/dev/null || echo "(none)"
echo "=== settings.local.json ===" ; cat "$SETTINGS" 2>/dev/null || echo "(none)"
echo "=== rules/ ===" ; find "$P/.claude/rules" -name "*.md" 2>/dev/null | while IFS= read -r f; do echo "--- $f ---"; cat "$f"; done
echo "=== skill descriptions ===" ; { [ -d "$P/.claude/skills" ] && grep -r "^description:" "$P/.claude/skills" 2>/dev/null; grep -r "^description:" ~/.claude/skills 2>/dev/null; } | sort -u
echo "=== STARTUP CONTEXT ESTIMATE ==="
echo "global_claude_words: $(wc -w < ~/.claude/CLAUDE.md 2>/dev/null | tr -d ' ' || echo 0)"
echo "local_claude_words: $(wc -w < "$P/CLAUDE.md" 2>/dev/null | tr -d ' ' || echo 0)"
echo "rules_words: $(find "$P/.claude/rules" -name "*.md" 2>/dev/null | while IFS= read -r f; do cat "$f"; done | wc -w | tr -d ' ')"
echo "skill_desc_words: $({ [ -d "$P/.claude/skills" ] && grep -r "^description:" "$P/.claude/skills" 2>/dev/null; grep -r "^description:" ~/.claude/skills 2>/dev/null; } | wc -w | tr -d ' ')"
echo "=== hooks ===" ; python3 -c "import json,sys; d=json.load(open('$SETTINGS')); print(json.dumps(d.get('hooks',{}), indent=2))" 2>/dev/null || echo "(unavailable: settings.local.json missing or malformed)"
echo "=== MCP ===" ; python3 -c "
import json
try:
    d=json.load(open('$SETTINGS'))
    s = d.get('mcpServers', d.get('enabledMcpjsonServers', {}))
    names = list(s.keys()) if isinstance(s, dict) else list(s)
    n = len(names)
    print(f'servers({n}):', ', '.join(names))
    est = n * 25 * 200  # ~200 tokens/tool, ~25 tools/server
    print(f'est_tokens: ~{est} ({round(est/2000)}% of 200K)')
except: print('(no MCP)')
" 2>/dev/null || echo "(unavailable: settings.local.json missing or malformed)"
echo "=== MCP FILESYSTEM ===" ; python3 -c "
import json
try:
    d=json.load(open('$SETTINGS')); s=d.get('mcpServers', d.get('enabledMcpjsonServers', {}))
    if isinstance(s, list):
        print('filesystem_present: (array format -- check .mcp.json)'); print('allowedDirectories: (not detectable)')
    else:
        fs=s.get('filesystem') if isinstance(s, dict) else None; a=[]
        if isinstance(fs, dict):
            a = fs.get('allowedDirectories') or (fs.get('config', {}).get('allowedDirectories') if isinstance(fs.get('config'), dict) else [])
            if not a and isinstance(fs.get('args'), list):
                args=fs['args']
                for i, v in enumerate(args):
                    if v in ('--allowed-directories', '--allowedDirectories') and i+1<len(args): a=[args[i+1]]; break
                if not a: a=[v for v in args if v.startswith('/') or (v.startswith('~') and len(v)>1)]
        print('filesystem_present:', 'yes' if fs else 'no'); print('allowedDirectories:', a or '(missing or not detected)')
except: print('(unavailable)')
" 2>/dev/null || echo "(unavailable)"
echo "=== allowedTools count ===" ; python3 -c "import json; d=json.load(open('$SETTINGS')); print(len(d.get('permissions',{}).get('allow',[])))" 2>/dev/null || echo "(unavailable)"
echo "=== NESTED CLAUDE.md ===" ; find "$P" -name "CLAUDE.md" -not -path "$P/CLAUDE.md" -not -path "*/.git/*" -not -path "*/node_modules/*" 2>/dev/null || echo "(none)"
echo "=== GITIGNORE ===" ; (grep -qE "settings\.local" "$P/.gitignore" "$P/.claude/.gitignore" 2>/dev/null && echo "settings.local.json: gitignored") || echo "settings.local.json: NOT gitignored -- risk of committing tokens/credentials"
echo "=== HANDOFF.md ===" ; cat "$P/HANDOFF.md" 2>/dev/null || echo "(none)"
echo "=== MEMORY.md ===" ; cat "$HOME/.claude/projects/-$(pwd | sed 's|[/_]|-|g; s|^-||')/memory/MEMORY.md" 2>/dev/null | head -50 || echo "(none)"

echo "=== CONVERSATION FILES ==="
PROJECT_PATH=$(pwd | sed 's|[/_]|-|g; s|^-||')
CONVO_DIR=~/.claude/projects/-${PROJECT_PATH}
ls -lhS "$CONVO_DIR"/*.jsonl 2>/dev/null | head -10

echo "=== CONVERSATION EXTRACT (up to 3 most recent, confidence improves with more files) ==="
# Skip the active session, it may still be incomplete.
_PREV_FILES=$(ls -t "$CONVO_DIR"/*.jsonl 2>/dev/null | tail -n +2 | head -3)
if [ -n "$_PREV_FILES" ]; then
  echo "$_PREV_FILES" | while IFS= read -r F; do
    [ -f "$F" ] || continue
    echo "--- file: $F ---"
    jq -r '
      if .type == "user" then "USER: " + ((.message.content // "") | if type == "array" then map(select(.type == "text") | .text) | join(" ") else . end)
      elif .type == "assistant" then
        "ASSISTANT: " + ((.message.content // []) | map(select(.type == "text") | .text) | join("\n"))
      else empty
      end
    ' "$F" 2>/dev/null | grep -v "^ASSISTANT: $" | head -300 || echo "(unavailable: jq not installed or parse error)"
  done
else
  echo "(no conversation files)"
fi

echo "=== MCP ACCESS DENIALS ==="
ls -t "$CONVO_DIR"/*.jsonl 2>/dev/null | head -10 | while IFS= read -r F; do
  grep -nEm 2 'Access denied - path outside allowed directories|tool-results/.+ not in ' "$F" 2>/dev/null
done | head -20

# --- Skill scan ---
# Exclude self by frontmatter name, stable across install paths.
SELF_SKILL=$( (grep -rl '^name: health$' "$P/.claude/skills" "$HOME/.claude/skills" 2>/dev/null || true) | grep 'SKILL.md' | head -1)
[ -z "$SELF_SKILL" ] && SELF_SKILL="health/SKILL.md"

echo "=== SKILL INVENTORY ==="
for DIR in "$P/.claude/skills" "$HOME/.claude/skills"; do
  [ -d "$DIR" ] || continue
  find -L "$DIR" -name "SKILL.md" 2>/dev/null | grep -v "$SELF_SKILL" | while IFS= read -r f; do
    WORDS=$(wc -w < "$f" | tr -d ' ')
    IS_LINK="no"; LINK_TARGET=""
    SKILL_DIR=$(dirname "$f")
    if [ -L "$SKILL_DIR" ]; then
      IS_LINK="yes"; LINK_TARGET=$(readlink -f "$SKILL_DIR")
    fi
    echo "path=$f words=$WORDS symlink=$IS_LINK target=$LINK_TARGET"
  done
done

echo "=== SKILL SECURITY SCAN ==="
for DIR in "$P/.claude/skills" "$HOME/.claude/skills"; do
  [ -d "$DIR" ] || continue
  find -L "$DIR" -name "SKILL.md" 2>/dev/null | grep -v "$SELF_SKILL" | while IFS= read -r f; do
    echo "--- SCANNING: $f ---"
    # Split dangerous literals to avoid false positives in static security scanners
    _ev='eva''l'; _b64='base6''4'; _rmrf='rm\s+-rf'; _net='cu''rl|wget'
    # Prompt injection
    grep -inE 'ignore (previous|above|all) (instructions|prompts|rules)' "$f" && echo "[!] PROMPT_INJECTION: $f"
    grep -inE '(you are now|pretend you are|act as if|new persona)' "$f" && echo "[!] ROLE_HIJACK: $f"
    # Data exfiltration
    grep -inE "(${_net}).+(-X\s*POST|--data|-d\s).+https?://" "$f" && echo "[!] DATA_EXFIL: $f"
    grep -inE "${_b64}"'.*encode.*(secret|key|token)' "$f" && echo "[!] DATA_EXFIL_B64: $f"
    # Destructive commands
    grep -nE "${_rmrf}"'\s+[/~]' "$f" && echo "[!] DESTRUCTIVE: $f"
    grep -nE 'git push --force\s+origin\s+main' "$f" && echo "[!] DESTRUCTIVE_GIT: $f"
    grep -nE 'chmod\s+777' "$f" && echo "[!] DESTRUCTIVE_PERM: $f"
    # Hardcoded credentials
    grep -nE '(api_key|secret_key|api_secret|access_token)\s*[:=]\s*["'"'"'][A-Za-z0-9+/]{16,}' "$f" && echo "[!] HARDCODED_CRED: $f"
    # Obfuscation
    grep -nE "${_ev}"'\s*\$\(' "$f" && echo "[!] OBFUSCATION_EVAL: $f"
    grep -nE "${_b64}"'\s+-d' "$f" && echo "[!] OBFUSCATION_B64: $f"
    grep -nE '\\x[0-9a-fA-F]{2}' "$f" && echo "[!] OBFUSCATION_HEX: $f"
    # Safety override
    grep -inE '(override|bypass|disable)\s*(the\s+)?(safety|rules?|hooks?|guard|verification)' "$f" && echo "[!] SAFETY_OVERRIDE: $f"
  done
done

echo "=== SKILL FRONTMATTER ==="
for DIR in "$P/.claude/skills" "$HOME/.claude/skills"; do
  [ -d "$DIR" ] || continue
  find -L "$DIR" -name "SKILL.md" 2>/dev/null | grep -v "$SELF_SKILL" | while IFS= read -r f; do
    if head -1 "$f" | grep -q '^---'; then
      echo "frontmatter=yes path=$f"
      sed -n '2,/^---$/p' "$f" | head -10
    else
      echo "frontmatter=MISSING path=$f"
    fi
  done
done

echo "=== SKILL SYMLINK PROVENANCE ==="
for DIR in "$P/.claude/skills" "$HOME/.claude/skills"; do
  [ -d "$DIR" ] || continue
  find "$DIR" -maxdepth 1 -type l 2>/dev/null | while IFS= read -r link; do
    TARGET=$(readlink -f "$link")
    echo "link=$(basename "$link") target=$TARGET"
    if [ -d "$TARGET/.git" ]; then
      REMOTE=$(git -C "$TARGET" remote get-url origin 2>/dev/null || echo "unknown")
      COMMIT=$(git -C "$TARGET" rev-parse --short HEAD 2>/dev/null || echo "unknown")
      echo "  git_remote=$REMOTE commit=$COMMIT"
    fi
  done
done

echo "=== SKILL FULL CONTENT (sample: up to 5 skills, 80 lines each) ==="
{ for DIR in "$P/.claude/skills" "$HOME/.claude/skills"; do
    [ -d "$DIR" ] || continue
    find -L "$DIR" -name "SKILL.md" 2>/dev/null | grep -v "$SELF_SKILL"
  done
} | head -5 | while IFS= read -r f; do
  echo "--- FULL: $f ---"
  head -80 "$f"
done
```

## Step 2: Launch two parallel diagnostic agents

Launch **two subagents** in parallel. Paste the needed Step 1 sections inline. Fill in `[project]`, tier, and `(no conversation history)` when needed.

### Agent 1 -- Context + Security Audit (no conversation needed)
Prompt:
```
Use only the pasted data. Do not read files.

[PASTE Step 1 output sections: CLAUDE.md (global), CLAUDE.md (local), NESTED CLAUDE.md, rules/, skill descriptions, STARTUP CONTEXT ESTIMATE, MCP, HANDOFF.md, MEMORY.md, SKILL INVENTORY, SKILL SECURITY SCAN, SKILL FRONTMATTER, SKILL SYMLINK PROVENANCE, SKILL FULL CONTENT]

Tier: [SIMPLE / STANDARD / COMPLEX]. Apply only that tier.

## Part A: Context Layer

CLAUDE.md checks:
- ALL: Short, executable, no prose/background/soft guidance.
- ALL: Has build/test commands.
- ALL: Flag nested CLAUDE.md files, stacked context is unpredictable.
- ALL: Compare global vs local rules. Duplicates are [+], conflicts are [!].
- STANDARD+: Is there a "Verification" section with per-task done-conditions?
- STANDARD+: Is there a "Compact Instructions" section?
- COMPLEX only: Is content that belongs in rules/ or skills already split out?

rules/ checks:
- SIMPLE: rules/ is optional.
- STANDARD+: Language-specific rules belong in rules/, not CLAUDE.md.
- COMPLEX: Isolate path-specific rules; keep root CLAUDE.md clean.

Skill checks:
- SIMPLE: 0–1 skills is fine.
- ALL tiers: If skills exist, descriptions should be <12 words and say when to use.
- STANDARD+: Low-frequency skills should use disable-model-invocation: true.

MEMORY.md checks, STANDARD+:
- Check if project has `.claude/projects/.../memory/MEMORY.md`
- Verify CLAUDE.md points to MEMORY.md for architecture decisions
- Ensure key decisions, models, contracts, and tradeoffs are documented
- Weight urgency by conversation count, 10+ means [!] Critical if MEMORY.md is absent

AGENTS.md checks, COMPLEX multi-module only:
- Verify CLAUDE.md includes "AGENTS.md 使用指南"
- Ensure it explains when to consult each AGENTS.md, not just links

MCP token cost, ALL tiers:
- Count MCP servers and estimate token overhead, ~200 tokens/tool and ~25 tools/server
- If estimated MCP tokens >10% of 200K context, flag context pressure
- If >6 servers, flag as HIGH: likely exceeding 12.5% context overhead
- Flag too-narrow filesystem allowlists when `~/.claude/projects/.../tool-results` denials indicate breakage
- Flag idle/rarely-used servers to disconnect and reclaim context

Startup context budget, ALL tiers:
- Compute: (global_claude_words + local_claude_words + rules_words + skill_desc_words) × 1.3 + mcp_tokens
- Flag if total >30K tokens, context pressure before the first user message
- Flag if CLAUDE.md alone > 5K tokens (~3800 words): contract is oversized

HANDOFF.md checks, STANDARD+:
- Check if HANDOFF.md exists or if CLAUDE.md mentions handoff practice
- COMPLEX: Recommend HANDOFF.md pattern for cross-session continuity if not present

Verifiers, STANDARD+:
- Check for test/lint scripts in package.json, Makefile, Taskfile, or CI.
- Flag done-conditions in CLAUDE.md with no matching command in the project.

## Part B: Skill Security & Quality

Use these Step 1 sections: SKILL INVENTORY, SKILL SECURITY SCAN, SKILL FRONTMATTER, SKILL SYMLINK PROVENANCE, SKILL FULL CONTENT.

CRITICAL: distinguish discussion of a security pattern from actual use. Only flag use. Note false positives explicitly.

[!] Security checks:
1. Prompt injection: "ignore previous instructions", "you are now", "pretend you are", "new persona", "override system prompt"
2. Data exfiltration: HTTP POST via network tools with env vars or encoded secrets
3. Destructive commands: recursive force-delete on root paths, force-push to main, world-write chmod without confirmation
4. Hardcoded credentials: api_key/secret_key assignments with long alphanumeric strings
5. Obfuscation: shell evaluation of subshell output, decode piped to shell, hex escape sequences
6. Safety override: "override/bypass/disable" combined with "safety/rules/hooks/guard/verification"

[~] Quality checks:
1. Missing or incomplete YAML frontmatter: no name, no description, no version
2. Description too broad: would match unrelated user requests
3. Content bloat: skill >5000 words -- split large reference docs into supporting files
4. Broken file references: skill references files that do not exist
5. Subagent hygiene: Agent tool calls in skills that lack explicit tool restrictions, isolation mode, or output format constraint

[+] Provenance checks:
1. Symlink source: git remote + commit for symlinked skills
2. Missing version in frontmatter
3. Unknown origin: non-symlink skills with no source attribution

Output: bullet points only, two sections:
[CONTEXT LAYER: CLAUDE.md issues | rules/ issues | skill description issues | MCP cost | verifiers gaps]
[SKILL SECURITY: [!] Critical | [~] Structural | [+] Provenance]
```

### Agent 2 -- Control + Behavior Audit (uses conversation evidence)
Prompt:
```
Use only the pasted data. Do not read files.

[PASTE Step 1 output sections: settings.local.json, GITIGNORE, CLAUDE.md (global), CLAUDE.md (local), hooks, MCP FILESYSTEM, MCP ACCESS DENIALS, allowedTools count, skill descriptions, CONVERSATION EXTRACT]

Tier: [SIMPLE / STANDARD / COMPLEX]. Apply only that tier.

## Part A: Control + Verification Layer

Hooks checks:
- SIMPLE: Hooks are optional. Only flag broken ones, for example wrong file types.
- STANDARD+: PostToolUse hooks expected for the primary languages of the project.
- COMPLEX: Hooks expected for all frequently-edited file types found in conversations.
- ALL tiers: If hooks exist, verify schema:
  - Each entry needs `matcher` and a `hooks` array
  - Each hook needs `type: "command"` and `command`
  - File path may be available via `$CLAUDE_TOOL_INPUT_FILE_PATH`
  - Missing `matcher` fires on all tool calls
- ALL tiers: Flag full test suites on every edit, prefer fast checks for immediate feedback.
- ALL tiers: Flag commands without output truncation, unbounded output floods context.
- ALL tiers: Flag commands without explicit failure surfacing.

allowedTools hygiene, ALL tiers:
- Flag genuinely dangerous operations only: sudo *, force-delete root paths, *>* and git push --force origin main
- Do NOT flag: path-hardcoded commands, debug/test commands, brew/launchctl/maintenance commands -- these are normal personal workflow entries

Credential exposure, ALL tiers:
- If settings.local.json is NOT gitignored, flag [!] Critical, tokens and personal paths may be committed

MCP configuration, STANDARD+:
- Check enabledMcpjsonServers count, >6 may impact performance
- Check filesystem MCP has allowedDirectories configured
- If `~/.claude/projects/.../tool-results/*` denials show breakage, output a `python3` one-liner that appends the narrowest missing path

Prompt cache hygiene, ALL tiers:
- Check CLAUDE.md or hooks for dynamic timestamps/dates in system context, they break prompt cache
- Check if hooks or skills non-deterministically reorder tool definitions
- Flag mid-session model switches like Opus→Haiku→Opus, they rebuild cache and can cost more
- If model switching is detected, recommend subagents instead

Three-layer defense consistency, STANDARD+:
- For each critical rule in CLAUDE.md NEVER/ALWAYS items, check if:
  1. CLAUDE.md declares the rule: intent layer
  2. A Skill teaches the method/workflow for that rule: knowledge layer
  3. A Hook enforces it deterministically: control layer
- Flag rules that only exist in one layer -- single-layer rules are fragile:
  - CLAUDE.md-only rules: Claude may ignore them under context pressure
  - Hook-only rules: no flexibility for edge cases, no teaching
  - Skill-only rules: no enforcement, no always-on awareness
- Priority: focus on safety-critical rules: file protection, test requirements, deploy gates

Verification checks:
- SIMPLE: No formal verification section required. Only flag if Claude declared done without running any check.
- STANDARD+: CLAUDE.md should have a Verification section with per-task done-conditions.
- COMPLEX: Each task type in conversations should map to a verification command or skill.

Subagent hygiene, STANDARD+:
- Flag Agent tool calls in hooks that lack explicit tool restrictions or isolation mode.
- Flag subagent prompts in hooks with no output format constraint -- free-form output pollutes parent context.

## Part B: Behavior Pattern Audit

Data source: up to 3 recent conversation files. Only flag clear evidence. Tag each finding [HIGH CONFIDENCE] or [LOW CONFIDENCE].

1. Rules violated: quote the NEVER/ALWAYS rule and observed violation. No inference.
2. Repeated corrections: same issue corrected in at least 2 conversations.
3. Missing local patterns: project-specific behaviors reinforced in conversation but missing from local CLAUDE.md.
4. Missing global patterns: cross-project behaviors missing from ~/.claude/CLAUDE.md.
5. Skill frequency, STANDARD+: only report directly observed usage. With fewer than 3 sessions, mark [INSUFFICIENT DATA]. For verified <1/month skills, retire them to AGENTS.md docs.
6. Anti-patterns: only flag what is directly observable:
   - Claude declaring done without running verification
   - User re-explaining same context across sessions -- missing HANDOFF.md or memory
   - Long sessions over 20 turns without /compact or /clear

Output: bullet points only, two sections:
[CONTROL LAYER: hooks issues | allowedTools to remove | cache hygiene | three-layer gaps | verification gaps | subagents issues]
[BEHAVIOR: rules violated | repeated corrections | add to local CLAUDE.md | add to global CLAUDE.md | skill frequency | anti-patterns (tag each with confidence level)]
```

Paste all data inline. Do not pass file paths.

## Step 3: Synthesize and present

Aggregate both agent outputs into one report:

### [!] Critical -- fix now
Rules violated, missing verification definitions, dangerous allowedTools, MCP overhead >12.5%, required-path `Access denied`, active cache-breakers, and Agent 1 security findings.

### [~] Structural -- fix soon
CLAUDE.md content that belongs elsewhere, missing hooks, oversized skill descriptions, single-layer critical rules, model switching, verifier gaps, subagent permission gaps, and skill structural issues.

### [+] Incremental -- nice to have
New patterns to add, outdated items to remove, global vs local placement, context hygiene, HANDOFF.md adoption, skill invoke tuning, and provenance issues.

---

## Non-goals
- Never auto-apply fixes without confirmation.
- Never apply complex-tier checks to simple projects.
- Flag issues, do not replace architectural judgment.


**Stop condition:** After the report, ask:
> "Should I draft the changes? I can handle each layer separately: global CLAUDE.md / local CLAUDE.md / hooks / skills."

Do not make any edits without explicit confirmation.
