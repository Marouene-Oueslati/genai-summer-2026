# LiteLLM Proxy — Sprint 1 Day 2

OpenAI-compatible proxy fronting **DGX Spark Ollama models** behind a single endpoint at `http://localhost:4000/v1`. Built on 2026-06-29 as the architectural keystone of the summer stack.

## Why this exists

Every future agent, MCP server, RAG pipeline, and benchmark in this repo will call **one URL**: `http://localhost:4000/v1`. LiteLLM routes the request to the right backend (Ollama local, vLLM local, AWS Bedrock cloud) based on the `model` field in the request body. Application code never changes when backends change.

---

## Conceptual foundations

Three concepts collide in this setup. Understanding them turns the configuration from magic into mechanics.

### OpenAI-compatible endpoint

An OpenAI-compatible endpoint is any HTTP server that accepts requests and returns responses in the **exact JSON shape OpenAI's official API uses**. Request: `POST /v1/chat/completions` with `{model, messages, temperature, max_tokens}`. Response: `{choices: [{message: {content: ...}}], usage: {prompt_tokens, completion_tokens}}`.

After ChatGPT launched, every modern provider (Ollama, vLLM, Mistral, Anthropic via wrappers, Bedrock via LiteLLM, SageMaker via custom handlers) adopted this format. Consequence: **any client written for OpenAI works against all of them by changing two strings** — `base_url` and the `model` parameter. That's why this stack is composable — local + cloud + fine-tuned models all swap behind one protocol.

### base_url anatomy

`http://localhost:4000/v1` has four parts:

| Part | Value | Meaning |
|---|---|---|
| Scheme | `http://` | Protocol |
| Host | `localhost` | Which machine answers |
| Port | `4000` | Which TCP port that machine listens on |
| Path prefix | `/v1` | The OpenAI-compat API namespace |

Client libraries append `/chat/completions`, `/embeddings`, `/models` automatically. You provide the `base_url`, the library handles the rest.

### "localhost" means three different things in this stack

`localhost` always means *"this machine, from the perspective of whoever is asking"* — but the asker changes. The word is the same; the destination is not.

| Where used | "localhost" resolves to | Why |
|---|---|---|
| Mac terminal: `curl localhost:4000` | The Mac | You're on the Mac asking |
| Inside Docker container: `localhost:11434` | The container itself, NOT the Mac | Each container has its own network namespace |
| SSH `-L 11435:localhost:11434` | The DGX (remote) | This `localhost` is interpreted at DGX, after the SSH tunnel arrives |

This is why `config.yaml` uses **`host.docker.internal`** instead of `localhost` for the `api_base`. From inside the LiteLLM container, `localhost` would mean the container itself; `host.docker.internal` is Docker's magic hostname that resolves to the Mac host.

Mental model: imagine three computers — Mac, Container (lives on Mac but isolated), DGX. Each thinks of itself as `localhost`. When they need to reach each other, they use non-localhost names: `host.docker.internal` (container → Mac), forwarded port via SSH (Mac → DGX).

---

## Architecture
## Connection scenarios

Where your Mac is determines how it reaches DGX Spark:

| Mac location | URL to reach DGX | Setup |
|---|---|---|
| Same home Wi-Fi as DGX | `http://10.10.4.116:11434/v1` | None — direct LAN |
| Away from home (café, mobile, different LAN) | `http://localhost:11435/v1` after `ssh -N -L 11435:localhost:11434 dgx-spark-wan-marouene` | SSH tunnel per session |
| Anywhere, encrypted, stable hostname | `http://spark-18e5:11434/v1` | Install Tailscale on both (~15 min, once) |
| WAN port forwarding | Not recommended | Ollama has zero built-in auth — exposes GPU publicly |

This setup uses the **SSH tunnel** path (row 2) because the Mac is currently on a different network than DGX even when "at home." Port 11435 instead of 11434 because something held 11434 on the Mac at setup time.

---

## Files

| File | Purpose |
|---|---|
| `config.yaml` | LiteLLM model routing — `model_name` → `api_base` |
| `.master-key` | Random 64-char hex Bearer token. **Gitignored.** |
| `.gitignore` | Excludes `.master-key` |
| `README.md` | This file |

## Start from scratch

### 1. SSH tunnel (dedicated Mac terminal tab)

```bash
ssh -N -L 11435:localhost:11434 dgx-spark-wan-marouene
```

Terminal goes silent. Leave it alone. If port 11435 is taken: `lsof -ti :11435 | xargs kill -9` then retry.

### 2. Verify tunnel

```bash
curl -sS http://localhost:11435/api/tags
```

Should return JSON listing DGX models.

### 3. Launch LiteLLM

```bash
cd ~/Project_Codes/genai-summer-2026/sprint1/litellm
export LITELLM_MASTER_KEY=$(cat .master-key)

docker run -d \
  --name litellm \
  -p 4000:4000 \
  -v $(pwd)/config.yaml:/app/config.yaml \
  -e LITELLM_MASTER_KEY=$LITELLM_MASTER_KEY \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml --port 4000

sleep 5
docker logs litellm 2>&1 | tail -10
```

Look for `Uvicorn running on http://0.0.0.0:4000`.

### 4. Smoke test

```bash
MASTER_KEY=$(cat .master-key)

curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3.1-8b-local","messages":[{"role":"user","content":"ping"}],"max_tokens":50}'
```

Coherent JSON response = end-to-end pipeline working.

## Stop / restart / inspect

```bash
docker stop litellm
docker start litellm
docker restart litellm    # after editing config.yaml
docker rm -f litellm      # delete container, recreate via Step 3
docker logs -f litellm    # follow live
```

Tunnel is independent of the container. Closing the tunnel tab leaves the container running but unable to reach Ollama — next request returns an upstream connection error.

## Adding a new model

1. `ollama pull <new-model>` on DGX Spark
2. Add a new entry to `config.yaml`:
```yaml
   - model_name: <client-alias>
     litellm_params:
       model: openai/<ollama-model-id>
       api_base: http://host.docker.internal:11435/v1
       api_key: "ollama"
```
3. `docker restart litellm`
4. Smoke test using the new `model_name` via the LiteLLM endpoint (not Ollama directly — that wouldn't test the routing layer)

## Verification commands

```bash
# Tunnel alive?           curl -sS http://localhost:11435/api/tags
# Container up?           docker ps | grep litellm
# Healthy?                curl http://localhost:4000/health/liveliness
# What models exposed?    curl http://localhost:4000/v1/models -H "Authorization: Bearer $(cat .master-key)"
```

## Config reference

`config.yaml` is the source of truth:

```yaml
model_list:
  - model_name: llama-3.1-8b-local    # client-facing alias
    litellm_params:
      model: openai/llama3.1:8b       # backend id (openai/ = backend speaks OpenAI protocol)
      api_base: http://host.docker.internal:11435/v1
      api_key: "ollama"               # dummy, Ollama doesn't enforce auth

  - model_name: llama-3.3-70b-local
    litellm_params:
      model: openai/llama3.3:70b
      api_base: http://host.docker.internal:11435/v1
      api_key: "ollama"

litellm_settings:
  drop_params: true                   # silently drops unsupported OpenAI params

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
```

The `openai/` prefix is the **protocol selector**, not the model family. It tells LiteLLM "this backend speaks OpenAI protocol — use the generic OpenAI client." For Bedrock you'd write `bedrock/anthropic.claude-3-5-sonnet-...` instead, and LiteLLM uses the AWS SDK with sigv4 auth.

## Lessons from setup (Jun 29)

These cost real time during the build — saved here so they don't cost more next time:

1. **Port 11434 conflict on Mac** — a stale process held it. Switched to `11435` for the Mac end of the tunnel; DGX side stays 11434. Kill stale processes: `lsof -ti :11435 | xargs kill -9`.
2. **`host.docker.internal`, not `localhost`** in `config.yaml` `api_base`. From inside Docker, `localhost` means the container; `host.docker.internal` means the Mac host. (See "localhost means three different things" above.)
3. **`openai/` model prefix** = protocol selector. Tells LiteLLM the backend speaks OpenAI protocol regardless of model family. Without it, LiteLLM tries to route via a native provider library and fails.
4. **`api_key: "ollama"`** is a dummy — Ollama ignores it but LiteLLM's parser requires a non-empty string. Any value works.
5. **`.master-key` only readable from this directory.** Running curl from monorepo root fails. Either `cd` here first or export `MASTER_KEY` in your shell profile.
6. **First request on a fresh model is slow** — Ollama loads weights to VRAM on demand. `OLLAMA_KEEP_ALIVE=10m` in DGX systemd override keeps the model warm.
7. **The SSH tunnel is per-session.** Closing the terminal tab kills it. For long sessions, consider Tailscale (stable hostname, survives across networks).

## What's next

This proxy is the foundation. Next sprint days extend it:

- **Day 3** — Add `success_callback: ["langfuse"]` to `litellm_settings` for traces. Every call gets logged with latency, cost, full prompt/response.
- **Day 4** — Add Bedrock entries with `model: bedrock/anthropic.claude-3-5-sonnet-...` and `aws_region_name: eu-west-1`. Now the same `localhost:4000/v1` serves local AND cloud models.
- **Day 1 Phase 2** — Add `qwen3-32b-local` pointing at vLLM on DGX port 8000 (separate port from Ollama, separate tunnel `-L 18000:localhost:8000`).

## References

- LiteLLM proxy docs: https://docs.litellm.ai/docs/proxy/quick_start
- Ollama OpenAI compatibility: https://github.com/ollama/ollama/blob/main/docs/openai.md
- Parent repo README: `../../README.md`
