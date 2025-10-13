# AI Stack Deployment Guide

**Date:** 2025-10-13  
**Architecture:** Service-based with Ansible automation

---

## Overview

The AI stack is now structured as individual service stacks following the repository pattern:
- **litellm** - OpenAI-compatible API gateway (Trusted server)
- **openwebui** - Chat interface (Trusted server)
- **ollama** - LLM runtime with GPU (Compute server)
- **qdrant** - Vector database (Compute server)

---

## Architecture

### Trusted Server: lxc-prod-2.prx-prod-1.homehub.dk (10.2.1.130)
**Services:**
- **Traefik** - Reverse proxy with TLS
- **Authentik** - SSO provider with outpost
- **LiteLLM** - API gateway at `https://openai.homehub.dk`
- **Open WebUI** - Chat interface at `https://chat.homehub.dk`

**Network:** `frontend` (external)  
**Authentication:** Authentik forward-auth middleware  
**Data:** `/srv/docker/{litellm,open-webui}/`

### Compute Server: srv-prod-1.prx-prod-2.homehub.dk (10.2.0.190)
**Services:**
- **Ollama** - LLM runtime (qwen2.5-coder:14b)
- **Qdrant** - Vector database

**GPU:** NVIDIA RTX 5060 Ti (16GB)  
**Network:** Standalone (no external exposure)  
**Data:** `/srv/docker/{ollama,qdrant}/`

---

## DNS Configuration

**Zone:** `homehub.dk` (Serial: 2025101311)

### Public Services (via Traefik)
```
chat.homehub.dk    CNAME  lxc-prod-2.prx-prod-1.homehub.dk.
openai.homehub.dk  CNAME  lxc-prod-2.prx-prod-1.homehub.dk.
```

### Internal Services (no public access)
```
ollama.homehub.dk  CNAME  srv-prod-1.prx-prod-2.homehub.dk.
qdrant.homehub.dk  CNAME  srv-prod-1.prx-prod-2.homehub.dk.
```

**Note:** vLLM DNS record has been removed.

---

## Stack Structure

Each service has its own directory with:
- `compose.yaml.j2` - Docker Compose template
- `env.<service>.j2` - Environment variables template
- `template.yaml` - Metadata for Ansible
- `config.yaml.j2` - Service-specific config (LiteLLM only)

---

## Deployment via Ansible

### Prerequisites
1. Ansible vault password in `.vault_pass`
2. SSH access to both servers
3. Git access to homelab-stacks repository

### Deploy Order

**1. Update DNS:**
```bash
cd /home/kingo/projects/homelab-ansible
ansible-playbook -i inventories/prod playbooks/deploy-bind9.yml --vault-password-file .vault_pass
```

**2. Deploy Compute Services (srv-prod-1):**
```bash
ansible-playbook -i inventories/prod playbooks/deploy-stacks.yml \
  --limit srv-prod-1 \
  --tags ollama,qdrant \
  --vault-password-file .vault_pass
```

This will:
- Clone homelab-stacks repository
- Render compose.yaml and .env files
- Create data directories
- Deploy ollama and qdrant containers
- Pull qwen2.5-coder:14b model

**3. Deploy Trusted Services (lxc-prod-2):**
```bash
ansible-playbook -i inventories/prod playbooks/deploy-stacks.yml \
  --limit lxc-prod-2 \
  --tags litellm,openwebui \
  --vault-password-file .vault_pass
```

This will:
- Render compose.yaml and .env files
- Create data directories
- Render LiteLLM config
- Deploy litellm and open-webui containers

---

## Configuration

### Template Variables
```yaml
domain: homehub.dk
ai_compute_ip: 10.2.0.190
webui_secret_key: "{{ vault_webui_secret_key }}"
qdrant_api_key: "{{ vault_qdrant_api_key }}"
```

### Network Configuration
- **Trusted services:** Use `frontend` network (external)
- **Compute services:** Standalone (no network declaration)
- **DNS Workaround:** `extra_hosts` maps internal CNAMEs to IPs

### Authentication
- Both OpenWebUI and LiteLLM protected by `authentik@docker` middleware
- No OAuth config in OpenWebUI (ForwardAuth handles everything)
- Authentik Outpost must be healthy on `frontend` network

---

## Verification

### DNS Resolution
```bash
dig +short CNAME chat.homehub.dk
dig +short CNAME openai.homehub.dk
dig +short CNAME ollama.homehub.dk
dig +short CNAME qdrant.homehub.dk
dig +short CNAME vllm.homehub.dk  # Should fail (removed)
```

### Container Status
```bash
# Trusted server
ssh root@10.2.1.130 "docker ps | grep -E 'open-webui|litellm'"

# Compute server
ssh root@10.2.0.190 "docker ps | grep -E 'ollama|qdrant'"
ssh root@10.2.0.190 "docker exec ollama ollama list"
ssh root@10.2.0.190 "nvidia-smi"
```

### Network Connectivity
```bash
# From open-webui container
ssh root@10.2.1.130 "docker exec open-webui getent hosts ollama.homehub.dk"
ssh root@10.2.1.130 "docker exec open-webui curl -s http://ollama.homehub.dk:11434/api/tags"

# From litellm container
ssh root@10.2.1.130 "docker exec litellm getent hosts ollama.homehub.dk"
```

### Public Access
```bash
# Should redirect to Authentik (401 or 302)
curl -Ik https://chat.homehub.dk
curl -Ik https://openai.homehub.dk/v1/models

# After login via browser
open https://chat.homehub.dk
```

---

## File Locations

### Repository: homelab-stacks
```
/home/kingo/projects/homelab-stacks/
├── litellm/
│   ├── compose.yaml.j2
│   ├── env.litellm.j2
│   ├── config.yaml.j2
│   └── template.yaml
├── openwebui/
│   ├── compose.yaml.j2
│   ├── env.openwebui.j2
│   └── template.yaml
├── ollama/
│   ├── compose.yaml.j2
│   ├── env.ollama.j2
│   └── template.yaml
└── qdrant/
    ├── compose.yaml.j2
    ├── env.qdrant.j2
    └── template.yaml
```

### On Servers

**Trusted (lxc-prod-2):**
```
/opt/stacks/litellm/
├── compose.yaml
├── .env.litellm
└── config/config.yaml

/opt/stacks/openwebui/
├── compose.yaml
└── .env.openwebui

/srv/docker/litellm/config/
/srv/docker/open-webui/data/
```

**Compute (srv-prod-1):**
```
/opt/stacks/ollama/
├── compose.yaml
└── .env.ollama

/opt/stacks/qdrant/
├── compose.yaml
└── .env.qdrant

/srv/docker/ollama/data/
/srv/docker/qdrant/storage/
```

---

## Troubleshooting

### Container DNS Resolution Issues
The `extra_hosts` workaround is in place to bypass Docker's embedded DNS:
```yaml
extra_hosts:
  - "ollama.homehub.dk:10.2.0.190"
  - "qdrant.homehub.dk:10.2.0.190"
```

### Authentik Middleware Not Found
1. Check outpost is healthy: `docker ps | grep outpost`
2. Verify outpost is on `frontend` network
3. Check Traefik logs: `docker logs traefik`
4. Restart Traefik if needed: `docker restart traefik`

### Ollama Model Not Loading
1. Check GPU memory: `nvidia-smi`
2. Ensure no other GPU processes running
3. Pull model manually: `docker exec ollama ollama pull qwen2.5-coder:14b`

### Slow Response Times
1. Verify Ollama is using GPU: `docker exec ollama ollama ps`
2. Check GPU utilization: `watch -n1 nvidia-smi`
3. Ensure model is loaded: `docker exec ollama ollama list`

---

## Migration from Old Structure

### Old Structure (deprecated)
```
/opt/stacks/ai/              # Trusted services (monolithic)
/opt/stacks/ai-compute/      # Compute services (monolithic)
```

### New Structure
Individual service stacks deployed via Ansible with proper template rendering.

### Migration Steps
1. Deploy new stacks via Ansible
2. Verify all services working
3. Stop old stacks: `docker compose down` in old directories
4. Remove old stack directories (keep data in `/srv/docker/`)

---

## Future Improvements

1. **Replace extra_hosts** - Deploy CoreDNS sidecar for proper DNS resolution
2. **Add monitoring** - Prometheus metrics for Ollama and GPU utilization
3. **Enable vLLM** - Optional high-throughput inference engine
4. **Add external providers** - ChatGPT, Claude via LiteLLM config
5. **Implement backups** - Automated backups of data directories

---

## References

- [LiteLLM Documentation](https://docs.litellm.ai/)
- [Open WebUI Documentation](https://docs.openwebui.com/)
- [Ollama Documentation](https://github.com/ollama/ollama)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Traefik Forward Auth](https://doc.traefik.io/traefik/middlewares/http/forwardauth/)
- [Authentik Proxy Provider](https://docs.goauthentik.io/docs/providers/proxy/)

---

**Last Updated:** 2025-10-13  
**Status:** Production-ready with Ansible automation

