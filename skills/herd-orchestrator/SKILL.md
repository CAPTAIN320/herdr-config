---
name: herd-orchestrator
description: "Orchestrate and coordinate work across multiple Herdr coding agents from the current pane. Use when the user asks to orchestrate / coordinate / plan-and-delegate across several herdr agents, open N agents, split work between workers, or run a ping-pong handshake and report-back protocol. Requires HERDR_ENV=1. Detects coordinator kind and picks worker agent/model per task difficulty. Uses git worktrees, a shared `.herd/` scratchpad, heartbeats, and automated teardown."
---

# Herd Orchestrator

Turn the current session into a **coordinator** that plans with the user and
delegates execution to sibling panes running other Herdr agents. The
coordinator does not search the repo or implement anything itself — that
keeps its context small and free to talk to the user. Repo search, reading,
editing, and testing all go to workers.

## Roles

- **Coordinator** — this session. Talks to the user, writes task specs,
  dispatches, verifies reports, summarizes. Never implements or searches.
  Owns `.herd/scratchpad.md` (single writer).
- **Workers** — sibling panes/agents. Do the actual explore, edit, and test
  work and report back themselves.

Reuse the same worker for similar tasks (its context carries over); a reused
worker keeps its worktree and branch, so start a new task on a clean state or
after the handoff is recorded. A good
default split is **one explore/review worker + one implement/test worker**,
but confirm the split, the goal, and the commit policy with the user before
dispatching: workers commit **only on their own branch**; the coordinator
never lets a worker commit to the base branch, push, or merge.

## Preflight

1. Confirm this is a Herdr-managed pane; otherwise stop:

   ```bash
   test "${HERDR_ENV:-}" = 1
   ```

   The coordinator's pane is `$HERDR_PANE_ID`, workspace `$HERDR_WORKSPACE_ID`.

2. Detect the coordinator kind (first match wins; record it in the scratchpad):

   ```bash
   # 1) herdr's own registry for this pane (authoritative if it exposes `kind`)
   herdr agent list   # find the entry whose pane == $HERDR_PANE_ID
   # 2) env markers (heuristic, verify per CLI version)
   [ -n "${KILO_SESSION:-}${KILO_CLI:-}" ] || [ -f ~/.config/kilo/kilo.jsonc ] || [ -f ~/.config/kilo/kilo.json ] && echo kilo
   [ -n "${CLAUDECODE:-}${CLAUDE_CODE_ENTRYPOINT:-}" ] && echo claude
   [ -n "${GEMINI_CLI:-}" ] && echo gemini
   env | grep -q '^CODEX_' && echo codex
   [ -n "${OPENCODE:-}" ] && echo opencode
   # 3) process ancestry
   ps -o comm= -p "$PPID"; ps -o comm= -p "$(ps -o ppid= -p $PPID | tr -d ' ')"
   # 4) installed CLIs usable as workers
   for c in claude codex gemini opencode; do command -v $c >/dev/null && echo "available: $c"; done
   ```

   If detection is ambiguous, ask the user. Never guess silently.

3. Discover existing workers — filter `agent list` to this workspace and
   exclude the coordinator:

   ```bash
   herdr agent list
   ```

   Report to the user how many workers are already open.

   **Orphan scan:** compare the scratchpad `Workers` table (if a previous
   `.herd/scratchpad.md` exists), `git worktree list`, and `herdr agent list`.
   Report leftovers from earlier runs (panes, worktrees, or `herd/*` branches
   without a matching row, or rows without a matching pane/worktree) and offer
   to tear them down (see Pane lifecycle & teardown).

4. Create the shared scratchpad directory and exclude it from git:

   ```bash
   HERD_DIR="${HERD_DIR:-$(git rev-parse --show-toplevel)/.herd}"
   mkdir -p "$HERD_DIR"/{log,results,worktrees,locks}
   grep -qx '.herd/' "$(git rev-parse --git-dir)/info/exclude" 2>/dev/null \
     || echo '.herd/' >> "$(git rev-parse --git-dir)/info/exclude"
   ```

   Initialize `$HERD_DIR/scratchpad.md` from the schema in Shared scratchpad
   (if an old scratchpad exists, archive it to `.herd/archive/` after the
   orphan scan instead of overwriting it).

5. If the user asked for **N agents**, open only the missing ones. For each:

   ```bash
   RUN=$(date +%Y%m%d-%H%M%S); BASE=$(git rev-parse --abbrev-ref HEAD)
   WT="$HERD_DIR/worktrees/worker-a"; BR="herd/$RUN/worker-a"
   git worktree add -b "$BR" "$WT" HEAD          # implement/edit workers only
   herdr pane layout --pane "$HERDR_PANE_ID"
   # wide pane -> right; tall/narrow pane -> down; avoid many same-direction splits
   herdr pane split --current --direction <right|down> --cwd "$WT" --no-focus
   # read the new pane id from .result.pane.pane_id
   herdr agent start <name> --kind <kind> [--model <m> if supported (verify)] --pane <new-pane-id>
   # Pick model from table based on detected kind and task tier
   herdr agent start <name> --kind <kind> --model <model-from-table> --pane <new-pane-id>
   ```

   Explore/review-only workers may keep `--cwd "$PWD"` instead of a worktree
   and are read-only. Uncommitted changes in the coordinator's checkout are
   **not** in the worktree. Warn the user or ask them to commit/stash first.
   Record pane, worktree and branch in the scratchpad `Workers` table with
   `created_by: coordinator`.

   Default `--kind` = detected coordinator kind, overridden by the Model
   strategy tier when another CLI is installed and the user approved mixing.
   Use unique short names like `worker-a`, `worker-b`. **Never close panes you
   did not create:** only panes recorded with `created_by: coordinator` in the
   scratchpad may be closed.

## Shared scratchpad (`.herd/`)

All run state lives in `$HERD_DIR` (default `<repo-root>/.herd`, created in
preflight and git-excluded). Always give workers the **absolute** `$HERD_DIR`;
their cwd differs (worktrees).

```
.herd/
  scratchpad.md        # coordinator-owned run state (schema below)
  log/<worker>.md      # worker-owned, append-only (progress/heartbeat/blockers)
  results/             # specs and long results (<task-id>.spec.md / .result.md)
  worktrees/<worker>/  # git worktrees (see Workspace isolation)
  locks/               # fallback file locks
```

**Write rules**

1. One writer per file. Only the coordinator edits `scratchpad.md`; each worker
   only appends (`>>`) to its own `log/<worker>.md`. This removes write races.
2. Log lines are single lines: `<ISO-UTC> <STATUS> <task-id> <text>`.
   STATUS ∈ `START|PROGRESS|HEARTBEAT|BLOCKED|DONE|FAILED`. Never rewrite or delete.
3. `Decisions` and `Handoffs` in the scratchpad are append-only: add rows,
   mark superseded rows `~~strikethrough~~`, never delete.
4. Coordinator reads `scratchpad.md` at the start of each turn and after each
   report (a file read is not polling). Workers read it before a task and on resume.
5. Chat messages carry a summary or pointer only; the file is the source of truth.
   Reading `.herd/` does not violate "coordinator never searches the repo".

**`scratchpad.md` schema**

```markdown
# Herd run <run-id>

- goal: <one line> - base: <branch@sha> - started: <ISO-UTC>
- coordinator: kind=<kind> pane=<id> model=<model|unknown>
- commit policy: <workers commit on own branch | no commits>

## Workers

| name | kind | model | tier | pane | created_by | worktree | branch | state | retries |
| ---- | ---- | ----- | ---- | ---- | ---------- | -------- | ------ | ----- | ------- |

## Task board

| id                                                                       | title  | owner      | tier | owns (globs) | depends_on | state | deadline | result |
| ------------------------------------------------------------------------ | ------ | ---------- | ---- | ------------ | ---------- | ----- | -------- | ------ |
| (state: queued → dispatched → working → reported → verified → integrated | failed | cancelled) |

## Wait-for (blocked edges)

| waiter | waits-on | since | reason |

## Decisions (append-only)

| ts | decision | by | rationale |

## Handoffs (append-only)

| ts | from | to | artifact (path/branch/commit) | notes |

## Results

- <task-id>: <one line> → <path/commit>

## Teardown checklist

- [ ] all tasks integrated/cancelled - [ ] worktrees removed - [ ] panes closed - [ ] .herd archived
```

## Model strategy

Classify each task into a tier **when writing the task spec** and record it in
the Task board.

| Tier       | Typical work                                                                                                      | Worker choice                              |
| ---------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| S (simple) | grep/read-only exploration, renames, formatting, docs, running tests                                              | cheapest/fastest model                     |
| M (normal) | single-module feature, bug fix with clear repro, tests                                                            | default model of the coordinator's kind    |
| H (hard)   | cross-module refactor, concurrency, security, architecture, flaky/unknown-cause bugs, integration/merge conflicts | strongest model / highest reasoning effort |

Editable mapping (adjust to what is installed and licensed; example names only):

| Coordinator kind | S                         | M                         | H                    |
| ---------------- | ------------------------- | ------------------------- | -------------------- |
| claude           | haiku-class               | sonnet-class              | opus-class           |
| kilo             | litellm/claude-haiku-4.5  | litellm/claude-sonnet-5.5 | litellm/gpt-6.1-sol  |
| codex            | mini / low effort         | default                   | high effort          |
| gemini           | flash-class               | pro                       | pro                  |
| opencode         | cheap configured provider | default                   | strongest configured |

Rules:

1. Default to the **same kind** as the coordinator (shared auth, known quirks).
   Use another installed kind only if it fits the tier better **and** the user
   approved mixing (cost/auth differ). Ask once per run; record in Decisions.
2. Reuse a worker if its tier ≥ the new task's tier. Do not reuse a tier-S worker for H.
3. Escalation: after one failed retry (see Health, recovery & escalation), respawn the task
   one tier up before escalating to the user.
4. If no model flag is available, select by `--kind` only and note
   "model=default" in the Workers table.

## Workspace isolation

Goal: two workers never write the same file at the same time.

1. **One worktree + branch per writing worker** (`git worktree add`, see Preflight).
   Read-only explore/review workers use the main checkout and must not write.
2. **File ownership.** Every task declares `Owns:` (paths or globs). Before
   dispatch the coordinator checks that `Owns` of all in-flight tasks are
   disjoint. If they overlap, serialize via `depends_on` (never run in parallel).
   Shared hot files (lockfiles, configs, barrels, `CHANGELOG`) are owned by the
   integration step only; workers report the needed change as a Handoff.
3. **Workers stay in `Owns`.** If a worker needs a path outside it, it logs
   `BLOCKED` and asks the coordinator; it does not edit it.
4. **Fallback when worktrees are impossible** (not a git repo, shared checkout
   required): file locks, created atomically and released on done:
   ```bash
   mkdir "$HERD_DIR/locks/$(echo -n "$path" | shasum | cut -c1-12).lock" \
     && echo "<worker> $(date -u +%FT%TZ) $path" > "$_/owner"   # fails if held
   ```
   A lock older than the task deadline with no heartbeat is stale; the
   coordinator may break it (log it in Decisions).
5. **Integration (coordinator decides, a worker executes).** When a task is
   `reported` and verified, the coordinator sends an integrate task to the
   integration worker (tier H when conflicts are expected): merge/rebase
   `herd/<run>/<worker>` into an integration branch `herd/<run>/integration`
   in order of `depends_on`, run tests, record the result in Handoffs. The
   coordinator never merges to the base branch itself; the user approves
   `integration → base`.
6. Never `git worktree add` the same branch twice and never delete a worktree with
   unmerged commits.

## Messaging protocol (critical)

- **Coordinator → worker:** `herdr agent prompt <worker> "<task>"` with **no
  `--wait`** — fire-and-forget. After dispatching, **end the turn.**
- **Never poll or wait on workers.** No `--wait`, no `agent wait` loops, no
  `agent read` polling. The user may ask questions meanwhile, and another
  worker may be trying to message the coordinator while it is blocked.
  Liveness is **push-based**: workers write heartbeats to `log/<worker>.md`,
  and an optional watchdog pane messages the coordinator only on an anomaly.
  The coordinator never loops or sleeps.
- **Worker → coordinator:** each worker reports by itself running
  `herdr agent prompt <coordinator-pane> "[worker X / <pane>] DONE|BLOCKED|FAILED: <result>"`
  with no `--wait`. Every task message you send must restate this reply
  instruction **including the coordinator's pane id**.
- **Long results:** the worker writes a file under `$HERD_DIR/results/` and
  sends only the path.
- **Long task specs:** write the spec to `$HERD_DIR/results/<task-id>.spec.md`
  and send the worker the path (avoids shell-quoting bugs).
- **Busy target:** if `agent prompt` returns `agent_blocked`/busy, wait ~10s
  and retry once. Do not blindly resend. The retry counts toward the task
  retry limit; max 1 retry, then mark `stuck`.
- **Concatenated reports:** two workers' reports can arrive in one message, so
  require every report to start with its `[worker X / pane]` tag on its own
  line.
- **Known pitfall:** building messages with `printf "$MSG" a b c` duplicated
  text when the arg count did not match the `%s` count. Prefer plain string
  interpolation or a spec file.

## Ping-pong handshake

Run this after preflight/opening panes and **before any real work**.

Send each worker: who the coordinator is (its pane id), the worker's own
label/pane, the reply protocol, and the instruction to touch no repo files
(only your own `.herd/log/` file) and just reply. The PONG also reports the
worker's `cwd`, `branch`, and `kind/model`, and the worker creates its log
file. Then end the turn; when all PONGs arrive, confirm to the user with a
table of workers and ask what to work on.

**Handshake template:**

```
[coordinator -> worker-a]
You are worker-a in pane <worker-a-pane>. I am the coordinator in pane $HERDR_PANE_ID.
This is a handshake only: touch no repo files (only your own .herd/log/ file).
First run: echo "$(date -u +%FT%TZ) START handshake" >> $HERD_DIR/log/worker-a.md
When done, report back by running (no --wait):
  herdr agent prompt $HERDR_PANE_ID "[worker-a / <worker-a-pane>] PONG - ready cwd=<pwd> branch=<branch> kind/model=<kind/model>"
Just send me: [worker-a / <worker-a-pane>] PONG - ready cwd=<pwd> branch=<branch> kind/model=<kind/model>
```

## Task lifecycle

1. Discuss the task with the user (goal, role split, file scope, whether
   workers may commit).
2. Write the task: scope, files, acceptance criteria, tier, `Owns`, "Commit on
   your branch `<branch>` only if the commit policy says so. Never push, merge
   or touch the base branch or paths outside `Owns`.", and the reply
   instruction (coordinator pane id + `[worker X / pane]` tag).
3. Dispatch with `herdr agent prompt <worker> "<task>"` (no `--wait`).
4. End the turn.
5. On report: verify -> integrate (see Workspace isolation) -> mark task
   `verified/integrated` -> if the worker has no queued task, teardown it.
   Optionally send to the review worker before moving to the next task.
6. At end of the run (or user abort) run Teardown (see Pane lifecycle &
   teardown).

**Task template:**

```
[coordinator -> worker-b] Task <id> (tier <S|M|H>): <one-line goal>.
Scope: <files / modules>. Owns: <globs>. Acceptance: <criteria>.
Worktree: <abs path>. Branch: <branch>. Scratchpad: $HERD_DIR (abs path).
Commit on your branch only if told; never push/merge or touch paths outside Owns.
Deadline: <N> min. Every ~2 min append HEARTBEAT to $HERD_DIR/log/worker-b.md;
append BLOCKED <id> waiting-on=<x> if stuck.
When done, report back by running (no --wait):
  herdr agent prompt $HERDR_PANE_ID "[worker-b / <worker-b-pane>] DONE|BLOCKED|FAILED: <concise result>"
Keep the report to a few lines; for long results write $HERD_DIR/results/<id>.result.md
and send only its path.
```

Heartbeat line format:
`echo "$(date -u +%FT%TZ) HEARTBEAT <task-id> <what>" >> $HERD_DIR/log/<worker>.md`

## Pane lifecycle & teardown

**Worker states** (tracked in scratchpad `Workers.state`):

`spawned → ready (PONG) → working → reported → idle → closed`, plus
`stuck`, `failed` (see Health, recovery & escalation) and `orphaned`.

| Transition             | Trigger                                       | Action                                |
| ---------------------- | --------------------------------------------- | ------------------------------------- |
| spawned → ready        | PONG received                                 | none                                  |
| ready/idle → working   | task dispatched                               | set task `dispatched`                 |
| working → reported     | `DONE` report                                 | verify; send next task or mark `idle` |
| idle → closed          | no queued task and user does not want it kept | Teardown (worker)                     |
| working → stuck/failed | stale heartbeat, `FAILED`, retries exhausted  | recovery, then Teardown               |

**Teardown (worker)**, run in order, and only if `created_by: coordinator`:

1. Confirm its branch is integrated (or the user said discard). Never remove
   a worktree with unmerged commits unless the user explicitly says discard.
2. Close the pane **(verify exact command: `herdr pane close --pane <id>` / `herdr agent stop <name>`)**.
3. `git worktree remove "$WT"` (use `--force` only for a failed/discarded worker); then
   `git branch -d "$BR"` (`-D` only if discarded).
4. Remove its locks (`$HERD_DIR/locks/*` owned by the worker) and temp specs/results
   that are already summarized in the scratchpad.
5. Update scratchpad: `state=closed`, tick the checklist.
6. Confirm with `herdr agent list` that the worker is gone.

**Teardown (run)** on completion, abort or irrecoverable failure:

1. Teardown every remaining worker (above), failed ones included.
2. `git worktree prune`; list leftovers with `git worktree list` and `git branch --list 'herd/<run>/*'`.
3. Move `.herd/scratchpad.md` and `.herd/log/` to `.herd/archive/<run-id>/`;
   delete `worktrees/`, `locks/` and `results/*.spec.md`. Keep `results/*.result.md`
   until the user confirms.
4. Report to the user: closed panes, removed worktrees, kept branches, archive path.

**No orphans:** every spawned pane/worktree must have a row in `Workers`. Do the
orphan scan in Preflight and again at Teardown. Never close panes not created by
the coordinator.

## Health, recovery & escalation

**Signals (no polling)**

- Worker heartbeat: a `HEARTBEAT` line in `log/<worker>.md` at least every 2 min
  of active work. Staleness is judged by the newest log line, not by the pane.
- Optional watchdog (a separate pane, no agent) that alerts the coordinator only
  on anomalies:
  ```bash
  # .herd/watchdog.sh <coordinator-pane> <stale-seconds>  (run in its own pane)
  while sleep 60; do
    for f in "$HERD_DIR"/log/*.md; do
      age=$(( $(date +%s) - $(stat -f %m "$f" 2>/dev/null || stat -c %Y "$f") ))
      [ "$age" -gt "$2" ] && herdr agent prompt "$1" "[watchdog] $(basename "$f" .md) silent ${age}s"
    done
  done
  ```
  Alert at most once per worker per incident (touch a `.alerted` marker).
- The coordinator also checks `log/` and deadlines whenever it wakes (a user
  message or any report). It does not sleep or loop to wait.

**Stuck classification** (the first matching one wins)
| Symptom | Class |
|---|---|
| no log line for > `stale` (default 5 min) and past deadline | `stuck` |
| `BLOCKED` with a reason | `blocked` |
| `FAILED` report or error text | `failed` |
| `agent list` shows the agent gone/exited (verify its status field) | `crashed` |
| two or more workers `BLOCKED` on each other | `deadlock` |

**Circular waits.** Workers only message the coordinator, never each other. All
waits are recorded in `Wait-for`. On any `BLOCKED`, add the edge and check for a
cycle (A→B→…→A) among `Wait-for` and `depends_on`. Task `depends_on` must be
acyclic **at dispatch time**. Break a cycle by re-ordering (finish the task
with the fewest dependencies first), merging the tasks into one worker, or
having the coordinator provide the missing input/decision (log in Decisions).

**Recovery procedure**

1. Ping once: `herdr agent prompt <worker> "[coordinator] status? reply with one line"` (no `--wait`).
   If a new log line appears before the next check, continue.
2. If no response, or the state is `crashed`/`failed`: **respawn with resume**
   - keep its worktree and branch (WIP is preserved) and close only the dead pane;
   - open a new pane with `--cwd "$WT"` and `agent start` (same name, bumped suffix, e.g. `worker-a2`);
   - send the original spec plus "Resume: read `log/worker-a.md` and `git status`/`git log` in your worktree; continue, do not restart".
3. Count `retries` in `Workers` and the Task board.

**Limits and escalation**

- Per task: max **2 retries** (1 on the same tier, 1 one tier up; see Model strategy).
- Per worker: max **1 respawn**. A second crash means replace the worker and `Teardown (worker)` the old one.
- Per run: if ≥ 3 workers fail, stop dispatching and ask the user.
- When limits are exceeded, or on a deadlock the coordinator cannot break: mark the task
  `failed`, **stop dispatching dependent tasks**, report to the user with: task,
  owner, last 5 log lines, branch/worktree, what was tried, options
  (retry / reassign / skip / abort). Never retry silently beyond the limits.
