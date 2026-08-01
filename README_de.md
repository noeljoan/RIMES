# RIMES

Ein modernes REST-API-Backend samt schlanker Weboberfläche für das
französische Reimwörterbuch.

> Datenbank mit 122.000+ Wörter.
> Diese Version: Python/FastAPI, SQLite oder PostgreSQL, Docker-fähig,
> plus eine eigenständige HTML-Weboberfläche.

---

## Was die Anwendung kann

- **Echte Reimsuche** — Wort eingeben, tatsächlich reimende Wörter aus dem
  260.000-Wörter-Wörterbuch erhalten (Endungsabgleich, kein starres
  Mustersystem).
- **Legacy-Wildcard-Prüfung** — die ursprüngliche VB3-Wildcard-Syntax
  (`*`, `?`, `!`, `$`, `[BD]`, `[!AE]` …) bleibt unterstützt, für alle,
  die eigene Muster definieren wollen.
- **Wörter hinzufügen** — neue Einträge live ins Wörterbuch aufnehmen.
- **Statistiken** — Wortanzahl, Musteranzahl, durchschnittliche Wortlänge.
- **Weboberfläche** (`rimes.html`) — einfache, lokal im Browser lauffähige
  Oberfläche ohne eigenen Build-Prozess.
- **Docker-Deployment** — PostgreSQL + API + Nginx-Reverse-Proxy als
  Gesamtpaket.

---

## Schnellstart (lokale Entwicklung)

### Voraussetzungen
- Python 3.12+

### 1. Abhängigkeiten installieren
```bash
pip install -r backend/requirements.txt
```

### 2. Datenbank vorbereiten
Die mitgelieferte `rime_data.db` ist bereits befüllt (260.352 Wörter,
14 Muster). Falls du sie neu aufbauen willst:
```bash
python backend/migrate_data.py
```

### 3. Server starten
```bash
uvicorn backend.main:app --host 0.0.0.0 --port 8080
```

Wichtig: Der Befehl muss **im `RIMES`-Hauptordner** ausgeführt werden
(dort, wo der `backend`-Unterordner liegt), nicht innerhalb von `backend/`.

### 4. Weboberfläche öffnen
`rimes.html` einfach per Doppelklick im Browser öffnen — sie spricht
automatisch `http://127.0.0.1:8080` an, solange der Server läuft.

### 5. Testen
```bash
curl http://localhost:8080/
# {"message": "Welcome to RIMES Rhyme API", "version": "1.0.0"}

curl -X POST http://localhost:8080/rime/rhymes \
  -H "Content-Type: application/json" \
  -d '{"word": "MAISON"}'
```

---

## API-Dokumentation

Sobald der Server läuft:
- Swagger UI: `http://localhost:8080/docs`
- OpenAPI JSON: `http://localhost:8080/openapi.json`

### `POST /rime/rhymes` — echte Reimsuche

Findet Wörter aus dem Wörterbuch, die sich auf das eingegebene Wort
reimen. Vergleicht dazu die Wortendung (bis zu 6 Buchstaben, verkürzt
sich automatisch, bis genug Treffer vorliegen). Funktioniert für
**jedes** Wort, ganz ohne vordefiniertes Muster.

```bash
curl -X POST http://localhost:8080/rime/rhymes \
  -H "Content-Type: application/json" \
  -d '{"word": "MAISON"}'
```
```json
{
  "word": "MAISON",
  "rhyme_suffix": "AISON",
  "word_count": 53,
  "words": ["CARGAISON", "COMPARAISON", "LIVRAISON", "RAISON", "SAISON", "..."],
  "truncated": false
}
```

### `POST /rime/check` — Legacy-Wildcard-Prüfung

Prüft ein Wort gegen die in `rime_patterns` gespeicherten Muster
(Legacy-Wildcard-Syntax). Liefert pro Treffer zusätzlich eine
Stichprobe echter, passender Wörter aus dem Wörterbuch.

```bash
curl -X POST http://localhost:8080/rime/check \
  -H "Content-Type: application/json" \
  -d '{"word": "NATION"}'
```

> Hinweis: Die von Haus aus enthaltenen Muster (`tG=F`, `tG=K`, `XKEY=42`
> …) sind reine Legacy-Kategorielabels aus der alten EXE — sie sind
> **keine** verwertbaren Wildcards und liefern daher normalerweise keine
> Treffer. Für sinnvolle Ergebnisse über `/check` eigene Wildcard-Muster
> in `rime_patterns` anlegen (z. B. `*TION`, `*MENT`, `*[!AE]*`). Für die
> normale Reimsuche `/rime/rhymes` verwenden — das ist der praktische
> Standardweg.

### `POST /rime/add` — Wort hinzufügen

```bash
curl -X POST http://localhost:8080/rime/add \
  -H "Content-Type: application/json" \
  -d '{"word": "FRANZ"}'
```
Antworten: `200` bei Erfolg, `409` falls das Wort bereits existiert,
`400` bei leerem Wort.

### `GET /rime/stats` — Statistiken

```bash
curl http://localhost:8080/rime/stats
```
```json
{
  "total_words": {"value": 260352, "description": "Total number of words in dictionary"},
  "total_patterns": {"value": 14, "description": "Number of phonetic patterns"},
  "avg_word_length": {"value": 9, "description": "Average word length"}
}
```

### `GET /` und `GET /health`

Willkommensnachricht bzw. Health-Check (letzterer wird auch von
`nginx.conf` für den Docker-Reverse-Proxy verwendet).

---

## Wildcard-Syntax (Legacy, für `/rime/check`)

| Zeichen    | Bedeutung                          |
|------------|-------------------------------------|
| `*`        | eine oder mehrere Buchstaben        |
| `?`        | genau ein Buchstabe                 |
| `!`        | kein Vokal (nicht `AEIOUY`)         |
| `$`        | nur `B` oder `D`                    |
| `[!AE]`    | schließt `A` und `E` aus            |
| `[BD]`     | nur `B` oder `D`                    |

---

## Mit Docker ausführen

```bash
docker-compose up --build
```

- PostgreSQL, API und Nginx-Reverse-Proxy starten gemeinsam
- API erreichbar über Nginx auf Port 80/443
- Migration gegen PostgreSQL:
  ```bash
  set DATABASE_URL=postgresql://rimeuser:rimepass@localhost:5432/rime_data
  python backend/migrate_postgres.py
  ```

```bash
docker-compose down
```

---

## Entwicklung

### Tests ausführen
```bash
pytest backend/tests/ -v
```

### Mit In-Memory-SQLite (schnelle Testläufe)
```bash
python run_local.py --mode test
python run_local.py --mode migrate
python run_local.py --mode server
```

---

## Projektstruktur

```
RIMES/
├── backend/
│   ├── main.py                  # FastAPI-App, CORS, /health
│   ├── models.py                # SQLAlchemy-ORM-Modelle
│   ├── database.py              # DB-Engine/Session
│   ├── api/
│   │   └── routes.py            # /rime/rhymes, /check, /add, /stats
│   ├── regex_converter.py       # Legacy-Wildcard → Regex
│   ├── migrate_data.py          # SQLite-Migration
│   ├── migrate_postgres.py      # PostgreSQL-Migration
│   ├── simulate_migration.py    # In-Memory-Simulation
│   ├── Dockerfile
│   ├── requirements.txt
│   └── tests/
├── rimes.html                   # Eigenständige Weboberfläche
├── docker-compose.yml
├── nginx.conf
├── init-scripts/01-init-schema.sql
├── extracted_words.txt          # 260.352+ französische Wörter (Quelldaten)
├── rime_data.db                 # Befüllte SQLite-DB (bereit zur Nutzung)
└── README.md
```

---

## Bekannte Einschränkungen

- Die vordefinierten `rime_patterns`-Einträge (`tG=F`, `XKEY=42` …) sind
  Legacy-Kategorielabels ohne echte Wildcard-Bedeutung — für die normale
  Reimsuche `/rime/rhymes` statt `/check` verwenden.
- `/rime/rhymes` nutzt einen einfachen In-Memory-Cache der Wortliste, der
  beim ersten Aufruf nach Serverstart ca. 1–2 s lädt und danach beim
  Hinzufügen eines Worts automatisch aktualisiert wird.
- Die Reimerkennung basiert auf Schriftbild-Endungen (Suffix-Abgleich),
  nicht auf echter Phonetik/IPA — für eine französische Wortliste ist das
  in der Praxis ein guter Näherungswert, aber kein linguistisch exaktes
  Reimmodell.

---

## Lizenz

MIT License

- MIT-Lizenz erlaubt freie Nutzung, Veränderung und Weiterverbreitung
