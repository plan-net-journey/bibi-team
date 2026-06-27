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
- `vault/case/<case>/data/` — **gesammelte/rohe Flat-File-Daten** (News, Kurse …),
  **gitignored** (siehe Daten-Hygiene unten).
- `data/` — Runtime-State (SQLite, Worktrees, Job-Logs), **gitignored**.

## Setup

Siehe [`INSTALL.md`](../INSTALL.md). Kurz: Repo klonen → `bibi`-Engine
installieren → `bibi-ctrl init`.

## Konventionen

Die maßgeblichen Bibi4-Konventionen und das Design leben (vorerst) im
Engine-/Design-Kontext des Vorhabens. Permanente, für alle geltende Regeln werden
committet hier ergänzt, sobald die ersten Skills migriert sind (Phase 1).

### Daten-Hygiene im Vault (DESIGN §3.5)

Der Vault trägt **alles** an einem Ort (Text, Binäres, Daten) — bewusst gegen
DB-Schichten. Damit das git-seitig nicht entgleist, zwei Regeln:

- **Binärdateien → git-LFS.** `.gitattributes` (im Repo) lenkt Bilder/PDF/DOC/…
  in LFS. **git-lfs muss auf jedem Knoten installiert sein** (sonst Pointer statt
  Inhalt). Nie ein großes Binär ohne passende LFS-Regel committen.
- **Gesammelte/rohe Daten → NICHT in git.** Wachsende/rotierende Flat-Files (News,
  Kurse) leben unter `vault/case/<case>/data/` — **gitignored**. Committet werden
  nur Code, Config und verdichtete Ergebnisse.

**Prüfen:** `bibi-ctrl doctor` meldet fehlendes git-lfs, große nicht-LFS-Blobs und
committete Sammeldaten (Exit ≠ 0 bei Befunden — pre-commit/CI-tauglich).
