# Capabilities

Dieses Projekt stellt einen **OpenAI-kompatiblen REST-API-Server** für das Spracherkennungsmodell [parakeet-tdt-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) bereit. Es läuft lokal, benötigt keinen API-Key und kann als Drop-in-Ersatz für die OpenAI Audio Transcriptions API verwendet werden.

---

## Transkription

Der Server transkribiert Audiodateien via `POST /v1/audio/transcriptions`.

**Beispiel mit curl:**
```bash
curl -X POST http://localhost:5092/v1/audio/transcriptions \
  -F "file=@meine-aufnahme.mp3" \
  -F "model=parakeet-tdt-0.6b-v3"
```

**Beispiel mit dem OpenAI Python SDK:**
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:5092/v1", api_key="sk-kein-key-noetig")

with open("meine-aufnahme.mp3", "rb") as f:
    result = client.audio.transcriptions.create(
        model="parakeet-tdt-0.6b-v3",
        file=f,
        response_format="text"
    )
print(result)
```

**Details:**
- Endpunkt: `POST /v1/audio/transcriptions`
- Kein API-Key erforderlich
- Kompatibel mit dem OpenAI Python SDK, openai-go, etc.
- Maximale Dateigröße: 2.000 MB (~2 GB) (`app.py:182`)

---

## Ausgabeformate (`response_format`)

Der Parameter `response_format` steuert, wie das Transkript zurückgegeben wird.

**Beispiele:**
```bash
# Nur Text
curl ... -F "response_format=text"

# JSON mit Zeitstempeln und Segmenten
curl ... -F "response_format=verbose_json"

# SRT-Untertiteldatei
curl ... -F "response_format=srt"

# WebVTT-Untertiteldatei
curl ... -F "response_format=vtt"
```

**Details:**

| Format | Beschreibung |
|---|---|
| `json` | `{"text": "..."}` (Standard) |
| `text` | Reiner Text ohne JSON-Wrapper |
| `verbose_json` | Text + Segmente mit Start/End-Zeiten, Sprache, Dauer |
| `srt` | SRT-Untertitel (z.B. für VLC, Premiere) |
| `vtt` | WebVTT-Untertitel (z.B. für Browser-Video-Tag) |

`verbose_json` liefert Segment-Objekte der Form:
```json
{
  "id": 0,
  "start": 0.0,
  "end": 3.4,
  "text": "Hallo, das ist ein Test.",
  "tokens": [],
  "temperature": 0.0,
  "avg_logprob": 0.0,
  "compression_ratio": 0.0,
  "no_speech_prob": 0.0
}
```

| Key | Beschreibung |
|---|---|
| `id` | Fortlaufende Segment-Nummer (0-basiert) |
| `start` | Startzeit des Segments in Sekunden |
| `end` | Endzeit des Segments in Sekunden |
| `text` | Transkribierter Text dieses Segments |
| `tokens` | Token-IDs des Segments (immer leer, vom Modell nicht befüllt) |
| `temperature` | Sampling-Temperatur (immer `0.0`, Modell nutzt kein Sampling) |
| `avg_logprob` | Durchschnittliche Log-Wahrscheinlichkeit (immer `0.0`, nicht berechnet) |
| `compression_ratio` | Verhältnis von Rohtext zu komprimiertem Text (immer `0.0`, nicht berechnet) |
| `no_speech_prob` | Wahrscheinlichkeit, dass das Segment kein Sprach enthält (immer `0.0`, nicht berechnet) |

---

## Unterstützte Sprachen

Das Modell erkennt die Sprache automatisch – keine manuelle Angabe nötig.

**Unterstützte Sprachen (25):**
Englisch, Spanisch, Französisch, Russisch, Deutsch, Italienisch, Polnisch, Ukrainisch, Rumänisch, Niederländisch, Ungarisch, Griechisch, Schwedisch, Tschechisch, Bulgarisch, Portugiesisch, Slowakisch, Kroatisch, Dänisch, Finnisch, Litauisch, Slowenisch, Lettisch, Estnisch, Maltesisch

**Details:**
- Automatische Spracherkennung, keine Konfiguration nötig
- Rückgabe der erkannten Sprache im `verbose_json`-Format (`"language": "german"`)

---

## Unterstützte Audioformate

Praktisch jedes Audio- oder Videoformat wird akzeptiert.

**Beispiele:**
```
MP3, WAV, FLAC, OGG, M4A, MP4, MKV, WEBM, ...
```

**Details:**
- FFmpeg konvertiert intern zu: WAV, 16 kHz, Mono, PCM 16-bit
- Bei Videodateien wird die Tonspur extrahiert
- Alle von FFmpeg unterstützten Formate funktionieren

---

## Modellvarianten und Präzision (`model`)

Drei Varianten stehen zur Wahl: schnell (INT8, Standard), ausgewogen (FP16) und vollständig (FP32).

**Beispiel:**
```bash
# Standard (INT8, empfohlen)
curl ... -F "model=parakeet-tdt-0.6b-v3"

# Halbpräzision
curl ... -F "model=grikdotnet/parakeet-tdt-0.6b-fp16"

# Volle Präzision
curl ... -F "model=istupakov/parakeet-tdt-0.6b-v3-onnx"
```

**Details:**

| Modell | Präzision | Speedup | WER (LibriSpeech) |
|---|---|---|---|
| `parakeet-tdt-0.6b-v3` | INT8 | 18.4x | 2.16 % |
| `grikdotnet/parakeet-tdt-0.6b-fp16` | FP16 | 18.8x | 2.16 % |
| `istupakov/parakeet-tdt-0.6b-v3-onnx` | FP32 | 19.4x | 2.16 % |

- INT8 hat **keinen Genauigkeitsverlust** gegenüber FP32
- Modelle werden beim ersten Aufruf geladen und dann im Speicher gehalten (Lazy Loading + Cache)
- Der INT8-Standard wird beim Serverstart vorgeladen

---

## Intelligentes Chunking langer Audiodateien

Lange Dateien werden automatisch an Sprechpausen aufgeteilt, damit kein Wort abgeschnitten wird.

**Details:**
- Chunk-Länge: ~90 Sekunden
- Stille wird per FFmpeg `silencedetect` erkannt (Schwellwert: -40 dB, Mindestdauer: 0,5 s)
- Das System sucht innerhalb eines ±30-Sekunden-Fensters um den Ziel-Schnittpunkt nach einer Pause
- Fallback: zeitbasiertes Splitting, wenn keine Stille gefunden wird
- Mindestabstand zwischen Schnitten: 5 Sekunden

---

## Echtzeit-Fortschritt und Monitoring

Während der Transkription kann der Fortschritt abgefragt werden.

**Beispiel:**
```bash
# Job-ID aus dem Response-Header lesen
JOB_ID=$(curl -si -X POST ... | grep -i x-job-id | awk '{print $2}' | tr -d '\r')

# Fortschritt abfragen
curl http://localhost:5092/progress/$JOB_ID
```

```json
{
  "status": "processing",
  "current_chunk": 2,
  "total_chunks": 5,
  "progress_percent": 40,
  "partial_text": "Bisher erkannter Text..."
}
```

**Weitere Endpunkte:**

| Endpunkt | Beschreibung |
|---|---|
| `GET /health` | Liveness-Check (`{"status": "ok"}`) |
| `GET /status` | Fortschritt des aktuell laufenden Jobs |
| `GET /metrics` | CPU- und RAM-Auslastung in Echtzeit |

---

## Web-Oberfläche

Unter `http://localhost:5092/` steht eine Browser-UI zur Verfügung.

**Funktionen:**
- Drag-and-Drop Upload
- Modellauswahl (INT8 / FP16 / FP32)
- Live-Fortschrittsanzeige (Prozent, Chunk-Zähler)
- CPU- und RAM-Auslastung als Live-Gauges
- Ergebnisanzeige mit Zeitstempeln (Chat-Bubble-Stil)
- Export: In Zwischenablage kopieren, als TXT oder SRT herunterladen
- Interaktive API-Dokumentation unter `/docs` (Swagger UI)

---

## Deployment: Docker

Der Server ist als CPU- und GPU-Variante containerisiert.

**Beispiel:**
```bash
# CPU
docker compose up parakeet-cpu -d

# GPU (benötigt NVIDIA Container Toolkit)
docker compose up parakeet-gpu -d

# Oder direkt über ghcr.io
docker run -p 5092:5092 ghcr.io/<owner>/parakeet-tdt-0.6b-v3-fastapi-openai:cpu
```

**Details:**
- CPU-Image: ~2 GB RAM
- GPU-Image: Basis `nvidia/cuda:12.1.1-cudnn8-runtime-ubuntu22.04`, ~4 GB VRAM
- Modelle werden in einem persistenten Docker-Volume (`parakeet-models`) gecacht
- Der Prozess läuft als nicht-privilegierter User (`parakeet`)
- Health-Check integriert

---

## Integration in andere Tools

**Open WebUI (z.B. für Ollama):**
1. Einstellungen → Audio → STT Engine: `OpenAI`
2. OpenAI Base URL: `http://localhost:5092/v1`
3. STT Model: `parakeet-tdt-0.6b-v3`

**Anki, Obsidian, n8n, Make, Zapier** – jedes Tool, das die OpenAI Audio API unterstützt, funktioniert ohne Änderungen.

---

## Nicht unterstützte Parameter

Folgende OpenAI-Parameter werden vom Endpunkt **nicht** ausgewertet:

| Parameter | Status |
|---|---|
| `language` | Ignoriert – Sprache wird automatisch erkannt |
| `prompt` | Nicht unterstützt |
| `temperature` | Nicht unterstützt |
| `timestamp_granularities` | Nicht unterstützt (Zeitstempel immer auf Segment-Ebene) |
| Stoppwörter | Nicht unterstützt |

Das Modell hat **keine Stoppwort-Funktion** – es transkribiert den gesamten Audioinhalt ohne Filter.
