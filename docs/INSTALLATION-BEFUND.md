# ili — Installations-Befund und Änderungen

> **Redigiert:** konkrete Werte (E-Mail, LAN-IPs, Domain, Zugangsdaten) sind hier
> bewusst ersetzt — dieses Dokument liegt in einem Repo, das veröffentlicht werden
> kann. Die Fundstellen sind mit Datei und Zeile benannt, das genügt zum Beheben.

Stand: 2026-08-21 · Prüfumgebung: Linux-Container (Docker 29.3.1, Python 3.11.15)

Auftrag war: `Toa1984/dashboard` nach `~/containers/dashboard` klonen, der dortigen
`INSTALL.md` folgen, per Docker installieren, **keinen Code ändern**, Probleme
dokumentieren. Kernfragen: **Läuft es? Sind private Daten drin?**

---

## Kurzantwort

| Frage | Antwort |
|---|---|
| Läuft ili? | **Ja.** API startet sauber, `/health` = 200, `/boards` liefert die drei Starter-Boards. |
| Hat **ili** private Daten? | **Nein.** Keine Keys, keine E-Mail, keine echten IPs, keine echte Domain. Nur dein Vorname in drei internen Doku-Dateien und einem Ollama-Modellnamen. |
| Hat der **Upstream `dashboard`** private Daten? | **Ja, erheblich** — und ein Teil davon landet im Container-Image bzw. wird vom Frontend ausgeliefert. Details unter [P6](#p6). |
| Docker-Build hier durchgeführt? | **Nein** — die Egress-Policy dieser Umgebung blockt den Docker-Hub-Blob-CDN. Alles andere wurde nativ verifiziert. Siehe [P1](#p1). |

---

## Was tatsächlich verifiziert wurde

Verifiziert heißt: hier ausgeführt, Ausgabe gesehen — nicht aus der Doku abgelesen.

* `pip install -r requirements.txt` läuft ohne Fehler durch (Python 3.11.15).
* `from app.main import app` importiert sauber, **35 Routen** registriert.
* ili unter `uvicorn app.main:app` gestartet:
  * `GET /health` → `200 {"status":"ok","service":"dashboard-api"}`
  * `GET /boards` → `200`, Top-Level: `willkommen`, `ideen`, `einrichtung`
* Upstream `dashboard` ebenso gestartet, `/health` und `/boards` → 200 (Demo-Boards).
* `docker compose build` im Upstream: **abgebrochen** (Basis-Image nicht ladbar, [P1](#p1)).
* Der `COPY`-Fehler aus [P2](#p2) wurde isoliert mit `FROM scratch` reproduziert —
  ohne Netz, damit unabhängig von [P1](#p1).

**Nicht verifiziert:** der vollständige `docker compose up` beider Container, das
nginx-Frontend im Browser, das optionale Projekt-Terminal. Alles drei hängt am
Basis-Image-Pull.

---

## Probleme

### P1 — Docker-Build in dieser Umgebung nicht möglich (Umgebung, nicht Repo)

`docker compose build` bricht beim Basis-Image ab:

```
failed to resolve source metadata for docker.io/library/python:3.11-slim:
Get "https://production.cloudfront.docker.com/registry-v2/..." : Forbidden
```

Erreichbarkeitstest der Registry-Hosts:

| Host | Ergebnis |
|---|---|
| `registry-1.docker.io` | 401 (normal — Auth-Challenge) |
| `auth.docker.io` | 404 (normal) |
| `production.cloudfront.docker.com` | **blockiert** (403 vom Egress-Proxy) |
| Retry nach Backoff | `429 Too Many Requests` (anonymes Docker-Hub-Limit) |

Der Blob-CDN ist per Organisations-Egress-Policy gesperrt; das ist keine
Fehlkonfiguration des Repos. **Auf einem Rechner mit normalem Docker-Zugang
(z. B. Docker Desktop) tritt das nicht auf.**

### P2 — Upstream: `docker compose build` schlägt nach frischem Clone fehl

Das `Containerfile` von `Toa1984/dashboard` kopiert zwei Dateien, die nach einem
frischen Clone nicht existieren, weil sie gitignored sind:

```
COPY board_templates.json .
COPY automat_limits.json .
```

Im Repo liegen nur `board_templates.example.json` und `automat_limits.example.json`
(`.gitignore` Zeilen 70 und 72). Isoliert reproduziert:

```
ERROR: failed to compute cache key: ... "/board_templates.json": not found
```

Mit der `.example`-Variante baut derselbe Layer fehlerfrei. Der Build ist damit
**nach jedem frischen Clone kaputt**, unabhängig von der Netzwerklage.

Naheliegende Behebung (nicht angewandt — kein Code geändert): entweder die beiden
`.example`-Dateien im `Containerfile` kopieren, analog zu den bereits so
behandelten `services_config.example.json` / `project_map.example.json`, oder die
Kopie im `docker-entrypoint.sh` erledigen.

**ili ist davon nicht betroffen** — dessen `Containerfile` kopiert nur `*.py`,
`app/`, `html/`, `demo/`, `requirements.txt`, `docker-entrypoint.sh`. Alle vorhanden.

### P3 — Es gibt keine `INSTALL.md`

Weder in `Toa1984/dashboard` noch in `Toa1984/ili-dashboard` existiert eine
`INSTALL.md`. Die Installationsanleitungen heißen:

* Upstream: Abschnitt „Installation" in `README.md`
* ili: `QUICKSTART.md`

`deploy/nginx-portable.conf` verweist im Kopfkommentar auf einen
„INSTALL-Abschnitt im README" — der Abschnitt heißt dort schlicht „Installation".
Kosmetisch, aber es erklärt, warum eine gesuchte `INSTALL.md` nicht auffindbar ist.

### P4 — Upstream-`docker-compose.yml` ist ausdrücklich nicht gepflegt

Erste Zeile der Datei:

```
# ⚠️  DEMO/Fremdnutzung nur — In Produktion läuft das Dashboard als Host-uvicorn
#     via dashboard-api.service. Dieses Compose-File ist nicht in Benutzung.
```

Es bindet außerdem Port **80** auf dem Host, was ein rootless-Nutzer nicht binden
darf. ili hat beides gelöst: fester Compose-Projektname `ili`, Default-Port
**8080** über `ILI_PORT`, und die API veröffentlicht gar keinen Host-Port mehr.

**Für eine Installation ist ili das richtige Repo, nicht der Upstream.**

### P5 — Upstream: `html/index.html`-Warnung im README ist überholt

Das README warnt, `html/index.html` fehle nach frischem Clone. Sie ist vorhanden
(663 Zeilen); `.gitignore` Zeile 31 nimmt sie ausdrücklich aus. Die Warnung führt
in die Irre.

### P6 — Private Daten im Upstream-Repo — teils im Image, teils ausgeliefert {#p6}

Das ist der ernste Befund. **Der Upstream ist ein privates Repo; falls er je
öffentlich wird oder ein Image daraus weitergegeben wird, geht Folgendes mit:**

**a) `html/services.html` — die vollständige Heimnetz-Karte.** 52 Service-Karten,
darunter:

| Enthalten | Beispiel |
|---|---|
| Deine E-Mail-Adresse | deine private E-Mail im Klartext (Zeile 262, n8n-Karte) |
| Dein Benutzername | dein Login-Name (Zeile 320, Cockpit) |
| Interne IPs | vier Hosts aus deinem LAN, mit Port — **100 Treffer** |
| Zugangsdaten-Paare | zwei Standard-Paare im Klartext, an vier Diensten (Grafana, Paperless NGX, FHEM) |
| Hardware-Details | GPU-Modell des LLM-Servers, Bezeichner der Zigbee-Koordinatoren |

Aus IP + Port + Benutzer/Passwort lässt sich jeder dieser Dienste direkt
ansprechen, sobald jemand im selben Netz steht.

**b) Deine echte Domain** in sechs ausgelieferten Frontend-Dateien
(`nav.js`, `leichen.html`, `whitelist.html`, `cost.html`, `ai-settings.html`,
`ki-advisor.html`), plus in `CLAUDE.md`, `HISTORY.md`, `tests/`, `tools/`.
Betroffen sind die Subdomains für Dashboard, LAN-Scanner, UI-Kit und Terminal 1–4.

Bemerkenswert: `CLAUDE.md` Zeile 88 verbietet genau das („eine absolute LAN-URL
… schreibt zusätzlich eine echte Domain ins Repo"), und `tests/test_frontend_asset_urls.py`
bewacht es — aber nur für `<script src>` und `<link href>`. Die sechs Treffer sind
`<a href>`-Querlinks, die der Test **bewusst erlaubt**. Die Regel greift also nicht
für den Fall, der hier eingetreten ist.

**c) Dein Home-Pfad `/home/<user>`** — 67 Treffer in 27 Dateien, überwiegend `deploy/systemd/*.service`
und Doku. Nur drei davon liegen in Dateien, die ins Image wandern
(`app/services/attachment_service.py` ×2, `kanban_coding_hook.py` ×1) und sind
dort Kommentar-Beispiele.

**Was davon wirklich im Container landet:** Das `Containerfile` kopiert `html/`
komplett. Damit sind (a) und (b) **im Image**. Zusätzlich mountet
`docker-compose.yml` `./html` direkt in den nginx-Container — `services.html` ist
also unter `http://<host>/services.html` **im Browser abrufbar**, ganz ohne Login.

Nicht im Image: `HISTORY.md`, `CLAUDE.md`, `ANLEITUNG_RESTARBEITEN.md`, `deploy/`,
`tests/`, `tools/`, `archiv/`, `html-beta/`.

**Keine Secrets gefunden:** kein `sk-ant-…`, kein `ghp_…`/`github_pat_…`, kein
AWS-Key, kein privater Schlüssel — weder im Upstream noch in ili. Die
`.env.example`/`config.env.example` enthalten nur Platzhalter. Der Scan lief über
den flachen Clone (`--depth 1`), deckt also **nicht** die Git-Historie ab.

### P7 — ili ist sauber, mit einer Randnotiz

Derselbe Scan über `Toa1984/ili-dashboard`:

| Muster | Ergebnis |
|---|---|
| API-Keys / Tokens / private Schlüssel | keine |
| E-Mail-Adressen | nur `git@github.com`, `token@github.com` (Git-URL-Formen) |
| Private IPs | nur `192.168.1.0/24` als generisches Doku-Beispiel im README |
| deine echte Domain | keine |
| dein Home-Pfad | keine |
| `services.html` | nicht vorhanden |

Verbleibend, harmlos, aber erwähnenswert: dein Vorname in `docs/WIDGETS.md`, `docs/JOBS.md` und
`docs/KATEGORIEN-PRIO.md` (jeweils als Begründung einer Design-Entscheidung) sowie
in einem Ollama-Modellnamen in `html/ai-settings.html:254`. Kein
Sicherheitsproblem — nur ein Hinweis auf die Herkunft, falls das Repo
veröffentlicht wird.

### P8 — Nav-Links auf Sub-Domains, die es bei einer Neuinstallation nicht gibt

`html/nav.js` leitet die Basis-Domain aus dem eigenen Hostnamen ab:

```js
const _h = location.hostname;
const BASE = _h.includes('.') ? _h.slice(_h.indexOf('.') + 1) : _h;
```

Auf der Caddy-Installation des Autors (`dashboard.intranet.<domain>`) ergibt das
die richtige Basis. Bei einer Container-Installation auf `http://localhost:8080`
enthält der Hostname keinen Punkt, `BASE` wird zu `localhost`, und drei
Nav-Einträge zeigen ins Leere:

| Eintrag | erzeugte URL | Problem |
|---|---|---|
| `nav-link-terminal` | `https://terminal.localhost/` | Sub-Domain existiert nicht; zusätzlich hartes `https://` und der Port `:8080` fällt weg |
| `nav-link-scan` | `https://shelly-scanner.localhost/lan` | Dienst ist in ili gar nicht enthalten |
| `nav-link-shelly` | `https://shelly-scanner.localhost/` | dito |

**Terminal — behoben.** Der Container-Stack bedient das Terminal same-origin
unter `/projterm/`: `deploy/nginx-setup.sh` schreibt dafür einen
`location ^~ /projterm/`-Block, den `deploy/nginx-portable.conf` per
`include /etc/nginx/ili-projterm*.conf` einbindet. Der Rest des Frontends nutzt
längst genau diesen Pfad — `leichen.html`, `fragen.html`, `m/m.js`,
`js/project-chat-terminal.js` und `ili-terminal-login.html` verlinken alle
`/projterm/`. `nav.js` war die einzige Stelle mit der Sub-Domain-Form. Der Link
zeigt jetzt ebenfalls auf `/projterm/`.

Das ist kein Container-Sonderweg: `/projterm/` existiert auf der Installation des
Autors genauso, sonst wären die fünf genannten Dateien dort schon kaputt. Ohne
`?arg=<board>` landet man laut `deploy/terminal/ili-term.sh` in der generischen
Session `proj-home` unter `$PROJECTS_DIR` — passend für einen allgemeinen
Nav-Eintrag.

**Shelly-Links — entfernt.** Entscheidung des Betreibers: der Shelly-Scanner
gehört in ein eigenes Projekt. Das deckt sich mit dem Code, der schon dort ist —
`app/api/misc.py`, `app/services/misc_service.py` und `app/api/config.py`
vermerken, dass `GET /shelly` am 2026-08-01 und `POST /trigger-scan` samt
`/config` am 2026-08-07 in den Container `shelly-scanner` gewandert sind. Es gibt
in ili **keine** dieser Routen mehr; Nav-Einträge, Sprachschlüssel und
README-Zeilen waren reine Karteileichen. Alle entfernt, siehe
den Abschnitt „Änderungen auf diesem Branch" unten.

Damit ist die Hostnamen-Ableitung (`BASE`) in `nav.js` ohne Verwendung und
ebenfalls entfallen — die Ursache aus diesem Abschnitt existiert nicht mehr,
statt nur an einer Stelle umgangen zu sein. Sämtliche Nav-Ziele sind jetzt
same-origin.

**Wichtig, unabhängig vom Link:** Das Projekt-Terminal startet **nicht**
automatisch mit. `docker compose up -d --build` allein lässt `/projterm/` mit
einem 503 antworten, das genau darauf hinweist. Es braucht die zweite
Compose-Datei:

```bash
docker compose -f docker-compose.yml -f docker-compose.terminal.yml up -d --build
docker compose logs web | grep ili-setup      # zeigt das generierte Passwort
```


### P9 — Feste `container_name` unterlaufen den festen Projektnamen

`docker-compose.yml` setzt in Zeile 15 bewusst `name: ili`, damit die Volumes nach
einem Umbenennen des Ordners nicht verwaisen. Zwei Zeilen weiter wird dieser
Nutzen für die Container wieder aufgehoben:

| Datei | Zeile | Wert |
|---|---|---|
| `docker-compose.yml` | 23 | `container_name: dashboard-api` |
| `docker-compose.yml` | 51 | `container_name: dashboard-web` |
| `docker-compose.terminal.yml` | 26 | `container_name: ili-terminal` |

Container-Namen sind in Docker **global**, nicht pro Projekt. Das Werkstatt-Repo
vergibt in seiner `docker-compose.yml` (Zeilen 17 und 42) exakt dieselben Namen
`dashboard-api` und `dashboard-web` und hat gar kein `name:`. Beide Stacks können
deshalb nie nebeneinander laufen — der zweite `up` scheitert am belegten Namen
oder hinterlässt eine gestoppte Leiche.

Auffällig ist die Inkonsistenz innerhalb von ili selbst: der Terminal-Service
heißt korrekt `ili-terminal`, `api` und `web` tragen noch die Namen aus der
Herkunft. Naheliegend wäre `ili-api` / `ili-web` — oder `container_name` ganz
weglassen, dann bildet Compose aus dem Projektnamen ohnehin `ili-api-1` und
`ili-web-1`.

Nicht geändert: das benennt laufende Container um, was bestehende Installationen
und jedes Skript trifft, das die Namen verwendet. Eine Entscheidung des
Betreibers, kein Bugfix.

### P10 — Der 503-Stub bleibt liegen, wenn das Terminal nachträglich startet

`deploy/nginx-setup.sh` läuft als `/docker-entrypoint.d/20-ili-setup.sh` und
entscheidet **einmal beim Start des `web`-Containers** in drei Stufen:

```sh
if probe;                                    then  # Terminal antwortet -> Route verdrahten
elif nslookup "$TERMINAL_HOST" >/dev/null;   then  # Name loest auf -> trotzdem verdrahten
else write_stub                                    # 503 mit Anleitung
```

Die mittlere Stufe fängt den Fall ab, dass der Terminal-Container noch bootet.
Der Stub entsteht also nur, wenn der Service zum Startzeitpunkt von `web`
überhaupt nicht im Compose-Netz war — der Normalfall bei einem laufenden Stack,
den man nachträglich um `-f docker-compose.terminal.yml` erweitert.

Der Skript-Kopf kennt das Verhalten („Writing the stub instead would leave a 503
behind that only a restart of this container could clear"), aber weder
`QUICKSTART.md` noch `docs/PROJECT-TERMINAL.md` sagen es dem Nutzer. Beide zeigen
nur:

```bash
docker compose -f docker-compose.yml -f docker-compose.terminal.yml up -d
```

Wer den Stack vorher schon laufen hatte, bekommt danach weiter den 503 und hat
keinen Hinweis, woran es liegt — der Stub-Text nennt genau den Befehl, den man
gerade ausgeführt hat. Nötig ist zusätzlich:

```bash
docker compose -f docker-compose.yml -f docker-compose.terminal.yml restart web
```

Zwei mögliche Behebungen: den Satz in beide Anleitungen aufnehmen, oder den
Stub-Text um den `restart web`-Hinweis ergänzen. Nicht umgesetzt — beides ändert
Text, der ins Werkstatt-Repo zurückfließen sollte.

---

## Änderungen auf diesem Branch

Basis: `main` (`8822c50`). Jede Änderung mit Begründung und Hinweis zum
Übertragen ins Werkstatt-Repo (`Toa1984/dashboard`).

### Überblick

| # | Datei | Art | Portierbar nach `dashboard`? |
|---|---|---|---|
| 1 | `html/nav.js` | Terminal-Link auf `/projterm/` | **Nur nach Entscheidung** — siehe [A1](#a1) |
| 2 | `html/nav.js` | Shelly-Einträge entfernt | Ja, wenn der Scanner auch dort aus der Nav soll |
| 3 | `html/nav.js` | Tote `BASE`-Ableitung entfernt | Nur zusammen mit 1 **und** 2 |
| 4 | `html/i18n/de.js` | Verwaiste Sprachschlüssel entfernt | Zusammen mit 2 |
| 5 | `README.md` | Doku zu entfernten Routen gelöscht | Ja, unabhängig |
| 6 | `docs/INSTALLATION-BEFUND.md` | Neu (dieses Dokument) | Nur als Information |

Kein Backend-Code, keine Konfiguration, keine Abhängigkeiten angefasst.

---

### A1 — Terminal-Link zeigt auf `/projterm/` {#a1}

**Datei:** `html/nav.js`, Eintrag `nav-link-terminal`

```diff
-{ id: 'nav-link-terminal', href: 'https://terminal.' + BASE + '/', icon: '💻', … },
+{ id: 'nav-link-terminal', href: '/projterm/',         icon: '💻', … },
```

**Warum:** `BASE` entstand aus `location.hostname`. Auf `http://localhost:8080`
enthält der Hostname keinen Punkt, `BASE` wurde `localhost`, und der Link zeigte
auf `https://terminal.localhost/` — Sub-Domain existiert nicht, `https://` hart
verdrahtet, Port `:8080` verloren.

`/projterm/` ist der Pfad, den der Rest des Frontends längst nutzt:
`leichen.html:299`, `fragen.html:232`, `m/m.js:505`,
`js/project-chat-terminal.js:249`, `ili-terminal-login.html:44`. Bedient wird er
von `deploy/nginx-setup.sh` (`location ^~ /projterm/`), eingebunden über
`deploy/nginx-portable.conf:55`.

**⚠️ Beim Portieren nach `dashboard` aufpassen — die beiden Ziele sind dort
nicht dasselbe.** `html/leichen.html` erklärt im Kommentar über `terminalUrl`:
`/projterm/?arg=<board-id>` wechselt per `tmux-project.sh` ins Projektverzeichnis
und startet dort Claude Code, während ein Terminal-Link ohne `/projterm/`-Präfix
auf eine **feste, geteilte tmux-Session ohne Query-Param-Handling** führt. Im
Werkstatt-Repo zeigt der Nav-Eintrag also absichtlich auf die geteilte Session.

Ein wörtliches Übernehmen dieses Diffs ändert dort still, welches Terminal
aufgeht. Zu entscheiden ist: soll der Nav-Eintrag die geteilte Session behalten
(dann nicht portieren, sondern nur die Adresse konfigurierbar machen) oder
ebenfalls auf das Projekt-Terminal zeigen?

In ili stellt sich die Frage nicht: es gibt nur ein Terminal. Ohne `?arg=` landet
man laut `deploy/terminal/ili-term.sh` in der generischen Session `proj-home`
unter `$PROJECTS_DIR`.

---

### A2 — Shelly-Einträge aus der Nav entfernt

**Datei:** `html/nav.js` — zwei Einträge ersatzlos gestrichen:

```diff
-{ id: 'nav-link-scan',   href: 'https://shelly-scanner.' + BASE + '/lan', icon: '🔍', key: 'nav.scan',   label: 'LAN-Scan' },
-{ id: 'nav-link-shelly', href: 'https://shelly-scanner.' + BASE + '/',    icon: '📡', key: 'nav.shelly', label: 'Shelly' },
```

**Warum:** Der Shelly-Scanner gehört in ein eigenes Projekt — und **ist es im
Code längst**. Drei Stellen dokumentieren den Umzug:

| Datei | Vermerk |
|---|---|
| `app/api/misc.py:55` | `GET /shelly` am 2026-08-01 in den Container `shelly-scanner` (Port 8808) gewandert; am 2026-08-07 folgten `POST /trigger-scan`, `scan_network.py`, `scan.html`, `scan_config.json` |
| `app/services/misc_service.py:6` | derselbe Umzug, mit Begründung (synchroner 20–30 s-Scan hatte den uvicorn-Threadpool erschöpft, Vorfall 2026-06-15) |
| `app/api/config.py:7` | `/config` (GET+POST) am 2026-08-07 entfallen, bediente nur die LAN-Scan-Konfiguration |

Verifiziert: in ili existiert weder eine `/shelly`- noch eine `/config`-Route.
Die Nav-Einträge zeigten also auf einen Dienst, den dieses Repo nicht mitbringt.

**Beim Portieren:** Im Werkstatt-Repo funktionieren die Links, weil der
`shelly-scanner`-Container dort läuft. Entfernen nur, wenn der Scanner auch dort
aus der Dashboard-Nav verschwinden soll. Beachte zusätzlich: dort steht in
`nav.js` beim Eintrag `nav-link-scan` die **echte Domain fest verdrahtet** statt
der `BASE`-Form — das ist zugleich einer der Datenschutz-Befunde aus
[INSTALLATION-BEFUND.md § P6](INSTALLATION-BEFUND.md).

---

### A3 — Tote `BASE`-Ableitung entfernt

**Datei:** `html/nav.js`, Kopf der IIFE

```diff
-    // Basis-Domain aus eigenem Hostnamen ableiten (dashboard.<base> → <base>),
-    // damit keine echte Domain im Repo steht.
-    const _h = location.hostname;
-    const BASE = _h.includes('.') ? _h.slice(_h.indexOf('.') + 1) : _h;
+    // Alle Nav-Ziele sind same-origin. Frueher wurde hier eine Basis-Domain aus
+    // location.hostname abgeleitet, um auf Nachbar-Sub-Domains zu verlinken; auf
+    // einer Installation ohne Punkt im Hostnamen (localhost) ergab das tote Links.
```

**Warum:** Nach A1 und A2 hatte `BASE` keine Verwendung mehr. Der Kommentar
versprach außerdem, „keine echte Domain im Repo" — ein Ziel, das die Ableitung
nur halb erreichte und das im Werkstatt-Repo ohnehin unterlaufen wird.

**Beim Portieren:** Nur zusammen mit A1 **und** A2 sinnvoll. Bleibt dort auch nur
einer der drei Sub-Domain-Links stehen, wird `BASE` weiter gebraucht.

**Ergebnis in ili:** sämtliche 23 Nav-Ziele sind same-origin, kein externer Link
mehr in der Navigation. Geprüft — jedes Ziel existiert als Datei in `html/`,
außer `/projterm/`, das die nginx-Route bedient.

---

### A4 — Verwaiste Sprachschlüssel entfernt

**Datei:** `html/i18n/de.js`

```diff
-    "nav.scan": "LAN-Scan",
-    "nav.shelly": "Shelly",
```

**Warum:** Nach A2 referenziert sie nichts mehr. `de.js` ist die einzige
Sprachdatei in `html/i18n/`.

---

### A5 — README: Dokumentation zu entfernten Routen gelöscht

**Datei:** `README.md`, vier Stellen:

| Stelle | Inhalt | Warum weg |
|---|---|---|
| Feature-Tabelle | `| **Shelly-Scanner** | Shelly-Geräte im LAN erkennen + MQTT-Config auslesen |` | Feature ist nicht in ili |
| Konfiguration | `# Shelly-Scanner Subnetz(e)` + `SHELLY_SUBNETS=…` | Variable wird von **keiner** Code-Stelle gelesen (verifiziert) |
| API-Tabelle | `| /config | GET/POST | Scan-Konfiguration |` | Route existiert nicht mehr |
| API-Tabelle | `| /shelly | GET | Shelly-Geräte scannen |` | Route existiert nicht mehr |

**Beim Portieren:** Diese vier Zeilen sind auch im Werkstatt-Repo falsch — die
Routen sind dort ebenso entfallen, das README hinkt nur nach. **Unabhängig von
allen anderen Punkten übertragbar.**

---

### A6 — Neues Dokument

`docs/INSTALLATION-BEFUND.md` — dieses Dokument: Prüfbericht (läuft ili, liegen
private Daten in den Repos, welche Installationsprobleme bestehen) plus dieses
Änderungsprotokoll.

Reine Dokumentation ohne Code-Bezug. Übernehmen nur, wenn der Befund auch im
Werkstatt-Repo festgehalten werden soll.

---

### Bewusst **nicht** geändert

Diese Punkte betreffen fast alle das Werkstatt-Repo und brauchen eine
Entscheidung. Details jeweils in den Abschnitt „Probleme" oben.

| Was | Wo | Warum offen |
|---|---|---|
| Feste `container_name: dashboard-api` / `dashboard-web` kollidieren mit dem Werkstatt-Repo (P9) | beide | Umbenennen trifft laufende Installationen und Skripte, die die Namen nutzen |
| `restart web` fehlt in den Terminal-Anleitungen (P10) | beide | Textänderung, die ins Werkstatt-Repo zurückfliessen sollte |
| `html/services.html` enthält E-Mail, Login-Namen, interne IPs und Zugangsdaten-Paare | nur `dashboard` | Datei ist generierter, personalisierter Inhalt; gehört in `.gitignore` + `.containerignore` statt ins Repo. Landet aktuell über `COPY html/ html/` im Image **und** wird per Bind-Mount ausgeliefert |
| Echte Domain in sechs `<a href>`-Links | nur `dashboard` | `tests/test_frontend_asset_urls.py` bewacht nur `<script src>` und `<link href>` und erlaubt `<a href>` ausdrücklich — die Guard-Regel greift für genau diesen Fall nicht |
| `Containerfile` kopiert die gitignoreten `board_templates.json` / `automat_limits.json` | nur `dashboard` | Build ist nach frischem Clone kaputt. Naheliegend: die `.example`-Varianten kopieren, analog zu `services_config.example.json` |
| `docker-compose.yml` bindet Port 80, Header sagt „nicht in Benutzung" | nur `dashboard` | ili hat das bereits gelöst (Port 8080 via `ILI_PORT`, fester Projektname) |
| README-Warnung, `html/index.html` fehle nach frischem Clone | nur `dashboard` | Datei ist vorhanden, `.gitignore` nimmt sie ausdrücklich aus |
| Vorname des Autors in `docs/WIDGETS.md`, `docs/JOBS.md`, `docs/KATEGORIEN-PRIO.md` und im Ollama-Modellnamen `timo-assistant:latest` | `ili-dashboard` | Kein Sicherheitsproblem, aber ein Herkunftshinweis in einem öffentlichen Repo |

---

### Wirksam werden lassen

`html/` ist read-only in den `web`-Container gemountet, und
`deploy/nginx-portable.conf` liefert `.js` mit `Cache-Control: no-cache` aus.
Für die Frontend-Änderungen genügt daher:

```bash
cd ~/containers/ili
git pull
# Browser neu laden — kein Rebuild, kein Neustart nötig
```

Das Projekt-Terminal startet **nicht** mit `docker compose up -d` allein. Damit
`/projterm/` mehr als ein erklärendes 503 liefert:

```bash
docker compose -f docker-compose.yml -f docker-compose.terminal.yml up -d --build
docker compose logs web | grep ili-setup      # zeigt das generierte Passwort
```

---

## Konzept: Erstlauf — Passwort personalisieren, Grunddaten erfassen

Entwurf, **nicht umgesetzt**. Frage war: kann ili Benutzer und Passwort erzeugen
und verlangen, dass beides nach der ersten Nutzung durch eigene Werte ersetzt
wird?

Kurz: ja, aber nicht als Einstellung. Basic Auth kennt kein „muss geändert
werden" — das braucht Zustand und eine Stelle, die ein Formular entgegennimmt.
Wie weit man geht, ist eine Abwägung, keine technische Zwangslage.

### Ausgangslage (verifiziert)

`deploy/nginx-setup.sh` erzeugt beim Start des `web`-Containers ein Passwort,
falls `TERMINAL_PASSWORD` leer ist, schreibt es als `{SHA}`-Hash nach
`/etc/nginx/ili-terminal.htpasswd` und druckt es ins Log. Die Variable
`GENERATED` im Skript hält fest, ob generiert oder aus der Umgebung übernommen
wurde.

Drei Eigenschaften dieser Konstruktion bestimmen alles Weitere:

1. **Die htpasswd liegt im Container-Dateisystem**, nicht in einem Volume. Sie
   ist nach jedem Neustart weg und wird neu geschrieben — ein generiertes
   Passwort wechselt also bei jedem Start.
2. **Es gibt keinen Prozess, der ein Formular annehmen könnte.** nginx liefert
   statische Dateien und proxyt; ein Passwortwechsel braucht jemanden, der
   POST-Daten verarbeitet und die htpasswd neu schreibt.
3. **`web` und `api` teilen kein Volume.** Der API-Container hat `/app/data`,
   der Web-Container nichts Schreibbares. Jede Lösung mit Zustand muss das
   zuerst ändern.

### Variante A — nur dokumentieren

README und `docs/PROJECT-TERMINAL.md` sagen deutlich: es wird eines generiert, es
steht im Log, es wechselt bei jedem Neustart, setz dir ein eigenes in `.env`.
Dazu der fehlende `restart web`-Hinweis aus P10.

Kein Code, kein Risiko. Erzwingt nichts — aber es behebt den heutigen Zustand,
dass die Anleitung den Neustart-Wechsel gar nicht erwähnt.

### Variante B — Hinweis im Dashboard, solange das generierte Passwort gilt

Der interessante Punkt: **das Setup-Skript weiss die Antwort bereits.** Es kann
seinen `GENERATED`-Zustand als winzigen statischen Endpunkt mit ausgeben, in
derselben Datei, die es ohnehin schreibt:

```nginx
location = /projterm/state {
    default_type application/json;
    return 200 '{"generated":true}';
}
```

Das Frontend fragt den Endpunkt ab und blendet einen Hinweis ein: „Das
Terminal-Passwort wurde automatisch erzeugt und wechselt bei jedem Neustart —
setz `TERMINAL_PASSWORD` in deiner `.env`."

Reizvoll, weil es **ohne** Zustandsspeicher, ohne API-Beteiligung und ohne
Änderung am Auth-Mechanismus auskommt. Ein Hinweis, kein Zwang — aber er
erscheint genau dann, wenn er zutrifft, statt in einer Doku, die niemand liest.

Aufwand: ein `location`-Block in `nginx-setup.sh`, ein `fetch` im Frontend, ein
Banner.

### Variante C — erzwungene Personalisierung

Die einzige Variante, die den Namen verdient. Sie ersetzt Basic Auth:

```
Browser ──► web (nginx)
              │  auth_request ──► api  (prüft Cookie/Session)
              │                    └─ Zustand in /app/data
              │  solange nicht personalisiert: 302 auf /terminal-setup.html
              ▼
           terminal (ttyd)
```

Was dazugehört:

| Baustein | Wo |
|---|---|
| Zugangsdaten + Flag `personalized` | `/app/data` im API-Container (Volume existiert) |
| Login-Seite und Setup-Seite | neu in `html/` |
| `POST /api/terminal-auth` (Login) und `POST /api/terminal-password` (Wechsel) | neuer Router in `app/api/` |
| `auth_request`-Block statt `auth_basic` | `deploy/nginx-setup.sh` |
| Einmal-Token für den allerersten Aufruf | beim ersten Start erzeugt, ins Log |

Nebeneffekt, der für die Variante spricht: das heutige Cookie-Konstrukt existiert
nur, weil Safari bei einem WebSocket-Handshake keine Basic-Credentials sendet
(steht so im Kopf von `nginx-setup.sh`). Eine formularbasierte Anmeldung mit
Session-Cookie hat dieses Problem gar nicht erst — der WS-Pfad und der HTML-Pfad
prüfen dann dasselbe Cookie.

Der Einwand: **das ist die einzige Schranke vor einer Root-Shell.** Der heutige
Aufbau ist klein, gut kommentiert und schwer falsch zu machen. Eine selbstgebaute
Session-Verwaltung ist es nicht. Wer diesen Weg geht, sollte Session-Ablauf,
Brute-Force-Bremse und die Cookie-Attribute bewusst festlegen — und `deploy/`
danach mit `/security-review` gegenlesen.

### Variante D — erster Login ohne Passwort

Der Vorschlag „beim ersten Mal ohne Passwort rein, dann muss man eines setzen"
hat ein Loch, das gegen ihn spricht: `docker-compose.yml` veröffentlicht den Port
als `${ILI_PORT:-8080}:80` — auf **allen** Interfaces, nicht nur auf
`127.0.0.1`. Zwischen `up -d` und dem Setzen des Passworts steht damit ein offener
Root-Shell-Zugang im ganzen LAN. Das Zeitfenster ist kurz, aber es ist real, und
wer den Stack startet und dann Kaffee holt, lässt es offen stehen.

Zwei Wege, dieselbe Bequemlichkeit ohne das Loch zu bekommen:

- **Einmal-Token statt gar keiner Schranke.** Beim ersten Start wird ein Token
  erzeugt und ins Log geschrieben; nur `/projterm/setup?token=…` ist ohne
  Anmeldung erreichbar, und der Token stirbt beim ersten erfolgreichen Setzen.
  Wer ins Log schauen kann, kommt ohnehin an alles — der Token verrät also
  nichts, was nicht schon offen läge. Das ist Variante C mit einem freundlichen
  Einstieg.
- **Während des Setups nur lokal binden.** `ILI_PORT` vorübergehend als
  `127.0.0.1:8080:80` — dann ist das Fenster auf den Rechner selbst begrenzt.
  Umständlich, aber ohne jeden Code machbar.

### Empfehlung

**B jetzt, C wenn ili an Fremde geht.** Variante B kostet wenig, erscheint genau
im richtigen Moment und ändert nichts an der Schranke. Solange ili im eigenen
Netz läuft, trägt das. Sobald andere Leute es installieren — und das ist der
erklärte Zweck des Repos — wird C zur richtigen Antwort, dann aber mit
Sicherheits-Review statt nebenbei.

D würde ich nicht in der wörtlichen Form bauen; der Einmal-Token liefert dieselbe
Bequemlichkeit ohne das offene Fenster.

### Vor dem Bauen zu verifizieren

Das liess sich hier nicht prüfen, weil das Basis-Image nicht ladbar war (P1):

- **Ist `ngx_http_auth_request_module` im offiziellen `nginx:alpine` enthalten?**
  Variante C steht und fällt damit. Prüfen mit:
  ```bash
  docker exec dashboard-web nginx -V 2>&1 | tr ' ' '\n' | grep auth_request
  ```
  Fehlt es, braucht C ein anderes nginx-Image oder einen anderen Ansatz.
- Verhalten von `auth_request` auf dem WebSocket-Upgrade-Pfad — der ist der
  empfindliche Teil, siehe die Safari-Notiz im Skriptkopf.


### Erweiterung: Konfigseite beim ersten Login

Vorschlag war, beim ersten Login eine Konfigseite zu öffnen und dort die
Grunddaten einzutragen. Zwei Dinge sprechen in den Entwurf hinein.

#### Es gibt bereits einen geführten Erstlauf

Eine frische Installation bringt das Projekt **Einrichtung** mit fünf
Unterprojekten mit — genau die Grunddaten, um die es geht:

| Unterprojekt | Karten (Auszug) |
|---|---|
| Daten & Datenbank | Wo deine Daten liegen · 🟡 JSON-Dateien oder Datenbank? · Projektcode ausserhalb des Klons · Sicherung ziehen |
| Domain & Aussenzugang | 🟡 Wie soll ili erreichbar sein? · `DASHBOARD_DOMAIN` setzen · Tunnel: Zugangsschutz ZUERST · Reverse Proxy: TLS im eigenen Netz · Terminal nur bewusst nach aussen geben |
| KI-Anbindung | 🟡 Welche KI soll ili nutzen? · Schlüssel eintragen · Modell zur Aufgabe wählen · 🟡 Kostenbremse? |
| Code & Beiträge | Deine Instanz ist ein git-Arbeitsbaum · 🟡 Wie hältst du eigene Änderungen? · Zugangstoken sicher ablegen |
| Betrieb | Was gesichert werden muss · 🟡 Wie sicherst du? · Updates einspielen · Logs im Blick |

Das ist eine bewusste Konstruktion: die 🟡-Karten sind Entscheidungskarten mit
Antwortknöpfen, und die Umsetzung macht die KI im Projekt-Terminal. Der Erstlauf
ist damit zugleich die Einführung in die Methode — das Board ist das Gedächtnis,
das Terminal die Werkbank.

Eine Konfigseite sollte das **nicht ersetzen.** Vieles davon lässt sich gar nicht
als Formular abbilden: „Wie sicherst du?", die Wahl zwischen Tunnel und Reverse
Proxy, die git-Strategie. Das sind Entscheidungen mit Folgearbeit, keine
Feldwerte.

#### Die harte Grenze: `.env` ist für die Container unerreichbar

Verifiziert:

- **Keine der Compose-Dateien mountet `.env`, und es gibt kein `env_file:`.**
  Compose liest die Datei auf dem *Host* und reicht einzelne Werte als
  Umgebungsvariablen weiter. Kein Container kann sie schreiben.
- **`constants.py:77` liest `ANTHROPIC_API_KEY` beim Modul-Import**, ebenso
  `project_links.py:30` das `DASHBOARD_DOMAIN`. Selbst wenn man die Variable
  im laufenden Prozess änderte, wirkte es nicht.
- `ILI_PORT`, `ILI_PROJECTS_DIR` und `COMPOSE_FILE` wirken ausschliesslich beim
  `up` — sie steuern Portbindung und Mounts, also etwas, das vor dem Start des
  Containers feststehen muss.

Damit zerfallen die Grunddaten in zwei Klassen:

| Klasse | Beispiele | Was eine Seite tun kann |
|---|---|---|
| **Laufzeit-Einstellungen** | Modellwahl, Effort/Temperatur, Kostenbremse, Theme, Widgets | Direkt speichern — `config_handler` (`ai_config.json`) und `user_settings_service` machen das heute schon |
| **Start-Einstellungen** | `ANTHROPIC_API_KEY`, `OLLAMA_URL`, `DASHBOARD_DOMAIN`, `ILI_PORT`, `ILI_PROJECTS_DIR`, `TERMINAL_PASSWORD` | **Nicht** schreiben — nur den fertigen `.env`-Block erzeugen und den Neustart-Befehl dazu anzeigen |

#### Entwurf, der damit funktioniert

Eine Erstlauf-Seite als **Generator mit Direktspeicherung für den Teil, der
geht**:

1. **Terminal-Zugang** — Benutzer und Passwort setzen. Das ist der einzige
   Grundwert, der sich sinnvoll erzwingen lässt (Variante C oben liefert die
   Mechanik).
2. **KI-Anbindung** — Schlüssel oder Ollama-URL abfragen; Modellwahl und
   Kostenbremse landen direkt in `ai_config.json`, der Schlüssel im
   `.env`-Block.
3. **Erreichbarkeit** — Domain und Port abfragen, aber nur als Vorbereitung:
   die Seite zeigt, was in die `.env` gehört.
4. **Ausgabe:** ein kopierfertiger `.env`-Block plus
   `docker compose up -d` — und der ehrliche Hinweis, dass die Start-Werte erst
   danach greifen.
5. **Weiterleitung** auf das Projekt *Einrichtung* für alles, was Entscheidung
   statt Feldwert ist.

Der Reiz: die Seite nimmt dem Erstlauf das Fummelige (Passwort, Schlüssel,
Domain-Schreibweise) ab, ohne zu behaupten, sie könne Dinge anwenden, die einen
Neustart brauchen. Der Ehrlichkeitsteil — „das hier musst du selbst in die `.env`
schreiben" — ist kein Makel, sondern die einzige korrekte Aussage.

#### Was es nicht werden sollte

Eine Seite, die so *aussieht*, als übernähme sie die Werte, es aber nicht tut.
Das ist die schlechteste Variante: der Nutzer trägt den Schlüssel ein, klickt
Speichern, und die KI-Funktionen bleiben trotzdem aus, weil `constants.py` den
Wert längst beim Start gelesen hat. Lieber gar kein Feld als ein Feld, dessen
Wirkung erst nach einem Neustart eintritt, den niemand angesagt hat.

---

## Empfehlung

1. **ili installieren, nicht den Upstream.** ili ist die für Fremdinstallation
   gedachte, bereinigte Fassung: eigener Compose-Projektname, Port 8080, benannte
   Volumes, `.env`, keine privaten Daten, kein kaputter `COPY`-Layer.
2. **Upstream-`html/services.html` behandeln, bevor daraus je ein Image weitergegeben
   oder das Repo öffentlich wird.** Die Datei ist generierter, personalisierter
   Inhalt und gehört in `.gitignore` + `.containerignore`, nicht ins Repo.
3. **Die echte Domain aus den sechs `<a href>`-Links entfernen** und die Guard-Regel
   in `tests/test_frontend_asset_urls.py` auf `<a href>` ausweiten — sonst tritt
   derselbe Fall wieder ein.
4. **`board_templates.json` / `automat_limits.json` im Upstream-`Containerfile`
   auf die `.example`-Varianten umstellen** ([P2](#p2)).

Alle vier Punkte sind hier **bewusst nicht umgesetzt** — die Vorgabe war, keinen
Code zu ändern.

---

## Installation von ili auf einem Mac (Docker Desktop)

Ungetestet in dieser Umgebung ([P1](#p1)), aber aus `QUICKSTART.md` und der
geprüften `docker-compose.yml` abgeleitet:

```bash
mkdir -p ~/containers && cd ~/containers
git clone https://github.com/Toa1984/ili-dashboard.git ili
cd ili
cp .env.example .env
docker compose up -d --build
```

Danach: **http://localhost:8080**

Nützlich:

```bash
docker compose logs api          # Backend-Log
docker compose logs web          # Frontend-Log
docker compose ps                # Status beider Container
docker compose down              # stoppen, Boards bleiben erhalten
docker compose down -v           # stoppen UND Boards/Daten löschen
```

Port 8080 belegt? `ILI_PORT=8090` in `.env` setzen, dann `docker compose up -d`.

Apple Silicon: `python:3.11-slim` und `nginx:alpine` gibt es beide als arm64 —
kein `--platform`-Flag nötig.

Der Anthropic-Key ist optional. Ohne ihn läuft das Kanban vollständig; nur die
KI-Funktionen bleiben aus. Mit: `ANTHROPIC_API_KEY=sk-ant-…` in `.env`, die Datei
ist gitignored.
