# Setup — bibi-team

Menschenlesbare Setup-Checkliste. Keine Credentials im Repo (DESIGN §4.10).

---

## Für Entwickler (Standard-Setup)

Skills sind committed — nach dem Clone sofort verfügbar, kein the-library nötig.

### Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- `git` mit hinterlegten **Gitea-Credentials** (credential-helper: `store` auf
  Linux, Keychain auf macOS) — die Repos sind privat.

### Schritte

**1. Instanz-Repo klonen** — das eigene Team-Repo, zuvor per Gitea-Template aus
dem Blueprint erzeugt (DESIGN §4.10 Schritt 0). `INSTANZ` = dein Repo-Name:

```bash
git clone http://sarasate.tail9f9173.ts.net:3000/m.rau/INSTANZ.git
cd INSTANZ
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
| `BIBI_REMOTE` | `http://sarasate…:3000/m.rau/INSTANZ.git` | Git-Remote für Synchronizer |

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

## Für Produktions-/Server-Deploy (z. B. sarasate)

Abweichend vom Entwickler-Setup oben: Engine + Instanz als **eigene,
separate Klone** unter `/srv/` (nicht `~/Project/`), Daemon läuft dauerhaft
über systemd. Reales Vorbild: `bibi-notes` auf sarasate (2026-07-04,
Details/Protokoll in dessen `vault/case/…/Migration.md`).

**1. Verzeichnisse anlegen** (einmalig `sudo` für `/srv/`):

```bash
sudo mkdir -p /srv/bibi /srv/INSTANZ
sudo chown "$(whoami):$(whoami)" /srv/bibi /srv/INSTANZ
```

**2. Klonen als Geschwister-Ordner** (Pflicht für den editierbaren Install
in Schritt 3 — `-e ../bibi` ist relativ):

```bash
git clone -b dev http://sarasate.tail9f9173.ts.net:3000/m.rau/bibi.git /srv/bibi
git clone -b trunk http://sarasate.tail9f9173.ts.net:3000/m.rau/INSTANZ.git /srv/INSTANZ
```

**3. venv + editierbares Install — mit `[daemon]`-Extra:**

```bash
cd /srv/INSTANZ
uv venv
uv pip install -e "../bibi[daemon]"
```

> **Falle:** `uv pip install -e ../bibi` (ohne `[daemon]`) installiert
> `fastapi`/`uvicorn` **nicht** mit — `daemon` ist im Engine-`pyproject.toml`
> ein optionales Extra, kein Basis-Dependency. Ohne es crash-loopt der
> Daemon mit `ModuleNotFoundError: No module named 'uvicorn'`.

**4. Knoten bootstrappen + Daemon installieren** — wie oben, Schritte 3+5
(`bibi-ctrl init`, dann `bibi-ctrl daemon install`).

> **Falle — editierbares Install ist fragil:** jeder `uv run bibi-ctrl …`-
> Aufruf synct das venv gegen die in `pyproject.toml`/`uv.lock` deklarierte
> Git-Abhängigkeit zurück — **auch der `ExecStart` der systemd/launchd-Unit
> bei jedem Neustart.** Der editierbare Install gegen `/srv/bibi` wird dabei
> durch einen regulären Install ersetzt — kein echtes „editable" mehr, sobald
> der Daemon einmal neu gestartet hat. Für alle direkten CLI-Aufrufe daher
> `.venv/bin/bibi-ctrl` verwenden, nicht `uv run bibi-ctrl` — Letzteres
> reproduziert die Falle bei jedem Aufruf.
>
> **Korrektur (2026-07-05, empirisch widerlegt):** der Re-Sync installiert
> **nicht** automatisch den aktuellen `origin/dev`-Commit, sondern exakt den in
> `uv.lock` **eingefrorenen** Commit-Hash (`bibi.git?rev=dev#<sha>`) — `uv`
> löst den Branch beim Lockfile-Erstellen einmalig auf ein festes SHA auf und
> rührt es danach nicht mehr an. Ein Engine-Fix auf `dev` kommt also **nicht**
> von selbst an, nur weil man den Daemon neu startet. Deploy-Reihenfolge für
> einen Engine-Fix:
> 1. Fix in `bibi` auf `dev` committen + pushen.
> 2. Im Instanz-Repo (`bibi-notes`/`INSTANZ`) `uv lock --upgrade-package bibi`
>    ausführen — hebt den gepinnten Commit im `uv.lock` an.
> 3. `uv.lock`-Änderung committen + pushen, dort ankommen lassen (Pull/Sync).
> 4. Erst dann bringt ein Daemon-Neustart (`systemctl restart …`) den neuen
>    Engine-Commit tatsächlich ins laufende venv.

> **Falle — Snap-`uv` bricht die systemd-Unit:** `bibi-ctrl daemon install`
> meidet bewusst `/snap/bin/uv` (Snap-Sandbox unverträglich mit systemd) und
> bevorzugt `~/.local/bin/uv` (astral-Standalone-Installer). Falls nur
> Snap-`uv` vorhanden ist, zuerst nachinstallieren:
> `curl -LsSf https://astral.sh/uv/install.sh | sh`.

> **Nicht vergessen — `auto_sync on` setzen:** ein unbeaufsichtigter Host hat
> niemanden, der einem Push zustimmen könnte; bleibt `auto_sync` auf `off`
> (Default), pusht der Synchronizer nie automatisch, während er weiterhin
> unauffällig pullt. Lokale Job-Run-Commits (Collector-/Digest-Läufe,
> `agent/*`-Mergebacks) akkumulieren dann unbegrenzt, bis irgendwann von
> woanders gepusht wird — dann trifft ein großer Rückstand auf einen frischen
> Fremd-Push, was Divergenz-/Sync-Konflikte provoziert (real passiert,
> 2026-07-05, Details in `bibi-notes`' `Migration.md`). Nach
> `bibi-ctrl daemon install` daher immer auch:
> ```bash
> .venv/bin/bibi-ctrl sync on
> ```

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

> **Noch offen (Track C):** the-library parst in `SKILL.md` bislang nur
> **GitHub**-Quell-URLs und klont `github.com/…`. Damit `/library use|sync` gegen
> Gitea funktioniert, müssen die „Source Format"-/„Source Parsing"-Regeln um das
> Gitea-Schema erweitert werden (`…/m.rau/<repo>/raw/branch/<branch>/<pfad>`,
> Klon-URL `…/m.rau/<repo>.git`) **oder** die Katalog-`source:` auf lokale Pfade
> umgestellt werden. Bis dahin reicht das hier eingerichtete Vendoring (Skills
> sind committed), `/library`-Upgrades sind erst nach Track C nutzbar.

### The-library einrichten (einmalig)

**1. Privaten Mirror anlegen** — *kein* Fork: ein Fork des public
`disler/the-library` wäre zwangsläufig public. Stattdessen ein eigenes privates
**Gitea**-Repo, das den Upstream spiegelt (bereits angelegt als `m.rau/the-library`):

```bash
# Gitea-Repo anlegen (falls noch nicht vorhanden):
curl -X POST -u USER:TOKEN -H 'Content-Type: application/json' \
  -d '{"name":"the-library","private":true}' \
  http://sarasate.tail9f9173.ts.net:3000/api/v1/user/repos
# Upstream hineinspiegeln:
git clone --bare https://github.com/disler/the-library.git /tmp/the-library.git
git -C /tmp/the-library.git push --mirror http://sarasate.tail9f9173.ts.net:3000/m.rau/the-library.git
rm -rf /tmp/the-library.git
```

**2. In globales Skills-Verzeichnis klonen:**

```bash
git clone http://sarasate.tail9f9173.ts.net:3000/m.rau/the-library.git ~/.claude/skills/library
```

`/library` ist damit in jeder Claude-Code-Session global verfügbar. (Ein
`upstream`-Remote auf das public `disler/the-library` für spätere Tool-Updates
ist optional und der einzige verbleibende GitHub-Berührungspunkt.)

**3. Repo-URL eintragen**

In `~/.claude/skills/library/SKILL.md` den `## Variables`-Abschnitt anpassen:

```markdown
- **LIBRARY_REPO_URL**: `http://sarasate.tail9f9173.ts.net:3000/m.rau/the-library`
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

> `m.rau/bibi` ist privat — the-library nutzt die im git-credential-helper
> hinterlegten Gitea-Credentials (siehe Track-C-Hinweis oben).

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
# Aktuell installiert von: m.rau/bibi@<commit> (dev)
```

> Branch: vor dem Release wird auf `dev` entwickelt (gruppierte `skills/case-*`-
> Struktur) — `source:` zeigt entsprechend auf `@dev`. Nach `dev → master` auf
> `@master` heben.

So ist immer nachvollziehbar, welche Engine-Version die vendored Skills lieferte.
