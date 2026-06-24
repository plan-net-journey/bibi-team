# Claude Code — bibi-team

Dies ist ein **Team-Repo** der bibi-Engine: es enthält die Team-Daten (`vault/`)
und die installierten `.claude`-Bausteine (Skills, Agents, Commands) — **nicht**
den Engine-Code. Die Engine `bibi` ist als Abhängigkeit deklariert
(`pyproject.toml`) und stellt `bibi-ctrl` bereit.

## Struktur (DESIGN §3.2)

- `.claude/skills|agents|commands/` — via `/library use` installierte Bausteine
  (Quelle: das `bibi`-Repo u. a.). Leer im Skelett.
- `.claude/.state.md` — repo-globaler Laufzeit-State, **gitignored**. Nie direkt
  editieren — `bibi-ctrl` verwaltet ihn.
- `vault/case/` — Cases; einziger der Engine bekannte Vault-Ordner (Scheduler
  parst Schedule-/At-MDs darin). Name per `case_dir:` konfigurierbar.
- `data/` — Runtime-State (SQLite, Worktrees, Job-Logs), **gitignored**.

## Setup

Siehe [`INSTALL.md`](../INSTALL.md). Kurz: Repo klonen → `bibi`-Engine
installieren → `bibi-ctrl init`.

## Konventionen

Die maßgeblichen Bibi4-Konventionen und das Design leben (vorerst) im
Engine-/Design-Kontext des Vorhabens. Permanente, für alle geltende Regeln werden
committet hier ergänzt, sobald die ersten Skills migriert sind (Phase 1).
