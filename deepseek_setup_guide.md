# Setting up DeepSeek-V3-0324 with 1.78-bit Dynamic Quantization on AWS EC2 G6.4xlarge for Cline

## Overview

This guide will walk you through setting up DeepSeek-V3-0324 with Unsloth's 1.78-bit Dynamic Quantization (UD-IQ1_S) on AWS EC2 G6.4xlarge and integrating it with Cline VS Code extension.

**Key Specifications:**
- **Model**: DeepSeek-V3-0324 (671B parameters, 37B activated per token)
- **Quantization**: UD-IQ1_S (1.78-bit dynamic quantization, ~151GB)
- **Instance**: AWS EC2 G6.4xlarge (1x NVIDIA L4 GPU with 22GB VRAM, 64GB RAM, 16 vCPUs)
- **Performance**: ~14 tokens/s single user inference, 140 tokens/s throughput

## Step 1: Launch AWS EC2 G6.4xlarge Instance

### 1.1 Instance Configuration
```bash
# Instance specs:
# - Instance Type: g6.4xlarge 
# - GPU: 1x NVIDIA L4 (22GB VRAM)
# - RAM: 64GB
# - CPU: 16 vCPUs (AMD EPYC 7R13)
# - Storage: 600GB NVMe SSD + additional EBS
# - Network: Up to 25 Gbps
```

### 1.2 Launch Instance
1. Go to AWS EC2 Console
2. Click "Launch Instance"
3. Choose **Ubuntu 22.04 LTS** AMI
4. Select **g6.4xlarge** instance type
5. Configure storage: Add at least **200GB EBS** volume for the model
6. Configure security group to allow SSH (port 22) and HTTP (port 11434 for Ollama)
7. Launch with your key pair

### 1.3 Connect to Instance
```bash
ssh -i your-key.pem ubuntu@your-instance-ip
```

## Step 2: Install System Dependencies

### 2.1 Update System
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install build-essential cmake curl libcurl4-openssl-dev pciutils -y
```

### 2.2 Install NVIDIA Drivers and CUDA
```bash
# Install NVIDIA drivers
sudo apt install nvidia-driver-535 -y

# Install CUDA Toolkit
wget https://developer.download.nvidia.com/compute/cuda/12.3.0/local_installers/cuda_12.3.0_545.23.06_linux.run
sudo sh cuda_12.3.0_545.23.06_linux.run

# Add CUDA to PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

# Reboot to ensure drivers are loaded
sudo reboot
```

### 2.3 Verify GPU Setup
```bash
nvidia-smi
nvcc --version
```

## Step 3: Build llama.cpp with CUDA Support

### 3.1 Clone and Build llama.cpp
```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

# Build with CUDA support
cmake . -B build \
    -DBUILD_SHARED_LIBS=OFF \
    -DGGML_CUDA=ON \
    -DLLAMA_CURL=ON

cmake --build build --config Release -j --clean-first \
    --target llama-quantize llama-cli llama-gguf-split llama-server

# Copy binaries for easy access
cp build/bin/llama-* .
```

## Step 4: Download DeepSeek-V3-0324 Model

### 4.1 Install Python Dependencies
```bash
pip install huggingface_hub hf_transfer
```

### 4.2 Download the 1.78-bit Dynamic Quantized Model
```python
# Create download script
cat > download_model.py << 'EOF'
import os
os.environ["HF_HUB_ENABLE_HF_TRANSFER"] = "1"

from huggingface_hub import snapshot_download

# Download the 1.78-bit dynamic quantized version (151GB)
snapshot_download(
    repo_id="unsloth/DeepSeek-V3-0324-GGUF",
    local_dir="DeepSeek-V3-0324-GGUF",
    allow_patterns=["*UD-IQ1_S*"],  # 1.78-bit dynamic quantization
)
print("Download completed!")
EOF

python download_model.py
```

## Step 5: Configure and Start the Model Server

### 5.1 Start llama.cpp Server
```bash
# Navigate to llama.cpp directory
cd llama.cpp

# Start server with optimal settings for G6.4xlarge
./llama-server \
    --model ../DeepSeek-V3-0324-GGUF/UD-IQ1_S/DeepSeek-V3-0324-UD-IQ1_S-00001-of-00003.gguf \
    --host 0.0.0.0 \
    --port 11434 \
    --ctx-size 8192 \
    --threads 16 \
    --n-gpu-layers 59 \
    --cache-type-k q8_0 \
    --temp 0.3 \
    --min-p 0.01 \
    --mlock
```

### 5.2 Server Configuration Options
- `--n-gpu-layers 59`: Offload layers to GPU (adjust based on VRAM usage)
- `--ctx-size 8192`: Context window size
- `--threads 16`: Use all CPU cores
- `--temp 0.3`: Temperature (DeepSeek recommended)
- `--min-p 0.01`: Minimum probability for better quality
- `--cache-type-k q8_0`: KV cache quantization (8-bit recommended)

### 5.3 Test the Server
```bash
# Test with curl
curl -X POST http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-v3",
    "messages": [
      {
        "role": "user", 
        "content": "Write a simple Python hello world function"
      }
    ],
    "temperature": 0.3
  }'
```

## Step 6: Set Up Cline in VS Code

### 6.1 Install VS Code (if running with GUI)
```bash
# If using desktop environment
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'
sudo apt update
sudo apt install code -y
```

### 6.2 Install Cline Extension
1. Open VS Code
2. Go to Extensions (Ctrl+Shift+X)
3. Search for "Cline"
4. Install the official Cline extension

### 6.3 Configure Cline for Local Model
1. Click the Cline icon in the VS Code sidebar
2. In Cline settings:
   - **API Provider**: Select "OpenAI Compatible"
   - **Base URL**: `http://localhost:11434/v1`
   - **Model ID**: `deepseek-v3` (or whatever model name your server uses)
   - **API Key**: `dummy` (not needed for local server)

## Step 7: Optimize Performance

### 7.1 Monitor Resources
```bash
# Monitor GPU usage
watch -n 1 nvidia-smi

# Monitor system resources
htop
```

### 7.2 Performance Tuning
```bash
# Adjust GPU layers based on VRAM usage
# Start with 59 layers, reduce if running out of VRAM
# Monitor with nvidia-smi and adjust --n-gpu-layers parameter

# For better performance on G6.4xlarge:
# - Use all 16 CPU threads: --threads 16
# - Enable memory locking: --mlock
# - Use 8-bit KV cache: --cache-type-k q8_0
```

## Step 8: Usage and Testing

### 8.1 Test with Cline
1. Open a project in VS Code
2. Open Cline (Ctrl+Shift+P → "Cline: Open In New Tab")
3. Try a coding task:
   ```
   Create a simple FastAPI application with a health check endpoint
   ```

### 8.2 Expected Performance
- **Single user inference**: ~14 tokens/s
- **Throughput**: ~140 tokens/s
- **Model accuracy**: 69.2% on coding benchmarks (1.78-bit version)
- **Memory usage**: ~151GB total (model + runtime)

### 8.3 Chat Template
DeepSeek-V3 uses this chat template:
```
<｜User｜>Your question here<｜Assistant｜>
```

## Step 9: Troubleshooting

### 9.1 Common Issues

**Out of Memory:**
```bash
# Reduce GPU layers
./llama-server --n-gpu-layers 30  # Instead of 59

# Reduce context size
./llama-server --ctx-size 4096    # Instead of 8192
```

**Slow Performance:**
```bash
# Ensure GPU is being used
nvidia-smi  # Should show GPU utilization

# Check if all layers are offloaded
# Look for "llm_load_tensors: offloaded X/Y layers to GPU"
```

**Connection Issues:**
```bash
# Check if server is running
curl http://localhost:11434/v1/models

# Check firewall
sudo ufw status
sudo ufw allow 11434
```

### 9.2 Alternative: Using Ollama

If you prefer Ollama:
```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull DeepSeek model (note: may not have the exact UD-IQ1_S version)
ollama pull deepseek-coder:7b

# Configure Cline with:
# API Provider: Ollama
# Base URL: http://localhost:11434
# Model: deepseek-coder:7b
```

## Step 10: Cost Considerations

- **G6.4xlarge cost**: ~$1.32/hour (varies by region)
- **Storage**: Additional EBS costs for 200GB
- **Data transfer**: Minimal for local development

**Estimated monthly cost** (8 hours/day usage): ~$320

## Conclusion

You now have DeepSeek-V3-0324 with 1.78-bit dynamic quantization running on AWS EC2 G6.4xlarge, integrated with Cline for autonomous coding assistance. The setup provides good performance for coding tasks while being more cost-effective than using API-based models for extensive development work.

**Key Benefits:**
- Local control and privacy
- No per-token costs after setup
- Good performance for coding tasks
- Integration with VS Code workflow via Cline

**Limitations:**
- Lower accuracy compared to full-precision models
- Requires significant infrastructure
- Initial setup complexity