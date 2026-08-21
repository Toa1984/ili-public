# Änderungen auf `claude/ili-installation-no8uax`

Stand: 2026-08-21 · Basis: `main` (`8822c50`)

Diese Datei listet **jede** Änderung dieses Branches, damit sie im
Werkstatt-Repo (`Toa1984/dashboard`) nachgezogen werden kann. Pro Eintrag:
was, warum, und was beim Übertragen zu beachten ist.

Zum Nachlesen der Befunde: [INSTALLATION-BEFUND.md](INSTALLATION-BEFUND.md).

---

## Überblick

| # | Datei | Art | Portierbar nach `dashboard`? |
|---|---|---|---|
| 1 | `html/nav.js` | Terminal-Link auf `/projterm/` | **Nur nach Entscheidung** — siehe [A1](#a1) |
| 2 | `html/nav.js` | Shelly-Einträge entfernt | Ja, wenn der Scanner auch dort aus der Nav soll |
| 3 | `html/nav.js` | Tote `BASE`-Ableitung entfernt | Nur zusammen mit 1 **und** 2 |
| 4 | `html/i18n/de.js` | Verwaiste Sprachschlüssel entfernt | Zusammen mit 2 |
| 5 | `README.md` | Doku zu entfernten Routen gelöscht | Ja, unabhängig |
| 6 | `docs/INSTALLATION-BEFUND.md` | Neu | Nur als Information |
| 7 | `docs/AENDERUNGEN.md` | Neu (diese Datei) | Nur als Information |

Kein Backend-Code, keine Konfiguration, keine Abhängigkeiten angefasst.

---

## A1 — Terminal-Link zeigt auf `/projterm/` {#a1}

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

## A2 — Shelly-Einträge aus der Nav entfernt

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

## A3 — Tote `BASE`-Ableitung entfernt

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

## A4 — Verwaiste Sprachschlüssel entfernt

**Datei:** `html/i18n/de.js`

```diff
-    "nav.scan": "LAN-Scan",
-    "nav.shelly": "Shelly",
```

**Warum:** Nach A2 referenziert sie nichts mehr. `de.js` ist die einzige
Sprachdatei in `html/i18n/`.

---

## A5 — README: Dokumentation zu entfernten Routen gelöscht

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

## A6/A7 — Neue Dokumente

`docs/INSTALLATION-BEFUND.md` — Prüfbericht: läuft ili, liegen private Daten in
den Repos, welche Installationsprobleme bestehen.

`docs/AENDERUNGEN.md` — diese Datei.

Beide sind reine Dokumentation ohne Code-Bezug. Übernehmen nur, wenn der Befund
auch im Werkstatt-Repo festgehalten werden soll.

---

## Bewusst **nicht** geändert

Diese Punkte betreffen fast alle das Werkstatt-Repo und brauchen eine
Entscheidung. Details jeweils in [INSTALLATION-BEFUND.md](INSTALLATION-BEFUND.md).

| Was | Wo | Warum offen |
|---|---|---|
| `html/services.html` enthält E-Mail, Login-Namen, interne IPs und Zugangsdaten-Paare | nur `dashboard` | Datei ist generierter, personalisierter Inhalt; gehört in `.gitignore` + `.containerignore` statt ins Repo. Landet aktuell über `COPY html/ html/` im Image **und** wird per Bind-Mount ausgeliefert |
| Echte Domain in sechs `<a href>`-Links | nur `dashboard` | `tests/test_frontend_asset_urls.py` bewacht nur `<script src>` und `<link href>` und erlaubt `<a href>` ausdrücklich — die Guard-Regel greift für genau diesen Fall nicht |
| `Containerfile` kopiert die gitignoreten `board_templates.json` / `automat_limits.json` | nur `dashboard` | Build ist nach frischem Clone kaputt. Naheliegend: die `.example`-Varianten kopieren, analog zu `services_config.example.json` |
| `docker-compose.yml` bindet Port 80, Header sagt „nicht in Benutzung" | nur `dashboard` | ili hat das bereits gelöst (Port 8080 via `ILI_PORT`, fester Projektname) |
| README-Warnung, `html/index.html` fehle nach frischem Clone | nur `dashboard` | Datei ist vorhanden, `.gitignore` nimmt sie ausdrücklich aus |
| Vorname des Autors in `docs/WIDGETS.md`, `docs/JOBS.md`, `docs/KATEGORIEN-PRIO.md` und im Ollama-Modellnamen `timo-assistant:latest` | `ili-dashboard` | Kein Sicherheitsproblem, aber ein Herkunftshinweis in einem öffentlichen Repo |

---

## Wirksam werden lassen

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
