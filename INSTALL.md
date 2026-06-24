# Setup — bibi-team

Menschenlesbare Setup-Checkliste für einen neuen Knoten. Keine Credentials im
Repo — nur der Ablauf (DESIGN §4.10).

## Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- Git-Zugang zum Team-Repo und zur `bibi`-Engine (beide privat auf GitHub-Org
  `plan-net-journey`; Auth z. B. via `gh auth login` oder Credential-Helper)

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

4. **(Optional) Daemon-Rollen installieren**

   ```bash
   bibi-ctrl daemon install
   ```

   Liest `~/.config/bibi/env` als Env-Quelle. Welche Rollen je Knotentyp sinnvoll
   sind, ergänzt das Team hier, sobald die Laufzeit steht (spätere Phasen).
