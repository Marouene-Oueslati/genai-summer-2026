# LiteLLM Proxy — Sprint 1 Day 2

OpenAI-compatible proxy fronting **DGX Spark Ollama models** behind a single endpoint at `http://localhost:4000/v1`. Built 2026-06-29, rebuilt 2026-06-30 with Mistral Small added.

---

## Mental model — read this first

Three machines are involved. The word `localhost` means something different on each.

### The three machines

| Machine | Type | What runs there |
|---|---|---|
| Mac | Physical | Docker Desktop, SSH tunnel process, your terminal |
| Container | Virtual (on Mac, isolated network namespace) | LiteLLM Python proxy |
| DGX Spark | Physical (at home, NVIDIA GB10) | Ollama serving models on GPU |

The container is physically on the Mac but networking-wise acts like a separate computer with its own loopback.

### The three meanings of `localhost`

`localhost` always means "this machine, from whoever is asking." The asker changes:

| Where the command runs | "localhost" resolves to |
|---|---|
| Mac terminal | The Mac |
| Inside the LiteLLM container | The container itself, NOT the Mac |
| Inside `ssh -L` second arg | The remote host (DGX), interpreted after the tunnel arrives |

This is why `config.yaml` uses `host.docker.internal`, not `localhost`. From inside the container, `localhost` is the container; `host.docker.internal` is Docker's magic hostname for the Mac host.

### The four ports

| Port | Where | Who listens | Purpose |
|---|---|---|---|
| 4000 | Mac (mapped to container) | LiteLLM | The proxy you call from clients |
| 4000 | Container | LiteLLM | Same service, internal view |
| 11435 | Mac | SSH tunnel listener | Forwards traffic to DGX |
| 11434 | DGX | Ollama systemd service | Actually serves the models |

Why 11435 on Mac and 11434 on DGX? Port 11434 was held by a stale process on Mac during setup. SSH `-L` lets local and remote ports differ.

### Request path

### Why this architecture

LiteLLM is the single front door (port 4000) hiding heterogeneous backends behind one OpenAI-compatible protocol. Client code says `model: "llama-3.1-8b-local"` and LiteLLM routes. Tomorrow add `claude-sonnet-bedrock` — same `localhost:4000`, different model name, AWS Bedrock underneath. Client code never changes.

---

## Files

| File | Purpose |
|---|---|
| `config.yaml` | LiteLLM model routing |
| `.master-key` | 64-char hex Bearer token. Gitignored. |
| `.gitignore` | Excludes `.master-key` |
| `README.md` | This file |

## Current config.yaml

```yaml
model_list:
  - model_name: llama-3.1-8b-local
    litellm_params:
      model: openai/llama3.1:8b
      api_base: http://host.docker.internal:11435/v1
      api_key: "ollama"

  - model_name: llama-3.3-70b-local
    litellm_params:
      model: openai/llama3.3:70b
      api_base: http://host.docker.internal:11435/v1
      api_key: "ollama"

  - model_name: mistral-small-local
    litellm_params:
      model: openai/mistral-small:latest
      api_base: http://host.docker.internal:11435/v1
      api_key: "ollama"

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
```

### Field-by-field

| Field | Meaning |
|---|---|
| `model_name` | Client-facing alias. Use in curl `model:` field. Your choice. |
| `model` | `openai/` (protocol selector) + exact Ollama name from `ollama list`. Must match exactly. |
| `api_base` | URL the container uses to reach the backend via SSH tunnel |
| `api_key` | Dummy `"ollama"` — Ollama ignores it, LiteLLM parser requires non-empty |
| `master_key` | `os.environ/LITELLM_MASTER_KEY` reads from env var at startup |

The `openai/` prefix is a protocol selector, not a model family. It tells LiteLLM which provider integration to use. `openai/...` = generic OpenAI HTTP client (works for Ollama). `bedrock/...` = AWS Bedrock SDK with sigv4 auth.

---

## Setup from scratch

### Prerequisites

- Docker Desktop running on Mac
- SSH access to DGX as `dgx-spark-wan-marouene`
- Ollama on DGX with `OLLAMA_HOST=0.0.0.0:11434`
- At least one model pulled: `ollama pull llama3.1:8b`

### Step 1 — SSH tunnel (dedicated Mac tab)

```bash
ssh -N -L 11435:localhost:11434 dgx-spark-wan-marouene
```

Terminal goes silent. Leave tab open. If port busy: `lsof -ti :11435 | xargs kill -9`.

### Step 2 — Verify tunnel

```bash
curl -sS http://localhost:11435/api/tags
```

### Step 3 — Master key

```bash
cd ~/Project_Codes/genai-summer-2026/sprint1/litellm
openssl rand -hex 32 > .master-key
chmod 600 .master-key
echo ".master-key" > .gitignore
```

### Step 4 — Launch

```bash
export LITELLM_MASTER_KEY=$(cat .master-key)

docker run -d \
  --name litellm \
  -p 4000:4000 \
  -v $(pwd)/config.yaml:/app/config.yaml \
  -e LITELLM_MASTER_KEY=$LITELLM_MASTER_KEY \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml --port 4000
```

### Step 5 — Verify health

```bash
sleep 5
docker ps | grep litellm
docker logs litellm 2>&1 | tail -20
```

Look for: `Uvicorn running on http://0.0.0.0:4000`.

---

## Smoke tests

Same pattern, only `model` field changes.

### Llama 3.1 8B (fast)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3.1-8b-local","messages":[{"role":"user","content":"ping"}],"max_tokens":50}'
```

### Llama 3.3 70B (slow first call, ~30-60s VRAM load)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3.3-70b-local","messages":[{"role":"user","content":"ping"}],"max_tokens":50}'
```

### Mistral Small (heavy template, use max_tokens 200+)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"mistral-small-local","messages":[{"role":"user","content":"ping"}],"max_tokens":200}'
```

Success = clean JSON, `choices[0].message.content` populated, `system_fingerprint: fp_ollama`.

---

## Daily operations

- **Start**: open tunnel tab, `export LITELLM_MASTER_KEY=$(cat .master-key)`, `docker start litellm`
- **Stop**: `docker stop litellm` (preserves container), close tunnel tab
- **Restart after config edit**: `docker restart litellm`
- **Wipe**: `docker rm -f litellm && docker rmi ghcr.io/berriai/litellm:main-latest`
- **Logs**: `docker logs litellm` or `docker logs -f litellm`
- **Health**: `curl http://localhost:4000/health/liveliness`
- **List exposed models**: `curl http://localhost:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY"`

---

## Adding a new model

1. On DGX: `ollama pull <model>`, then `ollama list` for exact name
2. Add entry to config.yaml: `model_name` (your alias), `model: openai/<exact-name>`, same api_base + api_key
3. `docker restart litellm`
4. Smoke test through `localhost:4000` (not Ollama directly — that doesn't test routing)

---

## Lessons from setup (Jun 29-30)

1. **Port 11434 conflict on Mac.** Use 11435 locally, 11434 on DGX. Kill stale: `lsof -ti :11435 | xargs kill -9`.
2. **`host.docker.internal`, not `localhost`** in `api_base`. From inside Docker, `localhost` means the container.
3. **`openai/` prefix** is the protocol selector. Without it, LiteLLM tries native libraries and fails.
4. **`api_key: "ollama"`** is dummy — Ollama ignores it, LiteLLM parser requires non-empty.
5. **Auth header mandatory in curl**: `-H "Authorization: Bearer $LITELLM_MASTER_KEY"`. Missing returns 401.
6. **`.master-key` only readable from this directory.** Either `cd` here or export `LITELLM_MASTER_KEY` once per shell.
7. **Model name in curl must match `model_name` alias exactly.** Not the Ollama ID. The alias you defined.
8. **First request slow** (~30-60s for 70B VRAM load). `OLLAMA_KEEP_ALIVE=10m` keeps warm.
9. **SSH tunnel is per-session.** Mac sleep kills it. Consider Tailscale for stable hostname.
10. **Heredoc + comments + zsh = `quote>` hazard.** Ctrl+C to recover. Avoid comments inside `<<'EOF'` blocks with quotes.

---

## What is next

- **Day 3** — LangFuse observability: add `success_callback: ["langfuse"]` to `litellm_settings`. Every call traces.
- **Day 4** — Bedrock cloud models: add `model: bedrock/anthropic.claude-3-5-sonnet-...` + AWS creds.
- **Day 1 Phase 2** — vLLM Qwen3 32B on DGX port 8000, separate SSH tunnel `-L 18000:localhost:8000`.

---

## References

- LiteLLM proxy quick start: https://docs.litellm.ai/docs/proxy/quick_start
- LiteLLM model config: https://docs.litellm.ai/docs/proxy/configs
- Ollama OpenAI compatibility: https://github.com/ollama/ollama/blob/main/docs/openai.md
- Parent repo README: ../../README.md
