# Setup — bibi-team

Menschenlesbare Setup-Checkliste für einen neuen Knoten. Keine Credentials im
Repo — nur der Ablauf (DESIGN §4.10).

## Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- [`gh`](https://cli.github.com/) installiert und eingeloggt (`gh auth login`)
- Git-Zugang zum Team-Repo und zur `bibi`-Engine (beide privat auf GitHub-Org
  `plan-net-journey`)

## Schritte

1. **Team-Repo klonen**

   ```bash
   git clone https://github.com/plan-net-journey/bibi-team.git
   cd bibi-team
   ```

2. **Engine installieren**

   ```bash
   uv pip install .            # zieht bibi aus dem deklarierten git-Dependency
   # ODER für lokale Engine-Entwicklung:
   uv pip install -e ../bibi   # editierbar gegen den lokalen bibi-Klon
   ```

3. **Knoten bootstrappen**

   ```bash
   bibi-ctrl init
   ```

   Fragt interaktiv ab und schreibt `~/.config/bibi/env` (außerhalb des Repos,
   nie versioniert):

   | Parameter | Beispiel | Bedeutung |
   |---|---|---|
   | `BIBI_SCHEDULER_URL` | `http://sarasate:8769` | wohin `--connect` zeigt |
   | `BIBI_ROLE` | `worker,synchronizer` | kombinierte Rollen dieses Knotens |
   | `BIBI_REMOTE` | `https://github.com/plan-net-journey/bibi-team.git` | Git-Remote für den Synchronizer |

4. **Skills installieren (the-library)**

   the-library ist ein externer Skill-Katalog, der `/library sync` bereitstellt.
   Er wird **einmalig pro Maschine** global installiert:

   ```bash
   git clone https://github.com/disler/the-library.git ~/.claude/skills/library
   ```

   Danach im Team-Repo die deklarierten Skills aus `library.yaml` ziehen:

   ```bash
   /library sync
   ```

   Die Skills landen in `.claude/skills/` (lokal, gitignored) und sind danach
   in Claude Code als `/open`, `/save`, `/close` usw. verfügbar.

   > Nach Engine-Updates (`git pull` im bibi-Repo) nochmals `/library sync`
   > ausführen, damit die Skill-Dateien aktualisiert werden.

5. **(Optional) Daemon-Rollen installieren**

   ```bash
   bibi-ctrl daemon install
   ```

   Liest `~/.config/bibi/env` als Env-Quelle. Welche Rollen je Knotentyp sinnvoll
   sind, ergänzt das Team hier, sobald die Laufzeit steht (spätere Phasen).
