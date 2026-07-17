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

Die für alle geltenden Vault-Konventionen — Sprache, Slash-Commands, Vokabular,
Top-Level-Ordner (`case`/`memo`/`etc`), Naming, Frontmatter und die Idee hinter
`case` und `memo` — stehen committet in
[`vault/CONVENTIONS.md`](../vault/CONVENTIONS.md). **Jedes bibi-team-Repo MUSS
diese Datei führen** — `bibi-ctrl doctor` meldet ihr Fehlen. Das maßgebliche
Engine-Design liegt separat im Projekt-Vault (`DESIGN.md`).

**Beim Schreiben/Editieren jeder `vault/`-Markdown-Datei aktiv gegen `CONVENTIONS.md` § „Markdown style" prüfen** — insbesondere: eine physische Zeile pro Absatz (nie bei ~80 Spalten hart umbrechen) und keine bloßen `<Platzhalter>`-artigen Winkelklammer-Tags außerhalb Code-Fences/Backticks (Obsidian interpretiert sie als offene, nie geschlossene HTML-Tags und rendert dann falsch — `` `<cutoff>` `` statt `<cutoff>` schreiben, oder in einen ```` ``` ````-Block). `bibi-ctrl doctor` prüft beides zusätzlich automatisiert — als Netz, nicht als Ersatz für die Selbstprüfung beim Schreiben selbst.

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

## Topologie

Jedes Team sollte in [`vault/TOPOLOGIE.md`](../vault/TOPOLOGIE.md) festhalten,
wo es tatsächlich läuft — eigene Repos, Hosts, aktive Daemon-Instanzen,
Rollen-Zuordnung. Anders als `CONVENTIONS.md` ist das **kein generisches
Muster**, sondern instanzspezifischer Live-Stand — die Datei hier ist nur ein
Platzhalter-Template zum Ausfüllen, nicht der echte Stand irgendeines Teams.

## Cross-Repo-Abstimmung

Diese Team-Repo-Instanz, das `bibi-team`-Blueprint, aus dem sie hervorging,
und die `bibi`-Engine, von der sie abhängt, entwickeln sich zusammen, leben
aber in getrennten Repos. Vor jeder Änderung in diesem Ökosystem klären:
welchem Repo sie eigentlich gehört (Engine-Code vs. private Instanz-Daten
vs. eine generische Skelett-/Konventions-Verbesserung, die auch andere Teams
bekommen sollten); welche Doku in jedem betroffenen Repo nachgezogen werden
muss (`README.md`, `CONVENTIONS.md`, `INSTALL.md`, `DESIGN.md`, diese Datei);
und ob eine abhängige Umgebung neu installiert/deployt werden muss, um eine
Engine-Änderung zu übernehmen. Eine generische Konventions-/Doku-Verbesserung
in einer Team-Instanz sollte gegen `bibi-team` abgeglichen werden, damit
künftige Teams sie erben — und umgekehrt.
