# Vault conventions (bibi-team repo)

The `vault/` folder is the home of all documents. These conventions apply
equally to human users and the AI, and to **every** bibi-team repo — the
skeleton and every instance derived from it. A bibi-team repo without a
`CONVENTIONS.md` is not conformant.

## Language (audience-based)

One language per document, chosen by audience:

- **English** — everything contributor-/model-facing: code (identifiers,
  docstrings, comments), the steering/reference docs (`CLAUDE.md`, `INSTALL.md`,
  this file), and the skill/command instruction files (`SKILL.md`,
  `commands/*.md`). These are instructions *to* the model and reference *for*
  contributors.
- **German** — end-user runtime text: CLI `print`/`help` strings, commit
  messages, and the `vault/` work products (case READMEs, memos, job output).
  These are narrated *for the user* during use.

Identifiers stay English regardless of the document they appear in (branch
names, vault folder slugs, command flags, frontmatter keys, status values) —
they are names, not prose. A user-facing **confirmation** the model speaks
follows the runtime rule and is German, even inside an otherwise-English
instruction file — the split is by audience, not by file.

## Markdown style

Write each paragraph or bullet as **one physical line**, however long — never
hard-wrap flowing prose across multiple lines the way traditional 80-column
text wrapping does; let the editor/viewer soft-wrap instead. List items, table
rows, blockquote lines, and code fences legitimately break per line; ordinary
flowing prose does not. This keeps diffs minimal (editing one clause doesn't
re-flow the whole paragraph) and matches how the `==name:==`-annotation
dialogues in this vault are written (e.g. `case/*/Routing-Analyse.md`).

## Top-level folders

The default vault layout is three folders plus this file:

```
vault/
├── case/            larger activities — each owns a subdirectory of files
├── memo/            single self-contained notes — one file each
├── etc/             config & auxiliary material (e.g. etc/Claude/ memory)
└── CONVENTIONS.md   ← this file
```

Only `case/` is known to the engine: the scheduler walks it for schedule/`at:`
MDs. Its name is configurable via `[tool.bibi] case_dir` in `pyproject.toml` or
the `BIBI_CASE_DIR` env var (default `case`; bibi3-compat: `project`). `memo/`
and `etc/` are pure vault conventions — the engine never parses them.

## Vocabulary

- **case** — a *larger activity* that needs its **own subdirectory**: a folder
  `case/YYYYmmdd.<slug>-<short>/` holding a `README.md` plus any further files,
  attachments, and sub-files. Carries a lifecycle (`open` → `paused`/`closed`),
  may host schedule or `at:` MDs (the scheduler fires them) and its own `data/`
  subfolder. `create_case` always places new cases flat, directly under
  `case/`; but a case may later be **moved** into subfolders for archiving
  (e.g. `case/2026/06/20260612.FooBar-deadbeef/`) — `/open`'s substring match
  searches recursively, so a moved case is still found and reactivated by its
  slug. This also covers legacy folders predating the `-<short>` suffix
  convention (`case/2026/20260531.LegacyThing/`, no hash in the name): as long
  as `README.md` carries a `slug` key, the folder is recognized as a case leaf
  by its frontmatter instead of by name pattern.
- **memo** — a *single, self-contained note*: **one file** under `memo/`,
  anchored to a date or a date + theme (a meeting, an event, a topic).
  Attachments (screenshots, embeddings) may sit alongside in `memo/`, but the
  memo itself is always one closed file.
- **etc** — configuration and auxiliary material that is neither a case nor a
  memo (e.g. the Claude memory mirror under `etc/Claude/`).
- **schedule** — recurring job definition (`schedule:` cron) inside a case.
- **at** — one-shot job definition (`at:` datetime) inside a case.
- **job** — a queued/running execution tracked by the daemon scheduler.
- **soul** — *(not used in bibi-team repos; bibi3 concept only).*
- **slug / short / status** — case README frontmatter fields (see below).

## The idea behind `case` and `memo`

The split is **folder of many files vs. a single file**:

- A **case is a larger activity that needs room.** It owns a subdirectory under
  `case/`, because it gathers more than one file: a `README.md`, notes,
  attachments, scripts, scheduled jobs, a `data/` subfolder. It is bounded in
  time and carries a lifecycle (`open` → `paused`/`closed`), and it is the
  engine's unit — the scheduler only ever looks inside `case/`. Think
  *engagement / investigation / project*.
- A **memo is a single, self-contained note.** One file, no folder, no
  lifecycle, no hash. It is anchored to a date (a meeting, an event) or a date +
  theme. Attachments may live next to it in `memo/`, but the note itself is one
  closed file. Think *meeting note / fact sheet / dated entry*.

Rule of thumb: if it needs more than one file, it is a case; if it is one
finished note, it is a memo. Cases often *produce* memos — e.g. a
`gmail-collector` case runs a `*/10` schedule that maintains the
`memo/202606.Billing.md` ledger.

## Naming convention

- **Date:** always `YYYYmmdd` (8 digits, no separators).
- **Case folder:** `YYYYmmdd.<slug>-<short>/` with `README.md` as the mandatory
  entry point. `<slug>` is CamelCase (non-alphanumerics stripped); `<short>` is
  the first 8 hex chars of a uuid4. Matched by `^\d{8}\.(.+)-([0-9a-f]{8})$`.
- **Memo file:** same scheme as a case folder, but for one file and without the
  `-<short>` suffix: `YYYYmmdd.<Topic>.md` (date + theme) or `YYYYmmdd.md`
  (date only); `<Topic>` is CamelCase. A periodic/running memo may use the month,
  `YYYYMM.<Topic>.md` (`202606.Billing.md`). No hash, no folder.
- **No umlauts in paths:** `Ue` for `Ü`, `ae` for `ä`, `oe` for `ö`, `ss` for
  `ß`. Identifiers and folder slugs are ASCII.

## Frontmatter schema

**Case README** (`case/<folder>/README.md`):

```yaml
---
slug: Billing                # CamelCase topic, no special characters
short: d7cb3e2b              # 8-char hex (uuid4 prefix), folder suffix
status: open                 # open | paused | closed
created: '2026-06-28'        # ISO date (YYYY-MM-DD)
protocol: ./protocol.json    # optional: set by /protocol on
---
```

| Field | Meaning |
|---|---|
| `slug` | CamelCase topic, no special characters |
| `short` | 8-char hex id (uuid4 prefix), folder suffix |
| `status` | `open` (set by `/open`), `paused` (`/close`), `closed` (`/done`) |
| `created` | ISO date of creation |
| `protocol` | optional `./protocol.json` (or `+debug`); toggled via `/protocol` |

**Schedule / job MD** (a flat MD inside a case dir, parsed by the scheduler):

```yaml
---
schedule: "*/10 * * * *"     # cron | on_demand | never  (recurring)
at: 2026-06-27T18:01:56      # ISO datetime (one-shot; mutually exclusive)
job: node fetch.mjs          # shell command to run …
claude: "summarize today"    # … OR an AI prompt (instead of job:)
app_port: 9100               # optional: long-running app / HITL port
exec_mode: container         # host | container
attempts: 3                  # optional retry count
backoff: exponential         # optional retry strategy
---
```

A schedule MD carries either `schedule:` (recurring) or `at:` (one-shot), and
either `job:` (shell) or `claude:` (AI prompt) as its payload.

**Memo files** carry **no required frontmatter** — they are plain documents.

## External job data & secrets

A scheduled job runs in a **fresh git worktree checked out from `trunk` on every fire** (`worktree.prepare()`) — anything gitignored *inside* that worktree, including `vault/case/*/data/` (the data-hygiene rule in `CLAUDE.md`), is wiped before the job even starts. A collector that must accumulate state across fires — a watermark, a growing NDJSON, cached API results — cannot live there; it needs a path outside any worktree entirely, on the host filesystem.

Convention: two XDG-style roots under the job-owning host's real home directory, both overridable per script via an env var.

| What | Default path | Override (example) |
|---|---|---|
| Secrets (OAuth tokens, API keys) | `~/.config/bibi-<name>/` | `BIBI_<NAME>_HOME` |
| Data (growing/cached state) | `~/.local/share/bibi/<subsystem>/` | `BIBI_<SUBSYSTEM>_DATA` |

`exec_mode: host` sees both paths directly — same OS user, real filesystem, nothing engine-specific needed. `exec_mode: container` needs help, and the two halves are solved differently:

- **Data** is mounted generically: `exec_backend.build_exec()` (bibi engine) bind-mounts the whole `~/.local/share/bibi` root into every container job. No per-script or per-job config needed — any script following the convention above just works in either mode.
- **Secrets are *not* mounted per-directory** (that would need one bind-mount per credential set); instead they ride the existing `BIBI_JOB_ENV_<NAME>` mechanism (`~/.config/bibi/env`, node config, see `INSTALL.md`) — every entry with that prefix is passed, prefix stripped, into **every** container job on the node. A script meant to work in both modes should read its secret from the matching env var first, falling back to the `~/.config/bibi-<name>/` file for interactive/host-only use.

**Team-trust model:** `BIBI_JOB_ENV_*` is deliberately unscoped — every container job on a node sees every credential declared there, not just the ones it needs. This mirrors `exec_mode: host`, where every job already runs as the same OS user with full filesystem access (zero job-to-job isolation there either); the data mount and the env-var mechanism both just extend that same existing trust boundary to container mode rather than inventing a stricter one. Weigh this before wiring up a **write**-scoped credential this way — a read-only scope is lower-stakes than one that can mutate, and deserves a deliberate decision, not a reflexive copy-paste.

## Slash commands

Thin wrappers around `bibi-ctrl`; installed under `.claude/skills/`.

| Command | Effect | Vault effect |
|---|---|---|
| `/open <topic>` | Open a new case or reactivate a matching one | `case/YYYYmmdd.<slug>-<short>/` |
| `/save` | Capture status in the active case README + commit + push | README updated |
| `/close` | Pause note + `status: paused` + commit + push + clear path | frontmatter `status: paused` |
| `/done` | Wrap-up + `status: closed` + commit + push + clear path | frontmatter `status: closed` |
| `/delete` | Remove the active case entirely (destructive; confirm) | folder gone |
| `/at "<when>" "<payload>"` | Write a one-shot `at:` MD + daemon rescan | `at:` MD in the case dir |
| `/job [...]` | List / show / kill / restart / rescan scheduler jobs | none |
| `/run <slug>` | Run a job once locally now, bypassing the scheduler | journal stays local |
| `/protocol on\|off\|debug` | Toggle per-turn logging in the active case | frontmatter `protocol:` |
| `/sync [on\|off]` | Toggle auto-push, or run a manual pull/push | `auto_sync` in state |
| `/state` | Show active case, `auto_sync`, `sync_conflict`, protocol | none (read-only) |

**Active case = parked cwd:** `/open` `cd`s the shell into the case folder; that
Bash-tool cwd *is* the active case (`bibi-ctrl` derives it from `Path.cwd()`).
One per shell; several can be active in parallel across sessions.

## Branches

- **`trunk`** — working default. All lifecycle commits go here, auto-pushed.
  The daemon pulls/fires from `trunk`.
- **`master`** — release line. Left untouched; raised from trunk by the
  maintainer.
