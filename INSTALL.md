# Setup — bibi-team

Menschenlesbare Setup-Checkliste für einen neuen Knoten. Keine Credentials im
Repo — nur der Ablauf (DESIGN §4.10).

## Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- [`gh`](https://cli.github.com/) installiert und eingeloggt (`gh auth login`)
  — wird für private Repos (bibi-Engine + the-library-Skills) benötigt
- Git-Zugang zum Team-Repo und zur `bibi`-Engine (beide privat auf GitHub-Org
  `plan-net-journey`)

## Schritte

### 1. Team-Repo klonen

```bash
gh repo clone plan-net-journey/bibi-team
cd bibi-team
```

### 2. Engine installieren

```bash
uv venv
uv pip install .            # zieht bibi aus dem deklarierten git-Dependency
# ODER für lokale Engine-Entwicklung:
uv pip install -e ../bibi   # editierbar gegen den lokalen bibi-Klon
source .venv/bin/activate
```

### 3. Knoten bootstrappen

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

### 4. The-library einrichten und Skills installieren

The-library ist ein Meta-Skill für private Skill-Distribution. Er wird
**einmalig pro Maschine** global eingerichtet und stellt `/library` in
jeder Claude-Code-Session bereit.

#### 4a. The-library forken und klonen (einmalig)

```bash
gh repo fork disler/the-library --private --clone=false
gh repo clone <dein-github-name>/the-library ~/.claude/skills/library
```

#### 4b. Fork-URL in SKILL.md eintragen (einmalig)

In `~/.claude/skills/library/SKILL.md` den `## Variables`-Abschnitt anpassen:

```markdown
- **LIBRARY_REPO_URL**: `https://github.com/<dein-github-name>/the-library.git`
```

Die anderen beiden Variablen (`LIBRARY_YAML_PATH`, `LIBRARY_SKILL_DIR`) bleiben
unverändert, sofern du nach `~/.claude/skills/library/` geklont hast.

#### 4c. bibi-Skills in den persönlichen Katalog eintragen (einmalig)

Die benötigten Skills sind in `library.yaml` im Team-Repo als Referenzliste
dokumentiert. Diese Einträge in den persönlichen Katalog übertragen:

```bash
cat library.yaml   # Einträge ansehen
```

Den `library:`-Block aus dieser Datei in
`~/.claude/skills/library/library.yaml` einfügen — oder jeden Skill einzeln
via Claude Code registrieren:

```
/library add open skill from https://github.com/plan-net-journey/bibi/blob/master/skills/open/SKILL.md
/library add save skill from https://github.com/plan-net-journey/bibi/blob/master/skills/save/SKILL.md
/library add close skill from https://github.com/plan-net-journey/bibi/blob/master/skills/close/SKILL.md
/library add done skill from https://github.com/plan-net-journey/bibi/blob/master/skills/done/SKILL.md
/library add delete skill from https://github.com/plan-net-journey/bibi/blob/master/skills/delete/SKILL.md
/library add protocol skill from https://github.com/plan-net-journey/bibi/blob/master/skills/protocol/SKILL.md
/library add sync skill from https://github.com/plan-net-journey/bibi/blob/master/skills/sync/SKILL.md
```

> Hinweis: `plan-net-journey/bibi` ist ein privates Repo. The-library nutzt
> automatisch die `gh`-Credentials für den Zugriff.

#### 4d. Skills im Team-Repo installieren (nach jedem Clone oder Engine-Update)

```bash
cd ~/Project/bibi-team   # im Team-Repo-Verzeichnis
/library use open
/library use save
/library use close
/library use done
/library use delete
/library use protocol
/library use sync
```

Die Skills landen in `.claude/skills/` (lokal, gitignored) und sind danach
als `/open`, `/save`, `/close` usw. in Claude Code sichtbar.

> Nach Engine-Updates (`git pull` im bibi-Repo): `/library sync` zieht alle
> installierten Skills auf den neuesten Stand.

### 5. Verify

```bash
bibi-ctrl status          # zeigt path/auto_sync/config
```

Starte Claude Code im Team-Repo — `/open`, `/save` etc. sollten als Befehle
erscheinen.

### 6. (Optional) Daemon-Rollen installieren

```bash
bibi-ctrl daemon install
```

Liest `~/.config/bibi/env` als Env-Quelle. Welche Rollen je Knotentyp sinnvoll
sind, ergänzt das Team hier, sobald die Laufzeit steht (spätere Phasen).
