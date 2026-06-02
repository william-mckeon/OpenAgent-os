Here is the complete, line-by-line rewrite of the umbrella README, transformed perfectly into the **OpenAgent OS** open-source release.

I have applied every rule from our `OpenAgent_Scrub_Plan.md`: I stripped all proprietary references to "Arcus" and "Islander Intelligence," applied your name and the Apache 2.0 license, removed all "planned" future roadmap items to prevent vaporware, and updated the architecture diagrams to reflect the BYOC (Bring Your Own Compute) model abstraction.

Please copy this exactly as provided.

---

# openagent-os

> The umbrella for the OpenAgent system — a version-pinned manifest of which commit
> of each service runs together, plus the runbook for bringing the whole thing
> up by hand. It builds nothing and owns nothing: every service runs from its
> own repo, with its own Docker setup and its own `.env`.

**Maintainer:** William McKeon ([github.com/william-mckeon](https://www.google.com/search?q=https://github.com/william-mckeon))  ·  **Status:** active  ·  Apache 2.0 License © 2026 William McKeon

---

## What this is

**OpenAgent** is an enterprise-grade AI agent scaffold — a decoupled, modular operating system for building and deploying stateful agents. The system is built as a set of small, single-responsibility services, each in its own repo with its own Docker setup and its own `.env`, each deployable on its own.

`openagent-os` is the umbrella over those services, and it is deliberately thin. It contains **no product code, no compose file, and no secrets.** It holds each service as a **git submodule** pinned to a specific commit, and it carries this README — the project’s explanation and the runbook for standing the whole system up by hand.

It runs nothing and owns nothing. Each service is brought up from its own folder, with its own `docker compose` and its own `.env`, exactly as that repo was designed. The shared `openagent-network` and the shared Postgres are owned by **openagent-logger** — its setup scripts create the network and its README brings up the database. `openagent-os` simply pins the versions and documents the order they come up in.

---

## The system at a glance

```text
User
  │  http://localhost:8000
  ▼
openagent-frontend (:8000)        Streamlit chat UI
  │
  ▼
openagent-api (:8001)             Identity gateway — persona, auth, SSE relay, event emission
  ├──────────────▶ openagent-infra (:8002) ──▶ BYOC Provider    (hot path: the response)
  └──────────────▶ openagent-logger (:8003) ──▶ Postgres         (side path: fire-and-forget capture)

```

Ports: **8000** frontend (user-facing), **8001** openagent-api, **8002** openagent-infra, **8003** openagent-logger, **5432** Postgres. Only 8000 is meant for users; the rest are internal to the stack.

---

## How the scaffold is designed

OpenAgent is built to be a stateful, enterprise-ready architecture out of the box. The core design principles are:

* **Identity off the client.** `openagent-frontend` is a pure UI shell. The persona and agent logic live upstream in `openagent-api`, so every future client — web, mobile, CLI — inherits one canonical agent instead of re-implementing it.
* **A robust identity gateway.** `openagent-api` acts as the traffic cop: it owns the persona, the compartmentalized auth chain, and the streaming relay. It is where OpenAgent stops being a UI idea and becomes a backend service.
* **Cryptographically verified memory.** `openagent-logger` serves as the capture layer. `openagent-api` emits a structured event stream and full conversation captures on every turn: append-only, HMAC-signed, retention-managed.
* **A nervous system architecture.** `openagent-infra` proxies requests to a dual-model setup — a **base model** for deep reasoning and everyday conversation, and a **nervous-system model** as the fast control layer for routing, history filtering, and agent decisions. A reasoning model paired with a fast control model is the shape of a true agent OS.

---

## Service status

| Service | Submodule | Version | Status | Role |
| --- | --- | --- | --- | --- |
| openagent-frontend | `openagent-frontend/` | 1.0.0 | live | Streamlit chat UI |
| openagent-api | `openagent-api/` | 1.0.0 | live | Identity gateway: persona, auth, SSE relay, event emission |
| openagent-infra | `openagent-infra/` | 1.0.0 | live | Model proxy → BYOC Provider (base reasoning + nervous-system control layer) |
| openagent-logger | `openagent-logger/` | 1.0.0 | live | Capture layer: ops events, conversation captures, audit; owns the network + shared Postgres |

---

## Architecture

```text
                            Browser  →  http://localhost:8000
                                          │
                                          ▼
                              ┌──────────────────────────┐
                              │ openagent-frontend :8000 │  Streamlit UI
                              └───────────┬──────────────┘
                                          │  X-API-Key: OPENAGENT_API_KEY
                                          ▼
                              ┌──────────────────────────┐
                              │ openagent-api      :8001 │  persona · auth · SSE relay
                              └─────┬───────────────┬────┘
                  HOT PATH          │               │   FIRE-AND-FORGET
       X-API-Key: INFRA_API_KEY     │               │   X-API-Key: LOGGER_API_KEY + HMAC
                                    ▼               ▼
                    ┌─────────────────────┐  ┌─────────────────────┐
                    │ openagent-infra:8002│  │openagent-logger:8003│
                    │ model proxy         │  │capture layer        │
                    └─────────┬───────────┘  └─────────┬───────────┘
                Bearer PROVIDER_API_KEY                │ PostgreSQL
                              ▼                        ▼
                    ┌─────────────────────┐  ┌─────────────────────┐
                    │ BYOC Cloud/vLLM     │  │ openagent-shared-db │
                    │ base model          │  │ schema              │
                    │ nervous-system      │  │ openagent_logger    │
                    └─────────────────────┘  └─────────────────────┘

```

* **Hot path:** frontend → openagent-api → openagent-infra → Compute Provider. This must succeed for the user to get a response.
* **Side path:** openagent-api → openagent-logger is fire-and-forget — if the logger is down, `/chat` is unaffected; events queue and are dropped per the overflow policy.
* **One network** (`openagent-network`), created and owned by **openagent-logger** (its `scripts/setup-network.*`). Every service attaches to it and addresses the others by name once they’re all on it.
* **Shared Postgres** (`openagent-shared-db`), also brought up via **openagent-logger**’s setup, hosts the `openagent_logger` schema.

---

## Security model (summary)

Authentication between services is **compartmentalized** — every boundary has its own independent secret, and no key is ever relayed unchanged to the next hop:

```text
frontend ──OPENAGENT_API_KEY──▶ openagent-api ──INFRA_API_KEY──▶ openagent-infra ──PROVIDER_API_KEY──▶ BYOC Provider
                               └──LOGGER_API_KEY (+ LOGGER_HMAC_SECRET signing)──▶ openagent-logger

```

A single-service compromise is bounded to that one boundary. The logger boundary adds payload integrity: every event is HMAC-signed and the signature is stored on the row, so captures can be re-verified offline. Full detail lives in the security sections of openagent-api and openagent-logger.

There is **no central `.env**` — each secret lives in the `.env` of the service that uses it, and the shared values must match on both sides of every boundary. Note one naming wrinkle: the api ↔ openagent-infra secret is the same value on both ends, but it is called `INFRA_API_KEY` in openagent-api and `API_KEY` in openagent-infra.

---

## Bringing the system up

This is a manual, directory-by-directory bring-up. `openagent-os` does not drive it — each service is started from its own folder, with its own `.env` and its own `docker compose`, exactly as that repo documents. What follows is the **order** and the **wiring context**; for the specifics of any one service, follow that repo’s own README.

**Prerequisites:** Docker; your choice of BYOC provider endpoints (e.g., RunPod serverless, OpenAI-compatible APIs) for the base and nervous-system models; and the four submodule folders present.

**0. Get the folders.** Clone openagent-os with its submodules (or initialize them after a plain clone):

```bash
git clone --recurse-submodules https://github.com/william-mckeon/openagent-os.git
cd openagent-os
# or, after a plain clone:
git submodule update --init --recursive

```

**1. Shared network + Postgres — owned by openagent-logger.** The `openagent-network` and the shared Postgres (`openagent-shared-db`) belong to openagent-logger, not openagent-os. Following **openagent-logger’s own README**, do this first:

* create the network (its `scripts/setup-network.*`, equivalently `docker network create openagent-network`), and
* bring up the shared Postgres — openagent-logger’s setup mounts its `database/init.sql` and passes the `openagent_logger` role password to Postgres via `PGOPTIONS`, so the role and schema are provisioned on first boot.

Everything below attaches to that network and that database, so it has to exist first.

**2. openagent-logger (:8003)** — the capture layer. Needs Postgres healthy first.

```bash
cd openagent-logger
cp .env.example .env        # then fill it in per openagent-logger's README
docker compose up -d --build
cd ..

```

**3. openagent-infra (:8002)** — the model proxy. Independent of the other services.

```bash
cd openagent-infra
cp .env.example .env        # fill in per openagent-infra's README
docker compose up -d --build
cd ..

```

**4. openagent-api (:8001)** — the identity gateway. Calls openagent-infra (hot path) and openagent-logger (fire-and-forget), so bring those up first.

```bash
cd openagent-api
cp .env.example .env        # fill in per openagent-api's README
docker compose up -d --build
cd ..

```

**5. openagent-frontend (host :8000)** — the UI. Talks only to openagent-api.

```bash
cd openagent-frontend
cp .env.example .env        # fill in per openagent-frontend's README
docker compose up -d --build
cd ..

```

**6. Open the UI:** [http://localhost:8000](https://www.google.com/search?q=http://localhost:8000)

> **Secrets must match across boundaries.** With no central `.env`, each repo carries its own — and the shared values have to agree on both sides of every hop: `OPENAGENT_API_KEY` (frontend ↔ openagent-api); the api ↔ openagent-infra value, which is `INFRA_API_KEY` in openagent-api and the same value as `API_KEY` in openagent-infra; `LOGGER_API_KEY` + `LOGGER_HMAC_SECRET` (openagent-api ↔ openagent-logger); and the `openagent_logger` DB password (the value you provision Postgres with in step 1 must equal openagent-logger’s `LOGGER_DB_PASSWORD`). Each repo’s `.env.example` documents its own variable names.

> **Reaching each other.** Each repo’s `.env` also sets how it addresses its neighbors. With everything on `openagent-network`, point each service at the others by their container name (e.g. openagent-api → `http://openagent-infra:8002`, `http://openagent-logger:8003`; frontend → `http://openagent-api:8001`); the per-repo `.env.example` files list these alongside their standalone (`host.docker.internal`) alternatives.

> **Cold Starts.** If using scale-to-zero serverless endpoints for your BYOC layer, the first call after a quiet period may wait for a worker to spin up. `openagent-frontend`'s health gate handles this — it polls and unlocks the chat input once the proxy confirms the model is warm.

---

## Maintaining the system

Each service stays a fully independent repo — its own git, its own CI, its own deploy. `openagent-os` only records *which commit of each* the system was last known to run together, via the submodule pointers.

```bash
# 1. Work in a service on its own remote, as usual
cd openagent-api
git commit -am "..."
git push

# 2. Record the new commit in openagent-os
cd ..
git submodule update --remote --merge openagent-api   # or: cd openagent-api && git pull && cd ..
git add openagent-api
git commit -m "Bump openagent-api to <short-sha>"
git push

```

Bumping `openagent-os` is just moving a submodule pointer. `openagent-os` is a version-controlled manifest of which commits stand up together, plus this README explaining how to stand them up.

---

## Repo layout

```text
openagent-os/
├── README.md             # this file — the project doc + manual bring-up runbook
├── .gitmodules           # pins each service to a commit (created by `git submodule add`)
├── openagent-frontend/   # submodule
├── openagent-api/        # submodule
├── openagent-infra/      # submodule
└── openagent-logger/     # submodule — owns the openagent-network + shared Postgres setup

```

No compose file, no `.env`, no Makefile. `openagent-os` is this README and the submodule pointers; everything that runs lives in the four folders.

---

## Note on the dual-model architecture

`openagent-infra` is designed to proxy requests to **two** separate functional models — a **base model** (for deep reasoning and standard response generation) and a **nervous-system model** (a fast control layer for routing, history filtering, and metadata extraction). `openagent-api` routes to the base model by default; explicit routing to the control layer is handled via model override parameters (e.g., `model="nervous_system"`). This allows you to mix and match providers or model sizes to balance compute cost and latency.
