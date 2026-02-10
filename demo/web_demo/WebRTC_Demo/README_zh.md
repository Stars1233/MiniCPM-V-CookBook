# WebRTC 实时视频交互演示

基于 WebRTC 实现的全双工实时视频交互方案，支持流式输入输出，具有高响应、低延迟的特性。

📖 [English Version](./README.md)

## 概述

本演示采用 WebRTC 技术实现了**全双工实时视频交互**方案。该方案填补了目前开源社区中**流式双工对话方案**的技术空白，为实时多模态交互提供了完整的解决方案。

## 硬件配置要求

全量模型（LLM Q4_K_M + Vision/Audio/TTS F16 + Token2Wav）总计约 **8.3 GB**，运行时 GPU 显存占用约 **10 GB**（含 KV cache 和计算缓冲区）。

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

| 模式 | 最低配置 | 推荐配置 | 说明 |
|------|---------|---------|------|
| **单工** | M1/M2/M3/M4, 16GB 内存 | M4 系列, 32GB+ 内存 | 所有 Apple Silicon 芯片均可运行；芯片越新推理越快 |
| **双工** | M4 Pro, 36GB+ 内存 | **M4 Max, 64GB+ 内存** | 实时流式交互对内存带宽要求高；M4 Max 实测 RTF < 1.0 |

> **说明**：macOS 使用统一内存架构——模型权重、KV cache、计算缓冲区共享系统内存。双工模式的主要瓶颈是计算吞吐量而非内存容量。M1/M2/M3 芯片的带宽和算力不足以支撑实时双工流式交互。

</details>

<details>
<summary><b>Linux / Windows (NVIDIA GPU)</b></summary>

| 模式 | 最低显存 | 推荐显存 | 示例显卡 |
|------|---------|---------|---------|
| **单工** | 10 GB | 12 GB+ | RTX 3060 12GB, RTX 4070 12GB |
| **双工** | 12 GB | 16 GB+ | RTX 4080 16GB, RTX 4090 24GB, RTX 3090 24GB |

**显卡适配参考**：

| 显卡 | 显存 | 单工 | 双工 | 说明 |
|------|------|------|------|------|
| RTX 4060 | 8 GB | ❌ | ❌ | 显存不足 |
| RTX 3060 | 12 GB | ✅ | ⚠️ 勉强 | 部分模块可能需要 CPU offload |
| RTX 4070 | 12 GB | ✅ | ✅ | 双工入门级 |
| RTX 4080 | 16 GB | ✅ | ✅ | 推荐双工配置 |
| RTX 3090 | 24 GB | ✅ | ✅ | 充裕 |
| RTX 4090 | 24 GB | ✅ | ✅ | 最佳性能 |

> **说明**：NVIDIA CUDA GPU 在此类负载下通常比 Apple Silicon Metal 更快。RTX 4070 即可流畅运行双工实时交互。

</details>

## 前置条件

### 1. 安装 Docker

<details>
<summary><b>macOS</b></summary>

```bash
# 使用 Homebrew 安装
brew install --cask docker

# 或从官网下载：https://www.docker.com/products/docker-desktop

# 启动 Docker Desktop
open -a Docker

# 验证安装
docker --version
```

</details>

<details>
<summary><b>Linux</b></summary>

```bash
# 安装 Docker Engine (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 启动 Docker 服务
sudo systemctl start docker
sudo systemctl enable docker

# （可选）将当前用户添加到 docker 组（免 sudo）
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker --version
```

**NVIDIA GPU 支持**（GPU 加速必须）：

```bash
# 安装 NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 验证 GPU 访问
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
```

</details>

<details>
<summary><b>Windows</b></summary>

1. 下载并安装 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
2. 确保启用 **WSL 2** 后端（Docker Desktop 设置 → General → Use the WSL 2 based engine）
3. 安装完成后重启电脑

```powershell
# 验证安装（PowerShell）
docker --version
```

</details>

### 2. 编译 llamacpp-omni 推理服务

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

```bash
# 克隆并进入项目目录
git clone https://github.com/OpenBMB/llama.cpp-omni.git
cd llama.cpp-omni

# 编译（macOS 默认启用 Metal 加速）
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target llama-server -j

# 验证编译结果
ls -la build/bin/llama-server
```

</details>

<details>
<summary><b>Linux (NVIDIA GPU)</b></summary>

```bash
# 克隆并进入项目目录
git clone https://github.com/OpenBMB/llama.cpp-omni.git
cd llama.cpp-omni

# 编译（启用 CUDA 加速）
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --target llama-server -j

# 验证编译结果
ls -la build/bin/llama-server
```

> **注意**：需要预先安装 NVIDIA 驱动和 CUDA Toolkit，可通过 `nvidia-smi` 验证。

</details>

<details>
<summary><b>Windows (NVIDIA GPU)</b></summary>

**前提**：已安装 Visual Studio 2019+（含 C++ 工作负载）、CMake、CUDA Toolkit。

```powershell
# 克隆并进入项目目录
git clone https://github.com/OpenBMB/llama.cpp-omni.git
cd llama.cpp-omni

# 编译（启用 CUDA 加速）
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --config Release --target llama-server -j

# 验证编译结果
dir build\bin\Release\llama-server.exe
```

> **注意**：如果没有 NVIDIA GPU，去掉 `-DGGML_CUDA=ON` 以使用纯 CPU 模式。

</details>

### 3. 准备 GGUF 模型文件

下载并按以下结构组织模型文件：

```
<MODEL_DIR>/
├── MiniCPM-o-4_5-Q4_K_M.gguf        # LLM 主模型 (~5GB)
├── audio/                            # 音频编码器
│   └── MiniCPM-o-4_5-audio-F16.gguf
├── vision/                           # 视觉编码器
│   └── MiniCPM-o-4_5-vision-F16.gguf
├── tts/                              # TTS 模型
│   ├── MiniCPM-o-4_5-tts-F16.gguf
│   └── MiniCPM-o-4_5-projector-F16.gguf
└── token2wav-gguf/                   # Token2Wav 模型
    ├── encoder.gguf
    ├── flow_matching.gguf
    ├── flow_extra.gguf
    ├── hifigan2.gguf
    └── prompt_cache.gguf
```

## 快速开始

我们提供了预构建的 Docker 镜像，方便快速部署和体验。Docker 镜像包含了所有必要的依赖和配置。

### 下载 Docker 镜像

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

**设备要求**：Apple Silicon Mac（M1/M2/M3/M4），**推荐使用 M4** 以获得最佳性能。

📦 [下载 Docker 镜像 (macOS)](https://drive.google.com/file/d/1i7HrGBZE3E-6lsrHjQgaEQK0Qxdi6tSN/view?usp=sharing)

</details>

<details>
<summary><b>Linux (NVIDIA GPU)</b></summary>

**设备要求**：推荐 16GB+ 显存的 NVIDIA GPU，驱动版本 525+。

📦 [下载 Docker 镜像 (Linux)](https://drive.google.com/file/d/1i7HrGBZE3E-6lsrHjQgaEQK0Qxdi6tSN/view?usp=sharing)

</details>

<details>
<summary><b>Windows</b></summary>

**设备要求**：推荐 NVIDIA GPU，Docker Desktop 需启用 WSL 2 后端。

📦 [下载 Docker 镜像 (Windows)](https://drive.google.com/file/d/1i7HrGBZE3E-6lsrHjQgaEQK0Qxdi6tSN/view?usp=sharing)

</details>

### 部署步骤

#### 第一步：解压并加载 Docker 镜像

<details>
<summary><b>macOS / Linux</b></summary>

```bash
# 解压压缩包
unzip omni_docker.zip
cd omni_docker

# 加载 Docker 镜像
docker load -i o45-frontend.tar
docker load -i omini_backend_code/omni_backend.tar
```

</details>

<details>
<summary><b>Windows</b></summary>

```powershell
# 解压压缩包（使用 7-Zip 或系统自带解压工具）
# 然后在解压目录打开 PowerShell

# 加载 Docker 镜像
docker load -i o45-frontend.tar
docker load -i omini_backend_code\omni_backend.tar
```

</details>

#### 第二步：安装 Python 依赖

```bash
# 安装推理服务所需的 Python 依赖
pip install -r cpp_server/requirements.txt
```

#### 第三步：一键部署（推荐）

<details>
<summary><b>macOS / Linux (deploy_all.sh)</b></summary>

> **注意**：`deploy_all.sh` 脚本位于 `omni_docker` 目录下。

```bash
# 单工模式（默认）
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf

# 双工模式
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --duplex
```

**macOS 专属选项**：

```bash
# 使用 CoreML/ANE 加速视觉编码器（仅 macOS）
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --duplex \
    --vision-backend coreml

# 手动指定 Python 路径（自动检测失败时使用）
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --python /path/to/python3
```

</details>

<details>
<summary><b>Windows (deploy_all_win.ps1)</b></summary>

> **注意**：在 PowerShell 中运行。`deploy_all_win.ps1` 脚本位于 `omni_docker` 目录下。

```powershell
# 允许脚本执行（首次运行）
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process

# 单工模式（默认）
.\deploy_all_win.ps1 `
    -CppDir "C:\path\to\llama.cpp-omni" `
    -ModelDir "C:\path\to\gguf"

# 双工模式
.\deploy_all_win.ps1 `
    -CppDir "C:\path\to\llama.cpp-omni" `
    -ModelDir "C:\path\to\gguf" `
    -Mode duplex
```

</details>

部署脚本自动完成以下任务：
- 检查 Docker 环境
- 自动更新 LiveKit 配置中的本机 IP
- 启动 Docker 服务（前端、后端、LiveKit）
- 安装 Python 依赖
- 启动 C++ 推理服务
- 注册推理服务到后端

#### 第四步：访问 Web 界面

浏览器打开：**http://localhost:3000**

### 服务端口说明

| 服务 | 端口 | 说明 |
|------|------|------|
| 前端 | 3000 | Web UI |
| 后端 | 8025 | 后端 API |
| LiveKit | 7880 | 实时通信 |
| 推理服务 | 9060 | C++ HTTP API |

### 常见问题

<details>
<summary><b>macOS：端口 8021 被系统服务占用</b></summary>

macOS 系统服务可能会占用 8021 端口。部署脚本默认使用 8025 端口以避免冲突。

```bash
# 检查端口是否被占用
lsof -i :8021
```

</details>

<details>
<summary><b>Linux：Docker 权限不足</b></summary>

```bash
# 将用户添加到 docker 组
sudo usermod -aG docker $USER
newgrp docker

# 或使用 sudo 运行
sudo ./deploy_all.sh ...
```

</details>

<details>
<summary><b>Windows：脚本执行策略错误</b></summary>

```powershell
# 允许当前会话执行脚本
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process

# 或直接运行
powershell -ExecutionPolicy Bypass -File .\deploy_all_win.ps1
```

</details>

## 核心特性

### 🔄 全双工通信
- 支持音视频双向同时传输
- 自然流畅的对话体验，无需等待轮次切换

### ⚡ 高响应低延迟
- 流式输入输出，实现实时交互
- 端到端延迟优化
- 对话过程中即时反馈

### 🚀 原生支持 llamacpp-omni
- 无缝集成 [llamacpp-omni](https://github.com/OpenBMB/llama.cpp-omni) 作为推理后端
- 快速部署，简单配置
- 高效的资源利用

### 🎯 快速体验 MiniCPM-o 4.5
- 快速体验 MiniCPM-o 4.5 的完整能力
- 实时多模态理解与生成
- 语音与视频交互一体化

## 技术亮点

- **WebRTC 协议**：业界标准的实时通信协议
- **流式架构**：连续数据流，交互流畅
- **双工设计**：填补开源社区流式双工对话方案的空白
- **跨平台支持**：支持 macOS (Metal)、Linux (CUDA)、Windows (CUDA)

## 相关资源

- [MiniCPM-o 4.5 模型](https://huggingface.co/openbmb/MiniCPM-o-4_5)
- [llamacpp-omni 推理后端](https://github.com/OpenBMB/llama.cpp-omni)
