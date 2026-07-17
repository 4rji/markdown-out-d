FROM python:3.12-slim

# ffmpeg: normalizes audio before local transcription
# libgomp1: required by faster-whisper (ctranslate2)
# tesseract-ocr: optional OCR for images (+ Spanish language pack)
RUN apt-get update && apt-get install -y --no-install-recommends \
        ffmpeg \
        libgomp1 \
        tesseract-ocr \
        tesseract-ocr-spa \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/markdown-converter

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt gunicorn

# DOWNLOAD_MODEL=1 bakes the Whisper model into the image.
# Default (0): mount the model as a volume at /opt/models instead.
ARG DOWNLOAD_MODEL=0
RUN if [ "$DOWNLOAD_MODEL" = "1" ]; then \
        pip install --no-cache-dir huggingface_hub && \
        hf download Systran/faster-whisper-base \
            --local-dir /opt/models/faster-whisper-base && \
        pip uninstall -y huggingface_hub; \
    fi

COPY app.py .
COPY static/ static/
COPY templates/ templates/

ENV WHISPER_MODEL_PATH=/opt/models/faster-whisper-base

EXPOSE 8082

# Single worker so the Whisper model is loaded once and the audio lock
# applies globally; threads handle concurrent light requests. The long
# timeout keeps gunicorn from killing the worker mid-transcription.
CMD ["gunicorn", "--bind", "0.0.0.0:8082", \
     "--workers", "1", "--threads", "4", \
     "--timeout", "600", "--graceful-timeout", "30", \
     "--access-logfile", "-", "--error-logfile", "-", \
     "app:app"]
