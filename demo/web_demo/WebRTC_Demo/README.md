# WebRTC Real-Time Video Interaction Demo

A full-duplex real-time video interaction solution based on WebRTC, enabling seamless streaming input/output with high responsiveness and low latency.

📖 [中文版本](./README_zh.md)

## Overview

This demo implements a **full-duplex real-time video interaction** solution using WebRTC technology. It fills a significant gap in the open-source community by providing a **streaming duplex conversation** capability that was previously unavailable.

> [!IMPORTANT]
> **GGUF Model Update**: The GGUF model files have been recently updated (including `prompt_cache.gguf` and other components). If you downloaded the models before, please re-download the latest version to ensure compatibility. Using outdated model files may cause initialization failures or degraded audio quality.

## Hardware Requirements

The full model set (LLM Q4_K_M + Vision/Audio/TTS F16 + Token2Wav) totals **~8.3 GB**, with a runtime GPU memory footprint of approximately **10 GB** (including KV cache and compute buffers).

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

| Mode | Minimum | Recommended | Notes |
|------|---------|-------------|-------|
| **Simplex** | M1/M2/M3/M4, 16GB RAM | M4 series, 32GB+ RAM | All Apple Silicon chips supported; newer chips provide faster inference |
| **Duplex** | M4 Pro, 36GB+ RAM | **M4 Max, 64GB+ RAM** | Real-time streaming requires high memory bandwidth; M4 Max verified to achieve RTF < 1.0 |

> **Note**: macOS uses unified memory — model weights, KV cache, and compute buffers all share system RAM. For duplex mode, the primary bottleneck is compute throughput rather than memory capacity. M1/M2/M3 chips lack the bandwidth and compute power needed for real-time duplex streaming.

</details>

<details>
<summary><b>Linux / Windows (NVIDIA GPU)</b></summary>

| Mode | Minimum VRAM | Recommended | Example GPUs |
|------|-------------|-------------|--------------|
| **Simplex** | 10 GB | 12 GB+ | RTX 3060 12GB, RTX 4070 12GB |
| **Duplex** | 12 GB | 16 GB+ | RTX 4080 16GB, RTX 4090 24GB, RTX 3090 24GB |

**GPU tier reference**:

| GPU | VRAM | Simplex | Duplex | Notes |
|-----|------|---------|--------|-------|
| RTX 4060 | 8 GB | ❌ | ❌ | Insufficient VRAM |
| RTX 3060 | 12 GB | ✅ | ⚠️ Marginal | May require CPU offload for some modules |
| RTX 4070 | 12 GB | ✅ | ✅ | Entry-level for duplex |
| RTX 4080 | 16 GB | ✅ | ✅ | Recommended for duplex |
| RTX 3090 | 24 GB | ✅ | ✅ | Comfortable |
| RTX 4090 | 24 GB | ✅ | ✅ | Best performance |

> **Note**: CUDA GPUs are generally faster than Apple Silicon Metal for this workload. An RTX 4070 can comfortably achieve real-time duplex streaming.

</details>

## Prerequisites

### 1. Install Docker

<details>
<summary><b>macOS</b></summary>

```bash
# Install via Homebrew
brew install --cask docker

# Or download from: https://www.docker.com/products/docker-desktop

# Launch Docker Desktop
open -a Docker

# Verify installation
docker --version
```

</details>

<details>
<summary><b>Linux</b></summary>

```bash
# Install Docker Engine (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# (Optional) Add current user to docker group (avoids sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
```

**NVIDIA GPU support** (required for GPU acceleration):

```bash
# Install NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify GPU access
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
```

</details>

<details>
<summary><b>Windows</b></summary>

1. Download and install [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
2. Ensure **WSL 2** backend is enabled (Docker Desktop settings → General → Use the WSL 2 based engine)
3. Restart your computer after installation

```powershell
# Verify installation (PowerShell)
docker --version
```

</details>

### 2. Build llamacpp-omni Inference Service

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

```bash
# Clone and enter the project directory
git clone https://github.com/OpenBMB/llama.cpp-omni.git
cd llama.cpp-omni

# Build (Metal acceleration enabled by default on macOS)
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target llama-server -j

# Verify build
ls -la build/bin/llama-server
```

**Optional: Apple Neural Engine (ANE) acceleration for Vision encoder**

macOS supports running the Vision encoder on the Apple Neural Engine (ANE/NPU) via CoreML, which offloads the ViT computation from the GPU and leaves more GPU bandwidth for the LLM and TTS models. To enable this:

1. Download the CoreML vision model (`coreml_minicpmo45_vit_all_f16.mlmodelc`) and place it in `<MODEL_DIR>/vision/`
2. Add `--vision-backend coreml` when deploying (see deployment steps below)

> **Note**: On some chips, Metal GPU may actually be faster than ANE for vision encoding. We recommend benchmarking both backends on your hardware. The default is Metal (GPU).

</details>

<details>
<summary><b>Linux (NVIDIA GPU)</b></summary>

```bash
# Clone and enter the project directory
git clone https://github.com/OpenBMB/llama.cpp-omni.git
cd llama.cpp-omni

# Build with CUDA support
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --target llama-server -j

# Verify build
ls -la build/bin/llama-server
```

> **Note**: Requires NVIDIA driver and CUDA toolkit installed. Verify with `nvidia-smi`.

</details>

<details>
<summary><b>Windows (NVIDIA GPU)</b></summary>

**Requirements**: Visual Studio 2019+ with C++ workload, CMake, CUDA Toolkit.

```powershell
# Clone and enter the project directory
git clone https://github.com/OpenBMB/llama.cpp-omni.git
cd llama.cpp-omni

# Build with CUDA support (using Visual Studio generator)
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --config Release --target llama-server -j

# Verify build
dir build\bin\Release\llama-server.exe
```

> **Note**: If you don't have an NVIDIA GPU, omit `-DGGML_CUDA=ON` to use CPU-only mode.

</details>

### 3. Prepare GGUF Model Files

Download and organize the model files with the following structure:

```
<MODEL_DIR>/
├── MiniCPM-o-4_5-Q4_K_M.gguf        # LLM main model (~5GB)
├── audio/                            # Audio encoder
│   └── MiniCPM-o-4_5-audio-F16.gguf
├── vision/                           # Vision encoder
│   └── MiniCPM-o-4_5-vision-F16.gguf
├── tts/                              # TTS model
│   ├── MiniCPM-o-4_5-tts-F16.gguf
│   └── MiniCPM-o-4_5-projector-F16.gguf
└── token2wav-gguf/                   # Token2Wav model
    ├── encoder.gguf
    ├── flow_matching.gguf
    ├── flow_extra.gguf
    ├── hifigan2.gguf
    └── prompt_cache.gguf
```

## Quick Start

We provide a pre-built Docker image for quick deployment and experience. The Docker image includes all necessary dependencies and configurations.

### Download Docker Images

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

**Requirements**: Apple Silicon Mac (M1/M2/M3/M4), **M4 recommended** for optimal performance.

📦 [Download Docker Image (macOS)](https://drive.google.com/file/d/1i7HrGBZE3E-6lsrHjQgaEQK0Qxdi6tSN/view?usp=sharing)

</details>

<details>
<summary><b>Linux (NVIDIA GPU)</b></summary>

**Requirements**: NVIDIA GPU with 16GB+ VRAM recommended, NVIDIA driver 525+.

📦 [Download Docker Image (Linux)](https://drive.google.com/file/d/1i7HrGBZE3E-6lsrHjQgaEQK0Qxdi6tSN/view?usp=sharing)

</details>

<details>
<summary><b>Windows</b></summary>

**Requirements**: NVIDIA GPU recommended, Docker Desktop with WSL 2 backend.

📦 [Download Docker Image (Windows)](https://drive.google.com/file/d/1i7HrGBZE3E-6lsrHjQgaEQK0Qxdi6tSN/view?usp=sharing)

</details>

### Deployment Steps

#### Step 1: Extract and Load Docker Images

<details>
<summary><b>macOS / Linux</b></summary>

```bash
# Extract the package
unzip omni_docker.zip
cd omni_docker

# Load Docker images
docker load -i o45-frontend.tar
docker load -i omini_backend_code/omni_backend.tar
```

</details>

<details>
<summary><b>Windows</b></summary>

```powershell
# Extract the package (use 7-Zip or built-in extractor)
# Then open PowerShell in the extracted directory

# Load Docker images
docker load -i o45-frontend.tar
docker load -i omini_backend_code\omni_backend.tar
```

</details>

#### Step 2: Install Python Dependencies

```bash
# Install required Python dependencies for the inference service
pip install -r cpp_server/requirements.txt
```

#### Step 3: One-Click Deployment (Recommended)

<details>
<summary><b>macOS / Linux (deploy_all.sh)</b></summary>

> **Note**: The `deploy_all.sh` script is located in the `omni_docker` directory.

```bash
# Simplex mode (default)
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf

# Duplex mode
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --duplex
```

**macOS specific options**:

```bash
# Use Apple Neural Engine (ANE/NPU) for vision encoder via CoreML
# Requires coreml_minicpmo45_vit_all_f16.mlmodelc in <MODEL_DIR>/vision/
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --duplex \
    --vision-backend coreml

# Specify Python path if auto-detection fails
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --python /path/to/python3
```

> **Tip**: `--vision-backend coreml` runs the Vision encoder on the NPU, freeing the GPU for LLM/TTS. Default is `metal` (GPU). Try both and compare latency on your specific hardware.

</details>

<details>
<summary><b>Windows (deploy_all_win.ps1)</b></summary>

> **Note**: Run in PowerShell. The `deploy_all_win.ps1` script is located in the `omni_docker` directory.

```powershell
# Allow script execution (run once)
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process

# Simplex mode (default)
.\deploy_all_win.ps1 `
    -CppDir "C:\path\to\llama.cpp-omni" `
    -ModelDir "C:\path\to\gguf"

# Duplex mode
.\deploy_all_win.ps1 `
    -CppDir "C:\path\to\llama.cpp-omni" `
    -ModelDir "C:\path\to\gguf" `
    -Mode duplex
```

</details>

The deployment script automatically:
- Checks Docker environment
- Updates LiveKit configuration with local IP
- Starts Docker services (frontend, backend, LiveKit)
- Installs Python dependencies
- Starts C++ inference service
- Registers inference service to backend

#### Step 4: Access the Web Interface

Open in your browser: **http://localhost:3000**

### Service Ports

| Service | Port | Description |
|---------|------|-------------|
| Frontend | 3000 | Web UI |
| Backend | 8025 | Backend API |
| LiveKit | 7880 | Real-time communication |
| Inference | 9060 | C++ HTTP API |

### Troubleshooting

<details>
<summary><b>macOS: Port 8021 occupied by system service</b></summary>

macOS system services may occupy port 8021. The deployment script uses port 8025 by default to avoid this conflict.

```bash
# Check if port is in use
lsof -i :8021
```

</details>

<details>
<summary><b>Linux: Docker permission denied</b></summary>

```bash
# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Or run with sudo
sudo ./deploy_all.sh ...
```

</details>

<details>
<summary><b>Windows: Script execution policy error</b></summary>

```powershell
# Allow script execution for current session
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process

# Or run directly
powershell -ExecutionPolicy Bypass -File .\deploy_all_win.ps1
```

</details>

## Key Features

### 🔄 Full-Duplex Communication
- Simultaneous bidirectional audio and video streaming
- Natural conversation flow without turn-taking delays

### ⚡ High Responsiveness & Low Latency
- Streaming input/output for real-time interactions
- Optimized for minimal end-to-end latency
- Immediate feedback during conversations

### 🚀 Native llamacpp-omni Support
- Seamlessly integrates with [llamacpp-omni](https://github.com/OpenBMB/llama.cpp-omni) as the inference backend
- Quick deployment and easy setup
- Efficient resource utilization

### 🎯 MiniCPM-o 4.5 Experience
- Rapidly experience the full capabilities of MiniCPM-o 4.5
- Real-time multimodal understanding and generation
- Voice and video interaction in one unified interface

## Technical Highlights

- **WebRTC Protocol**: Industry-standard real-time communication
- **Streaming Architecture**: Continuous data flow for smooth interactions
- **Duplex Design**: Fills the gap in open-source streaming duplex conversation solutions
- **Cross-Platform**: Supports macOS (Metal), Linux (CUDA), and Windows (CUDA)

## Related Resources

- [MiniCPM-o 4.5 Model](https://huggingface.co/openbmb/MiniCPM-o-4_5)
- [llamacpp-omni Backend](https://github.com/OpenBMB/llama.cpp-omni)
