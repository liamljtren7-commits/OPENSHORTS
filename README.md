 

<div align="left">

 # OpenShorts on your own GPU: a zero-paid-API setup guide 

</div>

>> Turn long videos into 9:16 Shorts, Reels and TikToks with local Whisper, a local LLM (Ollama) and FFmpeg on an NVIDIA GPU. 

---

<div align="left">

## What you get 

* A long video goes in. 3 to 15 vertical clips come out, with word-level captions and optional hook titles.
* Transcription: faster-whisper on your GPU.
* Picking moments, titles, descriptions and hook text: an Ollama model on your GPU.
* Reframing and encoding: MediaPipe, YOLOv8 and FFmpeg (NVENC if available). All local.

</div>

---

<div align="left">

##   **Requirements**

> A GPU. 8 GB of VRAM is comfortable

> Linux (Ubuntu) with Docker Engine and Docker Compose v2, or Windows with [Docker Desktop](https://www.docker.com/products/docker-desktop/) on the WSL2 backend.

> Google Gemini API KEY. Get from [HERE](https://aistudio.google.com/api-keys)

</div>

---

<div align="left">

## Prepare the host

### Step 1: NVIDIA driver

```
nvidia-smi
```

* You should see your GPU. If not, install the latest NVIDIA driver and reboot.

### Step 2: Install Docker Desktop & Docker Compose & set it up

* use GPT to install

> Windows + WSL2 : install the Windows NVIDIA driver and Docker Desktop with the WSL2 backend, then enable your Ubuntu distro under `Settings > Resources > WSL Integration`. 

</div>

---

<div align="left">

### Clone the Repo

```
git clone https://github.com/mutonby/openshorts.git
cd openshorts
```

### Create the working directories:

```
mkdir -p output uploads .cache/huggingface models
chmod -R a+rwX output uploads .cache models
```

---

### Fix the known Remotion issue

> The earlier repository audit found a possible version mismatch:

```
cd ~/openshorts/render-service
```


```
docker run --rm \
  -v "$(pwd):/w" \
  -w /w \
  node:18-bookworm-slim \
  npm install --package-lock-only --save-exact \
    @remotion/bundler@4.0.541 \
    @remotion/renderer@4.0.541 \
    remotion@4.0.541

cd ..
```

> This updates the renderer package lock before the renderer image is
built.

---

### Create `.env`

```
cat > .env <<'EOF'
# Local Ollama moment picker
LLM_BASE_URL=http://ollama:11434/v1
LLM_MODEL=qwen2.5:7b

# Local faster-whisper on NVIDIA GPU
TRANSCRIBE_BACKEND=whisper
WHISPER_MODEL=large-v3-turbo
WHISPER_DEVICE=cuda
WHISPER_COMPUTE=float16
ASR_GPU_CONCURRENCY=1

# Use NVENC when available; allow the app to fall back to x264
FFMPEG_ENCODER=auto

# One job at a time to reduce GPU memory contention
MAX_CONCURRENT_JOBS=1
EOF

AUTO_LAYOUT=1
GEMINI_MODEL=gemini-3.1-flash-lite
SUBTITLE_BAND_CHECK=0
```

> If u want to change the `GEMINI_MODEL` then u can make change in .env with `nano .env` 


---

### Create the compose override

```
cat > docker-compose.override.yml <<'EOF'
services:
  backend:
    build:
      context: .
      args:
        GPU: "1"
    env_file: .env
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu, video]

  ollama:
    image: ollama/ollama
    container_name: openshorts-ollama
    restart: unless-stopped
    environment:
      - OLLAMA_CONTEXT_LENGTH=16384
    volumes:
      - ollama_models:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  ollama_models:
EOF
```


---


###  Start Ollama and download the model

```
docker compose up -d ollama
```

```
docker exec openshorts-ollama ollama pull qwen2.5:7b
```

> If u have higher specs pc then u can choose `qwen2.5:14b` 

```
docker exec openshorts-ollama ollama run qwen2.5:7b 'Reply with the word ready'
```

```
docker exec openshorts-ollama ollama ps
```

> The `PROCESSOR` column of `ollama ps` should say 100% GPU.



---

## Build and start OpenShorts

```
docker compose up --build -d
```

> Its gonna take long time to build. Have patience

```
docker compose ps
```

> You gonna see four containers running with Its log fills with `GET /health/ready 200 OK` every few seconds. 

---

### Fixing the YOLO model error (IMP)

```
cd ~/openshorts
```

```
chmod a+r yolov8n.pt
chmod a+rwx .
```

```
docker run --rm --entrypoint ls openshorts-backend -la /app/yolov8n.pt
```

```
docker exec openshorts-backend sh -c 'touch /app/.w && echo writable && rm /app/.w || echo NOT-writable'
```

```
docker run --rm --entrypoint cat openshorts-backend /app/yolov8n.pt > yolov8n.pt
```

```
ls -la yolov8n.pt
```

---

### Restart docker compose

```
docker compose restart
```

---

## Verify the installation


### All 4 containers running? 

```
docker compose ps
```

### Check Whisper CUDA availability

```
docker exec openshorts-backend python -c "import ctranslate2; print('cuda devices:', ctranslate2.get_cuda_device_count())"
```

> Expected result:  `CUDA devices: 1` 


---

## Process your first video

### Open the dashboard: http://localhost:5175

> Go to settings >> put your Gemini API Key

> Back to Clip generator & Enjoy free Clipping. 

<img width="763" height="1150" alt="image" src="https://github.com/user-attachments/assets/a1033b86-aa91-4a5f-a08c-091e158142d5" />



---


## How to start next day? 

1.  Start the docker 

2. Move to working directory

```
cd openshorts
```

3. Up the containers

```
docker compose start
```

> All four containers gonna UP 

4. Move to the Deshboard

 > http://localhost:5175


### Stop all containers with 

```
docker compose stop
```

---


[Source](https://github.com/mutonby/openshorts)

</div>










