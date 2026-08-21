# Installations-Befund: ili & Upstream-`dashboard`

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
[AENDERUNGEN.md](AENDERUNGEN.md).

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
