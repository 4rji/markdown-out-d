# DigiTech Markdown Converter — Docker

Self-contained Docker setup for the Markdown converter. This folder holds a
copy of everything the image needs (`app.py`, `requirements.txt`, `static/`,
`templates/`), so you can copy just this folder to a server and build there.

The container runs the app with **gunicorn** (1 worker, 4 threads) instead of
Flask's development server. One worker keeps a single copy of the Whisper
model in memory and preserves the global audio-processing lock; the threads
handle concurrent uploads and downloads.

> **Note:** `app.py`, `static/`, and `templates/` here are copies of the ones
> at the repo root. If you change the app, re-copy them before rebuilding:
> `cp ../app.py ../requirements.txt . && cp -R ../static ../templates .`

## Prerequisites

- Ubuntu 22.04+ (or any Linux with Docker)
- Docker Engine with the Compose plugin:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER   # then log out and back in
```

## Whisper model

Audio transcription runs fully locally with faster-whisper and needs the model
files on disk. Two options:

**Option A — model on the host, mounted as a volume (default, smaller image):**

```bash
sudo mkdir -p /opt/models/faster-whisper-base
pip install huggingface_hub
hf download Systran/faster-whisper-base --local-dir /opt/models/faster-whisper-base
```

The compose file mounts `/opt/models/faster-whisper-base` read-only into the
container. If your model lives elsewhere, edit the volume path in
`docker-compose.yml`.

**Option B — model baked into the image (no volume needed):**

```bash
docker build --build-arg DOWNLOAD_MODEL=1 -t markdown-converter .
```

Then remove the `volumes:` section from `docker-compose.yml` (or run with
plain `docker run`, below).

Without a model the app still starts and converts documents/images; only
audio uploads fail with a configuration error.

## Build and run

With compose (from this folder):

```bash
docker compose up -d --build
```

Or with plain Docker:

```bash
docker build -t markdown-converter .
docker run -d --name markdown-converter \
  -p 8082:8082 \
  -v /opt/models/faster-whisper-base:/opt/models/faster-whisper-base:ro \
  --restart unless-stopped \
  markdown-converter
```

The app is then available at `http://<server>:8082`.

## Configuration

Set these under `environment:` in `docker-compose.yml` (or with `-e` flags):

- `WHISPER_MODEL_PATH` — model directory inside the container
  (default `/opt/models/faster-whisper-base`).
- `WHISPER_LANGUAGE` — force a transcription language, e.g. `es`
  (default: auto-detect).
- `MAX_UPLOAD_SIZE_MB` — upload size limit (default `500`).
- `FFMPEG_TIMEOUT_SECONDS` — audio transcode timeout (default `600`).
- `WHISPER_BEAM_SIZE` — transcription beam size (default `5`).

## Operations

```bash
docker compose logs -f          # follow logs
docker compose restart          # restart
docker compose down             # stop and remove
docker compose up -d --build    # rebuild after changes
```

Converted files live in a temp directory inside the container and are cleaned
up automatically after 30 minutes; nothing needs a persistent volume besides
the optional model mount.
