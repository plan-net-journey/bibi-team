# Vault conventions (bibi-team repo)

The `vault/` folder is the home of all documents. These conventions apply equally to human users and the AI, and to **every** bibi-team repo — the skeleton and every instance derived from it. A bibi-team repo without a `CONVENTIONS.md` is not conformant.

## Language (audience-based)

One language per document, chosen by audience:

- **English** — everything contributor-/model-facing: code (identifiers, docstrings, comments), the steering/reference docs (`CLAUDE.md`, `INSTALL.md`, this file), and the skill/command instruction files (`SKILL.md`, `commands/*.md`). These are instructions *to* the model and reference *for* contributors.
- **German** — end-user runtime text: CLI `print`/`help` strings, commit messages, and the `vault/` work products (case READMEs, memos, job output). These are narrated *for the user* during use.

Identifiers stay English regardless of the document they appear in (branch names, vault folder slugs, command flags, frontmatter keys, status values) — they are names, not prose. A user-facing **confirmation** the model speaks follows the runtime rule and is German, even inside an otherwise-English instruction file — the split is by audience, not by file.

## Markdown style

Write each paragraph or bullet as **one physical line**, however long — never hard-wrap flowing prose across multiple lines the way traditional 80-column text wrapping does; let the editor/viewer soft-wrap instead. Separate list items, table rows, blockquote lines, and code fences legitimately take one line each; a single paragraph or a single bullet never spans several. This keeps diffs minimal (editing one clause doesn't re-flow the whole paragraph) and matches how the `==name:==` annotation dialogues in this vault are written (see below).

Never write a bare `<placeholder>`-style angle-bracket tag in flowing text — outside a fenced/inline code span, Obsidian's permissive inline-HTML parsing treats `<cutoff>` as an opening HTML tag it never finds a closing match for, and renders the rest of the block wrong. Wrap it in backticks (`` `<cutoff>` ``) or a fenced code block instead. This is unrelated to real HTML being invalid — a `<script>`-looking fragment is exactly as risky as an obviously-not-a-tag placeholder word; Obsidian does not distinguish the two.

`bibi-ctrl doctor` checks both rules across the entire vault (`markdown-hardwrap`, `html-placeholder-tag`) and exits non-zero on findings, which makes it usable as a pre-commit/CI gate. Treat it as a net, not as a substitute for getting it right while writing.

### `==name:==` annotations and the signature file

Discussion inside a vault document happens **inline**, not in a separate thread: an annotator prefixes their remark with a highlighted `==name:==` marker (Obsidian's `==highlight==` syntax) and writes it exactly where it belongs. Several people — and the AI — answer each other in the same run of text, which is the practical reason a paragraph must stay one physical line: a hard-wrapped paragraph turns every inserted remark into a re-flow of the whole block, and the diff stops showing who said what.

Each person keeps their own marker in a small snippet file. `vault/etc/templates/sign.template.md` is committed and carries the placeholder form; every user copies it once to `vault/etc/templates/sign.md` and writes their own name into it. That personal instance is **gitignored** — it differs per user and per node — which is what the `.gitignore` pair `vault/etc/templates/*` plus the negation `!vault/etc/templates/*.template.md` implements. Setting it up is a one-time step per user, alongside `bibi-ctrl init` (see `INSTALL.md`).

## Top-level folders

The default vault layout is three folders plus this file:

```
vault/
├── case/            larger activities — each owns a subdirectory of files
├── memo/            single self-contained notes — one file each
├── etc/             config & auxiliary material (e.g. etc/Claude/ memory)
└── CONVENTIONS.md   ← this file
```

Only `case/` is known to the engine: the scheduler walks it for schedule/`at:` MDs. Its name is configurable via `[tool.bibi] case_dir` in `pyproject.toml` or the `BIBI_CASE_DIR` env var (default `case`; bibi3-compat: `project`). `memo/` and `etc/` are pure vault conventions — the engine never parses them.

## Vocabulary

- **case** — a *larger activity* that needs its **own subdirectory**: a folder `case/YYYYmmdd.<slug>-<short>/` holding a `README.md` plus any further files, attachments, and sub-files. Carries a lifecycle (`open` → `paused`/`closed`), may host schedule or `at:` MDs (the scheduler fires them) and its own `data/` subfolder. `create_case` always places new cases flat, directly under `case/`; but a case may later be **moved** into subfolders for archiving (e.g. `case/2026/06/20260612.FooBar-deadbeef/`) — `/open`'s substring match searches recursively, so a moved case is still found and reactivated by its slug. This also covers legacy folders predating the `-<short>` suffix convention (`case/2026/20260531.LegacyThing/`, no hash in the name): as long as `README.md` carries a `slug` key, the folder is recognized as a case leaf by its frontmatter instead of by name pattern.
- **memo** — a *single, self-contained note*: **one file** under `memo/`, anchored to a date or a date + theme (a meeting, an event, a topic). Attachments (screenshots, embeddings) may sit alongside in `memo/`, but the memo itself is always one closed file.
- **etc** — configuration and auxiliary material that is neither a case nor a memo (e.g. the Claude memory mirror under `etc/Claude/`, the signature snippet under `etc/templates/`).
- **schedule** — recurring job definition (`schedule:` cron) inside a case.
- **at** — one-shot job definition (`at:` datetime) inside a case.
- **job** — a queued/running execution tracked by the daemon scheduler.
- **soul** — a persona file under `.claude/souls/*.SOUL.md`, selected via `/soul`. The set ships with the repo and is editable per team — there is no hardcoded persona list in the engine.
- **slug / short / status** — case README frontmatter fields (see below).

## The idea behind `case` and `memo`

The split is **folder of many files vs. a single file**:

- A **case is a larger activity that needs room.** It owns a subdirectory under `case/`, because it gathers more than one file: a `README.md`, notes, attachments, scripts, scheduled jobs, a `data/` subfolder. It is bounded in time and carries a lifecycle (`open` → `paused`/`closed`), and it is the engine's unit — the scheduler only ever looks inside `case/`. Think *engagement / investigation / project*.
- A **memo is a single, self-contained note.** One file, no folder, no lifecycle, no hash. It is anchored to a date (a meeting, an event) or a date + theme. Attachments may live next to it in `memo/`, but the note itself is one closed file. Think *meeting note / fact sheet / dated entry*.

Rule of thumb: if it needs more than one file, it is a case; if it is one finished note, it is a memo. Cases often *produce* memos — e.g. a mail-collector case runs a `*/10` schedule that maintains a `memo/202606.Billing.md` ledger.

## Naming convention

- **Date:** always `YYYYmmdd` (8 digits, no separators).
- **Case folder:** `YYYYmmdd.<slug>-<short>/` with `README.md` as the mandatory entry point. `<slug>` is CamelCase (non-alphanumerics stripped); `<short>` is the first 8 hex chars of a uuid4. Matched by `^\d{8}\.(.+)-([0-9a-f]{8})$`.
- **Memo file:** same scheme as a case folder, but for one file and without the `-<short>` suffix: `YYYYmmdd.<Topic>.md` (date + theme) or `YYYYmmdd.md` (date only); `<Topic>` is CamelCase. A periodic/running memo may use the month, `YYYYMM.<Topic>.md` (`202606.Billing.md`). No hash, no folder.
- **No umlauts in paths:** `Ue` for `Ü`, `ae` for `ä`, `oe` for `ö`, `ss` for `ß`. Identifiers and folder slugs are ASCII.

## Bugs and change requests belong in the issue tracker, not the vault

A team that runs an issue tracker keeps **all** bug reports and change requests there — title, analysis, root cause, live findings, resolution. Do not open a bug dossier or a `Backlog.md` inside a case, and do not maintain a central bug list as a vault file: the tracker is the single source of truth for defects and requests, and a second copy in the vault will silently drift out of date. This repo's concrete tracker (URL, labels, templates) is named in `CLAUDE.md`, because it is instance-specific — the rule here is not.

The split is by *kind of content*, not by importance. A **case** remains the right home for an activity that needs room: an investigation, a migration, a project with attachments, scripts and scheduled jobs. A **bug or CR** is a tracked item with a lifecycle someone else may query — it belongs where its state can be filtered, sorted and closed. When a tracked item grows into real work, open a case for the work and link the two: the issue references the case folder, and the case `README.md` carries the issue reference as a **link in its opening line**, directly below the heading — not as a frontmatter field. Write it in the cross-repo form (`owner/repo#42`), so the reference stays unambiguous when read from another repo, resolves as a link in both the tracker and the vault editor, and allows more than one issue without list syntax.

This rule states where new material goes. Historical dossiers already archived inside a case are left where they are — moving finished history serves no one.

## Naming convention

- **Date:** always `YYYYmmdd` (8 digits, no separators).
- **Case folder:** `YYYYmmdd.<slug>-<short>/` with `README.md` as the mandatory entry point. `<slug>` is CamelCase (non-alphanumerics stripped); `<short>` is the first 8 hex chars of a uuid4. Matched by `^\d{8}\.(.+)-([0-9a-f]{8})$`.
- **Memo file:** same scheme as a case folder, but for one file and without the `-<short>` suffix: `YYYYmmdd.<Topic>.md` (date + theme) or `YYYYmmdd.md` (date only); `<Topic>` is CamelCase. A periodic/running memo may use the month, `YYYYMM.<Topic>.md` (`202606.Billing.md`). No hash, no folder.
- **No umlauts in paths:** `Ue` for `Ü`, `ae` for `ä`, `oe` for `ö`, `ss` for `ß`. Identifiers and folder slugs are ASCII.

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

**Schedule / job MD** (a flat MD inside a case dir, parsed by the scheduler). **The block below is an excerpt, not the reference.** It shows the keys needed most often; the parser understands twenty, among them every timeout and retry knob that decides when a hanging job is declared a zombie or how long a deferred one may keep deferring. All of them, with defaults and failure modes, are in [`JOBS.md`](../JOBS.md) — documented once, so the two cannot drift apart:

```yaml
---
schedule: "*/10 * * * *"     # cron | on_demand | never  (recurring)
at: 2026-06-27T18:01:56      # ISO datetime (one-shot; mutually exclusive)
job: node fetch.mjs          # shell command to run …
# job: "claude: summarize today"   # … OR an AI prompt, via the claude: prefix
app_port: 9100               # optional: long-running app / HITL port
exec_mode: container         # host | container
attempts: 3                  # optional: total attempts including the first run (default 1 = one run, no retry; 0 = never starts)
backoff: exponential         # optional retry strategy
docker_args: ["-p", "8780:8780"]   # optional, exec_mode: container only — see WARNING below
---
```

A schedule MD carries either `schedule:` (recurring) or `at:` (one-shot), and exactly one `job:` key as its payload — a shell command, or an AI prompt via the `claude:` prefix inside that same value (`job: "claude: <prompt>"`), never a separate `claude:` key (Unified Job Model, one payload key only).

**⚠️ WARNING — `docker_args`:** a generic, **completely unvalidated** escape hatch — a list of raw strings appended verbatim to the `docker run` invocation (`exec_mode: container` only, no-op otherwise). Nothing in bibi checks what's in it: `docker_args: ["--privileged"]` or `docker_args: ["-v", "/:/host"]` would both be honored exactly as written, silently undermining the arbitrary-UID/fixed-mount assumptions the rest of the container sandbox relies on. This is not a sandboxed mini-language — it is literally "whatever a job author puts here reaches the Docker CLI." Only use it if you understand what the added flags do to container isolation, and treat it with the same care as `sudo` in a job script. Legitimate uses: joining an additional Docker network (`["--network", "<network>"]`) or publishing a port beyond `app_port` (`["-p", "8780:8780"]`).

**Memo files** carry **no required frontmatter** — they are plain documents.

## External job data & secrets

A scheduled job runs in a **fresh git worktree checked out from `trunk` on every fire** (`worktree.prepare()`) — anything gitignored *inside* that worktree, including `vault/case/*/data/` (the data-hygiene rule in `CLAUDE.md`), is wiped before the job even starts. A collector that must accumulate state across fires — a watermark, a growing NDJSON, cached API results — cannot live there; it needs a path outside any worktree entirely, on the host filesystem.

This applies to **scheduler fires**. A local `/run` is different: it executes in place against the live checkout of a Client node, so it sees uncommitted edits and nothing is wiped beforehand — the same collector will therefore find leftover state from the previous local run that a scheduled fire would not. Don't infer a job's steady-state behaviour from a `/run` alone.

Convention: two XDG-style roots under the job-owning host's real home directory, both overridable per script via an env var.

| What | Default path | Override (example) |
|---|---|---|
| Secrets (OAuth tokens, API keys) | `~/.config/bibi-<name>/` | `BIBI_<NAME>_HOME` |
| Data (growing/cached state) | `~/.local/share/bibi/<subsystem>/` | `BIBI_<SUBSYSTEM>_DATA` |

`exec_mode: host` sees both paths directly — same OS user, real filesystem, nothing engine-specific needed. `exec_mode: container` needs help, and the two halves are solved differently:

- **Data** is mounted generically: `exec_backend.build_exec()` (bibi engine) bind-mounts the whole `~/.local/share/bibi` root into every container job. No per-script or per-job config needed — any script following the convention above just works in either mode.
- **Secrets are *not* mounted per-directory** (that would need one bind-mount per credential set); instead they ride the existing `BIBI_JOB_ENV_<NAME>` mechanism (`<repo>/data/env`, node config, see `INSTALL.md`) — every entry with that prefix is passed, prefix stripped, into **every** container job on the node. A script meant to work in both modes should read its secret from the matching env var first, falling back to the `~/.config/bibi-<name>/` file for interactive/host-only use.

**Team-trust model:** `BIBI_JOB_ENV_*` is deliberately unscoped — every container job on a node sees every credential declared there, not just the ones it needs. This mirrors `exec_mode: host`, where every job already runs as the same OS user with full filesystem access (zero job-to-job isolation there either); the data mount and the env-var mechanism both just extend that same existing trust boundary to container mode rather than inventing a stricter one. Weigh this before wiring up a **write**-scoped credential this way — a read-only scope is lower-stakes than one that can mutate, and deserves a deliberate decision, not a reflexive copy-paste.

**Credentials propagate host → client on their own.** A `BIBI_JOB_ENV_*` entry does not stay on the machine where it was typed: the host attaches its whole set to the heartbeat of every **approved** node, and the client writes it to `<repo>/data/distributed-env` (mode 0600, alongside its own `env`) — `config.distributable_config()` on the host, `config.write_distributed_env()` on the client. **Both paths are repo-local since `v0.7.3`** (`plan-net-journey/bibi#52`); they lived under `~/.config/bibi/` before, one file per *user* while the node registry keys per *repo*, and a second instance on one machine overwrote the first one's identity. That file is now a backup and nothing else — the engine neither reads nor writes it, and it should not be deleted, because it may hold the only copy of a credential. `python -c "import bibi.config as c; print(c.env_path())"` answers the question for any given checkout. Declare a credential **once on the host** instead of copying it onto each node by hand. No daemon restart is involved: the host re-reads the file on every heartbeat and ships a new bundle as soon as the hash over its contents changes. Precedence when a job is started is **distributed < local `env` < process env**, so a node can always override an inherited value locally, and a node reported `pending` or `blocked` never receives a bundle at all.

This widens the trust boundary described above by one step, and that step is easy to miss: a credential placed on the host is reachable by every job on **every approved node**, not only on the machine where it was entered. Weigh a write-scoped credential against that larger blast radius, not against a single host. A credential that is only ever used **interactively** has no business in `BIBI_JOB_ENV_*` at all — keep it in the workstation's own credential store (macOS Keychain, `git credential store` on Linux) and let only what jobs actually need travel.

### Per-job data subfolder and `bibi.job.data_dir()`

Recommended sub-convention for new scripts (since `bibi` dev `5e92f99`): `~/.local/share/bibi/<subsystem>/<job_id>/` instead of loose files sitting directly in the subsystem folder. The `job_id` comes from `BIBI_JOB_ID` — stable across every retry of one run and across the whole lifetime of a recurring job. The sanctioned helper is `bibi.job.data_dir(subsystem: str) -> Path`; don't assemble `Path.home() / ".local" / "share" / "bibi" / ...` by hand any more:

```python
import bibi.job
STATE = bibi.job.data_dir("my-collector") / "counter.txt"
```

The reason this matters: **RESET now deliberately wipes exactly that directory** (`job_db.wipe_job_data()`, a generic glob over `~/.local/share/bibi/*/<job_id>/`, needing no knowledge of individual subsystems) — START never touches it, whatever terminal state the job was in. Verb semantics: RESET = full wipe (bibi's own `job_db` fields *and* this directory), START = as before, preserving both. Uniform for JOB/CLAUDE/APP with no type branching — an app that stores its own session state here can come back up quickly after KILL→START, but loses it on RESET.

**Deliberately not a breaking change for existing collectors:** a script that keeps its files flat in the subsystem folder (`~/.local/share/bibi/<subsystem>/watermark.json` and the like) keeps working untouched — `wipe_job_data()`'s glob matches nothing there, so a RESET of that job cleans up nothing under the old layout. There is no migration pressure; switch an existing script to `bibi.job.data_dir()` deliberately, when you want the new RESET semantics for it.

## Job scripts: `uv run --script`, never `uv run python`

Applies to **any** `job:` payload that runs a Python script this way — not just collectors, and not only scripts with external dependencies. `uv run <file>.py` (no `--script`) runs in **project mode**: it resolves dependencies by walking **upward from the job's cwd looking for a `pyproject.toml`**, same as running that command by hand anywhere in the repo — and, critically, project mode syncs the *whole* project's declared dependencies before running anything, regardless of what the invoked script itself actually imports. Even a pure-stdlib script with zero real dependencies pays this cost.

In a job's worktree/container that upward search reliably finds the *team repo's own root* `pyproject.toml` — there usually isn't a closer one to stop the walk. This team repo's root `pyproject.toml` declares exactly one dependency: the `bibi` engine itself, pinned via a **private, self-hosted git remote** (`bibi[daemon] @ git+http://…/bibi.git@dev`).

On the host this is invisible — a developer's shell already has that remote authenticated (cached credential helper, prior clone) — so the script appears to work. Inside `exec_mode: container` there is no such credential (fresh filesystem, no `~/.git-credentials`, no keychain access), so `uv` tries to `git fetch` the private remote to resolve the project environment and fails outright: `fatal: could not read Username for '…': terminal prompts disabled`. The failure has nothing to do with the script's own imports (whatever they are, or nothing at all) — it never gets that far.

**Fix, and the rule going forward:** every job script gets its own PEP 723 header declaring exactly the dependencies it needs (an empty `dependencies = []` is fine for a stdlib-only script), and is invoked with `uv run --script <file>.py` — this resolves an isolated, ephemeral environment from the header alone, in **script mode**, which never walks upward for a project file at all. The team repo's own `bibi` dependency (and the private-remote auth it needs) never enters the picture:

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.9"
# dependencies = ["pandas>=2.0", "yfinance>=0.2"]
# ///
```

**Any** `job:` that shells out to `uv run python` on a `.py` file lacking this header is a latent container-mode failure waiting to happen, even if it currently only ever runs in `exec_mode: host` and even if it imports nothing but the stdlib — check for it the same way you'd check for the external-data convention above.

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
| `/job-create` | Interactive wizard for a new schedule: location, trigger, type, payload | schedule MD in a case dir |
| `/job-doctor` | Consistency diagnosis over `rescan`/`mergeback`/`status`/`doctor` | none (read-only) |
| `/run <slug>` | Run a job once locally now, in place, bypassing the scheduler | journal stays local |
| `/protocol on\|off\|debug` | Toggle per-turn logging in the active case | frontmatter `protocol:` |
| `/sync [on\|off]` | Toggle auto-push, or run a manual pull/push | `auto_sync` in state |
| `/state` | Show active case, `auto_sync`, `sync_conflict`, protocol | none (read-only) |
| `/soul [name]` | Show or switch the active persona (`.claude/souls/*.SOUL.md`) | none |
| `/bibi-setup` | Interview-guided onboarding of a node: init, daemon, web UI | none |

**Active case = parked case:** `/open` writes a **park marker** for the session (`data/park/<session_id>`) *and* `cd`s the shell into the case folder. `bibi-ctrl` resolves the active case from the Bash cwd first — an explicit `cd` into a case is a deliberate gesture and wins — and falls back to the marker otherwise, so the case survives parallel Bash calls overwriting each other's cwd, background shells, hooks/statusline subprocesses, and session restarts. The park marker is the only store for the active case; a `path:` mirror in `.claude/.state.md` existed until m.rau/bibi#99 and is gone — per-session state does not belong in a shared file. One case per session; several can be active in parallel across sessions, each with its own marker. `bibi-ctrl status` prints which source (`cwd` or `session`) the current case came from.

## Branches

- **`trunk`** — working default. All lifecycle commits go here, auto-pushed. The daemon pulls/fires from `trunk`.
- **`master`** — release line. Left untouched; raised from trunk by the maintainer.
