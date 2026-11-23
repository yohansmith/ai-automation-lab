# AI Automation Lab

A hands-on lab for building AI automation and agent workflows: self-hosted n8n, LLM APIs, RAG, LangGraph, and FastAPI, with an emphasis on measuring accuracy and data quality. This repo grows week by week; each week ends with something runnable.

**Current status:** Week 1, Day 1 complete (self-hosted n8n on Docker with Postgres, first webhook workflow).

## What's in here

```
ai-automation-lab/
├── n8n/
│   ├── docker-compose.yml     # n8n + Postgres with persistent volumes
│   ├── .env.example           # template for secrets (copy to .env)
│   └── workflows/             # exported n8n workflows (JSON)
├── python/                    # Python scripts (from Day 3)
├── docs/                      # notes and experiment write-ups
└── LEARNING_LOG.md            # daily learning notes and experiments
```

## Prerequisites

- Docker Desktop (WSL2 backend on Windows) with WSL integration enabled for your Linux distro
- Git
- Virtualization enabled in BIOS (on AMD: SVM Mode)
- About 40 GB of free disk space

Developed on Windows 11 with WSL2 (Ubuntu), on a laptop with 16 GB RAM and no GPU. Everything uses cloud LLM APIs rather than local models.

## Quickstart

Run these inside your Linux (WSL) terminal.

```bash
git clone git@github.com:yohansmith/ai-automation-lab.git
cd ai-automation-lab/n8n

# create your own secrets file
cp .env.example .env
# generate real values (replace the placeholders in .env):
openssl rand -hex 16   # use for POSTGRES_PASSWORD
openssl rand -hex 32   # use for N8N_ENCRYPTION_KEY

docker compose up -d
docker compose logs -f n8n   # wait until the editor is ready (Ctrl+C to stop reading)
```

Open <http://localhost:5678>, create the owner account, then import the example workflow:

1. Create a new workflow.
2. Menu (three dots) → **Import → From file** → `n8n/workflows/01-hello-webhook.json`.
3. Click **Publish**.

The first start can take a minute or two on modest hardware, and the first image pull is large.

> **Back up your `.env`.** `N8N_ENCRYPTION_KEY` encrypts the credentials stored in n8n. If you lose it, saved credentials cannot be decrypted. `.env` is gitignored and must never be committed.

## Workflow 01: Hello Webhook

**Flow:** Webhook (POST `/hello`) → Edit Fields → Respond to Webhook

Receives a JSON body with a `name`, and replies with a greeting and a timestamp.

Test URL (works only while the editor is listening, one request per click on "Execute workflow"):

```bash
curl -X POST http://localhost:5678/webhook-test/hello \
  -H "Content-Type: application/json" \
  -d '{"name":"Yohan"}'
```

Production URL (works any time the workflow is published):

```bash
curl -X POST http://localhost:5678/webhook/hello \
  -H "Content-Type: application/json" \
  -d '{"name":"Yohan"}'
```

Example response:

```json
{ "message": "Hello Yohan", "received_at": "2026-11-23T21:25:38.157+05:30" }
```

## Common commands

```bash
docker compose ps                 # what's running
docker compose logs -f n8n        # follow n8n logs
docker compose stop               # stop, keep everything
docker compose start              # start again
docker compose down               # remove containers, KEEP data (volumes)
docker compose down -v            # remove containers AND DELETE data. Destructive.
```

## What I learned (experiments)

I deliberately broke the setup to see what survives. Full notes are in [LEARNING_LOG.md](LEARNING_LOG.md).

| Action                                               | Result                                                                                                                                               |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docker compose restart`                             | Data intact. n8n is unavailable for about a minute while it starts.                                                                                  |
| `docker compose down` then `up -d`                   | Containers removed, **volumes kept**, so account, workflows and history all survived.                                                                |
| Call the test webhook URL without clicking Execute   | `404`, "webhook is not registered".                                                                                                                  |
| Change `POSTGRES_PASSWORD` after the database exists | n8n crashes with `password authentication failed`. Postgres only reads the password on first creation, and its healthcheck still reports healthy.    |
| `docker compose down -v`                             | Volumes deleted. n8n returned to first-run setup and all executions were lost. Workflows are recoverable only because they're exported to this repo. |

## Troubleshooting

**`docker: command not found` inside WSL**
Docker Desktop's WSL integration is off. Docker Desktop → Settings → Resources → WSL Integration → enable for your distro → Apply & restart. Then open a fresh terminal.

**Image pull fails with `TLS handshake timeout`**
A slow or unstable connection. Layers already downloaded are kept, so just retry `docker compose pull n8n` until it completes.

**`port is already allocated` on 5678**
An old container is using the port. Run `docker ps -a`, then `docker rm -f <name>`.

**n8n page won't load right after starting or restarting**
It takes about a minute. Check `docker compose logs --tail 20 n8n` and wait for the editor-ready message.

**`password authentication failed for user "n8n"` in n8n logs**
`POSTGRES_PASSWORD` in `.env` doesn't match the one the database was created with. Restore the original `.env`, or reset the data with `docker compose down -v` (this deletes everything).

**Webhook URL in the browser says "registered for POST requests"**
Browsers send GET. Use `curl` or Postman to send a POST.

**Production webhook returns 404**
The workflow isn't published, or you used `/webhook-test/` instead of `/webhook/`. In current n8n, use the **Publish** button.

**SSH `Host key verification failed` on first GitHub connection**
Type the full word `yes` at the fingerprint prompt, or run `ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts`.

## Notes and limitations

- The webhook currently has no authentication. Header auth and input validation are planned for Day 6.
- `WEBHOOK_URL` is set to `http://localhost:5678/`. For public access it must match the public URL (Cloudflare Tunnel or ngrok, and later a VPS).
- Local only for now; deployment to a VPS is planned for Week 7.

## Roadmap

| Week | Theme                                                                       |
| ---- | --------------------------------------------------------------------------- |
| 1    | Docker, n8n, JSON/webhooks, LLM APIs, structured outputs, evaluation basics |
| 2    | n8n production patterns and AI agents: Capstone 1                           |
| 3-4  | RAG,                                                                        |
