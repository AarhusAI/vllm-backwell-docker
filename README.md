# vLLM on Blackwell

[vLLM](https://docs.vllm.ai/) (with [LMCache](https://docs.lmcache.ai/), cu130
nightly build for NVIDIA Blackwell) behind a [LiteLLM](https://docs.litellm.ai/)
proxy. Traefik on the external `frontend` network is the only public entry point
and routes only to LiteLLM. vLLM, Postgres and Redis live on the internal `app`
network.

## Services

| Service    | Image                                   | Port | Exposed via         |
|------------|-----------------------------------------|------|---------------------|
| `vllm`     | `lmcache/vllm-openai:latest-nightly@sha256:…` (pinned by `VLLM_SHA`) | 8000 | internal only       |
| `litellm`  | `ghcr.io/berriai/litellm` (`LITELLM_IMAGE`) | 4000 | Traefik, `COMPOSE_LITELLM_DOMAIN` |
| `postgres` | `postgres:17`                           | 5432 | internal only       |
| `redis`    | `redis:8-alpine` (local) / `redis:7-alpine` (server) | 6379 | internal only |

The vLLM entrypoint starts `lmcache server` in the background, waits 5 s, then
`exec vllm serve "$@"`. Model and serve flags are passed as the container's
`command` args.

## Setup

```sh
cp .env.example .env
cp litellm_config.example.yaml litellm_config.yaml
```

Fill in `.env`. Variables the compose files **require** (`:?`):

| Variable                 | Used by           |
|--------------------------|-------------------|
| `HUGGING_FACE_HUB_TOKEN` | vllm              |
| `VLLM_API_KEY`           | vllm              |
| `LITELLM_MASTER_KEY`     | litellm           |
| `LITELLM_SALT_KEY`       | litellm           |
| `LITELLM_UI_USERNAME`    | litellm (server)  |
| `LITELLM_UI_PASSWORD`    | litellm (server)  |
| `POSTGRES_PASSWORD`      | litellm, postgres |
| `REDIS_PASSWORD`         | litellm, redis    |
| `COMPOSE_PROJECT_NAME`   | Traefik labels    |
| `COMPOSE_LITELLM_DOMAIN` | Traefik labels    |

Optional, with compose defaults: `VLLM_SHA`, `SHM_SIZE` (8gb), `LMCACHE_L1_SIZE_GB`
(128), `LMCACHE_MAX_JOBS` (32), `OMP_NUM_THREADS` (4), `LITELLM_IMAGE`,
`LITELLM_NUM_WORKS` (8), `POSTGRES_USER` (postgres), and the Blackwell kernel
flags `VLLM_USE_DEEP_GEMM`, `VLLM_USE_FLASHINFER_MOE_FP16`,
`VLLM_USE_FLASHINFER_SAMPLER`.

Validate interpolation before starting anything:

```sh
docker compose config
```

## Run

```sh
docker compose up -d                                   # local
docker compose -f docker-compose.server.yml up -d      # server
docker compose logs -f vllm
```

The server compose adds the NVIDIA GPU reservation, `restart: unless-stopped`,
and an HTTP → HTTPS Traefik redirect. Model load is slow; the vLLM healthcheck
allows a 480 s start period.

## LiteLLM config

`litellm_config.yaml` (gitignored) is mounted to `/app/config.yaml`. Models
pointing at the local vLLM use `openai/<model>` with `api_base` set to the vLLM
service. See `litellm_config.example.yaml`.

### Message overflow guardrail

`guardrails/message_overflow.py` is mounted into LiteLLM and registered in the
config as `message_overflow.MessageTrimmingGuardrail` (mode `pre_call`). It
clamps `max_tokens` and trims message history so the request fits the model's
context window. Tunables live under the guardrail's `default_config` in
`litellm_config.yaml`: `trim_ratio`, `max_output_tokens`, `safety_buffer`,
`debug`.

## Data

Everything persistent is under `.docker/data/` (gitignored): Hugging Face model
cache, vLLM and FlashInfer compile caches, Postgres, and Redis on the server
variant. Deleting the caches means a full re-download and recompile.

## Security

This setup does not come with any authentication beyond the LiteLLM master key,
the admin UI login (`LITELLM_UI_USERNAME` / `LITELLM_UI_PASSWORD`) and the vLLM
API key, and relies on correctly configured server firewalls to protect the
exposed end-points. The admin UI at `/ui` is publicly reachable; use a long
random UI password and keep it distinct from the master key.
