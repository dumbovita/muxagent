# muxagent

Run parallel **Antigravity CLI (`agy`)** agents in visible **`cmux`** tabs, orchestrated by **Codex**.

> **Note:** **macOS only.** This tool requires [cmux](https://github.com/manaflow-ai/cmux), a native macOS terminal multiplexer.

---

## What is it?

`muxagent` lets Codex act as an orchestrator that delegates tasks to parallel `agy` workers. Each worker runs in its own visible `cmux` tab inside your current workspace—without stealing your window focus.

- **Codex (Orchestrator):** Breaks down work, writes briefs, launches tabs, monitors progress, and verifies/integrates results.
- **cmux (Surface / Multiplexer):** Creates visible tabs, executes foreground commands, tracks terminal health, and delivers desktop notifications.
- **agy (Worker):** Antigravity CLI agent running headless inside an OS sandbox (`--sandbox --dangerously-skip-permissions`), executing tasks in either read-only (`plan`) or write (`accept-edits`) mode.

---

## Prerequisites

1. **macOS**
2. **[cmux](https://github.com/manaflow-ai/cmux)** running and accessible via socket (`cmux ping`)
3. **[Antigravity CLI (`agy`)](https://www.agy.dev)** installed and authenticated
4. **OpenAI Codex CLI**

---

## Setup

1. **Add Codex permissions rule:**
   Copy the bundled rule so Codex can run `cmux` commands without repeated permission prompts:
   ```bash
   cp rules/muxagent.rules ~/.codex/rules/muxagent.rules
   ```
   Restart Codex once after copying.

2. **Ensure runner script is executable:**
   ```bash
   chmod +x scripts/run-lane
   ```

3. **Install skill:**
   Symlink or place this directory into your Codex skills folder:
   ```bash
   ln -s "$(pwd)" ~/.codex/skills/muxagent
   ```

---

## Workflow

1. **Decomposition:** Codex divides a task into independent lanes (e.g., `api-review`, `auth-impl`, `unit-tests`).
2. **Briefing:** For each lane, Codex generates a `brief.md` containing acceptance criteria and boundaries.
3. **Execution:** Codex spins up a new tab via `cmux new-surface`, titles it, and runs `scripts/run-lane`.
4. **Monitoring:** Each lane outputs its live terminal feed in its own `cmux` tab and updates `status` (`queued`, `running`, `succeeded`, `failed`).
5. **Notification & Verification:** Upon completion, `cmux notify` sends a notification. Codex reads `result.md`, inspects changes, and verifies integration.

---

## Lane Modes

| Mode | Allowed Actions | Use Case |
| --- | --- | --- |
| `plan` | Read-only | Research, repository mapping, architecture review, root-cause debugging. |
| `accept-edits` | File modifications | Feature implementation, refactoring, writing tests (isolated via worktrees or distinct paths). |

---

## License

MIT
