# AI Stack Architecture

This stack runs on **lxc-prod-2.prx-prod-1.homehub.dk** (10.2.1.130) and provides the user-facing AI interfaces.

## Service Placement

### Trusted Server (lxc-prod-2.prx-prod-1.homehub.dk)
- Open WebUI: https://chat.homehub.dk
- LiteLLM Gateway: https://openai.homehub.dk
- Authentication: Authentik SSO (forward-auth)

### Management Server (srv-prod-1.prx-prod-2.homehub.dk)
- Ollama: http://ollama.homehub.dk:11434 (internal only)
- Qdrant: http://qdrant.homehub.dk:6333 (internal only)
- vLLM: http://vllm.homehub.dk:8000 (internal only)

## DNS Records

### Public (via Traefik on Trusted)
- `chat.homehub.dk` → lxc-prod-2.prx-prod-1.homehub.dk
- `openai.homehub.dk` → lxc-prod-2.prx-prod-1.homehub.dk

### Internal (Management server)
- `ollama.homehub.dk` → srv-prod-1.prx-prod-2.homehub.dk
- `qdrant.homehub.dk` → srv-prod-1.prx-prod-2.homehub.dk
- `vllm.homehub.dk` → srv-prod-1.prx-prod-2.homehub.dk

All services use internal CNAME records for service discovery. Public services route through Traefik on Trusted server.

## Data Paths

- Open WebUI: `/srv/docker/open-webui`
- LiteLLM: `/srv/docker/litellm`
- Ollama: `/srv/docker/ollama` (on Management)
- Qdrant: `/srv/docker/qdrant` (on Management)

## Models

- Ollama: qwen2.5-coder:14b
- vLLM: Qwen2.5-7B-Instruct

## Authentication

Both Open WebUI and LiteLLM are protected by Authentik SSO via Traefik forward-auth middleware (`authentik@docker`).

### Setting up Authentik Providers

1. **Open WebUI Provider**:
   - Name: `openwebui`
   - Redirect URIs: `https://chat.homehub.dk/oauth/oidc/callback`
   - Get Client ID and Secret, add to `.env`

2. **LiteLLM** uses API key authentication (master_key in config)

## Deployment

```bash
# Create directories
ssh root@10.2.1.130 "mkdir -p /opt/stacks/ai /srv/docker/open-webui /srv/docker/litellm/config"

# Copy files
scp compose.yaml root@10.2.1.130:/opt/stacks/ai/
scp config.yaml root@10.2.1.130:/srv/docker/litellm/config/

# Create .env (add Authentik credentials)
ssh root@10.2.1.130 "cat > /opt/stacks/ai/.env << 'EOF'
OAUTH_CLIENT_ID=<from-authentik>
OAUTH_CLIENT_SECRET=<from-authentik>
LITELLM_MASTER_KEY=$(openssl rand -hex 32)
EOF"

# Deploy
ssh root@10.2.1.130 "cd /opt/stacks/ai && docker compose up -d"
```

## Verification

```bash
# DNS
dig @10.2.1.130 +short A srv-prod-1.prx-prod-2.homehub.dk
dig @10.2.1.130 +short CNAME chat.homehub.dk
dig @10.2.1.130 +short CNAME ollama.homehub.dk

# Internal services (from Trusted server)
ssh root@10.2.1.130 "curl -s http://ollama.homehub.dk:11434/api/tags | jq"
ssh root@10.2.1.130 "curl -s http://qdrant.homehub.dk:6333/readyz"

# Public services
curl -Ik https://chat.homehub.dk  # Expect SSO redirect
curl -Ik https://openai.homehub.dk/v1/models  # Expect SSO redirect
```

## Usage

1. Open browser: `https://chat.homehub.dk`
2. Login via Authentik SSO
3. In Open WebUI Admin → Connections:
   - Set `BASE_URL=https://openai.homehub.dk/v1`
   - Add API key from LiteLLM master_key
4. Start chatting with qwen2.5-coder model

## Architecture Notes

- Service-to-service communication uses internal DNS CNAMEs (no IP addresses)
- No Traefik or Authentik Outpost runs in this stack - they're external
- All public access goes through Traefik with SSO
- GPU-heavy workloads run on Management server, UI runs on Trusted server


