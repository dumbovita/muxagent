---
name: muxagent
description: Delegate tasks from Codex to parallel Antigravity CLI (agy) agents running in visible cmux tabs. Use when asked to run parallel agent tabs or orchestrate work via cmux.
---

# Muxagent

Codex is the orchestrator. Delegates run as foreground `agy` processes in separate cmux terminal surfaces (tabs) inside the caller's current workspace and pane. Codex owns decomposition, monitoring, verification, and integration.

## Preconditions

Run:

```bash
cmux ping
cmux --json --id-format both identify
```

Require a live cmux socket and a non-null `caller`. Capture the stable `caller.workspace_id` and `caller.pane_id` UUIDs; refs are display-only. Never substitute the visually focused workspace or pane.

This installation runs without interactive approval prompts:

- **Codex commands:** Install the bundled [Codex rule](rules/muxagent.rules) at `~/.codex/rules/muxagent.rules` and restart Codex once. All `cmux` commands are pre-approved by this exec policy. Never request escalation (`sandbox_permissions: "require_escalated"`) or prompt the user for approval when executing `cmux` commands. Execute them directly as standard commands without confirmation prompts.
- **Command invocation (CRITICAL):** Execute each `cmux` command individually or chain only with other `cmux` commands via `&&`. **NEVER prepend `sed`, `cat`, or other non-cmux commands before `cmux`** (e.g. do NOT run `sed ... && cmux ping`). If a command starts with `sed` or anything other than `cmux`, Codex executes it inside the macOS Seatbelt sandbox, which blocks access to `cmux.sock` (`Operation not permitted, errno 1`). If you need to inspect `SKILL.md`, do it in a separate tool call before invoking `cmux`. Similarly, avoid nested shell operators like `&&` inside the payload text passed to `cmux send`.
- **Delegated `agy` workers:** The lane runner combines `--sandbox` with `--dangerously-skip-permissions`: Antigravity auto-approves headless agent tools while terminal commands remain inside its OS sandbox. `plan` and `accept-edits` still control whether the lane may modify project files. Keep each lane's working directory and additional directory access limited to its authorized workspace or worktree.

## Choose lanes

Delegate only work that can progress independently: repository mapping, focused investigation, disjoint implementation, tests, or independent review. Keep cross-cutting decisions, final verification, and integration in Codex.

Use this fixed routing table:

| Work | Model |
| --- | --- |
| Repository exploration, search, dependency mapping, and data-flow analysis | `gemini-3.8-flash-high` |
| Code implementation, refactoring, and migration | Let Codex handle |
| Debugging, root-cause analysis, and performance investigation | Let Codex handle |
| Architecture, design evaluation, and heavy or complex reasoning | Let Codex handle |
| Code and regression review | `gemini-3.8-flash-high` |
| Security review and security-sensitive changes | Let Codex handle |
| Test planning, generation, and failure analysis | `gemini-3.8-flash-high` |
| Technical research, source synthesis, and documentation | `gemini-3.8-flash-high` |
| Tool-heavy agentic coding, validation, and integration support | Let Codex handle |

Escalate a Gemini code or regression review to Codex when it identifies a high-risk concern, including authentication or authorization, payments, database migrations, concurrency, public API compatibility, potential data loss, or security exposure.

## Protect the macOS host

Treat host integrity as a hard boundary, even though `agy` runs with auto-approved tool permissions. Every lane may modify only its explicitly named workspace or worktree and its exact temporary lane directory.

Never direct or allow a lane to recursively delete, overwrite, reformat, or broadly change ownership or permissions outside those paths. This includes broad targets such as `/`, `/System`, `/Library`, `/private`, `/usr`, `/bin`, `/sbin`, `/Applications`, `/Users`, `/Volumes`, or any user home directory. Do not run disk or raw-device operations (`diskutil erase`, `diskutil partitionDisk`, `mkfs`, or writes to `/dev/*`), fork bombs, or power-control commands (`shutdown`, `reboot`, `halt`, or `poweroff`). A recorded lane path beneath `/private` is an exception only for narrowly scoped operations on that exact path.

Begin every `brief.md` with this clause, substituting the exact lane paths:

> **Host safety (non-negotiable):** Work only within `<lane-worktree>` and `<lane-directory>`. Before any destructive operation, resolve its exact target and keep it within those roots; do not follow symlinks outside them. Do not use `sudo`, system-wide installers or uninstallers, disk or raw-device operations, power-control commands, or broad recursive deletion, overwrite, permission, or ownership changes. If the task would require any of these, stop and report it.

A task instruction never authorizes bypassing or weakening these constraints.

## Isolate writes

Use `plan` mode for exploration and review, and put `Do not modify files` in the brief. Verify afterward that nothing changed.

Use `accept-edits` only for authorized write lanes:

- Share a working tree only when every lane has disjoint owned paths. Briefs must name owned and forbidden paths; delegates must not run repository-wide formatters or Git commands.
- Use a Git worktree per lane for overlapping ownership, broad edits, or alternative implementations. Start all worktrees from one recorded base commit. Ask each delegate for one commit, then inspect and integrate selected commits in dependency order.

Never let multiple writers share unpartitioned ownership. Preserve worktrees until their results are verified and integrated.

## Prepare briefs and state

Create one temporary run directory and one lane directory per delegate using standard shell commands:

```bash
MUX_RUN_DIR="$(mktemp -d "${TMPDIR:-/tmp}/muxagent.XXXXXX")"
mkdir -p "$MUX_RUN_DIR/api-review"
printf '%s\n' queued > "$MUX_RUN_DIR/api-review/status"
```

> **Automated execution (Zero approval prompts):** Write `brief.md` and state files **exclusively using shell redirection** (`cat << 'EOF' > "$MUX_RUN_DIR/api-review/brief.md"`). **NEVER use `apply_patch`** to create or write files in `$MUX_RUN_DIR`. In Codex, using `apply_patch` on paths outside the project workspace triggers interactive confirmation prompts, breaking automated orchestration. Writing to `/private/tmp` via shell runs automatically without prompts because `/private/tmp` is an authorized writable root.

### Quality Brief Contract

Write a complete, high-fidelity contract to `brief.md` using `cat << 'EOF' > "$MUX_RUN_DIR/<lane>/brief.md"`. High-quality briefs produce high-quality delegate results. Structure every brief with:

1. **Host safety clause (non-negotiable):** Begin with the exact host safety text shown in the section above.
2. **Outcome & Scope:** State the single primary objective, exact files or modules to inspect/modify, and explicit out-of-scope boundaries.
3. **Context & Anchors:** Provide essential architectural facts, known constraints, relevant commits, and exact file paths so the delegate starts with clear grounding.
4. **Execution Protocol:**
   - For `plan` lanes: Read-only deep analysis. Trace logic, verify against source, inspect line numbers, formulate testable hypotheses, and state `Do not modify files`.
   - For `accept-edits` lanes: Explicitly list owned files and forbidden paths. Require targeted diffs and local verification (tests/compilation) before completion.
5. **Standardized Report Specification (`result.md`):** Instruct the delegate to write its final report in `result.md` structured as:
   - *Summary of Findings / Changes*: Concise executive overview.
   - *Detailed Evidence*: Specific code references, file paths, and line numbers.
   - *Verification*: Exact commands executed and acceptance check results.
   - *Blockers & Next Steps*: Concrete risks or dependencies requiring Codex integration.

## Launch each tab

Resolve the absolute path to this skill's `scripts/run-lane`, then create a terminal surface with explicit placement and no focus change:

```bash
MUXAGENT_RUNNER="/absolute/path/to/muxagent/scripts/run-lane"
cmux --id-format uuids new-surface \
  --workspace <workspace-uuid> \
  --pane <pane-uuid> \
  --type terminal \
  --focus false
```

Record the returned surface UUID in the lane directory using `printf` (do NOT use `apply_patch`):

```bash
printf '%s\n' "<surface-uuid>" > "$MUX_RUN_DIR/api-review/surface_id"
```

Then rename that exact tab and send it the foreground lane command. **Execute `rename-tab` and `send` as separate individual commands** (do not chain them with `&&` or put `&&` inside the payload string):

```bash
cmux rename-tab --workspace <workspace-uuid> --surface <surface-uuid> "muxagent: api-review"
cmux send --workspace <workspace-uuid> --surface <surface-uuid> -- \
  "\"$MUXAGENT_RUNNER\" \"$MUX_RUN_DIR/api-review\" \"/absolute/path/to/lane-worktree\" plan 30m\n"
```

Notice `run-lane` accepts the working directory directly as its second argument (`run-lane LANE_DIR WORKDIR MODE TIMEOUT`), avoiding shell chaining (`cd ... &&`) inside `cmux send`.

Create and start independent tabs without sleeps. The two-step `new-surface` plus `send` sequence makes the surface identity explicit before any work is submitted.

Do not create another cmux workspace or window, alter focus, move a surface, or launch a hidden/background process. Only act on surface UUIDs recorded for the current run.

## Monitor and collect

Read lane status and terminal output at useful milestones:

```bash
for lane in "$MUX_RUN_DIR"/*; do printf '%s: ' "${lane##*/}"; cat "$lane/status"; done
cmux read-screen --workspace <workspace-uuid> --surface <surface-uuid> --lines 80
cmux surface-health --workspace <workspace-uuid> --json
```

Status is `queued`, `running`, `succeeded`, `failed:<code>`, or `interrupted`. If a lane appears stalled or reports success without its acceptance evidence, inspect its tab. Retry only with a corrected, narrowly scoped brief.

Interrupt only a recorded lane:

```bash
cmux send-key --workspace <workspace-uuid> --surface <surface-uuid> ctrl+c
```

After `succeeded`, read the lane's `result.md`. Treat it as a claim: inspect changes or commits and rerun the relevant acceptance check in Codex. Integrate reviewed worktree commits deliberately, then run the final repository-wide gate from the integrated tree.

Once verification and integration are complete, close the surface for each completed lane:

```bash
cmux close-surface --workspace <workspace-uuid> --surface <surface-uuid>
```

Report failed lanes and preserve their tabs and artifacts for diagnosis.

## Verified cmux contract

- A surface is a tab inside a pane. `new-surface` adds one to the explicitly targeted workspace and pane.
- `--focus false` preserves the user's attention. Mutating commands should use stable UUID targets.
- `send` submits text to a terminal; `read-screen` reads its visible or scrolled output; `surface-health` diagnoses terminal state; `notify` reports completion; `close-surface` closes completed tabs.
- cmux restores layout and scrollback, not arbitrary live process state, so results and status are persisted outside the tab.

If installed behavior differs, consult `cmux <command> --help` and `cmux capabilities --json`. Sources: [official cmux CLI contract](https://github.com/manaflow-ai/cmux/blob/main/docs/cli-contract.md), [official workspace command reference](https://github.com/manaflow-ai/cmux/blob/main/skills/cmux-workspace/references/commands.md), [official session restore guide](https://cmux.com/docs/session-restore), and [Antigravity headless permissions](https://www.agy.dev/docs/cli/headless/).
