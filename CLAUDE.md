# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Self-hosted LLM inference server (HomeLLM) running Qwen2.5-1.5B via Ollama with an Open WebUI frontend. Accessible over a private Tailscale VPN tunnel. Two deployment modes: local Docker Compose and production Kubernetes (k3s) on AWS EC2 (t3.small).

## Architecture

Three services share a Tailscale network:
- **Ollama** — LLM inference server on port 11434, auto-pulls `qwen2.5:1.5b` on startup
- **Open WebUI** — Chat frontend on port 8080, connects to Ollama via internal network
- **Tailscale** — VPN sidecar that provides private network access to all services

In Docker Compose, all three services use `network_mode: "service:tailscale"` to share the Tailscale network stack. Open WebUI reaches Ollama at `localhost:11434`.

In Kubernetes, Ollama and WebUI run as separate Deployments with ClusterIP/NodePort Services. WebUI reaches Ollama via `http://ollama-service:11434` (configured in `k8s/webui-configmap.yml`). WebUI is exposed externally on NodePort 30000. Tailscale has its own namespace (`tailscale`).

## Local Development (Docker Compose)

```bash
docker compose up -d          # start all services
docker compose down            # stop all services
docker compose logs -f ollama  # tail ollama logs
```

Requires `.env` with `WEBUI_SECRET_KEY` and `TS_AUTHKEY`. Host port 3000 maps to WebUI container port 8080.

## Kubernetes Deployment

K8s manifests live in `k8s/`. CI deploys automatically on push to `main` via `.github/workflows/deploy.yml`:
1. SCPs `k8s/` directory to EC2
2. Runs `sudo k3s kubectl apply -f /home/ubuntu/k8s -R`

Manual deployment:
```bash
scp -r k8s/ <user>@<ec2-host>:/home/ubuntu/
ssh <user>@<ec2-host> "sudo k3s kubectl apply -f /home/ubuntu/k8s -R"
```

## CI/CD

GitHub Actions workflow (`.github/workflows/deploy.yml`) triggers on push to `main`. Required GitHub Secrets:
- `EC2_KEY` — base64-encoded SSH private key
- `EC2_NAME` — EC2 username
- `EC2_HOST` — EC2 hostname/IP (Elastic IP)

## Key Configuration Files

- `k8s/webui-configmap.yml` — Ollama URL that WebUI uses (change if moving services to different namespaces)
- `k8s/secret.yml` — WebUI auth secret (gitignored)
- `k8s/ollama.yml` line 26 — model pulled on pod start (`qwen2.5:1.5b`); change here to swap models
- PVC sizes: ollama 10Gi, webui 3Gi, tailscale 1Gi

## Sensitive Files

`.env` and `k8s/secret.yml` contain secrets and are gitignored. `ollama_data/` contains SSH keys and model blobs — also gitignored.
