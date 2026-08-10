# Garden Agent Design

**Date:** 2026-04-21
**Status:** Approved

## Problem

Manual harvest sessions block R&D work and require a dedicated context budget.
The dedup backlog grows between sessions and then needs a large one-off sweep
to clear it. The goal is to eliminate the backlog entirely by processing new
entries automatically as they arrive.

## Solution

A self-contained Claude Code agent that lives in the garden folder and fires
automatically whenever a commit adds new garden entries. No tokens consumed at
install time. No human intervention required for routine dedup work.

---

## Components

### 1. `garden-agent.sh`

Entrypoint script. Lives at `~/.hortora/garden/garden-agent.sh`.

- Detects mode via TTY presence: hook (no TTY) vs. manual run (TTY)
- Hook mode: invokes `claude --print` with the agent task, appends output to
  `garden-agent.log`
- Manual run: invokes `claude` interactively (safety valve, not a primary workflow)
- Sources no env files — Vertex credentials expected in shell environment
  (`CLAUDE_CODE_USE_VERTEX=1` and Google Cloud vars set at login)

```bash
#!/usr/bin/env bash
GARDEN_ROOT="${HORTORA_GARDEN:-$HOME/.hortora/garden}"
LOG="$GARDEN_ROOT/garden-agent.log"
TASK="You are the Hortora garden deduplication agent. Run the dedup sweep as described in CLAUDE.md."

if [[ "$1" == "--hook" ]] || [[ ! -t 0 ]]; then
    echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] garden-agent starting" >> "$LOG"
    claude --print "$TASK" >> "$LOG" 2>&1
    echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] garden-agent done" >> "$LOG"
else
    claude "$TASK"
fi
```

### 2. `.claude/settings.json`

Pre-approves all commands the agent needs. No permission prompts during runs.

```json
{
  "defaultMode": "acceptEdits",
  "permissions": {
    "allow": [
      "Bash(git show *)",
      "Bash(git log *)",
      "Bash(git status *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git diff *)",
      "Bash(python3 */dedupe_scanner.py *)",
      "Bash(python3 */validate_garden.py *)"
    ],
    "deny": []
  }
}
```

Path patterns use `*` prefix — portable across machines, no hardcoded paths.

### 3. `CLAUDE.md`

Agent operating instructions. Read automatically by Claude Code when the agent
is invoked from the garden directory.

```markdown
# Garden Deduplication Agent

You are the Hortora garden deduplication agent. When invoked, run a full
dedup sweep and commit the results without asking for confirmation.

## Environment

- Garden root: current working directory
- Scanner: `python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py .`
- All reads via `git show HEAD:<path>` — never read files directly

## Workflow

1. Run `dedupe_scanner.py . --top 50` to get unchecked pairs, highest score first
2. For each pair, read both entries: `git show HEAD:<domain>/<id>.md | head -35`
3. Classify and act:

| Classification | Action |
|---|---|
| **Distinct** | `--record` as `distinct` |
| **Related** | Append `**See also:**` line to both files, `--record` as `related` |
| **Duplicate** | Apply duplicate rules below |

4. Commit: `git add -A && git commit -m "dedupe: sweep N pairs — M related, K duplicates resolved"`

## Duplicate Rules

Keep the entry with the higher `score:` in frontmatter. If tied, keep the
newer `submitted:` date. If still tied, keep the longer entry (line count).

Delete the discarded file. Append to `DISCARDED.md`. Remove from `GARDEN.md`
index if present. Record as `duplicate-discarded`.

## Tiebreaker order

Score → submitted date → line count (keep longer).
```

### 4. `post-commit` hook

Fires automatically after any commit to the garden. Checks whether new
`GE-*.md` files were added. If yes, runs the agent in the background.

```bash
#!/bin/bash
GARDEN_ROOT="$(git rev-parse --show-toplevel)"
LOG="$GARDEN_ROOT/garden-agent.log"

new_entries=$(git diff --name-only HEAD~1 HEAD 2>/dev/null \
  | grep -E "^[^/]+/GE-[0-9]{8}-[0-9a-f]{6}\.md$")

if [[ -n "$new_entries" ]]; then
    echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] new entries detected, starting agent:" >> "$LOG"
    echo "$new_entries" | sed 's/^/  /' >> "$LOG"
    nohup "$GARDEN_ROOT/garden-agent.sh" --hook >> "$LOG" 2>&1 &
fi
```

Silent no-op when no new entries are present.

### 5. `garden-agent-install.sh`

Idempotent installer. Lives in `soredium/scripts/`. Run once per machine.
No tokens consumed — pure bash.

Checks each component, creates what's missing, skips what exists, and prints
a status summary. Exit code 0 always (status, not error).

**Usage:**
```bash
cd ~/.hortora/garden
~/claude/hortora/soredium/scripts/garden-agent-install.sh
```

**Checks and installs:**
- `garden-agent.sh` — creates and `chmod +x` if absent
- `.claude/settings.json` — creates `.claude/` dir and file if absent
- `CLAUDE.md` — creates if absent
- `.git/hooks/post-commit` — appends agent block if not already present
  (hook may already exist with other logic — installer appends, does not replace)
- `.gitignore` — appends `garden-agent.log` entry if not already present

---

## Data Flow

```
forage CAPTURE
    → git commit (new GE-*.md)
        → post-commit hook fires
            → garden-agent.sh --hook (background)
                → claude --print task
                    → reads CLAUDE.md (instructions)
                    → runs dedupe_scanner.py (unchecked pairs)
                    → reads entry pairs via git show
                    → classifies: distinct / related / duplicate
                    → writes cross-refs, records pairs, discards dupes
                    → git commit "dedupe: ..."
                        → output → garden-agent.log
```

---

## Autonomy

Fully autonomous. No interactive mode designed — running `./garden-agent.sh`
manually from a TTY is a safety valve, not a workflow.

Duplicate tiebreaker (score → date → length) is deterministic and requires no
human input.

---

## Portability

The installer targets `~/.hortora/garden` by default but respects
`$HORTORA_GARDEN` if set. The agent script does the same. Any garden on any
machine installs identically by running `garden-agent-install.sh`.

Vertex credentials (`CLAUDE_CODE_USE_VERTEX=1` etc.) are expected in the shell
environment — not sourced from any file, not committed to the garden.

---

## Out of Scope (v1)

- Staleness review (REVIEW workflow) — agent handles DEDUPE only
- Multi-garden federation — single garden per install
- Interactive duplicate resolution — autonomous tiebreaker is sufficient
