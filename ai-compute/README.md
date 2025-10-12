# AI Compute Stack (Management Server)

This stack runs on **srv-prod-1.prx-prod-2.homehub.dk** (10.2.0.190) and provides GPU-accelerated AI compute services.

## Services

- **Ollama**: LLM runtime for local models (port 11434) - Always enabled
- **Qdrant**: Vector database for embeddings (port 6333) - Always enabled
- **vLLM**: High-performance LLM inference server (port 8000) - OPTIONAL, disabled by default

### Why is vLLM disabled by default?

Both Ollama and vLLM require significant GPU VRAM. On a 16GB GPU:
- qwen2.5-coder:14b (Ollama) needs ~9-10 GB
- Qwen2.5-7B-Instruct (vLLM) needs ~14 GB

Running both simultaneously causes **"out of memory" errors**. Choose one:
- **Use Ollama** (default): Best for interactive use, multiple models, easy model management
- **Use vLLM**: Best for high-throughput inference, API-only access, production workloads

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

### Default (Ollama + Qdrant only)
```bash
# Deploy default stack (no vLLM)
ssh root@10.2.0.190 "cd /opt/stacks/ai-compute && docker compose up -d"

# Pre-pull model
ssh root@10.2.0.190 "docker exec ollama ollama pull qwen2.5-coder:14b"
```

### Optional: Enable vLLM (if you need it)
```bash
# Stop Ollama to free GPU memory
ssh root@10.2.0.190 "cd /opt/stacks/ai-compute && docker compose stop ollama"

# Start with vLLM profile
ssh root@10.2.0.190 "cd /opt/stacks/ai-compute && docker compose --profile vllm up -d"
```

**Note**: With vLLM enabled and `gpu-memory-utilization=0.40`, you might be able to run both simultaneously with smaller models.

## Verification

```bash
curl -s http://ollama.homehub.dk:11434/api/tags
curl -s http://qdrant.homehub.dk:6333/readyz
curl -s http://vllm.homehub.dk:8000/v1/models
```

