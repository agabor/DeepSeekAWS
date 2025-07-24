# Setting up DeepSeek-V3 with Ollama on AWS EC2 G6.4xlarge for Cline

## Overview

This guide will walk you through setting up DeepSeek-V3 using Ollama on AWS EC2 G6.4xlarge and integrating it with Cline VS Code extension. This approach is much simpler than manual llama.cpp setup while still providing excellent performance.

**Key Specifications:**
- **Model**: DeepSeek-V3 (various sizes available through Ollama)
- **Platform**: Ollama (automatic optimization and management)
- **Instance**: AWS EC2 G6.4xlarge (1x NVIDIA L4 GPU with 22GB VRAM, 64GB RAM, 16 vCPUs)
- **Integration**: Cline VS Code extension

## Step 1: Launch AWS EC2 G6.4xlarge Instance

### 1.1 Instance Configuration
```bash
# Instance specs:
# - Instance Type: g6.4xlarge 
# - GPU: 1x NVIDIA L4 (22GB VRAM)
# - RAM: 64GB
# - CPU: 16 vCPUs (AMD EPYC 7R13)
# - Storage: 600GB NVMe SSD
# - Network: Up to 25 Gbps
```

### 1.2 Launch Instance
1. Go to AWS EC2 Console
2. Click "Launch Instance"
3. Choose **Ubuntu 22.04 LTS** AMI
4. Select **g6.4xlarge** instance type
5. Configure storage: The default 600GB NVMe SSD is sufficient for most models
6. Configure security group:
   - SSH (port 22) from your IP
   - HTTP (port 11434) for Ollama API
7. Launch with your key pair

### 1.3 Connect to Instance
```bash
ssh -i your-key.pem ubuntu@your-instance-ip
```

## Step 2: Install System Dependencies

### 2.1 Update System
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git -y
```

### 2.2 Install NVIDIA Drivers
```bash
# Install NVIDIA drivers (Ollama will handle CUDA)
sudo apt install nvidia-driver-535 -y

# Reboot to load drivers
sudo reboot

# After reboot, verify GPU
nvidia-smi
```

## Step 3: Install Ollama

### 3.1 Install Ollama
```bash
# Install Ollama with one command
curl -fsSL https://ollama.ai/install.sh | sh

# Verify installation
ollama --version

# Start Ollama service (it usually auto-starts)
sudo systemctl start ollama
sudo systemctl enable ollama

# Check service status
sudo systemctl status ollama
```

### 3.2 Configure Ollama for External Access
```bash
# Create systemd override directory
sudo mkdir -p /etc/systemd/system/ollama.service.d

# Create override configuration
sudo tee /etc/systemd/system/ollama.service.d/override.conf > /dev/null <<EOF
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_ORIGINS=*"
EOF

# Reload systemd and restart Ollama
sudo systemctl daemon-reload
sudo systemctl restart ollama

# Verify Ollama is accessible
curl http://localhost:11434/api/tags
```

## Step 4: Download and Run DeepSeek Models

### 4.1 Available DeepSeek Models in Ollama
```bash
# List all available DeepSeek models
ollama search deepseek

# Common options for G6.4xlarge:
# - deepseek-coder:7b        (Recommended for coding, ~4GB)
# - deepseek-coder:33b       (Better quality, ~19GB)
# - deepseek-coder:1.3b      (Lightweight, ~0.7GB)
# - deepseek-chat:7b         (General chat model)
```

### 4.2 Pull Your Chosen Model
```bash
# For coding tasks (recommended starting point)
ollama pull deepseek-coder:7b

# Or for better quality if you have the VRAM
ollama pull deepseek-coder:33b

# Check downloaded models
ollama list
```

### 4.3 Test the Model
```bash
# Quick test
ollama run deepseek-coder:7b "Write a Python function to calculate factorial"

# Test via API
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-coder:7b",
    "prompt": "Write a simple FastAPI hello world app",
    "stream": false
  }'
```

## Step 5: Optimize Ollama Performance

### 5.1 Configure GPU Usage
```bash
# Check GPU utilization while model is running
watch -n 1 nvidia-smi

# Ollama automatically uses GPU - verify with:
ollama ps  # Shows running models and GPU usage
```

### 5.2 Performance Tuning (Optional)
```bash
# Set environment variables for optimization
echo 'export OLLAMA_NUM_GPU=1' >> ~/.bashrc
echo 'export OLLAMA_GPU_OVERHEAD=0' >> ~/.bashrc
source ~/.bashrc

# Restart Ollama to apply changes
sudo systemctl restart ollama
```

### 5.3 Model-Specific Settings
```bash
# Create a custom Modelfile for fine-tuning (optional)
cat > Modelfile << 'EOF'
FROM deepseek-coder:7b

# Set parameters for coding tasks
PARAMETER temperature 0.1
PARAMETER top_p 0.95
PARAMETER repeat_penalty 1.1

# System prompt for coding
SYSTEM You are an expert software developer. Provide clean, efficient, and well-documented code.
EOF

# Create custom model
ollama create deepseek-coder-tuned -f Modelfile

# Use the tuned model
ollama run deepseek-coder-tuned
```

## Step 6: Set Up VS Code and Cline

### 6.1 Install VS Code (Local Machine)
Since you'll typically connect remotely, install VS Code on your local machine:

**Windows/Mac/Linux:**
1. Download from https://code.visualstudio.com/
2. Install the **Remote-SSH** extension
3. Connect to your EC2 instance via SSH

### 6.2 Connect VS Code to EC2
1. Open VS Code
2. Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac)
3. Type "Remote-SSH: Connect to Host"
4. Add your EC2 connection: `ubuntu@your-instance-ip`
5. Select your SSH key when prompted

### 6.3 Install Cline Extension
1. In VS Code (connected to EC2), go to Extensions (`Ctrl+Shift+X`)
2. Search for "Cline"
3. Install the official Cline extension
4. The extension will install on the remote server

## Step 7: Configure Cline with Ollama

### 7.1 Open Cline
1. Click the Cline icon in the VS Code sidebar
2. Or use `Ctrl+Shift+P` → "Cline: Open In New Tab"

### 7.2 Configure Ollama Connection
1. In Cline settings, click the gear icon
2. Configure the following:
   - **API Provider**: Select "Ollama"
   - **Base URL**: `http://localhost:11434`
   - **Model ID**: `deepseek-coder:7b` (or whichever model you pulled)

### 7.3 Test the Connection
1. Try a simple prompt in Cline:
   ```
   Create a simple Python script that reads a CSV file and prints the first 5 rows
   ```
2. Cline should respond and offer to create the file

## Step 8: Advanced Usage and Tips

### 8.1 Multiple Models for Different Tasks
```bash
# Install different models for different purposes
ollama pull deepseek-coder:7b    # For coding
ollama pull deepseek-chat:7b     # For general questions
ollama pull llama3.2:3b          # Lightweight for simple tasks

# Switch models in Cline settings as needed
```

### 8.2 Model Management
```bash
# List all models
ollama list

# Remove unused models to save space
ollama rm model-name

# Update a model
ollama pull deepseek-coder:7b

# Check model info
ollama show deepseek-coder:7b
```

### 8.3 Performance Monitoring
```bash
# Monitor system resources
htop

# Monitor GPU usage
nvidia-smi -l 1

# Check Ollama logs
sudo journalctl -u ollama -f
```

## Step 9: Production Optimization

### 9.1 Automatic Startup
```bash
# Create a script to preload models on startup
sudo tee /usr/local/bin/ollama-preload.sh > /dev/null << 'EOF'
#!/bin/bash
sleep 30  # Wait for Ollama to start
ollama run deepseek-coder:7b "Hello" > /dev/null 2>&1
EOF

sudo chmod +x /usr/local/bin/ollama-preload.sh

# Add to crontab
echo "@reboot /usr/local/bin/ollama-preload.sh" | sudo crontab -
```

### 9.2 Memory Management for Larger Models
```bash
# For 33B model on G6.4xlarge, you might need to adjust
# Check available memory
free -h

# Monitor memory usage during model loading
ollama run deepseek-coder:33b "test" &
watch -n 1 'free -h && nvidia-smi'
```

### 9.3 Backup Configuration
```bash
# Backup your Ollama models list
ollama list > ~/ollama-models-backup.txt

# Backup custom Modelfiles
cp -r ~/.ollama/models ~/ollama-models-backup/
```

## Step 10: Troubleshooting

### 10.1 Common Issues

**Ollama Service Not Starting:**
```bash
# Check service status
sudo systemctl status ollama

# Check logs
sudo journalctl -u ollama -n 50

# Restart service
sudo systemctl restart ollama
```

**GPU Not Being Used:**
```bash
# Verify NVIDIA drivers
nvidia-smi

# Check Ollama can see GPU
ollama run llama3.2:1b "test"
# Should show GPU usage in nvidia-smi
```

**Out of Memory Errors:**
```bash
# Switch to smaller model
ollama pull deepseek-coder:1.3b

# Or increase swap space
sudo fallocate -l 32G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Cline Can't Connect:**
```bash
# Test Ollama API
curl http://localhost:11434/api/tags

# Check firewall
sudo ufw status
sudo ufw allow 11434

# Verify Ollama is listening on all interfaces
sudo netstat -tlnp | grep 11434
```

### 10.2 Performance Issues

**Slow Response Times:**
```bash
# Check system load
top

# Verify GPU utilization
nvidia-smi

# Try smaller model or reduce context
ollama pull deepseek-coder:1.3b
```

**Model Loading Errors:**
```bash
# Clear Ollama cache and re-pull
ollama rm deepseek-coder:7b
ollama pull deepseek-coder:7b
```

## Step 11: Cost Optimization

### 11.1 Instance Management
```bash
# Stop instance when not in use
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Start when needed
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

### 11.2 Model Size vs Performance
- **deepseek-coder:1.3b**: $0.05/hour equivalent, basic coding
- **deepseek-coder:7b**: $0.15/hour equivalent, good balance
- **deepseek-coder:33b**: $0.30/hour equivalent, best quality

### 11.3 Spot Instances
Consider using EC2 Spot Instances for development:
- Up to 90% cost savings
- Good for non-critical development work
- Can handle interruptions gracefully

## Step 12: Integration Examples

### 12.1 Sample Cline Prompts
```
"Create a REST API with FastAPI that has CRUD operations for a User model"

"Debug this Python function and explain what's wrong: [paste code]"

"Convert this JavaScript function to TypeScript with proper types"

"Write unit tests for this Python class using pytest"

"Optimize this SQL query for better performance"
```

### 12.2 VS Code Workflow
1. Open your project in VS Code (connected to EC2)
2. Open Cline in a side panel
3. Ask Cline to analyze your codebase
4. Request specific changes or new features
5. Review and approve changes
6. Test using integrated terminal

## Conclusion

You now have DeepSeek-V3 running via Ollama on AWS EC2 G6.4xlarge, integrated with Cline for autonomous coding assistance. This setup provides:

**Advantages:**
- ✅ Simple setup and maintenance
- ✅ Automatic GPU optimization
- ✅ Easy model switching
- ✅ Built-in API server
- ✅ Cost-effective for development
- ✅ No per-token costs after setup

**Expected Performance:**
- **Response time**: 2-10 seconds for typical coding tasks
- **Quality**: Excellent for most coding scenarios
- **Cost**: ~$1.32/hour when running (G6.4xlarge)

**Next Steps:**
1. Experiment with different model sizes
2. Create custom Modelfiles for your specific use cases
3. Set up automated backups
4. Consider spot instances for cost savings

This setup gives you a powerful local AI coding assistant that rivals commercial solutions while maintaining full control over your code and data!
