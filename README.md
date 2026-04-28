# HomeLLM

Self-hosted LLM inference server running Qwen2.5-1.5B, accessible from any device via a private Tailscale VPN tunnel. Containerized with Docker, orchestrated with Kubernetes (k3s) on AWS EC2, and automatically deployed via GitHub Actions CI/CD.

![Open WebUI running Qwen2.5-1.5B](pics/Screenshot%202026-04-20%20at%204.14.54%20PM.png)

## Architecture

The system is composed of three services that work together:

1. **Ollama** serves the Qwen2.5-1.5B model and exposes an inference API on port 11434.
2. **Open WebUI** provides a ChatGPT-style browser interface on port 8080, connected to Ollama as its backend.
3. **Tailscale** creates a private VPN mesh so the server is reachable from any device on the Tailscale network without exposing ports to the public internet.

```
┌─────────────────────────────────────────────────────┐
│                   AWS EC2 (t3.small)                │
│                                                     │
│  ┌──────────────┐    ┌──────────────┐               │
│  │   Ollama     │◄───│  Open WebUI  │               │
│  │  :11434      │    │  :8080       │               │
│  └──────┬───────┘    └──────┬───────┘               │
│         │                   │                       │
│         └─────────┬─────────┘                       │
│                   │                                 │
│           ┌───────▼────────┐                        │
│           │   Tailscale    │                        │
│           │   VPN Tunnel   │                        │
│           └───────┬────────┘                        │
└───────────────────┼─────────────────────────────────┘
                    │
            ┌───────▼────────┐
            │  Your Devices  │
            │  (Tailnet)     │
            └────────────────┘
```

In **Docker Compose** mode, all three containers share Tailscale's network stack via `network_mode: "service:tailscale"`, so Ollama and WebUI communicate over `localhost`.

In **Kubernetes** mode, Ollama and WebUI run as separate Deployments with their own Services. WebUI reaches Ollama via the cluster-internal DNS name `http://ollama-service:11434`. WebUI is exposed externally on NodePort 30000.

## Tech Stack

| Component | Technology |
|-----------|------------|
| LLM | Qwen2.5-1.5B via Ollama |
| Frontend | Open WebUI |
| Containerization | Docker |
| Orchestration | Kubernetes (k3s) |
| Cloud | AWS EC2 (t3.small) with Elastic IP |
| Networking | Tailscale VPN |
| CI/CD | GitHub Actions |

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose (for local development)
- A [Tailscale](https://tailscale.com/) account and auth key
- An AWS EC2 instance with k3s installed (for production deployment)

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/nshu15/Local_LLM_Server.git
cd Local_LLM_Server
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
WEBUI_SECRET_KEY=<your-webui-secret>
TS_AUTHKEY=<your-tailscale-auth-key>
```

### 3. Run locally with Docker Compose

```bash
docker compose up -d
```

Once running, access Open WebUI at `http://localhost:3000` or via the Tailscale hostname `local-llm`.

Ollama will automatically pull the Qwen2.5-1.5B model on first startup. This may take a few minutes.

### 4. Stop services

```bash
docker compose down
```

## Production Deployment (Kubernetes)

The production deployment uses k3s on an AWS EC2 instance. Kubernetes manifests are in the `k8s/` directory:

| File | Purpose |
|------|---------|
| `ollama.yml` | Ollama Deployment and ClusterIP Service |
| `webui.yml` | Open WebUI Deployment and NodePort Service (port 30000) |
| `tailscale.yml` | Tailscale namespace |
| `webui-configmap.yml` | ConfigMap with Ollama URL for WebUI |
| `secret.yml` | WebUI auth secret |
| `ollama-pvc.yml` | 10Gi persistent volume for model data |
| `webui-pvc.yml` | 3Gi persistent volume for WebUI data |
| `tailscale-pvc.yml` | 1Gi persistent volume for Tailscale state |

### Manual deployment

```bash
scp -r k8s/ <user>@<ec2-host>:/home/ubuntu/
ssh <user>@<ec2-host> "sudo k3s kubectl apply -f /home/ubuntu/k8s -R"
```

## CI/CD

GitHub Actions automatically deploys to the EC2 instance on every push to `main`. The workflow (`.github/workflows/deploy.yml`) copies the k8s manifests to the server and applies them with `kubectl`.

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `EC2_KEY` | Base64-encoded SSH private key for the EC2 instance |
| `EC2_NAME` | SSH username (e.g., `ubuntu`) |
| `EC2_HOST` | EC2 Elastic IP or hostname |

## Changing the Model

To use a different Ollama model, update the `postStart` lifecycle hook in `k8s/ollama.yml`:

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "ollama pull <model-name>"]
```

For Docker Compose, the model is pulled from the persisted `ollama_data/` volume. Pull a new model by running:

```bash
docker exec ollama ollama pull <model-name>
```
