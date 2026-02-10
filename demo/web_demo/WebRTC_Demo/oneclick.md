# oneclick.sh — MiniCPM-o WebRTC Demo One-Click Deployment

One script to download, build, and run the entire MiniCPM-o WebRTC demo stack **without Docker**.

## Quick Start

```bash
# Start all services (auto-downloads everything on first run)
PYTHON_CMD=/path/to/python bash oneclick.sh start

# Check service status
bash oneclick.sh status

# View logs (all services, or pick one)
bash oneclick.sh logs
bash oneclick.sh logs cpp        # C++ inference only
bash oneclick.sh logs backend    # backend only

# Stop everything
bash oneclick.sh stop

# Restart (stop + start)
PYTHON_CMD=/path/to/python bash oneclick.sh restart

# Pull latest code + auto-rebuild
bash oneclick.sh update

# Download dependencies only (no start)
bash oneclick.sh download
```

## Commands

| Command    | Description |
|------------|-------------|
| `start`    | Start all 4 services (auto-downloads missing dependencies) |
| `stop`     | Stop all services |
| `restart`  | Stop then start all services |
| `status`   | Show running status of each service |
| `logs`     | Tail logs: `logs all` / `logs livekit` / `logs backend` / `logs cpp` / `logs frontend` |
| `download` | Download source code + models only (no start) |
| `update`   | Pull latest code from git, auto-rebuild if needed |

## Architecture & Startup Order

```
┌──────────────────────────────────────────────────────────────────┐
│                        oneclick.sh start                         │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                    preflight_check()
                    ┌──────┴──────┐
                    │ Auto-download│  (if missing)
                    │ & auto-build │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
  WebRTC_Demo        llama.cpp-omni     GGUF Models
  (sparse clone)     (git clone +       (huggingface-cli
                      cmake build)       download)
                           │
         ┌─────────────────┼─────────────────────────┐
         │                 │                          │
         ▼                 ▼                          ▼
  [1/4] LiveKit     [2/4] Backend              [3/4] C++ Inference
  Server             (FastAPI)                  (llama-server)
  :7880              :8021                       :9060
         │                 │                          │
         │                 │                          │
         └────────┬────────┘                          │
                  │    token auth                      │
                  │◄──────────────────────────────────┘
                  │    register as inference service
                  │
                  ▼
           [4/4] Frontend
           (Vue + Vite)
           :8088 (HTTPS)
```

### Service Details

| # | Service | Port | Description |
|---|---------|------|-------------|
| 1 | **LiveKit Server** | 7880 | WebRTC SFU (Selective Forwarding Unit) for real-time audio/video |
| 2 | **Backend (FastAPI)** | 8021 | API server, room management, LiveKit token generation |
| 3 | **C++ Inference** | 9060 | llama-server + MiniCPM-o model (LLM + TTS + Vision + Audio) |
| 4 | **Frontend (Vue)** | 8088 | HTTPS web UI, proxies API/WebSocket to backend & LiveKit |

### Startup Flow

1. **Preflight Check** — verifies all dependencies, auto-downloads/installs missing ones:
   - Source code: `WebRTC_Demo`, `llama.cpp-omni`
   - Models: GGUF files from HuggingFace
   - Tools: `livekit-server`, `node`, `pnpm`, `python`, `cmake`
   - Binary: `llama-server` (auto-compiles if missing)

2. **LiveKit Server** — starts with auto-synced config:
   - Updates `livekit.yaml` API secret to match the script
   - Updates `node_ip` / `domain` to local IP

3. **Backend** — installs Python deps on first run, then starts FastAPI:
   - Connects to LiveKit via WebSocket
   - Provides REST API for the frontend

4. **C++ Inference** — the heaviest service:
   - Python wrapper (`minicpmo_cpp_http_server.py`) launches `llama-server`
   - Loads LLM, TTS, Vision, Audio, and Token2Wav models
   - Registers itself with the Backend as an available inference service
   - Health check waits up to 5 minutes for model loading

5. **Frontend** — dev mode (Vite hot reload) or prod mode (static build):
   - Generates HTTPS self-signed certificate (WebRTC requires secure context)
   - Proxies `/api/*` to Backend and `/rtc/*` to LiveKit

## Environment Variables

All variables have sensible defaults. Override as needed:

### Essential

| Variable | Default | Description |
|----------|---------|-------------|
| `PYTHON_CMD` | `python` | Python interpreter path (>= 3.9, recommend 3.11) |

### C++ Inference

| Variable | Default | Description |
|----------|---------|-------------|
| `LLAMACPP_ROOT` | `$SCRIPT_DIR/llama.cpp-omni` | llama.cpp-omni source directory |
| `MODEL_DIR` | `$SCRIPT_DIR/models/openbmb/MiniCPM-o-4_5-gguf` | GGUF model directory |
| `CPP_MODE` | `duplex` | Inference mode: `duplex` (full-duplex) or `simplex` |
| `VISION_BACKEND` | `coreml` (macOS) / `""` (Linux) | Vision encoder: `metal`, `coreml`, or `""` (default) |
| `N_GPU_LAYERS` | `99` | Number of LLM layers to offload to GPU |
| `TOKEN2WAV_DEVICE` | `gpu:1` | Token2Wav device: `gpu:1`, `gpu:0`, or `cpu` |

### Ports

| Variable | Default | Description |
|----------|---------|-------------|
| `LIVEKIT_PORT` | `7880` | LiveKit server port |
| `BACKEND_PORT` | `8021` | Backend API port |
| `FRONTEND_PORT` | `8088` | Frontend HTTPS port |
| `CPP_SERVER_PORT` | `9060` | C++ inference HTTP port (+1 for health, +10000 for llama-server) |

### Frontend

| Variable | Default | Description |
|----------|---------|-------------|
| `FRONTEND_MODE` | `dev` | `dev` (Vite hot reload) or `prod` (production build) |
| `FORCE_BUILD` | `0` | Set to `1` to force frontend rebuild in prod mode |

### Download Mirrors (for China)

| Variable | Default | Description |
|----------|---------|-------------|
| `GITHUB_PROXY` | `https://ghfast.top` | GitHub download proxy (empty = direct) |
| `HF_ENDPOINT` | `https://hf-mirror.com` | HuggingFace mirror (empty = official) |
| `NODE_MIRROR` | `https://npmmirror.com/mirrors/node` | Node.js binary mirror |
| `NPM_REGISTRY` | `https://registry.npmmirror.com` | npm registry mirror |

### Repository URLs

| Variable | Default | Description |
|----------|---------|-------------|
| `COOKBOOK_REPO` | `https://github.com/OpenSQZ/MiniCPM-V-CookBook.git` | WebRTC_Demo source repo |
| `COOKBOOK_BRANCH` | `webrtc-demo` | Branch to checkout |
| `LLAMACPP_REPO` | `https://github.com/tc-mb/llama.cpp-omni.git` | llama.cpp-omni repo |
| `HF_MODEL_REPO` | `openbmb/MiniCPM-o-4_5-gguf` | HuggingFace model repo |

## Common Scenarios

### First Run on a New Server

```bash
# Everything is automatic — downloads, builds, and starts
PYTHON_CMD=/path/to/conda/envs/py311/bin/python bash oneclick.sh start
```

### Duplex / Simplex Mode

```bash
# Duplex (default) — speak and listen simultaneously, natural conversation
PYTHON_CMD=/path/to/python CPP_MODE=duplex bash oneclick.sh start

# Simplex — turn-based, one speaks at a time
PYTHON_CMD=/path/to/python CPP_MODE=simplex bash oneclick.sh start
```

### 16GB macOS (Apple Silicon)

GPU memory is tight. Use Metal backend and offload Token2Wav to CPU:

```bash
PYTHON_CMD=/path/to/python \
  VISION_BACKEND=metal \
  TOKEN2WAV_DEVICE=cpu \
  bash oneclick.sh start
```

### Production Mode (Stable Frontend)

```bash
PYTHON_CMD=/path/to/python FRONTEND_MODE=prod bash oneclick.sh start
```

### Direct Connection (No China Mirrors)

```bash
PYTHON_CMD=/path/to/python \
  GITHUB_PROXY="" \
  HF_ENDPOINT="" \
  NODE_MIRROR="" \
  NPM_REGISTRY="" \
  bash oneclick.sh start
```

### Update to Latest Code

```bash
bash oneclick.sh update    # pulls code + auto-rebuilds if needed
bash oneclick.sh restart   # apply changes
```

### View Logs

```bash
bash oneclick.sh logs              # all services
bash oneclick.sh logs cpp          # C++ inference only
bash oneclick.sh logs backend      # backend only
```

## Directory Layout

```
$SCRIPT_DIR/
├── oneclick.sh                    # This script
├── WebRTC_Demo/                   # Project source (auto-downloaded)
│   ├── omini_backend_code/
│   │   ├── config/livekit.yaml    # LiveKit server config (auto-synced)
│   │   └── code/                  # Backend Python code
│   │       └── config/local.yaml  # Backend config
│   ├── o45-frontend/              # Vue frontend
│   └── cpp_server/                # C++ inference Python wrapper
├── llama.cpp-omni/                # C++ inference engine (auto-downloaded)
│   └── build/bin/llama-server     # Compiled binary (auto-built)
├── models/openbmb/MiniCPM-o-4_5-gguf/  # GGUF models (auto-downloaded)
│   ├── MiniCPM-o-4_5-Q4_K_M.gguf       # LLM (Q4_K quantized)
│   ├── tts/                             # TTS model + projector
│   ├── audio/                           # Audio processing model
│   ├── vision/                          # Vision encoder (GGUF + CoreML)
│   └── token2wav-gguf/                  # Token2Wav vocoder
├── .pids/                         # PID files for service management
├── .logs/                         # Log files
│   ├── livekit.log
│   ├── backend.log
│   ├── cpp_server.log             # Includes [CPP] prefixed llama-server output
│   └── frontend.log
└── .certs/                        # Auto-generated HTTPS certificates
```

## Troubleshooting

### Port Already in Use

The script auto-kills processes occupying ports on restart. If auto-kill fails:

```bash
# Find what's using the port
ss -tlnp | grep :8021
# Or on macOS
lsof -i :8021

# Kill it manually
kill <PID>
```

### 401 Unauthorized (LiveKit)

The script auto-syncs `livekit.yaml` API secret on every start. If you still see 401:
- Check that `livekit.yaml` `keys.devkey` matches `LIVEKIT_API_SECRET` in the script
- Restart LiveKit: `bash oneclick.sh restart`

### GPU Out of Memory (macOS)

Reduce GPU usage:

```bash
TOKEN2WAV_DEVICE=cpu VISION_BACKEND=metal bash oneclick.sh restart
```
