# AI Compute Stack (Management Server)

This stack runs on **srv-prod-1.prx-prod-2.homehub.dk** (10.2.0.190) and provides GPU-accelerated AI compute services.

## Services

- **Ollama**: LLM runtime for local models (port 11434)
- **Qdrant**: Vector database for embeddings (port 6333)
- **vLLM**: High-performance LLM inference server (port 8000)

## DNS Records (Internal Only)

- `ollama.homehub.dk` → srv-prod-1.prx-prod-2.homehub.dk
- `qdrant.homehub.dk` → srv-prod-1.prx-prod-2.homehub.dk
- `vllm.homehub.dk` → srv-prod-1.prx-prod-2.homehub.dk

## Data Paths

- Ollama: `/srv/docker/ollama/data`
- Qdrant: `/srv/docker/qdrant/storage`

## GPU Configuration

All services share the same NVIDIA GPU via nvidia-docker runtime.

## Models

- Ollama: qwen2.5-coder:14b (pre-pulled)
- vLLM: Qwen/Qwen2.5-7B-Instruct

## Deployment

```bash
# Create directories
ssh root@10.2.0.190 "mkdir -p /opt/stacks/ai-compute /srv/docker/ollama/data /srv/docker/qdrant/storage"

# Copy compose file
scp compose.yaml root@10.2.0.190:/opt/stacks/ai-compute/

# Deploy
ssh root@10.2.0.190 "cd /opt/stacks/ai-compute && docker compose up -d"

# Pre-pull model
ssh root@10.2.0.190 "docker exec ollama ollama pull qwen2.5-coder:14b"
```

## Verification

```bash
curl -s http://ollama.homehub.dk:11434/api/tags
curl -s http://qdrant.homehub.dk:6333/readyz
curl -s http://vllm.homehub.dk:8000/v1/models
```

