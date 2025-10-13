# LiteLLM vs vLLM - Understanding the Difference

**Date:** 2025-10-13  
**TL;DR:** LiteLLM is for API gateway/routing (what you need). vLLM is an alternative to Ollama (optional).

---

## 🤔 The Confusion

You asked: *"In regards of vLLM, how come we needed it in the first place. I thought it has to value in regards adding API connections to other models? Like chatGPT and claude?"*

**Short Answer:** You're thinking of **LiteLLM**, not **vLLM**. They have similar names but do completely different things!

---

## 🆚 LiteLLM vs vLLM

### **LiteLLM** (API Gateway) ✅

**What it is:**
- API gateway/proxy/router
- OpenAI-compatible API wrapper

**What it does:**
- Routes requests to **multiple AI providers**:
  - ✅ Your local Ollama (qwen2.5-coder:14b)
  - ✅ OpenAI (ChatGPT, GPT-4, etc.)
  - ✅ Anthropic (Claude)
  - ✅ Google (Gemini)
  - ✅ Cohere, Together AI, etc.
- Provides a **unified API** for all providers
- Load balancing, fallbacks, retries

**Location:** Trusted server (lxc-prod-2) - has internet access  
**URL:** `https://openai.homehub.dk`  
**Status:** ✅ **Running and working**

**Example usage:**
```bash
# Use local Ollama
curl -X POST https://openai.homehub.dk/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "local/ollama-qwen2.5-coder-14b", "messages": [...]}'

# Use ChatGPT (after config)
curl -X POST https://openai.homehub.dk/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4", "messages": [...]}'

# Use Claude (after config)
curl -X POST https://openai.homehub.dk/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-3-opus", "messages": [...]}'
```

---

### **vLLM** (Inference Engine) ⏸️

**What it is:**
- Fast LLM inference engine
- Alternative to Ollama

**What it does:**
- Runs LLM models **locally on your GPU**
- Optimized for high throughput and low latency
- Better for production/high-traffic scenarios
- More complex setup than Ollama

**Location:** Compute server (srv-prod-1) - GPU server  
**Status:** ⏸️ **Disabled by default** (optional)

**Why optional:**
- You already have **Ollama** doing the same job
- Both use your GPU to run models
- vLLM is faster but more complex
- For most use cases, Ollama is sufficient

**When to use vLLM:**
- You need maximum throughput (100+ requests/sec)
- You want to benchmark against Ollama
- You need advanced batching/optimization
- You're running production AI services

---

## 🎯 What You Actually Need

### **For ChatGPT & Claude Integration:**

You use **LiteLLM** (already running!) by editing its config:

**File:** `/srv/docker/litellm/config/config.yaml` on lxc-prod-2

```yaml
model_list:
  # ===== Local Models (On your GPU server) =====
  - model_name: local/ollama-qwen2.5-coder-14b
    litellm_params:
      model: "ollama/qwen2.5-coder:14b"
      api_base: "http://ollama.homehub.dk:11434"

  - model_name: local/vllm-llama3-8b
    litellm_params:
      model: "hosted_vllm/meta-llama/Meta-Llama-3-8B-Instruct"
      api_base: "http://vllm.homehub.dk:8000"

  # ===== OpenAI (ChatGPT) =====
  - model_name: gpt-4
    litellm_params:
      model: "gpt-4"
      api_key: "sk-YOUR-OPENAI-API-KEY"  # Get from https://platform.openai.com

  - model_name: gpt-4-turbo
    litellm_params:
      model: "gpt-4-turbo-preview"
      api_key: "sk-YOUR-OPENAI-API-KEY"

  - model_name: gpt-3.5-turbo
    litellm_params:
      model: "gpt-3.5-turbo"
      api_key: "sk-YOUR-OPENAI-API-KEY"

  # ===== Anthropic (Claude) =====
  - model_name: claude-3-opus
    litellm_params:
      model: "claude-3-opus-20240229"
      api_key: "sk-ant-YOUR-ANTHROPIC-API-KEY"  # Get from https://console.anthropic.com

  - model_name: claude-3-sonnet
    litellm_params:
      model: "claude-3-sonnet-20240229"
      api_key: "sk-ant-YOUR-ANTHROPIC-API-KEY"

  - model_name: claude-3-haiku
    litellm_params:
      model: "claude-3-haiku-20240307"
      api_key: "sk-ant-YOUR-ANTHROPIC-API-KEY"

  # ===== Google (Gemini) =====
  - model_name: gemini-pro
    litellm_params:
      model: "gemini/gemini-pro"
      api_key: "YOUR-GOOGLE-API-KEY"  # Get from https://makersuite.google.com/app/apikey

general_settings:
  telemetry: false
```

**Then restart LiteLLM:**
```bash
ssh root@10.2.1.130 "cd /opt/stacks/ai && docker compose restart litellm"
```

---

## 📊 Comparison Table

| Feature | LiteLLM | vLLM | Ollama |
|---------|---------|------|--------|
| **Type** | API Gateway | Inference Engine | Inference Engine |
| **Purpose** | Route to multiple providers | Run models locally | Run models locally |
| **Internet** | Needs access | No internet needed | No internet needed |
| **GPU** | No GPU needed | Requires GPU | Requires GPU |
| **Models** | Any provider's models | Local models only | Local models only |
| **Complexity** | Easy | Complex | Easy |
| **Location** | Trusted server | Compute server | Compute server |
| **Status** | ✅ Running | ⏸️ Optional | ✅ Running |
| **Use Case** | Multi-provider routing | High-throughput inference | General-purpose inference |

---

## 🛠️ How to Add ChatGPT/Claude to Open WebUI

### **Step 1: Get API Keys**

**OpenAI:**
1. Go to https://platform.openai.com
2. Sign up / log in
3. Go to API Keys section
4. Create new secret key
5. Copy the key (starts with `sk-...`)

**Anthropic (Claude):**
1. Go to https://console.anthropic.com
2. Sign up / log in
3. Go to API Keys
4. Create key
5. Copy the key (starts with `sk-ant-...`)

### **Step 2: Update LiteLLM Config**

```bash
ssh root@10.2.1.130
nano /srv/docker/litellm/config/config.yaml
```

Add your API keys to the models (see config example above).

### **Step 3: Restart LiteLLM**

```bash
cd /opt/stacks/ai
docker compose restart litellm
```

### **Step 4: Verify in Open WebUI**

1. Go to `https://chat.homehub.dk`
2. Click on model selector dropdown
3. You should now see:
   - ✅ `local/ollama-qwen2.5-coder-14b` (your GPU)
   - ✅ `gpt-4` (OpenAI)
   - ✅ `claude-3-opus` (Anthropic)
   - ... and any other models you added

### **Step 5: Configure in Open WebUI Admin**

1. Go to `https://chat.homehub.dk` → Admin panel → Settings → Connections
2. Set **OpenAI API Base URL:** `https://openai.homehub.dk/v1`
3. Set **OpenAI API Key:** Any dummy key (e.g., `sk-dummy-key`)
   - LiteLLM will use the keys from its config, not this one
4. Save

---

## 💰 Cost Considerations

**Local Models (Ollama):**
- ✅ Free (uses your GPU)
- ✅ Private (data never leaves your network)
- ⚠️ Limited to open-source models

**ChatGPT (OpenAI):**
- 💵 Paid per token
- 💵 GPT-4: ~$0.03 per 1K tokens (expensive)
- 💵 GPT-3.5: ~$0.001 per 1K tokens (cheap)

**Claude (Anthropic):**
- 💵 Paid per token
- 💵 Claude 3 Opus: ~$0.015 per 1K tokens
- 💵 Claude 3 Haiku: ~$0.00025 per 1K tokens (very cheap)

**Recommendation:**
- Use **local Ollama** for most tasks (free, private)
- Use **Claude Haiku** for quick/simple tasks (very cheap)
- Use **GPT-4** or **Claude Opus** for complex reasoning (expensive)

---

## 🎯 Summary

| Question | Answer |
|----------|--------|
| **Do I need vLLM?** | ❌ No, Ollama is sufficient for most use cases |
| **How do I add ChatGPT/Claude?** | ✅ Edit LiteLLM config (see above) |
| **Is LiteLLM already set up?** | ✅ Yes, running on `https://openai.homehub.dk` |
| **Can I use multiple providers?** | ✅ Yes, that's what LiteLLM is for! |
| **Should I enable vLLM?** | ⏸️ Only if you need high-throughput optimization |

---

## 🚀 Next Steps

1. **Get API keys** from OpenAI and/or Anthropic
2. **Update LiteLLM config** with your keys
3. **Restart LiteLLM** container
4. **Test in Open WebUI** - select different models
5. **(Optional)** Enable usage tracking/logging in LiteLLM

---

**References:**
- LiteLLM docs: https://docs.litellm.ai/
- vLLM docs: https://docs.vllm.ai/
- Ollama docs: https://ollama.ai/
- OpenAI pricing: https://openai.com/pricing
- Anthropic pricing: https://www.anthropic.com/pricing

