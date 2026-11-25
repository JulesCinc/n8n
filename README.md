# Medical Information Assistant (Vidal-based)

An assistant that answers questions about medications, their uses, side effects and interactions, based on the French medical handbook **VIDAL**.

> ⚠️ For educational purposes only. This is not a substitute for professional medical advice.

## Use Case

Helps healthcare professionals and patients get reliable, up-to-date information on medications available in France, so they can use them safely.

## Challenges

- Handling medical terminology
- Ensuring accuracy in safety-critical information
- Following legal and ethical guidelines for drug-related advice

## Reference Data

- **Dictionnaire VIDAL (2018)**
- **VIDAL online**: only patient information is public. Technical information is proprietary and reserved for healthcare professionals.

## Deployment

The stack runs on a VM with Docker Compose (`docker-compose.yml`):

| Service      | Role                          | Access                                   |
|--------------|-------------------------------|------------------------------------------|
| `ollama`     | Local LLM server              | Internal only: `http://ollama:11434`     |
| `open-webui` | Chat interface                | Port `80`                                |
| `n8n`        | Workflow / AI agent editor    | Port `5678` (`http://<vm-ip>:5678`)      |
| `qdrant`     | Vector database (RAG)         | `127.0.0.1:6333` on the VM only (SSH tunnel for the dashboard) |

All services share the Compose network, so they can reach each other by service name (e.g. Open WebUI can call n8n at `http://n8n:5678`). Data is kept in Docker volumes (`ollama`, `open-webui`, `n8n_data`, `qdrant_data`).

### Setup

Create a `.env` file next to `docker-compose.yml` with the Qdrant API key (do not commit it):

```bash
echo "QDRANT_API_KEY=$(openssl rand -hex 32)" > .env
```

The Vidal JSONL files go in `./vidal_out/`; they are mounted read-only in n8n at `/data/vidal` for the indexing workflow.

### Start / update

```bash
git pull
docker compose up -d
```

If the containers were first created from another folder, reuse the original project name to keep the existing volumes:

```bash
docker inspect ollama --format '{{ index .Config.Labels "com.docker.compose.project" }}'
docker compose -p <project-name> up -d
```

### n8n configuration

- `WEBHOOK_URL`: public URL of the VM (`https://k2vm-136.mde.epf.fr/`), used to build webhook URLs. Change it if your VM differs.
- `N8N_SECURE_COOKIE=false`: allows logging in over plain HTTP. Fine for the lab, not for production.
- On first access, n8n asks you to create an owner account.

### Using the LLM in n8n

In an **Ollama Chat Model** node, create an Ollama credential with one of these base URLs:

- `http://10.3.50.48:11434`: school GPU server
- `http://ollama:11434`: local Ollama container

For agents that use tools, pick a model that supports tool calling (e.g. `llama3.2`).

### Using Qdrant in n8n

Create a **Qdrant API** credential with URL `http://qdrant:6333` and the `QDRANT_API_KEY` from `.env`.
