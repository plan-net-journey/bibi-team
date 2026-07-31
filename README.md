# `<Team name>`

> **Replace this block when you instantiate the template — it is the only part of this README that belongs to your team.** Everything below the horizontal rule is inherited from `bibi-team` and holds for every team. If something there is wrong or unclear, fix it in the blueprint so every team gets the fix, rather than patching this copy.
>
> **What this team does:** *one paragraph — the domain, who works here, what the jobs produce. Or a link to the case that explains it.*

---

A team repository that is **just Markdown in git** — plus an optional runtime that can execute parts of it on a schedule.

You can use this repo with nothing installed but a text editor: write notes, keep them in `vault/`, commit them. Everything beyond that is opt-in. Install the `bibi` engine and you get slash commands for the document lifecycle, and the ability to run a job on the spot. Start its daemon and the repository gains a runtime — jobs that fire on a schedule (shell commands or AI prompts, written as Markdown files in the vault), a web UI, live output. Connect a second machine and the two share one job queue.

Nothing here is a database. A case is a folder, a job is a file, state is frontmatter. That is the whole design premise: if the runtime disappears, the repository is still readable and still yours.

## What is in this repo

Three places matter. [`vault/`](vault/) holds the documents and is the actual content. [`.claude/`](.claude/) holds the agent side: the vendored slash commands, the personas, and the rules the AI reads. `data/` is gitignored runtime state, as is `vault/case/*/data/` where collected data accumulates — neither ever belongs in a commit.

The full layout is written down in [`.claude/CLAUDE.md`](.claude/CLAUDE.md) and, for the vault side, in [`vault/CONVENTIONS.md`](vault/CONVENTIONS.md#top-level-folders). It is not repeated here, so the two cannot drift apart.

## Three ways to use it, each optional

| Level | What it takes | What you get |
|---|---|---|
| **Repository** | a clone and an editor | Markdown notes in git. Cases, memos, conventions. |
| **Engine** | install `bibi` — gives you `bibi-ctrl` | The slash commands: `/open`, `/save`, `/close`, `/done`, `/sync`, `/state`, and `/run` to execute a job on the spot. Ordinary CLI calls. Nothing runs in the background. |
| **Runtime** | *start* the daemon — same install, no extra package | Depends entirely on which roles you start it with. See below. |

Each level builds on the one above and none of them is required. Stopping after the engine is a perfectly reasonable place to be: you get every slash command, you sync by hand, and nothing runs while you sleep.

**What the daemon adds is narrower than it sounds**, because "start the daemon and you get jobs" is wrong on most machines. The slash commands never need it. On a laptop it syncs in the background, serves the web UI, and executes nothing at all — the scheduled jobs run on the one machine that is set up to run them. Which machine that is, is the next section.

## Host and client

Two kinds of machine, and for most teams that is the whole story:

- **A host** runs the jobs. Usually a server that stays on. A team has at most one.
- **Everything else is a client.** Your laptop. It does not run the team's scheduled jobs — it writes the documents that define them, and watches what the host is doing.

As a client you can work **git-only** — a clone, an editor, `git push`, and the host picks your changes up on its next pull — or **inside the bibi environment**, with the engine installed and a daemon running: slash commands, background sync, the web UI, live job output, and `/run`. Both are normal. Only the host needs the full setup.

A team **with no host at all** is a legitimate arrangement, not a broken one: `bibi-ctrl run` executes a job in-process on a client, with no daemon and no scheduler involved. What you give up is precisely two things — jobs firing on a schedule, and jobs being distributed across machines. Everything else works from the first day, which makes "clients only" a sound way to start and add a host later.

`/run` is client-only on purpose: it runs a job in place against your live checkout, which is handy on a laptop and unsafe on the host, where the synchronizer is pulling into that same checkout. On the host you use `bibi-ctrl job start` instead.

Underneath, *host* and *client* are not types but combinations of five roles — `synchronizer`, `scheduler`, `worker`, `controller`, and the `connect` modifier — chosen per machine in `~/.config/bibi/env`, never in the repo. On a host the scheduler and the worker sit together, which is the normal arrangement; separating them across machines is supported but advanced. The values to set are in [`INSTALL.md`](INSTALL.md).

Credentials travel host → client on their own: anything named `BIBI_JOB_ENV_*` on the host rides the heartbeat out to every approved node, so a secret is declared once and never copied by hand. It is deliberately unscoped — every job on every approved node can read it — so weigh a write-scoped token against that before adding one. Details in [`vault/CONVENTIONS.md`](vault/CONVENTIONS.md).

## job, app, claude

There is **one** execution type. A job is a Markdown file whose frontmatter carries a trigger (*when*) and a payload (*what*):

```yaml
---
schedule: "0 9 * * *"
job: python collect.py
---
```

The words *app* and *claude* are not separate types — they are **patterns over that one type**, and the engine distinguishes them only by looking at the payload and one optional field:

| You call it | What actually makes it that | How it differs in practice |
|---|---|---|
| **job** | the default — payload is a shell command | Runs, produces output, exits. |
| **claude** | payload starts with `claude:` | The rest of the value is a prompt; the worker runs Claude Code with it. `model:`, `soul:` and `session:` apply. Never enters `awaiting`. |
| **app** | `app_port:` is set | Long-running server, typically started at daemon boot. Gets a 48h silence timeout instead of 2h, can request human input, and is reachable in the browser. |

This is why there is no `app:` or `claude:` frontmatter key: one payload key, `job:`, and the shape of its value decides the rest. The full field list, all defaults, and copy-paste patterns for each are in **[`JOBS.md`](JOBS.md)**.

## The three repos, and the-library

| Repo | What it is | You touch it when |
|---|---|---|
| **`bibi`** | The engine. Ships `bibi-ctrl`, the daemon, and the canonical `SKILL.md` sources. A Python dependency, not a checkout you need. | You change engine behaviour. |
| **`bibi-team`** | The skeleton this repo came from. A template repository: conventions, `.claude/` building blocks, empty vault. | You improve something *every* team should inherit. |
| **this repo** | An instance created from that template. Your team's actual documents and jobs. | Daily work. |

The split exists because the three change at different speeds and for different reasons. Engine code is versioned and deployed; a team's vault is edited hourly and is private; the conventions in between should improve once and propagate to everyone. Before changing anything in this ecosystem, ask which of the three it belongs to — a fix in an instance that should have been a blueprint fix is a fix every future team will miss.

**the-library** is none of these. It is an external, general-purpose meta-skill for distributing agentics (skills, agents, prompts) privately across repos and machines — not bibi-specific, and not a dependency. bibi uses it for exactly one job: pulling `SKILL.md` files out of the engine repo into this repo's `.claude/skills/`, listed in `library.yaml`.

The model is **vendoring**: skills are committed here, not fetched at runtime. Clone this repo and the slash commands work immediately, with no the-library installed and no network. It is a maintainer's tool for the initial install and later upgrades (`/library use`, `/library sync`), nothing more. That also means a vendored skill can silently go stale — when upgrading, date the engine's `SKILL.md` against the last behaviour change of the command it describes, not against the copy sitting here.

## Where to go next

| Question | Document |
|---|---|
| How do I set up a node? | [`INSTALL.md`](INSTALL.md) |
| How do I write a job? What does `defer_max` do? | [`JOBS.md`](JOBS.md) |
| How do I write in the vault? What is a case vs. a memo? | [`vault/CONVENTIONS.md`](vault/CONVENTIONS.md) |
| Where does this team actually run? | [`vault/TOPOLOGIE.md`](vault/TOPOLOGIE.md) |
| What are the rules for the AI in this repo? | [`.claude/CLAUDE.md`](.claude/CLAUDE.md) |
| Why is it built this way? | `DESIGN.md` — the engine design, kept in the project vault |

Verify before trusting: `TOPOLOGIE.md` describes live infrastructure and can go stale, the runtime cannot. `bibi-ctrl doctor` checks this repo for the conventions that are machine-checkable.
