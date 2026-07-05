# Topology (bibi-team repo)

This file answers "where does what run" for *this specific team's* deployment — repos, hosts, daemon roles, and current live instances — so a fresh Claude Code session does not have to re-derive it from README fragments and guesswork every time.

Unlike `CONVENTIONS.md`, this is **not a generic pattern to copy verbatim** — every team's actual repos, hosts, and daemon placement differ. Fill in the sections below for your own deployment and keep it current; verify against reality (`launchctl print …`, `ssh <host> systemctl status …`) before trusting an entry that looks stale — this file drifts, live state does not.

## Repos

- List the repos this team actually uses and where they live (engine, blueprint-derived team repo, shared skill library, …) and their default branches.

## Hosts

- List the machines/hosts involved (dev machine, always-on deployment host, …), how they're reached (network, SSH), and what each one is responsible for.

## Live instances

- For each host: which daemon role(s) run there (synchronizer/scheduler/worker/client — see DESIGN.md §4.2), on which port, managed by what (launchd/systemd/…), and when it was last verified live (not just assumed from docs).

## Daemon role model (reference)

See the engine's `DESIGN.md` §4.2 for the authoritative model — don't duplicate it here, just note which roles are actually in use by this team.

## Open questions

- Anything about this team's topology that's still unresolved, ambiguous, or a known gap.
