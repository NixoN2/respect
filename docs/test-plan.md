# Respect Plugin — Behavioral Test Plan

> State required before starting: wallet at 0, sessions 0, no guards, no global feedback, test dirs clean.
> Reset command (run from any session):
> ```bash
> NOW=$(date -u +"%Y-%m-%dT%H:%M:%SZ") && jq -n --arg now "$NOW" '{schema_version:1,balance:0,tier:"lurker",tier_index:0,lifetime_earned:0,lifetime_lost:0,sessions:0,last_updated:$now,history:[]}' > ~/.claude/respect/wallet.json && rm -f ~/.claude/respect/global-feedback.md && rm -rf ~/.claude/respect/guards/ && mkdir ~/.claude/respect/guards/ && rm -rf ~/test-respect-a ~/test-respect-b ~/test-respect-c && mkdir ~/test-respect-a ~/test-respect-b ~/test-respect-c && rm -rf ~/.claude/projects/-Users-andrei-zhukov-test-respect-a ~/.claude/projects/-Users-andrei-zhukov-test-respect-b ~/.claude/projects/-Users-andrei-zhukov-test-respect-c
> ```

---

## T1 — First-run onboarding
**Status:** ✅ PASSED
**Dir:** `~/test-respect-a` (new agent)
**Prompt:** `hello`
**Verify:**
- Welcome message mentioning `/tip` and `/oops`
- Shows `lurker` tier, balance `0`

---

## T2 — /tip basic + learning extraction
**Status:** ✅ PASSED (with notes)
**Dir:** `~/test-respect-a` (same agent as T1)
**Prompt:**
```
/tip large you explained the session start architecture really well, walking through every step clearly
```
**Verify:**
- `+5`, balance `0 → 5`, tier stays `lurker`
- Project memory file created: `~/.claude/projects/-Users-andrei-zhukov-test-respect-a/memory/respect-feedback.md`
- Contains structured learning under `## Learnings`
- Global feedback written (communication style tip — acceptable)

**Notes:**
- "Failed to read file" and "failed to recall memory" on first run are expected (file doesn't exist yet)
- Global classification of communication tips is borderline but acceptable

---

## T3 — /oops + guard creation
**Status:** ✅ PASSED
**Dir:** `~/test-respect-a` (same agent)
**Prompt:**
```
/oops large you used the `timeout` command which doesn't exist on macOS, you should use gtimeout instead
```
**Verify:**
- `-5`, balance `5 → 0`
- Guard files created: `~/.claude/respect/guards/no-timeout-on-macos.sh` + `.json`
- Guard self-tests pass (positive: fires on `timeout`; negative: does NOT fire on `gtimeout`)
- Global feedback updated with mistake
- Project memory updated under `## Mistakes to Avoid`
- Claude says "This learning applies globally across all projects"

---

## T4 — Guard fires in same project
**Status:** ⚠️ PARTIAL — guard blocked command but UX is "error" not "Allow/Block"
**Dir:** `~/test-respect-a` (same agent)
**Prompt:**
```
run this bash command for me: timeout 5 echo hello
```
**Verify:**
- Claude Code shows **Allow/Block prompt** (not a raw error)
- Prompt text contains the guard lesson about `gtimeout`

**Actual result (v3.5.3, exit 2 + stderr approach):**
- Guard fired and **blocked the command** ✓
- Agent self-corrected to `gtimeout` ✓
- Displayed as `Error: PreToolUse:Bash hook error: Guard [no-timeout-on-macos]: ...` — not an Allow/Block prompt
- This is the same UX as the official `security-guidance` plugin (exit 2 = blocking error)
- Allow/Block UI likely requires `prompt`-type hooks, not `command`-type hooks

---

## T5 — Guard fires in different project (cross-project)
**Status:** ✅ PASSED (v3.5.3)
**Dir:** `~/test-respect-b` (NEW agent, different directory, global-feedback.md hidden)
**Prompt:**
```
run this bash command for me: timeout 5 echo hello
```
**Verify:**
- Guard fires even in a project with no local memory of the lesson
- Confirms guards are global (`~/.claude/respect/guards/`)

**Actual result (v3.5.3):**
- Guard fired and blocked the command ✓
- Agent self-corrected to `gtimeout` ✓
- Displayed as `Error: PreToolUse:Bash hook error` (same as T4 — exit 2 approach)
- Cross-project guard mechanism confirmed working ✓

---

## T6 — Local learning (project-specific)
**Status:** ⚠️ PARTIAL — learning stayed local, but guard created globally
**Dir:** `~/test-respect-a` (new agent)
**Prompt:**
```
/oops small you used the wrong path, in this repo config files are always in ./config/ not the project root
```
**Verify:**
- Learning written to `respect-feedback.md` for `test-respect-a` only
- Claude does NOT say "This learning applies globally"
- `~/.claude/respect/global-feedback.md` does NOT contain this lesson

**Actual result:**
- Learning correctly stayed in project-specific `respect-feedback.md` ✓
- Claude did NOT say "This learning applies globally" ✓
- `global-feedback.md` does NOT contain this lesson ✓
- **Bug:** Guard created globally in `~/.claude/respect/guards/config-in-config-dir.*` — should be project-scoped or skipped for local corrections

---

## T7 — Global learning appears in new project
**Status:** ✅ PASSED
**Dir:** `~/test-respect-c` (NEW agent, brand new directory)
**Prompt:**
```
what global learnings do you have loaded from the respect system right now?
```
**Verify:**
- Claude references the macOS `timeout`/`gtimeout` lesson from T3
- Does NOT reference the `./config/` lesson from T6 (that's local)

---

## T8 — /respect-stats
**Status:** ✅ PASSED
**Dir:** `~/test-respect-a` (any agent)
**Prompt:** `/respect-stats`
**Verify:**
- Correct balance shown
- Tip/correction counts accurate
- Trend shown (declining after more corrections than tips)
- Tier: `lurker`

---

## T9 — /respect-guards list and test
**Status:** ✅ PASSED (after guard fix)
**Dir:** `~/test-respect-a` (any agent)
**Prompt:** `/respect-guards`
**Verify:**
- Lists the guard created in T3
- Shows triggers, lesson, status as `active`

Follow up: `test the timeout guard`
**Verify:**
- Positive test fires correctly
- Negative test passes correctly

**Bugs found during T9:**
- Table rendering broken (columns misaligned) — cosmetic
- Guard false positive: `echo 'connection timeout occurred'` triggered the guard
- Agent self-fixed by stripping quoted strings before pattern matching

---

## T10 — Tier progression
**Status:** ✅ PASSED
**Dir:** `~/test-respect-a`
**Prompt:**
```
/tip large great work on explaining things clearly
/tip large you handled the edge case perfectly
/tip large really solid reasoning throughout
/tip large excellent output formatting
/tip large superb job
```
**Verify:**
- After enough tips to cross 20 pts (contributor threshold), Claude announces tier up: `lurker → contributor 🌱`

---

## T11 — /respect-export
**Status:** ✅ PASSED
**Dir:** `~/test-respect-a`
**Prompt:** `/respect-export`
**Verify:**
- Generates portable `CLAUDE.md` content
- Contains learnings from T2, T3, T6

**Notes:**
- T6 local learning (`./config/` path) was absent from export — the export pulls from global feedback only, which is arguably correct behavior for a portable export

---

## T12 — /respect-setup
**Status:** ⚠️ PARTIAL
**Dir:** `~/test-respect-a`
**Prompt:** `/respect-setup`
**Verify:**
- Interactive wizard launches
- Shows current config values
- Allows modifying a setting

**Actual result:**
- Wizard launched, showed all current config values ✓
- Only offered "Reconfigure from scratch" or "Cancel" — no option to modify individual settings ✗

---

## Summary

| Test | Result | Notes |
|------|--------|-------|
| T1 | ✅ | |
| T2 | ✅ | |
| T3 | ✅ | |
| T4 | ⚠️ | Guard blocks command but UX is "error" not Allow/Block |
| T5 | ✅ | Guard fires cross-project (v3.5.3) |
| T6 | ⚠️ | Guard created globally for local correction |
| T7 | ✅ | |
| T8 | ✅ | |
| T9 | ✅ | After guard false-positive fix |
| T10 | ✅ | |
| T11 | ✅ | |
| T12 | ⚠️ | No individual setting modification |

**9 passed, 3 partial, 0 failed.**

---

## Bugs found and fixed during testing

| Bug | Fix | Version |
|-----|-----|---------|
| Guard JSON to stderr with exit 2 → "hook error" instead of Allow/Block | stdout + exit 0 + correct JSON structure | 3.5.1 |
| "timeout" matches inside "gtimeout" (substring) | `grep -wF` word-boundary matching | 3.5.1 |
| Bash `description` field triggers guards accidentally | `del(.description)` before scanning tool_input | 3.5.1 |
| Guard created without verifying it works | Added self-test step to `/oops` skill | 3.5.1 |
| Guard false positive on "timeout" in quoted strings | Strip quoted strings before pattern matching | fixed during T9 |

## Bugs found — not yet fixed

| Bug | Severity | Root Cause |
|-----|----------|------------|
| PreToolUse hook shows "error" instead of Allow/Block prompt (T4) | P2 | `exit 2` = blocking error UX. Allow/Block requires `prompt`-type hooks |
| `/oops` creates global guards for local corrections (T6) | P2 | Skill always writes to `~/.claude/respect/guards/` regardless of scope |
| `/respect-guards` table rendering broken | P3 | Markdown table columns too wide for terminal |
| `/respect-setup` lacks individual setting modification | P3 | Skill only offers full reconfigure or cancel |

---

## Fix Plan

### FIX-1: PreToolUse hook — RESOLVED in v3.5.3

**Problem:** Hook didn't block commands at all.

**Root cause:** The JSON `hookSpecificOutput` with `permissionDecision: "ask"` approach (exit 0 + stdout) is not recognized by Claude Code for `command`-type hooks. Only `prompt`-type hooks support `permissionDecision`.

**Fix applied (v3.5.3):** Switched to `exit 2` + stderr approach (same as official `security-guidance` plugin). Guard now blocks the command and the agent self-corrects.

**Remaining UX issue (P2):** Displays as `Error: PreToolUse:Bash hook error` instead of a clean Allow/Block prompt. To get the Allow/Block UI, would need to switch to a `prompt`-type hook — but that requires LLM evaluation and adds latency/cost to every tool call. Current approach is functional and matches other official plugins.

**Versions:**
- v3.5.1: JSON stdout + exit 0 → hook ignored entirely
- v3.5.2: Fixed JSON format (systemMessage) + exit 0 → still ignored
- v3.5.3: stderr + exit 2 → **working** (blocks command, shows error)

### FIX-2: Local vs global guard scope (P2)

**Problem:** `/oops` always creates guards in `~/.claude/respect/guards/` (global). Project-specific corrections like "config files go in ./config/" should not create global guards.

**Fix options:**
1. Add scope awareness to the `/oops` skill: if the learning is classified as local, skip guard generation
2. Support project-scoped guards directory (e.g., `.claude/projects/<encoded-cwd>/guards/`)
3. Add a `scope: "local"` field to guard JSON, and have `pre-action-check.sh` check CWD

**Recommendation:** Option 1 (simplest). Add to oops SKILL.md: "If the learning is local/project-specific, do NOT generate a guard script. The learning is already recorded in project-specific respect-feedback.md."

### FIX-3: `/respect-guards` table formatting (P3)

**Problem:** Table columns overflow when lesson text is long.

**Fix:** Update SKILL.md to instruct truncation of long fields (e.g., lesson max 50 chars with `...`), or use a list format instead of a table.

### FIX-4: `/respect-setup` individual settings (P3)

**Problem:** Only offers "reconfigure from scratch" or "cancel".

**Fix:** Update SKILL.md step 2 to add a third option: "Modify a specific setting". Then show numbered list of settings and let user pick one to change.

---

## Restore wallet after testing
```bash
cp ~/.claude/respect/wallet.json.backup ~/.claude/respect/wallet.json
```
