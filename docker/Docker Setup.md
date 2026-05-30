# Docker setup — CSxAI + RAGFlow

This folder documents how to run **RAGFlow** (and how it connects to **CSxAI-Backend**) using Docker. The monorepo does **not** ship a single Compose file for the whole stack; **RAGFlow** is a **separate clone** with its own `docker/` directory. CSxAI apps (Nest, Vite) usually run on the host via `pnpm` / `uv`.

For a shorter, workspace-specific walkthrough, see also [`docs/ragflow-local-docker.md`](../docs/ragflow-local-docker.md). For Nest API behaviour (`vectorBackend`, env vars, uploads), see [`docs/ragflow-backend-integration.md`](../docs/ragflow-backend-integration.md).

---

## 1. What you are running

| Piece | Where it runs | Role |
|--------|----------------|------|
| **RAGFlow** | Docker (`ragflow/docker`) | Datasets, document upload/parse, embeddings inside RAGFlow, retrieval API |
| **CSxAI-Backend** | Host (`pnpm dev`, port **3100** typical) | Knowledge API, BullMQ jobs, calls RAGFlow over HTTP when `vectorBackend=ragflow` |
| **CSxAI-Frontend-Web** | Host (`pnpm dev`, port **4200** typical) | Knowledge UI, upload queue |
| **Qdrant / Mongo / Redis** (standard KB path) | Your existing dev setup | Used when `vectorBackend` is Qdrant (default); RAGFlow path does **not** require Qdrant vectors for those indexes |

RAGFlow’s own reference for every Compose variable is upstream: `ragflow/docker/README.md` (in the RAGFlow repo clone).

---

## 2. Repository layout (important)

RAGFlow is **not** inside `csxai-turborepo/`. The usual layout is:

```text
csxai-repo/                    # parent folder (example name)
  csxai-turborepo/             # this monorepo (Nest + React)
    docker/
      README.md                # this file
    apps/
      CSxAI-Backend/
  ragflow/                     # separate git clone — https://github.com/infiniflow/ragflow
    docker/
      docker-compose.yml
      docker-compose-base.yml
      .env
      ...
```

If you do not have `ragflow/` yet:

```bash
cd /path/to/csxai-repo
git clone --depth 1 https://github.com/infiniflow/ragflow.git
```

---

## 3. Prerequisites

- **Docker Engine** and **Docker Compose v2** (`docker compose`, not only legacy `docker-compose`).
- **RAM:** plan for **8 GB+** free for Elasticsearch + RAGFlow; **16 GB** on the host is more comfortable.
- **Disk:** first pull is large (Elasticsearch, MySQL, MinIO, RAGFlow image, etc.).
- **Linux:** Elasticsearch needs `vm.max_map_count` (see section 4).

---

## 4. Linux: Elasticsearch `vm.max_map_count`

Required on many Linux hosts (once per machine, persist if you want it after reboot):

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elasticsearch.conf
sudo sysctl --system
```

On **Docker Desktop** (macOS/Windows) this is often unnecessary; if Elasticsearch exits immediately, check `docker compose logs` for the ES container.

---

## 5. Configure RAGFlow `docker/.env`

```bash
cd ragflow/docker
```

1. **Base env file**  
   Upstream ships `.env` in this directory. If your clone has `.env.example` instead, copy it to `.env` per upstream README.

2. **CPU vs GPU profile**  
   `docker-compose.yml` uses Compose **profiles** (`cpu` / `gpu`). Your `.env` should set the profile upstream expects (often `COMPOSE_PROFILES` or `DEVICE` — **read the comments at the top of your `ragflow/docker/.env`**). Typical local dev without NVIDIA GPUs:
   - Use the **CPU** profile so the **`ragflow-cpu`** service starts (not `ragflow-gpu`).

3. **Avoid port clashes with CSxAI / local services**  
   If `REDIS_PORT=6379` or web `80` conflicts with Nest or system services, override in `ragflow/docker/.env`, for example:

   ```bash
   REDIS_PORT=16379
   SVR_WEB_HTTP_PORT=8088
   EXPOSE_MYSQL_PORT=55455
   ```

   Leave **`SVR_HTTP_PORT=9380`** unless that port is taken; Nest uses **`RAGFLOW_URL=http://127.0.0.1:9380`** by default.

4. **Image tag**  
   `RAGFLOW_IMAGE` pins the server version. Keep it aligned with what your team tests (see upstream release notes).

---

## 6. Start RAGFlow

From **`ragflow/docker`** (not `csxai-turborepo/docker`):

```bash
cd /path/to/csxai-repo/ragflow/docker
docker compose up -d
docker compose ps
```

**Logs (RAGFlow app container — name may vary by profile):**

```bash
docker compose logs -f ragflow-cpu
# or, if you enabled GPU profile:
# docker compose logs -f ragflow-gpu
```

**All services (MySQL, ES, Redis, MinIO, RAGFlow):**

```bash
docker compose logs -f
```

**Note:** Upstream does **not** name the main service `ragflow`. On CPU profile the service is typically **`ragflow-cpu`**. If `docker compose ps` shows a different name, use that in `logs -f`.

---

## 7. First-time RAGFlow UI and API key

RAGFlow must be able to run **embeddings** (and often an **LLM** for some features). Short version:

1. Open the **web UI** (often `http://127.0.0.1:8088` if you set `SVR_WEB_HTTP_PORT=8088`; otherwise check `.env` → mapped host port for container `80`).
2. Sign in / complete initial setup per upstream.
3. Under **avatar / settings**, add a **model provider** and at least one **embedding** model (e.g. OpenAI embeddings, or **Builtin** / TEI if your image supports it).
4. Create an **API key** in the UI (**API** / **API keys**).  
   - REST base URL for automation is **`http://127.0.0.1:9380`** (API port), **not** the browser port alone.
   - Send `Authorization: Bearer <key>` on HTTP requests.

Detailed UI steps and smoke `curl` are in [`docs/ragflow-local-docker.md`](../docs/ragflow-local-docker.md) (sections 6–7).

---

## 8. Wire CSxAI-Backend to RAGFlow

In **`apps/CSxAI-Backend/.env`** (see **`apps/CSxAI-Backend/.env.example`** for all placeholders):

```bash
RAGFLOW_URL=http://127.0.0.1:9380
RAGFLOW_API_KEY=<paste key from RAGFlow UI>
# Optional tuning:
# RAGFLOW_REQUEST_TIMEOUT_MS=60000
# RAGFLOW_DEFAULT_CHUNK_METHOD=naive
```

Start the backend from **`csxai-turborepo`** (see root `CLAUDE.md`):

```bash
cd csxai-turborepo
pnpm install
pnpm --filter @repo/types build
pnpm dev --filter @csxai/backend
```

In the **Knowledge** UI, choose **RAGFlow** as the upload processing backend when testing the RAGFlow path. Queued uploads use `POST /api/knowledge/uploads?vectorBackend=ragflow` (optional `ragflowChunkMethod=…`).

---

## 9. Verify end-to-end

1. **RAGFlow alive**

   ```bash
   export RAGFLOW_API_KEY='your-key'
   curl -sS -H "Authorization: Bearer $RAGFLOW_API_KEY" \
     http://127.0.0.1:9380/api/v1/datasets
   ```

2. **Nest can reach RAGFlow**  
   Trigger a small RAGFlow-backed upload from the UI or curl with a JWT; job should leave `failed` if URL/key wrong.

3. **Agent / search**  
   Search still goes through Nest (`GET /api/knowledge/:id/search`); Nest calls RAGFlow retrieval for RAGFlow-backed indexes.

---

## 10. Day-to-day operations

| Task | Command / action |
|------|-------------------|
| Stop RAGFlow | `cd ragflow/docker && docker compose down` |
| Stop and **delete volumes** (wipe all RAGFlow data) | `docker compose down -v` (destructive) |
| Restart after `.env` change | `docker compose up -d` |
| Disk usage | `docker system df` |
| Update RAGFlow image | Change `RAGFLOW_IMAGE` in `.env`, then `docker compose pull && docker compose up -d` |

---

## 11. Troubleshooting

| Symptom | Things to check |
|---------|------------------|
| Elasticsearch exits / OOM | `vm.max_map_count`; host RAM; `docker compose logs` for ES |
| Nothing listens on 9380 | `docker compose ps`; correct **profile** (cpu vs gpu); service name `ragflow-cpu` vs `ragflow-gpu` |
| Port already allocated | Change host ports in `ragflow/docker/.env`, then `docker compose up -d` again |
| Nest: RAGFlow errors / 401 | `RAGFLOW_URL` must match API port; `RAGFLOW_API_KEY` must be the **API** key from RAGFlow UI |
| Uploads work in UI but Nest fails | Firewall / Docker network: Nest on host must reach `127.0.0.1:9380` (or host IP if RAGFlow runs on another machine) |
| Slow first document | Model download / cold start inside RAGFlow; watch `docker compose logs -f ragflow-cpu` |

---

## 12. Further reading

| Document | Content |
|----------|---------|
| [`docs/ragflow-local-docker.md`](../docs/ragflow-local-docker.md) | CSxAI-focused ports, profiles, UI + API key flow, `curl` smoke test |
| [`docs/ragflow-backend-integration.md`](../docs/ragflow-backend-integration.md) | Nest env, `vectorBackend`, uploads vs Qdrant, search mapping |
| `ragflow/docker/README.md` | Upstream variable reference, HTTPS example, `service_conf.yaml.template` |
| [`../CLAUDE.md`](../CLAUDE.md) | Monorepo-wide commands (pnpm filters, ports) |

---

## 13. Scope of this `docker/` folder

- **Included:** instructions and checklists for running **RAGFlow via its own repo** next to CSxAI, plus **env wiring** for CSxAI-Backend.
- **Not included here:** production hardening, TLS termination, multi-host Swarm/Kubernetes, or a full Docker Compose for Nest + Mongo + Redis + Qdrant (add those separately if your team standardizes on Compose for the whole stack).
