> **New to this layer?** Start with [CONCEPTS.md](./sprint1/litellm/CONCEPTS.md) for the localhost/port/config refresher, then come back here for setup.

# GenAI Summer 2026

Personal learning repo built during a structured 7-week sprint plan (Jun 24 – Aug 14, 2026). Goal: master end-to-end GenAI orchestration, AWS ML certification (MLA-C01), and the architectural patterns needed for defense/NATO consulting work.

## What this is

A local + cloud LLM platform combining:

- **DGX Spark (NVIDIA GB10)** running Ollama and vLLM for sovereign on-prem inference
- **LiteLLM proxy** unifying local + cloud models behind one OpenAI-compatible endpoint
- **AWS Bedrock** for frontier models (Claude, Mistral Large) when needed
- **LangFuse** for observability across every LLM call
- **Three build projects**: SE Copilot, Defense OSINT Brief Agent, Privacy-Preserving Fine-Tune

## Architecture
Single agent codebase swaps backends via model name + config. OpenAI-compatible protocol throughout.

## Sprint structure

| Sprint | Dates | Focus |
|--------|-------|-------|
| 1 | Jun 24 – Jul 1 | Foundations: DGX serving, LiteLLM, LangFuse, bench framework |
| 2 | Jul 2 – Jul 10 | Hybrid RAG pipeline (ColPali / ColQwen2 + GraphRAG) |
| 3 | Jul 6 – Jul 12 | MCP server build + PyPI publish |
| — | Jul 13 – Jul 25 | Vacation |
| 4 | Jul 27 – Aug 4 | MCP remote + ECS Fargate + multi-agent SOA |
| 5 | Aug 5 – Aug 11 | LoRA fine-tuning + SageMaker async inference |
| 6 | Aug 12 – Aug 14 | MLA-C01 cert exam Aug 14 + white paper finalization |

## Repo layout
## Sprint 1 status (current)

- [x] AWS Day 0 — IAM admin user, MFA, CLI profile (`genai-mgmt`, eu-west-1)
- [x] DGX Spark Ollama service — systemd, LAN/WAN reachable
- [x] Llama 3.1 8B + Llama 3.3 70B serving on GB10
- [x] LiteLLM proxy → both models routed via SSH tunnel
- [ ] LangFuse self-hosted observability (Day 3)
- [ ] Bedrock model access + cloud routing in LiteLLM (Day 4)
- [ ] Bench framework v1 (Day 5)
- [ ] vLLM Qwen3 32B serving (Day 1 Phase 2)

## Stack

| Layer | Tech | Why |
|-------|------|-----|
| Local inference | Ollama, vLLM | Sovereignty + cost (≈€0 per inference on owned hardware) |
| Proxy / abstraction | LiteLLM | One OpenAI-compatible facade, swap backends via config |
| Observability | LangFuse | EU-hosted, GDPR-friendly, traces every call |
| Cloud LLM | AWS Bedrock | Frontier models (Claude, Mistral Large) in EU regions |
| IaC | OpenTofu | Open-source Terraform fork, reproducible infra |
| Orchestration | MCP, LangGraph | Tool-calling agents with standardized protocol |
| Cert path | AWS MLA-C01 | ML Engineer Associate, exam Aug 14, 2026 |

## Security notes

- **No secrets in this repo.** `.master-key`, `.env`, `*.pem` are gitignored. If you fork, regenerate your own keys.
- **AWS profile**: `genai-mgmt`, IAM user with MFA, no root usage for daily work
- **DGX Spark access**: SSH key + multi-tier user model (sudo operators, internal users, external collaborators)

## License

Personal learning repo — code under MIT for portfolio purposes, write-ups under CC-BY-NC.

---

Built during medical leave Jun – Aug 2026 by Marouene Oueslati. Targeting defense/NATO consulting alongside automotive and smart manufacturing.
