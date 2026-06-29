# Concepts — localhost, ports, and config.yaml

Refresher for the mental model behind LiteLLM + Ollama. Read this if the README's commands stop making sense.

---

## What is `localhost`?

`localhost` is a nickname for **"this machine, talking to itself."** It's an address that always points to the computer where the command is running. The request never leaves the box.

Think of it like talking to yourself in a mirror — no phone, no address, you just speak. `localhost` is that mirror, for computers.

The technical name: **loopback address**, which maps to the special IP `127.0.0.1`.

## What is a port?

If `localhost` is a building, **ports are doors on that building**.

One computer runs many services at once — a web server, a database, an LLM proxy, a chat app. Each one listens at a different door (port) so traffic doesn't collide.

Port numbers range from 1 to 65535. Common conventions:

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP (web) |
| 443 | HTTPS (secure web) |
| 4000 | LiteLLM default |
| 5432 | Postgres database |
| 11434 | Ollama default |

These are just defaults — any free port works.

## Putting it together

`http://localhost:4000` means: **"Open the door numbered 4000 on this machine."**

- If LiteLLM is listening → you reach LiteLLM
- If nothing is listening → "connection refused"
- If a different service is listening → you reach the wrong thing

## The stack, as doors on three buildings

```
Building 1: Mac
  Door 4000  → LiteLLM container (you knock here from terminal)
  Door 11435 → SSH tunnel listener (forwards to DGX)

Building 2: Container (lives inside Mac but acts as a separate building)
  Door 4000  → LiteLLM Python process

Building 3: DGX Spark (at home)
  Door 11434 → Ollama service
  Door 22    → SSH server
```

When you type `curl http://localhost:4000/...` in your Mac terminal:
- `localhost` = Building 1 (Mac), because that's where curl runs
- `4000` = the LiteLLM door on Building 1

## The trap: `localhost` is relative

Same word, different building depending on **who's asking**:

| Speaker | "localhost" means |
|---|---|
| Mac terminal | The Mac |
| LiteLLM (inside container) | The container |
| Ollama (on DGX) | DGX |

That's why `config.yaml` can't say `localhost` for the Ollama address — LiteLLM lives inside the container, so its "localhost" is the container, not the Mac. To call back out to the Mac, the container uses Docker's special hostname `host.docker.internal`, which means **"the Mac that runs me."**

### Office analogy that sticks

Three people in three offices:
- **Mac** says "my desk" → Mac's desk
- **Container** says "my desk" → the container's desk (NOT Mac's)
- **DGX** says "my desk" → DGX's desk

When the container needs to reach Mac's desk, it can't say "my desk." It has to say something like "the desk of the person whose office I'm in." That's `host.docker.internal`.

## One sentence to remember

> **`localhost:PORT` always means "the service at door PORT on whatever machine I'm running on right now."**

The "whatever machine" changes depending on who's asking. Once you know that, the rest follows.

---

## What is `config.yaml`?

It's LiteLLM's routing table. When a client request arrives at `localhost:4000` with `"model": "<alias>"`, LiteLLM looks up that alias in `config.yaml` and forwards the request to the matching `api_base`.

### Structure

```yaml
model_list:
  - model_name: <client-facing alias>
    litellm_params:
      model: <provider-prefix>/<backend-model-name>
      api_base: <URL where the backend lives>
      api_key: <auth token for the backend>

general_settings:
  master_key: <auth token clients must send to LiteLLM>
```

### Three things to internalize

1. **`model_name` is your invention.** It's the name your client uses (`"model": "llama-3.1-8b-local"`). Pick whatever makes sense to you.

2. **`litellm_params.model` has two parts:**
   - `openai/` or `bedrock/` or `anthropic/` — **protocol selector**: tells LiteLLM which provider integration to use
   - `llama3.1:8b` — the actual backend model name, exactly as the backend knows it
   - Example: `openai/llama3.1:8b` means "use generic OpenAI HTTP client to talk to a model named llama3.1:8b" (which is how Ollama serves it)

3. **`api_base` is the door** the LiteLLM container knocks on to reach the backend. From inside the container, it can't use `localhost` for the Mac — it uses `host.docker.internal`. Port number on the end tells LiteLLM which door on that machine.

### How to create or edit it

Three options:

**On GitHub** (web UI): browse to file → pencil icon → paste → commit → `git pull` locally
**With `nano`** (interactive terminal editor): `nano config.yaml` → paste → `Ctrl+O` Enter → `Ctrl+X`
**With heredoc** (one-shot from terminal):

```bash
cd ~/Project_Codes/genai-summer-2026/sprint1/litellm

cat > config.yaml <<'EOF'
model_list:
  - model_name: llama-3.1-8b-local
    litellm_params:
      model: openai/llama3.1:8b
      api_base: http://host.docker.internal:11435/v1
      api_key: "ollama"

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
EOF
```

Verify:
```bash
cat config.yaml
```

### YAML rules that bite

- **2-space indentation, never tabs** (YAML is whitespace-sensitive)
- **Strings with special chars in quotes**: `"ollama"`, `"os.environ/LITELLM_MASTER_KEY"`
- **`model:` value must match `ollama list` exactly**, including any `:tag`
- **Lists use `- ` (dash + space)**, properly indented under `model_list:`

If LiteLLM fails to start after a config edit, run `docker logs litellm 2>&1 | tail -30` — YAML parse errors are usually clearly labeled.

---

## Connection back to the README

The README tells you **what to type**.
This file tells you **why it works**.

When you next forget why port 4000 isn't the same as 11434, or why `config.yaml` uses `host.docker.internal`, come here first.
