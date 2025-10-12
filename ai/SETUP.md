# AI Stack Setup Guide

## Prerequisites

Before deploying the AI stack, ensure:
1. DNS records are configured and resolving
2. Authentik is running and accessible at https://authentik.homehub.dk
3. Traefik is running with the `frontend` network

## Phase 1: Configure Authentik Forward Auth

### Option A: Using Built-in Forward Auth (Recommended)

1. Log into Authentik Admin: https://authentik.homehub.dk/if/admin/
2. Go to **Applications** → **Providers** → **Create**
3. Select **Proxy Provider**
4. Configure:
   - Name: `openwebui-proxy`
   - Authorization flow: `default-provider-authorization-implicit-consent`
   - Type: `Forward auth (single application)`
   - External host: `https://chat.homehub.dk`
   - Skip path regex: `^/(api|static)/.*`
5. Create **Application**:
   - Name: `Open WebUI`
   - Slug: `openwebui`
   - Provider: `openwebui-proxy`
6. Create **Outpost**:
   - Name: `authentik-proxy-outpost`
   - Type: `Proxy`
   - Providers: Select `openwebui-proxy`
   - Configuration: Docker integration

### Option B: Using OAuth2/OpenID (Current Configuration)

1. Log into Authentik Admin
2. Go to **Applications** → **Providers** → **Create**
3. Select **OAuth2/OpenID Provider**
4. Configure:
   - Name: `openwebui`
   - Authorization flow: `default-provider-authorization-implicit-consent`
   - Client type: `Confidential`
   - Redirect URIs: `https://chat.homehub.dk/oauth/oidc/callback`
   - Signing Key: `authentik Self-signed Certificate`
5. Save and note the **Client ID** and **Client Secret**
6. Update `/opt/stacks/ai/.env` with these credentials

### Adding Traefik Middleware

If using Forward Auth (Option A), add to Authentik compose or create Traefik dynamic config:

```yaml
# In Authentik compose labels or Traefik dynamic config
- traefik.http.middlewares.authentik.forwardauth.address=http://authentik-server:9000/outpost.goauthentik.io/auth/traefik
- traefik.http.middlewares.authentik.forwardauth.trustForwardHeader=true
- traefik.http.middlewares.authentik.forwardauth.authResponseHeaders=X-authentik-username,X-authentik-groups,X-authentik-email,X-authentik-name,X-authentik-uid
```

If middleware doesn't exist, temporarily remove `middlewares=authentik@docker` from AI stack labels until configured.

## Phase 2: Deploy AI Stack

### On Trusted Server (lxc-prod-2)

```bash
# Files should already be in place from automated deployment
cd /opt/stacks/ai

# Edit .env to add Authentik credentials if using OAuth
nano .env

# Deploy the stack
docker compose up -d

# Check status
docker compose ps
docker compose logs -f
```

## Phase 3: Verification

### DNS Resolution
```bash
dig @10.2.1.130 +short CNAME chat.homehub.dk
dig @10.2.1.130 +short CNAME openai.homehub.dk
dig @10.2.1.130 +short CNAME ollama.homehub.dk
```

### Internal Connectivity (from lxc-prod-2)
```bash
curl -s http://ollama.homehub.dk:11434/api/tags | jq
curl -s http://qdrant.homehub.dk:6333/readyz
curl -s http://vllm.homehub.dk:8000/v1/models | jq
```

### Public Access
```bash
# Should redirect to Authentik or show 401 if middleware is active
curl -Ik https://chat.homehub.dk
curl -Ik https://openai.homehub.dk/v1/models
```

## Phase 4: Configure Open WebUI

1. Access: https://chat.homehub.dk
2. Login via Authentik SSO
3. First user becomes admin
4. Go to **Admin Settings** → **Connections**
5. Set OpenAI API Configuration:
   - API Base URL: `https://openai.homehub.dk/v1`
   - API Key: Use the `LITELLM_MASTER_KEY` from `/opt/stacks/ai/.env`
6. Test by sending a message

## Troubleshooting

### Open WebUI can't reach Ollama

Check from container:
```bash
docker exec open-webui curl -s http://ollama.homehub.dk:11434/api/tags
```

If this fails, DNS might not be resolving inside container. Verify DNS server is set to 10.2.1.130.

### Authentik middleware not found

If you see errors about `authentik@docker` middleware:

1. **Temporary fix**: Edit `/opt/stacks/ai/compose.yaml` and remove the middleware lines
2. **Permanent fix**: Configure forward-auth in Authentik as described above

### LiteLLM errors

Check logs:
```bash
docker compose logs litellm
```

Verify config:
```bash
cat /srv/docker/litellm/config/config.yaml
```

### Models not available in Open WebUI

Ensure models are pulled on Management server:
```bash
ssh root@10.2.0.190 "docker exec ollama ollama list"
```

Pull missing models:
```bash
ssh root@10.2.0.190 "docker exec ollama ollama pull qwen2.5-coder:14b"
```

## Security Notes

- **Internal CNAMEs only**: Compute services (ollama, qdrant, vllm) use internal DNS and are not exposed publicly
- **SSO Required**: All public access requires Authentik authentication
- **API Keys**: LiteLLM master key should be kept secure and rotated regularly
- **Network Isolation**: Compute stack runs on separate server from UI stack

