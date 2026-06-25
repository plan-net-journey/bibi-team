# Setup — bibi-team

Menschenlesbare Setup-Checkliste. Keine Credentials im Repo (DESIGN §4.10).

---

## Für Entwickler (Standard-Setup)

Skills sind committed — nach dem Clone sofort verfügbar, kein the-library nötig.

### Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- [`gh`](https://cli.github.com/) installiert und eingeloggt (`gh auth login`)

### Schritte

**1. Repo klonen**

```bash
gh repo clone plan-net-journey/bibi-team
cd bibi-team
```

**2. Engine installieren**

```bash
uv venv
uv pip install .
source .venv/bin/activate
```

Für lokale Engine-Entwicklung stattdessen:

```bash
uv pip install -e ../bibi   # editierbar gegen lokalen bibi-Klon
```

**3. Knoten bootstrappen**

```bash
bibi-ctrl init
```

Fragt interaktiv ab und schreibt `~/.config/bibi/env` (außerhalb des Repos,
nie versioniert):

| Parameter | Beispiel | Bedeutung |
|---|---|---|
| `BIBI_SCHEDULER_URL` | `http://sarasate:8769` | wohin `--connect` zeigt |
| `BIBI_ROLE` | `worker,synchronizer` | kombinierte Rollen dieses Knotens |
| `BIBI_REMOTE` | `https://github.com/plan-net-journey/bibi-team.git` | Git-Remote für Synchronizer |

**4. Verify**

```bash
bibi-ctrl status
```

Starte Claude Code im Repo-Verzeichnis — `/open`, `/save` etc. erscheinen
als Befehle (Skills sind in `.claude/skills/` committed).

**5. (Optional) Daemon-Rollen installieren**

```bash
bibi-ctrl daemon install
```

Welche Rollen je Knotentyp sinnvoll sind, ergänzt das Team sobald die
Laufzeit steht (spätere Phasen).

---

## Für den Skills-Maintainer

Der Maintainer verwaltet das Plugin-System: er installiert neue Skills und
führt Upgrades durch. Andere Entwickler erhalten Änderungen via `git pull`.

### Modell: Vendoring + the-library

```
library.yaml          ← Manifest (wie package.json) — committed
.claude/skills/       ← Vendored Skills (committed, nicht gitignored)
the-library           ← Werkzeug für Install/Upgrade (nur für Maintainer)
```

The-library liest `library.yaml`, zieht Skill-Dateien aus dem Engine-Repo
und legt sie in `.claude/skills/`. Der Maintainer prüft den Diff und committet.

### The-library einrichten (einmalig)

**1. Privaten Mirror anlegen** — *kein* Fork: ein Fork von `disler/the-library`
(public) wäre auf GitHub zwangsläufig public. Stattdessen ein eigenes privates
Repo, das den Upstream spiegelt:

```bash
gh repo create plan-net-journey/the-library --private
git clone --bare https://github.com/disler/the-library.git /tmp/the-library.git
git -C /tmp/the-library.git push --mirror https://github.com/plan-net-journey/the-library.git
rm -rf /tmp/the-library.git
```

**2. In globales Skills-Verzeichnis klonen** (`upstream` für spätere Updates):

```bash
gh repo clone plan-net-journey/the-library ~/.claude/skills/library
git -C ~/.claude/skills/library remote add upstream https://github.com/disler/the-library.git
```

`/library` ist damit in jeder Claude-Code-Session global verfügbar.

**3. Repo-URL eintragen**

In `~/.claude/skills/library/SKILL.md` den `## Variables`-Abschnitt anpassen:

```markdown
- **LIBRARY_REPO_URL**: `https://github.com/plan-net-journey/the-library`
```

`LIBRARY_YAML_PATH` und `LIBRARY_SKILL_DIR` bleiben unverändert.

**4. Verify**

Neue Claude-Code-Session starten → `/library list` zeigt den leeren Katalog.

### Skills installieren oder upgraden

**Initialer Install** (nach Einrichten von the-library):

```
/library use open
/library use save
/library use close
/library use done
/library use delete
/library use protocol
/library use sync
/library use state
```

> `plan-net-journey/bibi` ist privat — the-library nutzt automatisch die
> `gh`-Credentials.

> **Naming (Option A):** Die Engine-Quellordner sind gruppiert (`skills/case-*`,
> `skills/bibi-*`), der Katalog-`name:` ist aber **bare**. the-library installiert
> nach `.claude/skills/<name>/`, und CC leitet den Slash-Befehl aus dem
> Installationsordner ab → Quelle `case-open` ⇒ `name: open` ⇒ Befehl `/open`.
> (`status` heißt `state`, da `/status` ein CC-Builtin ist.)

**Upgrade** (nach Engine-Update):

```
/library sync
```

Danach im Terminal:

```bash
git diff .claude/skills/        # Änderungen prüfen
git add .claude/skills/
git commit -m "upgrade: bibi skills <version/commit>"
git push origin trunk
```

**Neuen Skill hinzufügen:**

1. Eintrag in `library.yaml` ergänzen (Source-URL auf neue SKILL.md)
2. `/library use <name>` aufrufen
3. Commit + Push

### library.yaml aktuell halten

Nach jedem Install/Upgrade den Kommentar in `library.yaml` aktualisieren:

```yaml
# Aktuell installiert von: plan-net-journey/bibi@<commit> (dev)
```

> Branch: vor dem Release wird auf `dev` entwickelt (gruppierte `skills/case-*`-
> Struktur) — `source:` zeigt entsprechend auf `@dev`. Nach `dev → master` auf
> `@master` heben.

So ist immer nachvollziehbar, welche Engine-Version die vendored Skills lieferte.
