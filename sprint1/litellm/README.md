# LiteLLM Proxy — Sprint 1 Day 2

OpenAI-compatible proxy fronting **DGX Spark Ollama models** behind a single endpoint at `http://localhost:4000/v1`. Built on 2026-06-29 as the architectural keystone of the summer stack.

## Why this exists

Every future agent, MCP server, RAG pipeline, and benchmark in this repo will call **one URL**: `http://localhost:4000/v1`. LiteLLM routes the request to the right backend (Ollama local, vLLM local, AWS Bedrock cloud) based on the `model` field in the request body. Application code never changes when backends change.

## Architecture
## Files in this directory

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

Tunnel is independent of the container. Closing the tunnel tab leaves the container running but unable to reach Ollama.

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
4. Smoke test

## Verification commands

```bash
# Tunnel alive?           curl -sS http://localhost:11435/api/tags
# Container up?           docker ps | grep litellm
# Healthy?                curl http://localhost:4000/health/liveliness
# What models exposed?    curl http://localhost:4000/v1/models -H "Authorization: Bearer $(cat .master-key)"
```

## Lessons from setup (Jun 29)

1. **Port 11434 conflict on Mac** — a stale process held it. Switched to `11435` for the Mac end of the tunnel; DGX side stays 11434. No conflicts since.
2. **Use `host.docker.internal`, not `localhost`** in `config.yaml` `api_base`. From inside Docker, `localhost` means the container itself; `host.docker.internal` means the Mac host.
3. **`openai/` model prefix** tells LiteLLM the backend speaks OpenAI protocol. Works for Ollama because of its `/v1` endpoint.
4. **`api_key: "ollama"`** is a dummy — Ollama ignores it but LiteLLM's parser requires a non-empty string.
5. **`.master-key` only readable from this directory.** Running curl from monorepo root fails. Either `cd` here first or export `MASTER_KEY` in your shell profile.
6. **First request on a fresh model is slow** — Ollama loads weights to VRAM on demand. `OLLAMA_KEEP_ALIVE=10m` keeps the model warm.

## Next sprint days build on this

- **Day 3** — Add `success_callback: ["langfuse"]` to `litellm_settings` for traces
- **Day 4** — Add `bedrock/anthropic.claude-...` entries for cloud models
- **Day 1 Phase 2** — Add `qwen3-32b-local` pointing at vLLM on DGX port 8000

## References

- LiteLLM proxy docs: https://docs.litellm.ai/docs/proxy/quick_start
- Ollama OpenAI compatibility: https://github.com/ollama/ollama/blob/main/docs/openai.md
- Parent repo README: `../../README.md`
