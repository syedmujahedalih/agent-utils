---
name: setup-llama-server
description: Sets up a persistent llama.cpp OpenAI-compatible 
inference server. Use when asked to install or configure a 
local LLM inference server, set up llama.cpp, or configure 
a local model backend for AI agents.
---

# Setup llama.cpp Inference Server

## 1. System Requirements Check

Detect the OS and verify available RAM before proceeding.

```bash
# Detect OS
OS=$(uname -s)
if [ "$OS" = "Darwin" ]; then
  echo "macOS detected"
elif [ "$OS" = "Linux" ]; then
  echo "Linux detected"
else
  echo "Unsupported OS: $OS" && exit 1
fi

# Verify minimum 16GB RAM
if [ "$OS" = "Darwin" ]; then
  RAM_GB=$(( $(sysctl -n hw.memsize) / 1024 / 1024 / 1024 ))
elif [ "$OS" = "Linux" ]; then
  RAM_GB=$(( $(grep MemTotal /proc/meminfo | awk '{print $2}') / 1024 / 1024 ))
fi

if [ "$RAM_GB" -lt 16 ]; then
  echo "Error: at least 16GB RAM required (found ${RAM_GB}GB)" && exit 1
fi
echo "RAM check passed: ${RAM_GB}GB available"
```

## 2. Download the Model

Install the huggingface CLI and download Qwen3-14B-Q4_K_M.gguf.

```bash
pip install -U huggingface_hub

mkdir -p ~/models
hf download unsloth/Qwen3-14B-GGUF \
  Qwen3-14B-Q4_K_M.gguf \
  --local-dir ~/models
```

## 3. Install llama.cpp

### macOS

```bash
brew install llama.cpp
```

### Linux — NVIDIA GPU

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)
```

### Linux — CPU only

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build
cmake --build build --config Release -j$(nproc)
```

## 4. Create a Background Service

Replace YOUR_USERNAME with your actual system username.

### macOS — launchd

Create ~/Library/LaunchAgents/com.llamacpp.server.plist:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.llamacpp.server</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/llama-server</string>
    <string>--model</string>
    <string>/Users/YOUR_USERNAME/models/Qwen3-14B-Q4_K_M.gguf</string>
    <string>--host</string>
    <string>0.0.0.0</string>
    <string>--port</string>
    <string>8080</string>
    <string>-ngl</string>
    <string>99</string>
    <string>--ctx-size</string>
    <string>16384</string>
    <string>--no-slots</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>NetworkState</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/Users/YOUR_USERNAME/Library/Logs/llamacpp-server.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/YOUR_USERNAME/Library/Logs/llamacpp-server-error.log</string>
</dict>
</plist>
```

Load the service:

```bash
launchctl load ~/Library/LaunchAgents/com.llamacpp.server.plist
```

### Linux — systemd

Create /etc/systemd/system/llama-server.service:

```ini
[Unit]
Description=llama.cpp OpenAI-compatible inference server
After=network.target

[Service]
ExecStart=/home/YOUR_USERNAME/llama.cpp/build/bin/llama-server \
  --model /home/YOUR_USERNAME/models/Qwen3-14B-Q4_K_M.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  -ngl 99 \
  --ctx-size 16384 \
  --no-slots
Restart=always
RestartSec=5
User=YOUR_USERNAME
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> For CPU-only builds drop the `-ngl 99` flag.

Enable and start:

```bash
sudo systemctl enable --now llama-server
```

## 5. Verify Local Access

```bash
curl http://localhost:8080/v1/models
```

You should receive a JSON response listing the loaded model.

## 6. Tailscale Remote Access

### macOS

```bash
brew install --cask tailscale
```

Open the Tailscale app and create a free account at
tailscale.com. Sign in.

### Linux / VPS

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Install Tailscale on your phone or remote device and sign
in with the same account. Then expose the server:

```bash
tailscale serve --bg http://localhost:8080
```

## 7. Verify Remote Access

Two ways to reach the server remotely:

Via Tailscale Serve (HTTPS, no port):
```bash
curl https://<your-machine>.tail-xxxxx.ts.net/v1/models
```

Via Tailscale IP directly (HTTP, with port):
```bash
curl http://$(tailscale ip -4):8080/v1/models
```

Get your Tailscale hostname and IP:
```bash
tailscale status
```
