# Local LLM Inference Service Infra

A Docker-based local LLM inference service infrastructure that runs an Ollama serivce and serves the local models inferense service securel via tailscale.

> Note: The Agent used to serve as python client to consume the inference API is just a simple chatbot to demonestrate the client-server interaction.

## Project Overview

This project builds a secure local LLM inference service infrastructure using:
- `Debian` headless server setup in a machine as `homelab` 
- `ollama/ollama:latest` as the LLM serving backend
- a custom Python client `agent` container for interaction
- Docker Compose for orchestration and isolation
- `Tailscale` setup for exposing the service securely to the internet

The python client is designed to run inside Docker and connect to Ollama serice over an isolated internal network.

## Architecture

### Components

- `ollama` service
  - Runs the Ollama server
  - Serves any model `OLLAMA_MODEL=<model>` configured in `.env` file
  - Uses `/scripts/start_ollama.sh` to configure `OLLAMA_HOST` and launch the server
  - Exposes port `11434` internally only

- `agent` service
  - Builds from `agent/Dockerfile`
  - Contains the Python app in `agent/app/main.py`
  - Loads environment variables from `.env`
  - Uses `docker compose exec agent python3 main.py` for interactive sessions

### Network

- `local_code_network`
  - `driver: bridge`

### Storage

- `ollama_data` volume
  - Persists downloaded Ollama models
  - Keeps model files across restarts

## Security

This project uses several Docker hardening measures:
<!-- 
- `internal: true` network
  - prevents external containers or hosts from joining the same Docker network -->

- `expose: 11434` for Ollama
  - does not publish a host port
  - only makes the port available to containers on the internal network

- `read_only: true` on agent container
  - prevents write operations to the container filesystem

- `cap_drop: - ALL`
  - removes Linux capabilities from the agent container

- `no-new-privileges: true`
  - blocks privilege escalation

- Non-root agent user
  - agent container runs as `agent`
  - reduces risk from container compromise

> Note: If you want host-only access to Ollama for testing, use `127.0.0.1:11434:11434` instead of `expose`.

## Infrastructure

### Docker Compose

The project is orchestrated with `docker-compose.yaml`.

Key services:
- `ollama`
- `agent`

Key files:
- `docker-compose.yaml`
- `agent/Dockerfile`
- `agent/app/main.py`
- `agent/app/requirements.txt`
- `scripts/start_ollama.sh`
- `.env`

## Prerequisites

- Docker Engine
- Docker Compose (v2 or greater)
- Git (optional)
- Sufficient disk space for Ollama model files

## Tools Used

- Docker | Compose
- Tailscale
- Debian-based Python contaier Image
- FastAPI / SQLAlchemy dependencies in the agent environment
- `httpx` for Ollama API requests
- `python-dotenv` for environment variable loading
<!-- 
## Configuration Guide

This project supports two deployment modes controlled by `.env` and `docker-compose.yaml`:

1. Secured Mode `(SECURED_NETWORK=true, internal: true)`
- Maximum isolation
- Manual model management
- Recommended for production/shared hosts

2. Convenience Mode `(SECURED_NETWORK=false, open network)`
- Automatic model pulling from .env
- Accessible to other Docker containers
- Recommended for personal development

###  Configuration Matrix 

| Mode        | .env                                  | docker-compose.yaml          | Model Management           | Security          |
| ----------- | ------------------------------------- | ---------------------------- | -------------------------- | ----------------- |
| Secured     | `SECURED_NETWORK=true`                  | `internal: true`               | Manual `docker compose exec` | Highest           |
| Convenience | `SECURED_NETWORK=false` `OLLAMA_MODEL=<model>` | `# internal: true` (commented) | Auto-pull                  | Docker-accessible | -->

## Setup

### 1. Clone repository

```bash
git clone https://github.com/himmat12/local-llm-inference-service-infra.git
cd local-llm-inference-service-infra
```

### 2. Create `.env`

Create or update the `.env` file with:

```env
OLLAMA_HOST=http://ollama:11434
OLLAMA_MODEL=<model> # qwen2.5-coder:7b
AGENT_WORKSPACE=/workspace
```

### 3. Build and start services

```bash
docker compose build
docker compose up -d
```

This will start the Ollama and agent containers.

### 4. Verify services

```bash
docker compose ps
```

You should see both `ollama` and `agent` running.

## Getting Started

<!-- ### Setup Ollama Models

Initially the Ollama service container does not includes any models. Therefore, you need to install your preferred models.

```
docker compose exec ollama bash

```

Once into bash terminal of Ollama service container, pull the models

bash
```
ollama pull <model> 
# or 
# ollama pull qwen2.5-coder:7b
# or 
# ollama pull lamma3.2:3b
``` -->

### Run the agent interactively

The agent is designed for interactive terminal sessions.

```bash
docker compose exec agent python3 main.py
```

### What happens next

- The agent starts
- It prints startup info
- It prints a list of available commands
- You can type questions or use utility commands

## Agent Commands

Within the interactive agent terminal, use these commands:

- `help`, `--help`, `?`
  - Display available commands
- `history`, `show history`
  - Print the chat history with timestamps
- `clear`
  - Clear the current chat session history
- `status`
  - Display session status and model info
- `exit`, `quit`
  - Exit the agent cleanly

## Project Files

- `docker-compose.yaml`
  - Defines the Ollama and agent services
  - Configures the internal Docker network

- `agent/Dockerfile`
  - Builds the agent container from Debian
  - Installs Python and required tools
  - Installs Python dependencies from `agent/app/requirements.txt`

- `scripts/start_ollama.sh`
  - Starts Ollama with `OLLAMA_HOST=0.0.0.0`
  - Ensures Ollama listens on the internal Docker network

- `agent/app/main.py`
  - Contains the interactive Python agent logic
  - Uses HTTP requests to Ollama to generate responses
  - Includes loading animation and built-in command support

## Notes

- The agent is intentionally not started automatically in the container CMD because it is interactive.
- Use `docker compose exec agent python3 main.py` when you want a live session.
- The `agent` container is read-only and uses `tmpfs` for temporary runtime data.        

## Troubleshooting

### Logging

To see services logs:

```bash
docker compose logs -f ollama # for ollama running service logs
```

```bash
docker compose logs -f agent # for agent running service logs
```

### If the agent is not interactive

Make sure you launch it via:

```bash
docker compose exec agent python3 main.py
```

### If Ollama cannot be reached

Confirm the internal network and service health:

```bash
docker compose ps
```

### If the agent process exits with 137

This usually means the interactive terminal session was killed.
Run again with `docker compose exec agent python3 main.py`.

## Future Improvements

- Add a dedicated CLI wrapper script inside the agent container
- Add model download validation to `scripts/start_ollama.sh`
- Add logs and runtime metrics for the agent
- Add a documented `docker-compose.override.yml` for development
