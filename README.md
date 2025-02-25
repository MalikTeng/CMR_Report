# DeepSeek Model Deployment with Ollama and Open WebUI

This guide provides comprehensive instructions for deploying DeepSeek models locally using Docker Compose, Ollama, and Open WebUI with NVIDIA GPU support.

## Author
**Yu Deng**  
*School of Biomedical Engineering & Imaging Sciences*  
*King's College London*  

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Model Management](#model-management)
- [Troubleshooting](#troubleshooting)
- [User Guide](#user-guide)

## Prerequisites

### Hardware Requirements
- NVIDIA GPU with at least 16GB VRAM (recommended for DeepSeek 8B model)
- Sufficient system RAM (minimum 32GB recommended)
- Compatible NVIDIA drivers installed

### Software Requirements
- Docker Engine
- Docker Compose
- NVIDIA Container Toolkit
- CUDA drivers

## Installation

### 1. Install NVIDIA Container Toolkit
```bash
# Add NVIDIA package repositories
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
&& curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
&& curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
   sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
   sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install the toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configure Docker runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 2. Verify GPU Support
```bash
# Test GPU accessibility
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

## Configuration

### 1. Create Docker Compose Configuration
Create a `docker-compose.yml` file with the following content:

```yaml
version: '3.8'
services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    volumes:
      - openwebui_data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=your_secret_key_here
    depends_on:
      - ollama
    restart: unless-stopped

volumes:
  ollama_data:
  openwebui_data:
```

### 2. Configure User Permissions
```bash
# Add your user to the docker group
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker

# Set correct permissions for Docker socket
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

## Deployment

### 1. Start Services
```bash
# Navigate to your project directory
cd /path/to/docker-compose-folder

# Start the containers
docker compose up -d

# Verify containers are running
docker ps
```

### 2. Pull DeepSeek Model
```bash
# Pull the DeepSeek model (adjust size as needed)
docker exec ollama ollama pull deepseek-r1:8b
```

## Model Management

### Starting and Stopping Services
```bash
# Stop services temporarily
docker compose stop

# Restart services
docker compose start

# Stop and remove everything
docker compose down

# Remove with volumes (caution: deletes model data)
docker compose down -v
```

### Accessing the Web Interface
1. Open your browser and navigate to `http://localhost:3000`
2. Create an account
3. Select the DeepSeek model from the model dropdown

## Troubleshooting

### Common Issues and Solutions

#### 1. Docker Permission Denied
```bash
sudo chmod 666 /var/run/docker.sock
```

#### 2. Container Not Found
```bash
# Check container status
docker ps -a | grep ollama

# Check logs
docker logs ollama
docker logs open-webui
```

#### 3. Port Conflicts
```bash
# Check port usage
sudo lsof -i :11434
sudo lsof -i :3000

# Kill conflicting processes
sudo kill -9 $(sudo lsof -t -i:11434)
```

#### 4. GPU Not Detected
```bash
# Verify NVIDIA drivers
nvidia-smi

# Check Docker GPU support
docker info | grep -i runtime

# Reinstall NVIDIA Container Toolkit if needed
sudo apt install --reinstall nvidia-container-toolkit
```

#### 5. Docker Compose Version Issues
If you encounter "unsupported Compose file version" errors:
```bash
# Check Docker Compose version
docker compose version

# Update Docker Compose if needed
sudo apt install docker-compose-plugin
```

## User Guide

### Basic Usage
1. Access the Open WebUI interface at `http://localhost:3000`
2. Select the DeepSeek model from the available models
3. Start a new chat session
4. Type your prompt and press Enter

### Model Configuration
- Temperature: Adjust for creativity vs. consistency
- Max tokens: Control response length
- Top P/Top K: Fine-tune response diversity

### Best Practices
- Start with simple prompts to test model functionality
- Monitor GPU memory usage with `nvidia-smi`
- Keep chat context concise for better performance
- Use system prompts for consistent behavior

### Resource Management
- Monitor GPU usage: `watch -n 1 nvidia-smi`
- Check container resources: `docker stats`
- Clear GPU memory if needed: `docker compose restart ollama`

## Security Considerations
- Change the default `WEBUI_SECRET_KEY` in docker-compose.yml
- Use strong passwords for the web interface
- Restrict network access if deploying in production
- Keep Docker and all components updated

For additional support or information, refer to:
- [Ollama Documentation](https://ollama.ai/docs)
- [Open WebUI Documentation](https://docs.openwebui.com)
- [DeepSeek Model Documentation](https://github.com/deepseek-ai/DeepSeek-LLM)