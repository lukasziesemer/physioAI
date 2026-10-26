# PhysioDoc

Lokale Web-App fuer Physiotherapie-Praxen. Behandlungsgespraeche aufnehmen, transkribieren (Faster-Whisper), Fachbegriffe korrigieren und strukturierte Dokumentation generieren (Ollama/Qwen3.5) — alles lokal, keine Cloud.

## Features

- **Live-Aufnahme** mit Mikrofon, Pegel-Anzeige und Echtzeit-Transkript-Vorschau
- **Datei-Upload** per Drag & Drop (MP3, WAV, M4A, OGG, WEBM)
- **3-Schritt-Pipeline**: Transkription → Fachbegriff-Korrektur → strukturierte Zusammenfassung
- **Dokumentationstyp** umschaltbar: Befundung oder Verlaufsdoku (eigener Prompt je Typ)
- **Ergebnis-Ansicht** mit 3 Tabs (Zusammenfassung, korrigiertes Transkript, Roh-Transkript), Zusammenfassung als Markdown gerendert
- **Inline-Editing** der Zusammenfassung und des korrigierten Transkripts direkt im Browser
- **Kopieren** pro Tab in die Zwischenablage und **PDF-Export** des Ergebnisses
- **Assistent**: Chat mit dem LLM (Streaming-Antworten, Vorschlags-Prompts)
- **Verlauf**: Archiv aller Ergebnisse mit Liste, Volltextsuche, Detailansicht und Loeschen
- **Session-Persistenz**: laufende Verarbeitung bleibt nach einem Seiten-Reload erhalten
- **Offline-Entwicklung** mit Mock-Services (`USE_MOCKS=true`)

## Tech Stack

| Komponente | Technologie |
|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS, react-markdown |
| Backend | FastAPI (Python), uvicorn |
| STT | Faster-Whisper (OpenAI-kompatible API, `/v1/audio/transcriptions`) |
| LLM | Ollama mit Qwen3.5:9b |
| PDF | fpdf2 |
| Speicher | JSON-Dateien (kein Datenbank-Setup noetig) |

## Voraussetzungen

- Python 3.12+
- Node.js 20+
- [Faster-Whisper Server](https://github.com/fedirz/faster-whisper-server) (fuer STT)
- [Ollama](https://ollama.ai/) mit `qwen3.5:9b` Modell (fuer LLM)

Fuer die Entwicklung ohne externe Services: `USE_MOCKS=true` in `.env` setzen.

## Setup

```bash
# Repository klonen
git clone https://github.com/lukasziesemer/physioAI.git
cd physioAI

# Umgebungsvariablen konfigurieren
cp .env.example .env
# .env anpassen (Server-IPs, Modell, etc.)

# Backend
cd backend
pip install -r requirements.txt
cd ..

# Frontend
cd frontend
npm install
cd ..
```

## Starten

```bash
# Terminal 1: Backend
cd backend
uvicorn app:app --host 0.0.0.0 --port 8080

# Terminal 2: Frontend
cd frontend
npm run dev
```

Frontend: http://localhost:5173 (Vite leitet `/api` an das Backend auf Port 8080 weiter)
Backend-API: http://localhost:8080/docs

## Docker

Das `Dockerfile` baut das Frontend und liefert es zusammen mit dem Backend aus einem Container aus (FastAPI serviert `static/`, wenn vorhanden):

```bash
docker compose up -d --build
```

App: http://localhost:8080

Uploads und Ergebnisse liegen in den Volumes `physiodoc-uploads` und `physiodoc-data`. `LLM_MODEL`, `UPLOAD_DIR`, `DATA_DIR` und `USE_MOCKS` sind in `docker-compose.yml` fest gesetzt; `WHISPER_API_URL` und `OLLAMA_API_URL` werden aus der `.env` gelesen (siehe unten).

## Entwicklung mit Mocks

Ohne Whisper/Ollama-Server entwickeln:

```bash
# In .env setzen:
USE_MOCKS=true
```

Die Mock-Services geben realistische Physio-Texte nach kurzer Verzoegerung zurueck.

## Tests

```bash
cd backend
pytest tests/ -v
```

## Umgebungsvariablen

| Variable | Beschreibung | Standard |
|---|---|---|
| `WHISPER_API_URL` | Faster-Whisper Server URL | `http://localhost:8100` |
| `OLLAMA_API_URL` | Ollama Server URL | `http://localhost:11434` |
| `LLM_MODEL` | Ollama Modell | `qwen3.5:9b` |
| `UPLOAD_DIR` | Pfad fuer Audio-Uploads | `./uploads` |
| `DATA_DIR` | Pfad fuer Ergebnis-JSON | `./data` |
| `USE_MOCKS` | Mock-Services aktivieren | `false` |

Mit `docker compose` zeigen die Standardwerte auf `host.docker.internal` (den Host-Rechner). Laufen Whisper oder Ollama auf einem anderen Rechner, die URLs in der `.env` neben `docker-compose.yml` setzen.

## API-Endpunkte

| Methode | Pfad | Zweck |
|---|---|---|
| GET | `/api/health` | Health Check |
| POST | `/api/recording/start` | Aufnahme-Session starten |
| POST | `/api/transcribe/chunk` | Audio-Chunk transkribieren (Live-Vorschau) |
| POST | `/api/recording/stop` | Aufnahme stoppen, Pipeline starten |
| POST | `/api/process` | Audio-Datei hochladen und verarbeiten |
| GET | `/api/recording/{session_id}/status` | Verarbeitungsfortschritt abfragen |
| GET | `/api/results` | Alle Ergebnisse auflisten (neueste zuerst) |
| GET | `/api/results/search?q=...` | Ergebnisse durchsuchen |
| GET | `/api/results/{id}` | Ergebnis abrufen |
| PUT | `/api/results/{id}` | Zusammenfassung / korrigiertes Transkript bearbeiten |
| DELETE | `/api/results/{id}` | Ergebnis inkl. Audio-Datei loeschen |
| GET | `/api/results/{id}/pdf` | Ergebnis als PDF herunterladen |
| POST | `/api/chat` | Chat mit dem Assistenten (Server-Sent Events) |

`/api/recording/stop` und `/api/process` akzeptieren optional das Formularfeld `doc_type` (`befund` oder `verlauf`, Standard `befund`).

## Projektstruktur

```
physioAI/
├── frontend/
│   └── src/
│       ├── components/       # React-Komponenten (RecordingView, ResultView, ...)
│       ├── hooks/            # useRecorder, useAudioMeter
│       └── services/         # API-Client
├── backend/
│   ├── app.py                # FastAPI App + Routen
│   ├── services/             # Whisper, LLM, Chat, PDF, Storage
│   ├── mock_services/        # Fake-Services fuer Offline-Dev
│   ├── prompts.py            # Korrektur-, Zusammenfassungs- (Befund/Verlauf) und Chat-Prompts
│   └── tests/
├── docs/                     # Specs und Plaene
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

## Lizenz

Privat
