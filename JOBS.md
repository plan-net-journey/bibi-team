# Jobs

Everything the scheduler can run, and every frontmatter key it understands. For what bibi *is*, start at [`README.md`](README.md); for how to write in the vault, see [`vault/CONVENTIONS.md`](vault/CONVENTIONS.md).

## A job is a Markdown file

The scheduler walks `vault/case/` and parses every `.md` it finds. A file becomes a job when its frontmatter carries a **trigger** — `schedule:` or `at:`. Files without one are ignored entirely, which is why ordinary notes can live in the same folder as jobs without being touched.

```yaml
---
schedule: "*/10 * * * *"
job: uv run --script collect.py
attempts: 2
---

Free prose below the frontmatter. Never parsed, never executed — write whatever the job needs documented.
```

Three rules the parser enforces, and violating any of them makes the file a reported error rather than a silently skipped note:

- Exactly one trigger. `schedule:` **or** `at:`, never both.
- Exactly one payload: the key `job:`. There is no `claude:` or `app:` key — a Claude prompt is a `job:` value that begins with `claude:`.
- Integer fields must be integers. `attempts: "2"` and `attempts: true` are both rejected.

The job's **slug** — its name everywhere in the UI and CLI — is derived: an explicit `slug:` wins; otherwise a file called `README.md` or `SCHEDULE.md` takes its parent folder's name; otherwise the filename without extension.

## Triggers

| Value | Fires | Use for |
|---|---|---|
| `schedule: "0 9 * * *"` | on a cron expression, in node-local wall-clock time | Anything recurring. Standard 5-field cron via croniter. |
| `schedule: now` | once, the moment the scheduler discovers the file | Testing a new job. |
| `schedule: startup` | on every daemon start | Long-running apps and servers. |
| `schedule: never` | not at all | A job you only ever launch by hand with START. |
| `schedule: on_demand` | not at all | Same as `never`; a more honest name for "manual only". |
| `at: 2026-08-01T09:00` | once, at that timestamp | One-shot reminders and deferred work. Created by `/at`. |

Cron expressions are evaluated in **local** wall-clock time, not UTC — an early bug where "next run in 1h" was consistently off by the UTC offset was fixed by converting to a local naive datetime before handing it to croniter.

> **Do not use `schedule: autostart`.** The parser accepts it, so the file will not report an error, but nothing downstream understands the value: the job is classified as *recurring* while its next-fire time can never be computed, so it never fires and START is the only way to launch it. `startup` is the working trigger for apps that should come up with the daemon. (Verified against `bibi` dev: `is_recurring("autostart")` returns true while `_next_cron` returns `None`. `DESIGN.md` §5.3 still recommends `autostart` for long-running servers — that recommendation predates the value being dropped from the scheduler's special-value list and should not be followed.)

## Payloads

The value of `job:` decides how the job is executed. The prefix is the only switch.

```yaml
job: python collect.py                     # shell — bash -c
job: "claude: summarize today's mail"      # AI prompt — claude -p
```

A shell job runs under `bash -c`, can be anything the node can execute, and is the only kind that can request human input. A Claude job hands the rest of the value to Claude Code as a prompt, and the fields `model:`, `soul:` and `session:` apply to it — they are ignored on a shell job.

Both run through the same wrapper, produce the same append-only `output.jsonl`, and share one lifecycle. There is no second code path.

Python payloads have one rule of their own, from `CONVENTIONS.md`: use `uv run --script file.py`, never `uv run file.py`. Without `--script`, uv resolves in *project mode* — it walks upward for a `pyproject.toml` and syncs the entire project's dependencies before running anything, even for a pure-stdlib script.

## Usage patterns

### Recurring collector

The workhorse. Runs on cron, writes its results somewhere durable, exits.

```yaml
---
schedule: "*/10 * * * *"
job: uv run --script collect.py
attempts: 2
backoff: exponential
priority: 5
---
```

A scheduled fire runs in a **fresh git worktree checked out from `trunk`**, so anything gitignored inside it — including `vault/case/*/data/` — is wiped before the job starts. State that must survive across fires belongs outside any worktree: `~/.local/share/bibi/<subsystem>/` for data, `~/.config/bibi-<name>/` for secrets. Use `bibi.job.data_dir("<subsystem>")` rather than assembling the path by hand.

### Scheduled AI work

```yaml
---
schedule: "0 7 * * *"
job: "claude: Read vault/case/*/README.md, list what changed since yesterday, write it to vault/memo/$(date +%Y%m%d).Digest.md"
model: claude-sonnet-4-6
soul: Data
attempts: 1
---
```

`soul:` picks a persona from `.claude/souls/*.SOUL.md` — the set ships with the repo and is editable per team. `session:` continues an existing conversation instead of starting fresh. Claude jobs never enter `awaiting`; they cannot ask a human anything mid-run.

### One-shot

```yaml
---
at: 2026-08-01T09:00
job: "claude: Draft the follow-up mail for the meeting in this case folder"
---
```

Written for you by `/at`. After it fires it stays in the vault as a record — it simply has no future fire time.

### Long-running app

```yaml
---
schedule: startup
job: uv run --script dashboard.py
app_port: 9100
app_prefix: /dash
exec_mode: container
---
```

`app_port:` is what makes this an app rather than a job: it raises the silence timeout from 2h to 48h, exposes the process in the browser, and enables human-in-the-loop. Do **not** add `wall_time:` — that is a kill switch, and an app is supposed to keep running.

An app can ask for input. It calls `bibi.job.awaiting("What now?", input_format="text", port=9100)`, which writes a `BIBI:{...}` line to stdout; the wrapper parses it and moves the job to `awaiting`. The UI then links straight to the app's own address — there is no relay and no generated form, the app serves its own input page. `bibi.job.running()` moves it back.

### Manual only

```yaml
---
schedule: never
job: bash scripts/restore.sh
---
```

Defined, visible in the UI, fires only when someone presses START. The right shape for anything destructive or expensive.

There is a second, sharper trick worth knowing: an **intentionally invalid** `schedule:` value makes the scheduler report a parse error and drop the file from discovery without deleting it. That is a deliberate way to hide a job while keeping the file — but it shows up as an error, so prefer `never` unless hiding is exactly what you want.

## Frontmatter reference

Every key the parser reads. Anything not listed here is ignored, so you can keep your own metadata in the same frontmatter without confusing the scheduler.

### Trigger and payload

| Key | Type | Default | Meaning |
|---|---|---|---|
| `schedule` | string | — | Cron expression or `now` / `startup` / `never` / `on_demand`. Mutually exclusive with `at`. |
| `at` | ISO 8601 | — | One-shot timestamp. Timezone-aware values are converted to local time. |
| `job` | string | — | **Required.** Shell command, or an AI prompt via a leading `claude:`. |
| `slug` | string | derived | Job name. Falls back to folder name (for `README.md`/`SCHEDULE.md`) or filename stem. |
| `priority` | int | `0` | Higher wins when several jobs are queued. Ties break FIFO. |

### Claude payloads only

| Key | Type | Default | Meaning |
|---|---|---|---|
| `model` | string | `claude-sonnet-4-6` | Model for this job. |
| `soul` | string | none | Persona from `.claude/souls/*.SOUL.md`. |
| `session` | string | none | Continue an existing session instead of starting a new one. |

### Retries and timeouts

| Key | Type | Default | Meaning |
|---|---|---|---|
| `attempts` | int | `1` | **Total** attempts, the first run included. `1` means one run and no retry; `3` means up to three runs. `0` means the job never starts — not even manually. Changed in `v0.8.10` (plan-net-journey/bibi#168): the old reading counted retries *in addition to* the first run, so `attempts: 3` produced four runs. |
| `backoff` | string | `fixed` | `fixed` \| `linear` \| `exponential`. |
| `silence_timeout` | int (s) | context-dependent | No stdout/stderr for this long while `running` → `zombie`. Default is `3600` for Claude payloads, `172800` (48h) when `app_port`/`app_prefix` is set, `7200` otherwise. |
| `wall_time` | int (s) | none | Hard kill after this long. Opt-in only — there is deliberately no global default, because it would kill every app. |
| `defer_time` | int (s) | none (falls back to `360`) | How long a job that deferred itself waits before retrying. |
| `defer_max` | int (s) | `1200` | Total time a job may spend deferring, measured from its first defer, before it is given up on as `inactive`. |
| `error_time` | int (s) | none | Counterpart to `defer_time` for the failure path. |

**A running job is expected to say something.** Every line on stdout/stderr and every `bibi.job` signal counts as activity; the wrapper records the moment in `jobs.last_ping_at`, and since v0.7.6 that value travels to the client alongside `silence_timeout`, so the FE can show when the deadline runs out. A job that says nothing for `silence_timeout` is killed as a `zombie` — that is the promise, not an accident.

> **Buffer your output at your peril.** The wrapper only sees a line once it reaches its pipe. An app that buffers stdout in blocks can work and log for minutes without a single line arriving, and from the deadline's point of view it is **silent** the whole time. Use `print(..., flush=True)` or `PYTHONUNBUFFERED=1`. The trap is nasty because it only shows up under load: short runs flush their buffer when they exit. If your job genuinely has nothing to print — an app idling between requests — call `bibi.job.activity()`, which is a heartbeat without a line. If it cannot talk over stdout at all, `POST /-/job/{id}/ping` feeds the same column.

The `attempts` semantics are the field most likely to surprise you, and they were surprising in production: the wrapper checks `attempt_cur < attempts_max`, so `attempts: 1` runs the job **twice** before reporting `error`. The default was changed from `1` to `0` in July 2026 after exactly that happened to a job that never asked for a retry.

### Execution environment

| Key | Type | Default | Meaning |
|---|---|---|---|
| `app_port` | int | none | Port the process listens on. Setting it marks the job as an app. |
| `app_prefix` | string | none | URL path prefix for routing. |
| `exec_mode` | string | node config | `host` \| `container`. Overrides the node's `BIBI_EXEC_MODE`. |
| `image` | string | node default | Container image for this job. |
| `docker_args` | list of strings | none | Raw arguments appended to `docker run`. See the warning. |

> **`docker_args` is unvalidated by design.** The strings are passed to the Docker CLI verbatim, and nothing in bibi inspects them. `["--privileged"]` and `["-v", "/:/host"]` are both honoured exactly as written, quietly voiding the arbitrary-UID and fixed-mount assumptions the rest of the container sandbox depends on. Treat it like `sudo` in a job script. Legitimate uses so far: joining an extra network (`["--network", "gitea_default"]`) and publishing a port beyond `app_port` (`["-p", "8780:8780"]`). It is a no-op outside `exec_mode: container`.

## Lifecycle

A job's state is owned by whichever component is responsible for moving it out of that state.

| State | Owned by | Means |
|---|---|---|
| `pending` | scheduler | Queued, waiting for a worker. |
| `starting` | worker | Reserved and being set up — worktree, container, image. No process yet. |
| `running` | worker | Process is alive. |
| `awaiting` | worker | Shell job waiting for human input. Claude jobs never reach this. |
| `complete` | scheduler | Exited zero. |
| `failed` | worker | Exited non-zero, retries may remain. |
| `error` | scheduler | Failed and out of retries. |
| `deferred` | scheduler | Asked to be retried later. |
| `inactive` | scheduler | Deferred for longer than `defer_max`; given up on. |
| `zombie` | worker | Alive but silent past its timeout — an *unexpected* hang, worth investigating. |
| `killed` | worker | Terminated. Cause is recorded as `by_user`, `by_wall_time`, or `no_process`. |

`zombie` and `inactive` stay separate on purpose: `inactive` is an expected resting state the scheduler chose, `zombie` is something being stuck. Collapsing them would hide the only one that warrants an alarm.

The distinction between `starting` and `running` carries an invariant that matters operationally: **`running` implies a live PID**. A job found in `starting` after a daemon restart is unambiguously an orphan — its setup was interrupted and no process was ever created — which is what lets orphan detection work on PIDs rather than on time heuristics.

### Verbs

Three manual transitions sit above the automatic lifecycle:

| Verb | Effect |
|---|---|
| `START` | Run now, ignoring the trigger. |
| `RESET` | Back to `pending`, rescheduled normally. |
| `KILL` | Terminate a running job (`killed`, cause `by_user`). |

For apps, `START` and `RESET` both mean "restart" and `KILL` means "stop" — there is no separate app lifecycle and no `stop` verb.

**`RESET` also wipes the job's data directory** (`~/.local/share/bibi/*/<job_id>/`), `START` never does. That is the practical difference between the two: an app that keeps session state there survives KILL→START and loses it on RESET. Note that collectors written before this convention write flat into the subsystem folder rather than into a `<job_id>` subfolder, so RESET cleans up nothing for them — there is no migration pressure, but do not assume RESET gives you a clean slate.

RESET respects the trigger rather than firing blindly: a recurring job gets its next cron tick, everything else (`never`, `on_demand`, `startup`, `at:`) simply becomes un-due until someone presses START.

## Where jobs run

| | Scheduled fire | Local `/run` |
|---|---|---|
| Checkout | fresh worktree from `trunk` | the live checkout, in place |
| Uncommitted edits | invisible | visible |
| Gitignored state | wiped first | left alone |
| Allowed on | `worker` nodes | **client nodes only** |

The consequence is easy to trip over: a collector tested with `/run` sees leftover state from the previous local run that a scheduled fire never would. Do not infer steady-state behaviour from a `/run` alone.

`/run` is refused on `scheduler` and `worker` nodes because it writes into a shared checkout that the synchronizer is concurrently pulling and merging. On such a node, use `bibi-ctrl job start <id>`.

It needs **no daemon and no scheduler** — it calls the worker in-process and polls the job row until it reaches a terminal state. That is what makes a team with no host workable at all: every job can still be run by hand from a client, and what you lose is the schedule, not the ability to execute. One difference follows directly from having no daemon: `/run` always uses `attempts=0`, whatever the MD says. A retry would have to be picked up by the pinned worker loop that only a running daemon provides, so a queued one would wait forever.

## Checking your work

```bash
bibi-ctrl rescan          # re-parse the vault; reports parse errors per file
bibi-ctrl job list        # what the scheduler currently knows
bibi-ctrl doctor          # repo hygiene: conventions, git-lfs, committed collected data
```

A job MD that reports an error is not scheduled at all — it is not partially working. Check `rescan` after writing one.
