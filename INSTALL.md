# Setup — bibi-team

Menschenlesbare Setup-Checkliste. Keine Credentials im Repo (DESIGN §4.10).

---

## Für Entwickler (Standard-Setup)

Skills sind committed — nach dem Clone sofort verfügbar, kein the-library nötig.

### Voraussetzungen

- [`uv`](https://docs.astral.sh/uv/) installiert
- `git` — **die Engine braucht keine Credentials.** `bibi` und `bibi-team` sind seit dem 2026-08-06 öffentlich auf GitHub; `uv pip install .` zieht die Engine ohne jede Anmeldung. Nur für den Klon des **Instanz-Repos** brauchst du Zugang, wenn es privat ist (credential-helper: `store` auf Linux, Keychain auf macOS). Bis zur Öffnung stand hier das Gegenteil.
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

**Jetzt gleich: die geerbte `LICENSE` (`m.rau/bibi#178`).** Das Template kopiert den gesamten Dateibestand, also auch die MIT-Lizenz des Blueprints. Der Blueprint **braucht** sie — ein öffentliches Repo ohne Lizenz ist „alle Rechte vorbehalten" und dürfte gar nicht als Vorlage benutzt werden. **Deine Instanz erbt damit eine Aussage, die für sie meist nicht stimmt.**

- **Privates Repo?** Dann `LICENSE` löschen — sie stammt aus dem Blueprint und gilt für ihn, nicht für dich. Eine MIT-Datei auf einem Repo mit Kundeninhalten sagt, jeder dürfe sie verwenden, verändern und weitergeben. Das ist keine Formalie, sondern eine Falschaussage über die Rechtelage — und sie steht dort unbemerkt, weil niemand eine Datei liest, die von selbst erschienen ist.
- **Öffentliches Team-Repo?** Dann braucht es eine **eigene** Lizenzentscheidung statt der geerbten.

`/bibi-setup` fragt in Schritt 0b von selbst danach. Wer diese Anleitung von Hand abarbeitet, muss selbst daran denken — deshalb steht es hier und nicht weiter unten.

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

> **Es gibt auch einen geführten Weg, und er ist inzwischen der bessere.** Der Skill `/bibi-setup` fragt dich durch das Setup, statt dich eine Anleitung abarbeiten zu lassen — er installiert, konfiguriert, bringt den richtigen Daemon hoch und öffnet die Oberfläche. Er liegt in `.claude/skills/bibi-setup/` und ist nach dem Klon sofort da.
>
> Er kennt seit dem 2026-08-06 **alle vier Knotenarten** und fragt als erstes, welche diese Maschine ist (`m.rau/bibi#179`, `#180`). Vorher setzte er einen Scheduler voraus und installierte immer einen Dienst; beides ist weg. Alles, was unten steht, macht er selbst — diese Anleitung ist der Weg von Hand, nicht der Notweg.

**Vorher prüfen, ob auf diesem Rechner schon ein bibi-Knoten wohnt (`m.rau/bibi#173`).** `bibi-ctrl init` schreibt `~/.config/bibi/env` und legt **kein Backup** an. Ein zweites `init` auf derselben Maschine zerstört die Konfiguration der ersten Instanz: `BIBI_NODE_ID` (der Knoten verliert seine Identität und seine Freigabe am Scheduler), alle `BIBI_JOB_ENV_*`-Werte aus dem Verteilweg, gesetzte Poll-Intervalle, `BIBI_PUBLIC_HOST`.

```bash
test -f ~/.config/bibi/env && grep -oE '^[A-Za-z_][A-Za-z0-9_]*=' ~/.config/bibi/env | tr -d '='
```

Das zeigt die Variablennamen **ohne die Werte** — genug, um zu sehen, ob dort schon jemand wohnt, und unbedenklich in einem Terminal, dem jemand zusieht. Gehört der Eintrag zu einem anderen Checkout, gib dieser Instanz ihre eigene Datei und benutze sie für **jeden** `bibi-ctrl`-Aufruf:

```bash
export BIBI_CONFIG_PATH="$PWD/data/bibi-env"
```

`config.py` löst sie vor `XDG_CONFIG_HOME` auf. Der sarasate-Host und sein Client fahren seit dem 2026-07-11 genau so — die Fähigkeit ist erprobt, nur ihre Erwähnung fehlte hier. Zwei Anschlüsse, ohne die es nicht hält: **die Variable gehört auch in die systemd-/launchd-Unit** (Schritt 6), sonst liest der Daemon beim nächsten Start wieder die geteilte Datei; und **`BIBI_WORKER_NAME` gehört dazu**, sonst meldet sich die zweite Instanz unter `socket.gethostname()` an und kollidiert mit der ersten in der Team-Registry.

```bash
bibi-ctrl init
```

**Er stellt dir genau eine Entscheidung: Was ist diese Maschine?** Und darauf gibt es **vier Antworten** (`m.rau/bibi#174`) — den Rest leitet er selbst ab:

| Antwort | wann | Daemon |
|---|---|---|
| `client` | ein Arbeitsplatz. Du arbeitest hier, du siehst die Oberfläche. **Der Normalfall am Anfang**, auch ganz ohne Server. | Sitzungs-Daemon |
| `worker` | reiner Ausführungsknoten: nimmt Aufträge entgegen, zeigt nichts an | Dienst |
| `scheduler` | der Server. Er hält die Job-Datenbank und die Uhr. | Dienst |
| `scheduler+worker` | der Server, und er führt die Jobs selbst aus — die übliche Form für den **ersten** Server eines Teams | Dienst |

Dasselbe ohne Rückfrage, etwa in einem Skript:

```bash
bibi-ctrl init --non-interactive --profile client --scheduler-url http://<host>:8780
```

**Was du dabei nicht abwägen musst, weil es keine Wahl ist.** `synchronizer` gehört auf **jeden** Knoten — ohne ihn gleicht sich das Repo nicht ab, und der Daemon lehnt eine Rollenmenge ohne ihn seit `v0.7.2` ab (`m.rau/bibi#163`). `controller` — die Oberfläche — trägt ein Client immer, ein Worker nie. Und `connect` ist keine Vorliebe, sondern folgt aus der Frage, ob es einen Scheduler gibt: ein Scheduler darf es überhaupt nicht tragen, er ist das Verbindungsziel und verbindet sich nicht zu sich selbst.

**Ein Worker ohne Scheduler ist kein Aufbau, sondern ein Fehler.** Er startet, meldet sich gesund und bekommt nie einen Auftrag. `init` lehnt das deshalb ab und sagt, warum. Wenn es keinen Scheduler gibt, ist die Antwort `client`, nicht `worker`.

**Soll der Server eine Oberfläche zeigen?** Standardmäßig nicht — er ist Backend, und eine zweite Oberfläche wäre ein zweiter Ort, an dem man nachsieht (so hat sarasate es am 2026-08-04 entschieden). Ist er der **erste** Knoten deines Teams und es gibt noch keinen Client, hänge `--with-ui` an: sonst hat niemand etwas anzusehen.

**Ohne Scheduler fehlt nichts als zwei Dinge:** zeitgesteuerte Jobs und die Verteilung über mehrere Rechner. Der Case-Zyklus, `bibi-ctrl run`, die Oberfläche und `doctor` laufen ab Tag 1. „Nur Clients" ist ein gültiger Aufbau, kein halber.

**Nach der Scheduler-URL fragt er von selbst**, wenn deine Antwort einen Scheduler zulässt (`client`) oder verlangt (`worker`) — vorher entschied darüber das Wort `connect` in der Rollenliste, was niemand ahnen konnte.

**Die Rollenliste gibt es weiterhin**, für den, der sie kennt: `--role synchronizer,controller` statt `--profile`, oder interaktiv als zweite zulässige Antwort auf dieselbe Frage. Sie ist nicht verschwunden, sie ist nur nicht mehr die erste Frage an einen neuen Menschen.

Fragt interaktiv ab und schreibt `~/.config/bibi/env` (außerhalb des Repos,
nie versioniert):

| Parameter | Beispiel | Bedeutung |
|---|---|---|
| `BIBI_SCHEDULER_URL` | `http://<host>:8780` | wohin `--connect` zeigt (leer lassen, wenn es keinen Server gibt) |
| `BIBI_ROLE` | `synchronizer,controller` | die Rollen dieses Knotens — **abgeleitet** aus deiner Antwort oben, nicht selbst einzutippen |
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

**6. Daemon — und hier hängt es an der Knotenart, nicht am Betriebssystem**

**Client: du bist fertig, es gibt nichts zu installieren.** Der Daemon kommt mit der Sitzung: `bibi` startet ihn als eigenes Kind (`--session --port auto`) und beendet ihn mit der letzten Sitzung. Das ist Absicht — ein Arbeitsplatz-Daemon existiert, solange jemand arbeitet, und ein Dienst, der den Rechner überlebt, ist einer, den niemand bestellt hat (`m.rau/bibi#180`). Der Port ist dynamisch; lies ihn aus `bibi-ctrl status`, schreib ihn nirgends fest.

**Starte einen `--session`-Daemon nicht von Hand aus einer normalen Shell.** Er meldet dann keine Sitzung an, der Aufräumer findet null und fährt ihn beim nächsten Durchlauf wieder herunter — das sieht aus wie ein Start, der still fehlgeschlagen ist. Nimm `bibi`.

**Worker, Scheduler, Scheduler+Worker: hier gehört ein Dienst hin**, denn sie müssen da sein, wenn niemand zusieht.

```bash
bibi-ctrl daemon install [--connect]
```

`--connect` auf einem Worker, nie auf einem Scheduler. Läuft eine **zweite Instanz** auf derselben Maschine (Schritt 3), trag `BIBI_CONFIG_PATH` und `BIBI_WORKER_NAME` von Hand in die geschriebene Unit nach — `daemon install` nimmt sie nicht mit, und ohne sie liest der Daemon beim nächsten Start wieder die geteilte Konfiguration.

Steht auf einem **Client** schon eine Unit, weil sie aus einer früheren Anleitung stammt: `bibi-ctrl daemon uninstall`.

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

### Einen Skill reparieren: immer zuerst die Quelle

**Ein Skill-Fix ist erst fertig, wenn er in `bibi/skills/<name>/SKILL.md` steht.** Die Datei unter `.claude/skills/` ist die Kopie. Wer nur sie repariert, hat den Fehler nicht behoben, sondern lokal überdeckt — und der nächste `/library sync` schreibt ihn zurück, ohne dass jemand hinsieht: der Sync hat ja nur „Quellstand hergestellt".

Das ist kein hypothetisches Risiko. `m.rau/bibi#70` (`attempts:` falsch dokumentiert) wurde am 2026-08-01 in den Instanzen gefixt und geschlossen; die Quelle trug den Fehler bis zum 2026-08-06 weiter. Alle Instanzen standen damit *vor* der Quelle, und ein Sync hätte ein geschlossenes Ticket still wieder geöffnet.

**Ein Skill-Fix braucht kein Release** — er kommt per git, nicht per `uv sync`. Aber er darf nur Kommandos nennen, die der **gepinnte** Engine-Tag schon kennt. Gegen den Tag prüfen, nicht gegen `dev`:

```bash
git -C ../bibi show v0.7.1:bibi/ctrl/daemon_cmd.py | grep -- '--session'
```

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
