# LiteLLM Proxy — Sprint 1 Day 2

OpenAI-compatible proxy fronting DGX Spark Ollama models at http://localhost:4000/v1. Single endpoint, multiple backends, OpenAI-compatible protocol throughout.

## Mental model (read first)

Three machines, three meanings of localhost.

Machines:
- Mac (physical): Docker Desktop, SSH tunnel process, your terminal
- Container (virtual on Mac, isolated network namespace): LiteLLM Python proxy
- DGX Spark (physical, at home, NVIDIA GB10): Ollama serving models on GPU

The container is physically on the Mac but networking-wise is a separate machine with its own loopback.

Three meanings of localhost - always means this machine from the asker perspective:
- Mac terminal: localhost = the Mac
- Inside LiteLLM container: localhost = the container itself, NOT the Mac
- Inside ssh -L second arg: localhost = the remote host (DGX), interpreted after the tunnel arrives

This is why config.yaml uses host.docker.internal, not localhost. From inside the container, localhost is the container; host.docker.internal is Docker magic name for the Mac host.

Four ports:
- 4000 on Mac (mapped to container): LiteLLM listener you call
- 4000 inside container: same service, internal view
- 11435 on Mac: SSH tunnel listener
- 11434 on DGX: Ollama systemd service

Why 11435 on Mac and 11434 on DGX? Port 11434 was held by a stale process on Mac during setup. SSH -L lets the local and remote ports differ.

Request path:
1. Mac terminal: curl http://localhost:4000/...
2. Docker port-map: Mac:4000 to container:4000
3. LiteLLM in container reads config.yaml, finds api_base host.docker.internal:11435
4. Container DNS resolves to Mac IP, sends HTTP to Mac:11435
5. Mac:11435 is the SSH tunnel listener, forwards encrypted to DGX
6. On DGX, SSH delivers to localhost:11434 (Ollama)
7. Ollama runs model on GB10 GPU, returns JSON back

Why this architecture: LiteLLM is the single front door (port 4000) hiding heterogeneous backends. Tomorrow add claude-sonnet-bedrock, same localhost:4000, different model name, AWS Bedrock underneath. Client code never changes.

## Current config.yaml

Three Ollama models routed via SSH tunnel:

model_list with three entries:
- llama-3.1-8b-local (Llama 3.1 8B)
- llama-3.3-70b-local (Llama 3.3 70B)
- mistral-small-local (Mistral Small 23.6B)

Each entry uses:
- model: openai/<exact-ollama-name>
- api_base: http://host.docker.internal:11435/v1
- api_key: ollama (dummy, required non-empty)

general_settings.master_key: os.environ/LITELLM_MASTER_KEY (read from env var)

The openai/ prefix is a protocol selector - tells LiteLLM to use generic OpenAI HTTP client. For Bedrock you would write bedrock/anthropic.claude-... and LiteLLM uses AWS SDK instead.

## Setup from scratch

Prerequisites: Docker Desktop running, SSH config has dgx-spark-wan-marouene, Ollama on DGX with OLLAMA_HOST=0.0.0.0:11434, at least one model pulled.

Step 1 - SSH tunnel in dedicated tab:
  ssh -N -L 11435:localhost:11434 dgx-spark-wan-marouene
  Goes silent. Leave tab open. If port busy: lsof -ti :11435 | xargs kill -9

Step 2 - Verify tunnel:
  curl -sS http://localhost:11435/api/tags
  Returns JSON of DGX models = working

Step 3 - Master key:
  openssl rand -hex 32 > .master-key
  chmod 600 .master-key
  echo .master-key > .gitignore

Step 4 - Launch:
  export LITELLM_MASTER_KEY=$(cat .master-key)
  docker run -d --name litellm -p 4000:4000 -v $(pwd)/config.yaml:/app/config.yaml -e LITELLM_MASTER_KEY=$LITELLM_MASTER_KEY ghcr.io/berriai/litellm:main-latest --config /app/config.yaml --port 4000

Step 5 - Verify health:
  sleep 5 && docker ps | grep litellm && docker logs litellm 2>&1 | tail -20
  Look for: Uvicorn running on http://0.0.0.0:4000

## Smoke tests (one per model)

All three follow the same pattern, only model field changes. Required: Authorization Bearer header, Content-Type header.

Pattern:
  curl http://localhost:4000/v1/chat/completions -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H "Content-Type: application/json" -d (JSON body with model alias, messages, max_tokens)

Three aliases to test: llama-3.1-8b-local, llama-3.3-70b-local, mistral-small-local

Notes:
- 70B is slow first call (~30-60s VRAM load), then fast
- Mistral has heavy template (~160 prompt tokens for ping), bump max_tokens to 200+
- Success = JSON with choices[0].message.content populated, system_fingerprint fp_ollama

## Daily operations

- Start: open tunnel tab, export LITELLM_MASTER_KEY, docker start litellm
- Stop: docker stop litellm (preserves container), close tunnel tab
- Restart after config change: docker restart litellm
- Wipe and reinstall: docker rm -f litellm && docker rmi ghcr.io/berriai/litellm:main-latest
- Logs: docker logs litellm or docker logs -f litellm (follow)
- Health: curl http://localhost:4000/health/liveliness
- List models exposed: curl http://localhost:4000/v1/models with Bearer auth

## Adding a new model

1. ssh dgx-spark-wan-marouene then ollama pull <model>
2. Verify exact name with ollama list
3. Add entry to config.yaml with model_name (your alias) and model openai/<exact-ollama-name>
4. docker restart litellm
5. Smoke test through localhost:4000 (not Ollama directly)

## Lessons from setup (Jun 29-30)

1. Port 11434 conflict on Mac - use 11435 locally
2. host.docker.internal, not localhost, in api_base
3. openai/ prefix is the protocol selector
4. api_key ollama is dummy, required non-empty
5. Auth header mandatory in curl: Bearer LITELLM_MASTER_KEY
6. .master-key only readable from this directory, cd here or export the env var
7. Model name in curl must match model_name alias exactly, not the Ollama ID
8. First request slow (~30-60s for 70B VRAM load), OLLAMA_KEEP_ALIVE=10m keeps warm
9. SSH tunnel is per-session, Mac sleep kills it, consider Tailscale long-term
10. Multi-line heredoc + comments + zsh can hang on quote prompt, Ctrl+C to recover

## What is next

- Day 3: LangFuse observability via success_callback langfuse in litellm_settings
- Day 4: Bedrock cloud models with bedrock/anthropic.claude-... and AWS creds
- Day 1 Phase 2: vLLM Qwen3 32B with separate SSH tunnel on port 18000

## References

- LiteLLM proxy quick start: https://docs.litellm.ai/docs/proxy/quick_start
- LiteLLM model config: https://docs.litellm.ai/docs/proxy/configs
- Ollama OpenAI compatibility: https://github.com/ollama/ollama/blob/main/docs/openai.md
- Parent repo README: ../../README.md
