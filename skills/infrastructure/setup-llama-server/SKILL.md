---
name: setup-llama-server
description: Sets up a persistent llama.cpp OpenAI-compatible inference server. Use when asked to install or configure a local LLM inference server, set up llama.cpp, or configure a local model backend for AI agents.
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

Install `huggingface-cli` and download `Qwen3-14B-Q4_K_M.gguf` from the `unsloth/Qwen3-14B-GGUF` repository.

```bash
pip install -U huggingface_hub

mkdir -p ~/models
huggingface-cli download unsloth/Qwen3-14B-GGUF Qwen3-14B-Q4_K_M.gguf --local-dir ~/models
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

### macOS — launchd

Create the plist at `~/Library/LaunchAgents/com.llamacpp.server.plist`:

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
    <string>/usr/local/bin/llama-server</string>
    <string>-m</string>
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
  <key>StandardOutPath</key>
  <string>/tmp/llama-server.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/llama-server.err</string>
</dict>
</plist>
```

Load the service:

```bash
launchctl load ~/Library/LaunchAgents/com.llamacpp.server.plist
```

### Linux — systemd

Create `/etc/systemd/system/llama-server.service`:

```ini
[Unit]
Description=llama.cpp OpenAI-compatible inference server
After=network.target

[Service]
ExecStart=/usr/local/bin/llama-server \
  -m /home/YOUR_USERNAME/models/Qwen3-14B-Q4_K_M.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --ctx-size 16384 \
  --no-slots
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> For NVIDIA GPU builds, add `-ngl 99` to the `ExecStart` flags. Omit it for CPU-only.

Enable and start the service:

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

Open the Tailscale app and sign in.

### Linux

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Expose the server over Tailscale:

```bash
tailscale serve --bg http://localhost:8080
```

## 7. Verify Remote Access

From another device connected to the same Tailscale network, run:

```bash
curl http://<tailscale-hostname>:8080/v1/models
```

Replace `<tailscale-hostname>` with the machine's Tailscale hostname or IP (visible via `tailscale status`).
