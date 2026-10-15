# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Docker Compose deployment of a local LLM stack: **vLLM** (with LMCache, built for NVIDIA Blackwell / cu130 nightly) serving a model behind a **LiteLLM** proxy, with Postgres and Redis backing LiteLLM. Traefik (external `frontend` network, managed outside this repo) is the only public entry point and routes only to LiteLLM. vLLM is reachable only on the internal `app` network.

There is no application code to build or test. The only code file is `guardrails/message_overflow.py`, a LiteLLM `CustomGuardrail` that trims messages / clamps `max_tokens` so requests fit the model context window. No auth beyond API keys; the README states the setup relies on server firewalls.

## Commands

```bash
cp .env.example .env                       # then fill in tokens/keys
cp litellm_config.example.yaml litellm_config.yaml

docker compose up -d                                              # local
docker compose -f docker-compose.server.yml up -d                 # server (GPU reservation, restart policies, HTTPS redirect)
docker compose logs -f vllm                                       # model load takes minutes; healthcheck start_period is 480s
docker compose config                                             # validate env interpolation (fails on missing `:?` vars)
```

## Layout and gotchas

- `docker-compose.yml` vs `docker-compose.server.yml`: keep both in sync when changing a service. Server variant adds `deploy.resources` for GPU, `restart: unless-stopped` everywhere, Traefik HTTP→HTTPS labels, and bind-mounts Redis data. Local variant runs LiteLLM with `--detailed_debug`.
- vLLM image is pinned by SHA via `VLLM_SHA` (default in compose). The entrypoint starts an `lmcache server` in the background, sleeps, then `exec vllm serve "$@"`. Model/options are passed as container `command` args, not via `VLLM_MODEL` / `VLLM_OPTIONS` (those keys exist in `.env.example` but nothing in the compose files reads them).
- Env var name mismatches between `.env.example` and compose: compose requires `COMPOSE_LITELLM_DOMAIN` (example has `COMPOSE_DOMAIN`) and `REDIS_PASSWORD` (absent from example). Check `docker compose config` before assuming a var is wired.
- `litellm_config.yaml` and `.env` are gitignored; only `.example` versions are committed. The guardrail reads `config.yaml` relative to CWD inside the container (`/app/config.yaml`) to pull its `default_config`.
- Guardrail is wired per-model in `litellm_config.yaml` under `guardrails:` and referenced as `message_overflow.MessageTrimmingGuardrail` (file is mounted at `/app/message_overflow.py`). Its tuning knobs (`trim_ratio`, `max_output_tokens`, `safety_buffer`, `debug`) live in the config, not the code.
- Persistent data lives in `.docker/data/` (HF model cache, vLLM/flashinfer compile caches, Postgres). Everything there is gitignored. Deleting the caches forces a long re-download / recompile.
- Blackwell kernel flags (`VLLM_USE_DEEP_GEMM`, `VLLM_USE_FLASHINFER_*`) are set in `.env`; they exist because the cu130 nightly build misbehaves with defaults. Do not "clean up" by removing them.
