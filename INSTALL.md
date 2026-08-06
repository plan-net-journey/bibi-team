# Setup — bibi-team

Menschenlesbare Setup-Checkliste. Keine Credentials im Repo (DESIGN §4.10).

---

## Für Entwickler (Standard-Setup)

Skills sind committed — nach dem Clone sofort verfügbar, kein the-library nötig.

### Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- `git` mit hinterlegten **Gitea-Credentials** (credential-helper: `store` auf
  Linux, Keychain auf macOS) — die Repos sind privat.
- **`git-lfs` installiert** (Paket `git-lfs`) — der Vault führt Bilder/PDFs und
  andere Binärdateien über LFS (`.gitattributes`, s. `.claude/CLAUDE.md`
  § Daten-Hygiene). Ohne das Binär kommen sie als Pointer-Textdateien statt als
  Inhalt an.

### Schritte

**1. Instanz-Repo klonen** — das eigene Team-Repo, zuvor per Gitea-Template aus
dem Blueprint erzeugt (DESIGN §4.10 Schritt 0). `INSTANZ` = dein Repo-Name:

```bash
git lfs install          # einmalig pro NUTZER, nicht pro Maschine
git clone https://github.com/plan-net-journey/INSTANZ.git
cd INSTANZ
```

`git lfs install` schreibt die LFS-Filter in die `~/.gitconfig` des **aktuellen
Nutzers**. Ein frisch angelegter Account hat sie also nicht, auch wenn
`git-lfs` systemweit installiert ist — dann landen alle LFS-Dateien als
Pointer-Text im Checkout, und das fällt erst auf, wenn jemand ein Bild öffnet.
Nachträglich heilen: `git lfs install && git lfs pull` (der Filter allein wirkt
nur auf künftige Checkouts).

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

**Vorher die einzige Entscheidung, die dieser Schritt von dir verlangt: Was ist diese Maschine?**

| | `BIBI_ROLE` eingeben | wann |
|---|---|---|
| **Client** ohne Scheduler | `synchronizer,controller` | es gibt (noch) keinen Server. Der Normalfall am Anfang. |
| **Client** an einem Scheduler | `synchronizer,controller` + `--connect <url>` | ein Server läuft, du hängst dich an |
| **Scheduler** | `scheduler,worker,synchronizer` | der Server selbst |

**Zwei Dinge musst du dabei nicht abwägen.** `synchronizer` gehört auf **jeden** Knoten — ohne ihn gleicht sich das Repo nicht ab. Und `controller` ist auf einem Client **immer** dabei: er serviert die Oberfläche, und die ist der Weg, auf dem ein Mensch hier arbeitet. Auf dem Scheduler gehört er **nicht** hin — der ist Backend, und eine zweite Oberfläche wäre ein zweiter Ort, an dem man nachsieht.

**Ohne Scheduler fehlt nichts als zwei Dinge:** zeitgesteuerte Jobs und die Verteilung über mehrere Rechner. Der Case-Zyklus, `bibi-ctrl run`, die Oberfläche und `doctor` laufen ab Tag 1. „Nur Clients" ist ein gültiger Aufbau, kein halber.

**Nach der Scheduler-URL wird nur gefragt, wenn `connect` in den Rollen steht** — ohne Server bleibt das Feld leer, und das ist richtig so.

Fragt interaktiv ab und schreibt `~/.config/bibi/env` (außerhalb des Repos,
nie versioniert):

| Parameter | Beispiel | Bedeutung |
|---|---|---|
| `BIBI_SCHEDULER_URL` | `http://<host>:8780` | wohin `--connect` zeigt (leer lassen, wenn es keinen Server gibt) |
| `BIBI_ROLE` | s. Tabelle oben | kombinierte Rollen dieses Knotens |
| `BIBI_REMOTE` | `https://github.com/<org>/INSTANZ.git` | Git-Remote für Synchronizer |
| `BIBI_STATUS_POLL_INTERVAL` | `30` | Poll-Intervall (Sekunden) der Feed-Status-Kacheln, Default 30 |

**4. Persönliche Signatur anlegen**

```bash
cp vault/etc/templates/sign.template.md vault/etc/templates/sign.md
# darin `enter-your-name` durch den eigenen Namen ersetzen
```

Einmal pro Nutzer, nicht pro Maschine im Team: `sign.md` ist gitignored (jeder Nutzer/Knoten hat eine eigene), committet ist nur das Template. Der Schnipsel liefert die `==name:==`-Marke, mit der Anmerkungen inline in Vault-Dokumenten signiert werden — siehe [`vault/CONVENTIONS.md`](vault/CONVENTIONS.md) § „`==name:==` annotations and the signature file".

**5. Verify**

```bash
bibi-ctrl status
```

Starte Claude Code im Repo-Verzeichnis — `/open`, `/save` etc. erscheinen
als Befehle (Skills sind in `.claude/skills/` committed).

**6. (Optional) Daemon-Rollen installieren**

```bash
bibi-ctrl daemon install
```

Welche Rollen je Knotentyp sinnvoll sind, ergänzt das Team sobald die
Laufzeit steht (spätere Phasen).

---

## Für Produktions-/Server-Deploy

Abweichend vom Entwickler-Setup oben: Engine + Instanz als **eigene,
separate Klone** unter `/srv/` (nicht `~/Project/`), Daemon läuft dauerhaft
über systemd. Das Verfahren ist im Betrieb erprobt; die Schritte unten stammen
aus einem realen Server-Deploy, nicht aus der Theorie.

**1. Verzeichnisse anlegen** (einmalig `sudo` für `/srv/`):

```bash
sudo mkdir -p /srv/bibi /srv/INSTANZ
sudo chown "$(whoami):$(whoami)" /srv/bibi /srv/INSTANZ
```

**2. Klonen als Geschwister-Ordner** (Pflicht für den editierbaren Install
in Schritt 3 — `-e ../bibi` ist relativ):

```bash
git lfs install          # als der Nutzer, dem die Checkouts gehören sollen
git clone -b master https://github.com/plan-net-journey/bibi.git /srv/bibi
git clone -b trunk https://github.com/plan-net-journey/INSTANZ.git /srv/INSTANZ
```

Das `git lfs install` gilt pro Nutzer (s. Abschnitt oben) — auf einem Server
mit eigenem Service-Account also erneut, auch wenn ein anderer Account auf
derselben Maschine es längst hat.

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

> **Falle — jede lokale bibi-Installation ist fragil, editable oder nicht:**
> jeder `uv run bibi-ctrl …`-Aufruf synct das venv gegen die in
> `uv.lock` **eingefrorene** Git-Abhängigkeit zurück — **auch der
> `ExecStart` der systemd/launchd-Unit bei jedem Neustart.** Der editierbare
> Install gegen `/srv/bibi` wird dabei durch einen regulären Install
> ersetzt — kein echtes „editable" mehr, sobald der Daemon einmal neu
> gestartet hat. Für alle direkten CLI-Aufrufe daher `.venv/bin/bibi-ctrl`
> verwenden, nicht `uv run bibi-ctrl` — Letzteres reproduziert die Falle bei
> jedem Aufruf.
>
> **Korrektur (2026-07-05, empirisch widerlegt):** der Re-Sync installiert
> **nicht** automatisch den aktuellen `origin/dev`-Commit, sondern exakt den in
> `uv.lock` **eingefrorenen** Commit-Hash (`bibi.git?rev=dev#<sha>`) — `uv`
> löst den Branch beim Lockfile-Erstellen einmalig auf ein festes SHA auf und
> rührt es danach nicht mehr an. Ein Engine-Fix auf `dev` kommt also **nicht**
> von selbst an, nur weil man den Daemon neu startet.
>
> **Zweite Korrektur (2026-07-22, `bibi-notes` live erlebt):** die Falle
> betrifft nicht nur das editierbare `-e ../bibi`-Setup dieses Abschnitts,
> sondern GENAUSO ein reguläres, nicht-editierbares Install direkt gegen die
> Git-URL (`uv pip install "bibi[daemon] @ git+…@dev"`, das Entwickler-Setup
> oben oder ein manueller Ad-hoc-Deploy) — der Re-Sync-Mechanismus fragt
> nicht danach, WIE die vorherige Installation zustande kam, nur ob `uv.lock`
> zufrieden ist. Auch `--reinstall`, `--no-cache` und sogar ein explizites
> `uv pip uninstall` davor halfen dabei **nicht** — der nächste `uv run`
> setzt die venv trotzdem auf den in `uv.lock` eingefrorenen Commit zurück,
> weil das Lockfile selbst unverändert blieb. Deploy-Reihenfolge für einen
> Engine-Fix, unabhängig vom Install-Modus:
> 1. Fix in `bibi` auf `dev` committen + pushen.
> 2. Im Instanz-Repo (`bibi-notes`/`INSTANZ`) `uv lock --upgrade-package bibi`
>    ausführen — hebt den gepinnten Commit im `uv.lock` an. Ein reines
>    `uv pip install`/`uninstall` gegen die venv reicht nicht, das Lockfile
>    selbst muss sich ändern.
> 3. `uv.lock`-Änderung committen + pushen, dort ankommen lassen (Pull/Sync).
> 4. Erst dann bringt ein Daemon-Neustart (`systemctl restart …`) den neuen
>    Engine-Commit tatsächlich ins laufende venv.

> **Falle — Snap-`uv` bricht die systemd-Unit:** `bibi-ctrl daemon install`
> meidet bewusst `/snap/bin/uv` (Snap-Sandbox unverträglich mit systemd) und
> bevorzugt `~/.local/bin/uv` (astral-Standalone-Installer). Falls nur
> Snap-`uv` vorhanden ist, zuerst nachinstallieren:
> `curl -LsSf https://astral.sh/uv/install.sh | sh`.

> **Nicht vergessen — `auto_sync on` setzen:** ein unbeaufsichtigter Scheduler hat
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

> **Bis zur Publikation stand hier Track C** — ein offener Punkt, weil
> the-library nur **GitHub**-Quell-URLs parst und die Engine auf einer privaten
> Gitea lag. Mit der Veröffentlichung von `bibi` auf GitHub ist er
> **gegenstandslos**: die Quellen in `library.yaml` zeigen jetzt dorthin, und
> `/library use|sync` funktioniert ohne Erweiterung.

### The-library einrichten (einmalig)

Ein direkter Klon genügt. Der frühere Umweg über einen privaten Mirror hatte
genau einen Grund — die Engine war privat und the-library kam nicht an sie
heran. Der ist mit der Publikation weg.

```bash
git clone https://github.com/disler/the-library ~/.claude/skills/library
```

`/library` ist damit in jeder Claude-Code-Session global verfügbar.

**Repo-URL eintragen:** in `~/.claude/skills/library/SKILL.md` den
`## Variables`-Abschnitt anpassen:

```markdown
- **LIBRARY_REPO_URL**: `https://github.com/plan-net-journey/bibi`
```

`LIBRARY_YAML_PATH` und `LIBRARY_SKILL_DIR` bleiben unverändert.

**Verify:** neue Claude-Code-Session starten → `/library list` zeigt den Katalog.

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
# Aktuell installiert von: plan-net-journey/bibi@<tag> (master)
```

> Branch: vor dem Release wird auf `dev` entwickelt (gruppierte `skills/case-*`-
> Struktur) — `source:` zeigt entsprechend auf `@dev`. Nach `dev → master` auf
> `@master` heben.

So ist immer nachvollziehbar, welche Engine-Version die vendored Skills lieferte.

---

## Diese Anleitung prüfen

Sie veraltet leise. Wer sie geschrieben hat, liest über Lücken hinweg — er weiß ja, was gemeint ist. Deshalb: **einmal von vorn durchgehen, bevor ein neuer Mensch es tut.** In einem leeren Verzeichnis, mit einer Sitzung, die nichts über dieses Projekt weiß.

Der Prompt dafür, zum Kopieren:

```text
Ich gehe die INSTALL.md eines bibi-Team-Repos von vorn durch und prüfe,
ob sie für jemanden trägt, der das System nicht kennt.

Begleite mich Schritt für Schritt: sag mir, was ich eingeben soll und was
ich danach sehen müsste, und frag dann, was ich tatsächlich gesehen habe.

Zwei Regeln:
1. Repariere nichts. Was hakt, kommt in befunde.md, dann gehen wir weiter.
   Wer unterwegs repariert, weiß am Ende nicht mehr, wo es gehakt hat.
2. Frag mich bei jedem Schritt: "Hätte ein Fremder das gewusst?" Ich neige
   dazu, Lücken zu überlesen, weil ich das System kenne.

Läuft auf dieser Maschine schon eine bibi-Instanz? Dann halte mich auf,
bevor ich `bibi-ctrl init` ohne BIBI_CONFIG_PATH ausführe — es überschreibt
sonst deren Konfiguration ohne Backup.

Am Ende: sortiere die Befunde danach, wem sie gehören — dem Blueprint, der
Engine oder dieser Instanz. Im Zweifel Blueprint: ein Fehler dort trifft
jedes künftige Team.
```

**Was der Prompt bewusst nicht enthält:** die Liste dessen, was gerade schon bekannt ist. Die steht im Issue-Board und veraltet dort, wo sie hingehört — in einem Prompt würde sie mitaltern und irgendwann Befunde unterdrücken, die längst wieder neu sind.
