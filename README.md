# openagent-os

> The umbrella for the OpenAgent system — a version-pinned manifest of which commit
> of each service runs together, plus the runbook for bringing the whole thing
> up by hand. It builds nothing and owns nothing: every service runs from its
> own repo, with its own Docker setup and its own `.env`.

**Maintainer:** William McKeon ([github.com/william-mckeon](https://github.com/william-mckeon))  ·  **Status:** actively maintained  ·  Apache 2.0 License © 2026 William McKeon

---

## What this is

**OpenAgent** is a decoupled, multi-service reference implementation of a stateful AI agent — a set of small, single-responsibility services that together stand up a working agent. I built it to get the infrastructure right: clean boundaries between services, compartmentalized auth across every hop, and structured capture of what happens. Each service lives in its own repo with its own Docker setup and its own `.env`, and each is deployable on its own.

The system is **four core services plus one optional fifth.** The four core services (frontend, api, infra, logger) stand up a complete working agent. The fifth, **openagent-memory**, is an opt-in session-scoped retrieval layer (RAG); enable it for conversation memory, or leave it out entirely and the rest of the system is unaffected.

`openagent-os` is the umbrella over those services, and it is deliberately thin. It contains **no product code, no compose file, and no secrets.** It holds each service as a **git submodule** pinned to a specific commit, and it carries this README — the project's explanation and the runbook for standing the whole system up by hand.

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
openagent-api (:8001)             Identity gateway — persona, auth, prompt assembly, SSE relay, event emission
  ├──────────────▶ openagent-infra  (:8002) ──▶ BYOC Provider   (hot path: the response)
  ├──────────────▶ openagent-memory (:8004) ──▶ pgvector        (OPTIONAL: retrieve before, ingest after)
  └──────────────▶ openagent-logger (:8003) ──▶ Postgres        (side path: fire-and-forget capture)
```

Ports: **8000** frontend (user-facing), **8001** openagent-api, **8002** openagent-infra, **8003** openagent-logger, **8004** openagent-memory (optional), **5432** shared Postgres. Only 8000 is meant for users; the rest are internal to the stack. `openagent-memory`, when enabled, owns its **own** PostgreSQL + pgvector instance (separate from the shared Postgres) and reaches `openagent-infra` for embeddings.

---

## How it's put together

A few principles shaped the design:

* **Identity off the client.** `openagent-frontend` is a pure UI shell. The persona and agent logic live upstream in `openagent-api`, so any client — web, mobile, CLI — inherits one canonical agent instead of re-implementing it.
* **A robust identity gateway.** `openagent-api` acts as the traffic cop: it owns the persona, the compartmentalized auth chain, the prompt assembly, and the streaming relay. It is where OpenAgent stops being a UI idea and becomes a backend service.
* **Cryptographically verified capture.** `openagent-logger` is the capture layer. `openagent-api` emits a structured event stream and full conversation captures on every turn: append-only, HMAC-signed, retention-managed.
* **A reasoning model, a control model, and an embedding model.** `openagent-infra` proxies requests to three models — a **base model** for deep reasoning and everyday conversation, a **nervous-system model** as the fast control layer for routing, history filtering, and agent decisions, and an **embedding model** that turns text into vectors for retrieval. Deep work, quick decisions, and semantic lookup each handled by the right tool.
* **Optional session memory.** `openagent-memory` is an opt-in retrieval layer (RAG). It embeds each conversation turn into its own pgvector store and ranks the most relevant prior turns back into the prompt. The gateway assembles the prompt; memory only ranks. Retrieval is fail-open on the hot path, so the agent runs the same with or without it — memory is an enhancement, never a hard dependency.

---

## Service status

| Service | Submodule | Version | Status | Role |
| --- | --- | --- | --- | --- |
| openagent-frontend | `openagent-frontend/` | 1.0.0 | working | Streamlit chat UI |
| openagent-api | `openagent-api/` | 1.0.0 | working | Identity gateway: persona, auth, prompt assembly, SSE relay, event emission (+ optional memory retrieve/ingest) |
| openagent-infra | `openagent-infra/` | 1.0.0 | working | Model proxy → BYOC Provider (base reasoning + nervous-system control layer + embedding) |
| openagent-logger | `openagent-logger/` | 0.1.0 | working (pre-production) | Capture layer: ops events, conversation captures, audit; owns the network + shared Postgres |
| openagent-memory | `openagent-memory/` | 0.1.0 | working (pre-production) · **optional** | Session-scoped RAG layer: embeds and ranks prior turns; owns its own PostgreSQL + pgvector |

---

## Architecture

The diagram below shows the **core four-service system** — the two always-present boundaries off `openagent-api`: the `openagent-infra` hot path and the `openagent-logger` fire-and-forget sibling. The optional `openagent-memory` boundary is documented separately, just below, so the core diagram stays readable.

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
                              │ openagent-api      :8001 │  persona · auth · prompt assembly · SSE relay
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
                    │ BYOC Provider       │  │ openagent-shared-db │
                    │ base model          │  │ schema              │
                    │ nervous-system      │  │ openagent_logger    │
                    │ embedding           │  │                     │
                    └─────────────────────┘  └─────────────────────┘
```

* **Hot path:** frontend → openagent-api → openagent-infra → Compute Provider. This must succeed for the user to get a response.
* **Side path:** openagent-api → openagent-logger is fire-and-forget — if the logger is down, `/chat` is unaffected; events queue and are dropped per the overflow policy.
* **One network** (`openagent-network`), created and owned by **openagent-logger** (its `scripts/setup-network.*`). Every service attaches to it and addresses the others by name once they're all on it.
* **Shared Postgres** (`openagent-shared-db`), also brought up via **openagent-logger**'s setup, hosts the `openagent_logger` schema.

### Optional: openagent-memory (session retrieval)

When enabled, `openagent-api` gains a third outbound boundary to `openagent-memory` (:8004). It is consulted twice per turn — once on the hot path (retrieve, before prompt assembly) and once off the user's path (ingest, after a clean stream):

```text
   openagent-api (:8001)
     │
     ├─[hot path, before assembly]──▶ openagent-memory  POST /retrieve   (fail-open, bounded; never blocks the first token)
     │
     └─[off path, after the answer]─▶ openagent-memory  POST /ingest ×2  (signalled on failure, never blocks /chat)

   openagent-memory (:8004)  —  session-scoped RAG, OPTIONAL
     │   Auth in:  X-API-Key: MEMORY_API_KEY   (transport-key only — NO HMAC today)
     │
     ├──▶ openagent-infra  POST /embed         (X-API-Key: INFRA_API_KEY — turns text into vectors)
     └──▶ memory-owned PostgreSQL + pgvector   (its OWN database, NOT the shared openagent-shared-db)
```

* **Optional / opt-in.** Memory is enabled only when `openagent-api` is configured for it (`MEMORY_URL` + `MEMORY_API_KEY`). Absent that configuration, the gateway forwards full history exactly as before and never calls memory. It is never a refuse-to-boot dependency.
* **Memory ranks; the gateway builds.** `openagent-memory` returns ranked prior turns; `openagent-api` decides what actually goes into the prompt (`[bio] + [retrieved older turns] + [recent N turns] + [current turn]`), so the retrieval layer can be swapped without touching prompt policy.
* **Its own store.** `openagent-memory` owns its own PostgreSQL + pgvector instance for the conversation turns — distinct from the logger's shared Postgres. That database holds conversation content at rest; treat it with the same care as the logger's captures.
* **Reached only by openagent-api**, and only when enabled. It is never called from a browser.

---

## Security model (summary)

Authentication between services is **compartmentalized** — every boundary has its own independent secret, and no key is ever relayed unchanged to the next hop:

```text
frontend ──OPENAGENT_API_KEY──▶ openagent-api ──INFRA_API_KEY──▶ openagent-infra ──PROVIDER_API_KEY──▶ BYOC Provider
                               ├──LOGGER_API_KEY (+ LOGGER_HMAC_SECRET signing)──▶ openagent-logger
                               └──MEMORY_API_KEY (transport only, no HMAC)──▶ openagent-memory ──INFRA_API_KEY──▶ openagent-infra (/embed)
```

A single-service compromise is bounded to that one boundary. The logger boundary adds payload integrity: every event is HMAC-signed and the signature is stored on the row, so captures can be re-verified offline. The **memory boundary is transport-key only today** — `openagent-memory` uses `X-API-Key: MEMORY_API_KEY` and defines no HMAC contract yet (the client scaffolds signing for a future addition), so it has wire-access control but no offline payload-integrity proof for stored turns. Full detail lives in the security sections of openagent-api, openagent-logger, and openagent-memory.

There is **no central `.env`** — each secret lives in the `.env` of the service that uses it, and the shared values must match on both sides of every boundary. Two naming/sharing wrinkles to know:

* The api ↔ openagent-infra secret is the same value on both ends, but it is called `INFRA_API_KEY` in openagent-api and `API_KEY` in openagent-infra. When memory is enabled, **openagent-memory also holds that same value** (as `INFRA_API_KEY`) because it is a second caller of openagent-infra's `/embed`.
* `MEMORY_API_KEY` is a separate, independent value shared only between openagent-api and openagent-memory. openagent-memory also provisions its **own** database password for its own Postgres, internal to its stack.

---

## Bringing the system up

This is a manual, directory-by-directory bring-up. `openagent-os` does not drive it — each service is started from its own folder, with its own `.env` and its own `docker compose`, exactly as that repo documents. What follows is the **order** and the **wiring context**; for the specifics of any one service, follow that repo's own README.

**Prerequisites:** Docker; your choice of BYOC provider endpoints (e.g., RunPod serverless, OpenAI-compatible APIs) for the base and nervous-system models, plus an embedding endpoint **if you enable openagent-memory**; and the submodule folders present (four core, plus `openagent-memory` if you want it).

**0. Get the folders.** Clone openagent-os with its submodules (or initialize them after a plain clone):

```bash
git clone --recurse-submodules https://github.com/william-mckeon/openagent-os.git
cd openagent-os
# or, after a plain clone:
git submodule update --init --recursive
```

**1. Shared network + Postgres — owned by openagent-logger.** The `openagent-network` and the shared Postgres (`openagent-shared-db`) belong to openagent-logger, not openagent-os. Following **openagent-logger's own README**, do this first:

* create the network (its `scripts/setup-network.*`, equivalently `docker network create openagent-network`), and
* bring up the shared Postgres — openagent-logger's setup mounts its `database/init.sql` and passes the `openagent_logger` role password to Postgres via `PGOPTIONS`, so the role and schema are provisioned on first boot.

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

**4. openagent-memory (:8004) — OPTIONAL.** The session-scoped retrieval layer (RAG). **Skip this entire step to run without memory.** Unlike the other services, it brings up its **own** PostgreSQL + pgvector (not the shared Postgres). It calls openagent-infra's `/embed`, so bring openagent-infra up first.

```bash
cd openagent-memory
cp .env.example .env        # fill in per openagent-memory's README;
                            # set its INFRA_API_KEY to match openagent-infra's API_KEY
docker compose up -d --build
cd ..
```

**5. openagent-api (:8001)** — the identity gateway. Calls openagent-infra (hot path) and openagent-logger (fire-and-forget), plus openagent-memory when enabled, so bring those up first. If you ran step 4, also set `MEMORY_URL` + `MEMORY_API_KEY` (and `MEMORY_SESSION_ID`) in openagent-api's `.env`; leave them unset to run without memory.

```bash
cd openagent-api
cp .env.example .env        # fill in per openagent-api's README
docker compose up -d --build
cd ..
```

**6. openagent-frontend (host :8000)** — the UI. Talks only to openagent-api.

```bash
cd openagent-frontend
cp .env.example .env        # fill in per openagent-frontend's README
docker compose up -d --build
cd ..
```

**7. Open the UI:** http://localhost:8000

> **Secrets must match across boundaries.** With no central `.env`, each repo carries its own — and the shared values have to agree on both sides of every hop: `OPENAGENT_API_KEY` (frontend ↔ openagent-api); the api ↔ openagent-infra value, which is `INFRA_API_KEY` in openagent-api and the same value as `API_KEY` in openagent-infra; `LOGGER_API_KEY` + `LOGGER_HMAC_SECRET` (openagent-api ↔ openagent-logger); and the `openagent_logger` DB password (the value you provision Postgres with in step 1 must equal openagent-logger's `LOGGER_DB_PASSWORD`). **If you enabled memory:** `MEMORY_API_KEY` (openagent-api ↔ openagent-memory), and openagent-memory's `INFRA_API_KEY` must also equal openagent-infra's `API_KEY` since memory is a second caller of `/embed`. Each repo's `.env.example` documents its own variable names.

> **Reaching each other.** Each repo's `.env` also sets how it addresses its neighbors. With everything on `openagent-network`, point each service at the others by their container name (e.g. openagent-api → `http://openagent-infra:8002`, `http://openagent-logger:8003`, and when enabled `http://openagent-memory:8004`; openagent-memory → `http://openagent-infra:8002` for `/embed`; frontend → `http://openagent-api:8001`); the per-repo `.env.example` files list these alongside their standalone (`host.docker.internal`) alternatives.

> **Cold starts.** If you use scale-to-zero serverless endpoints for your BYOC layer, the first call after a quiet period waits for a worker to spin up. Note that `openagent-infra`'s `/health` reports a reachable-but-cold worker as `ok` — it answers "is the provider reachable?", not "is the model warm?" (it reports `degraded` only when a host genuinely cannot be reached). So the cold-start wait isn't gated away up front; it is absorbed on the first `/chat` (or `/embed`) call, where the read timeout is unbounded. Any "wait until warm" behavior in `openagent-frontend` therefore can't rely on `/health` alone to detect warmth. The same cold-start applies to the embedding endpoint behind `openagent-memory`: a cold embedder makes retrieval fail open (recent-only) until it warms, which never fails `/chat`.

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
├── openagent-logger/     # submodule — owns the openagent-network + shared Postgres setup
└── openagent-memory/     # submodule — OPTIONAL; session-scoped RAG, owns its own Postgres + pgvector
```

No compose file, no `.env`, no Makefile. `openagent-os` is this README and the submodule pointers; everything that runs lives in the service folders (four core, plus the optional `openagent-memory`).

---

## Note on the model setup

`openagent-infra` proxies requests to **three** separate functional models. Two are reasoning models reached over the `/chat` route: a **base model** (for deep reasoning and standard response generation) and a **nervous-system model** (a fast control layer for routing, history filtering, and metadata extraction). `openagent-api` routes to the base model by default; explicit routing to the control layer is handled via a model override parameter (e.g., `model="nervous_system"`).

The third is an **embedding model**, reached over a separate `/embed` route. It is a different *kind* of model — it turns text into vectors for retrieval rather than generating a reply, so it has no reasoning level and does not stream (it returns a single JSON response). It is optional: when no embedding endpoint is configured, `/embed` reports "not configured" and the `/chat` path is unaffected. When `openagent-memory` is enabled, it is the consumer of `/embed` — embedding each turn on ingest and each query on retrieval.

All three are reached as OpenAI-compatible endpoints behind the one proxy, which lets you mix and match providers or model sizes to balance compute cost against latency.