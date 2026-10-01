# IMPLEMENTATION_PLAN.md: ANVAYA MVP Build Plan

| Attribute | Detail |
|---|---|
| Version | 1.0.0 |
| Build window | 14 days (2 weeks) |
| Team roles | R1 tech lead / backend · R2 parsers & schema · R3 AI/ML · R4 frontend · R5 DevOps · R6 data / QA / pitch |
| Depends on | All other spec files. BUILD_PROMPT.md turns this plan into per-module prompts. |
| Version rule | Install the latest stable versions at build time and commit the lockfiles (`uv.lock` or `requirements.lock`, `package-lock.json`). Container base images are set by build args; replace them with the exact tag you tested. |

---

## 1. Definition of "done" for the whole MVP

All of the following hold on the recording laptop:
1. `./demo.sh --speed 10` brings the stack to the first frame of DEMO_SCRIPT.md §2.
2. The golden-path checklist (§9) passes 3 times in a row, then 10 times in a row before recording.
3. `make test` passes.
4. `make e2e` passes.
5. `make offline-test` passes.
6. `npm run lint` passes (including `no-raw-hex`).

## 2. Repository tree

```
anvaya/
├── README.md                         # pitch, a–k claims table (PRD.md §6), how to run, demo, roadmap
├── Makefile
├── docker-compose.yml
├── demo.sh
├── .gitignore                        # data/, **/keys/, node_modules/, dist/, .venv/
├── docs/
│   ├── spec/                         # the eight spec files, committed as-is
│   ├── ARCHITECTURE.md               # ≤ 2 pages for evaluators (built from TRD.md §2, §14 and BACKEND_SCHEMA.md §7)
│   ├── BENCHMARKS.md                 # written by `make bench`; measured numbers only
│   └── PROGRESS.md                   # session log: done / next / decisions / assumptions
├── backend/
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── app/
│   │   ├── main.py                   # create_app(); startup: keys, schema, packs, tasks; supervisor
│   │   ├── selfcheck.py              # used by make offline-test
│   │   ├── core/        interfaces.py  models.py  container.py  clock.py  ids.py  queue.py  config.py
│   │   ├── storage/     schema.sql  sqlite_repo.py  flatten.py
│   │   ├── ingest/      udp.py  http.py  vetting.py
│   │   ├── vault/       store.py  merkle.py  ledger.py  keys.py  witness.py
│   │   ├── pipeline/    worker.py  replay.py
│   │   ├── parsers/     syslog_header.py  registry.py  engine.py  extractors.py  jsonspan.py  shape.py
│   │   │   └── packs/   cisco-asa.yaml  suricata-eve.yaml  fortigate-kv.yaml
│   │   ├── ocsf/        mapper.py  enums.py  validate.py  lineage.py  roundtrip.py  confidence.py
│   │   │   └── schema/  network_activity.json  detection_finding.json  authentication.json  VERSION
│   │   ├── intel/       correlate.py  silence.py
│   │   ├── search/      nl2sql.py  templates.py  sql_guard.py  llm.py  prompt.py
│   │   ├── aiparser/    drafts.py  sources.py  prebake.py  prompt.py
│   │   │   └── cache/   acme-edge.json
│   │   ├── delivery/    parquet.py  siem.py  retention.py
│   │   ├── demo/        controller.py  netprobe.py
│   │   ├── stream/      bus.py  sse.py
│   │   └── api/         routes_health.py  routes_ingest.py  routes_events.py  routes_ledger.py
│   │                    routes_sources.py  routes_parsers.py  routes_incidents.py  routes_search.py
│   │                    routes_delivery.py  routes_demo.py  routes_stream.py  errors.py
│   └── tests/
│       ├── samples/     cisco-asa/*.log *.json   suricata-eve/…   fortigate-kv/…   acme-edge/…
│       └── test_*.py    (list in §8)
├── generator/           __init__.py  scenario.py  formats.py  br01.py  sinks.py
├── bench/               load_test.py
├── frontend/
│   ├── Dockerfile  nginx.conf  package.json  vite.config.ts  tailwind.config.ts  tsconfig.json
│   ├── scripts/no-raw-hex.mjs
│   ├── e2e/golden-path.spec.ts       # Playwright
│   └── src/
│       ├── main.tsx  App.tsx  routes.tsx
│       ├── styles/tokens.css  styles/globals.css
│       ├── lib/         api.ts  types.ts  queryKeys.ts  stream.ts  tokens.ts  format.ts  sqlHighlight.ts  lineage.ts
│       ├── state/       stream.ts  feed.ts  stats.ts  incidents.ts  sources.ts  integrity.ts  replay.ts  demo.ts  lineage.ts  index.ts
│       ├── components/  ui/ (shadcn)  shell/  presenter/  common/  badges/
│       └── features/    overview/  feed/  event/  incidents/  search/  integrity/  sources/  delivery/
└── data/                             # bind-mounted for local dev only (gitignored)
```

## 3. Setup commands

Run these once, while online:

```bash
# 0. Prerequisites: Docker Engine with the Compose plugin, Python 3 (for local tests), Node.js LTS, git.
git clone "$ANVAYA_REPO_URL" anvaya && cd anvaya   # set ANVAYA_REPO_URL to your team repository first

# 1. Backend dev environment
python3 -m venv .venv && source .venv/bin/activate
pip install -e "backend[dev]"            # installs the latest stable deps declared in pyproject.toml
pip freeze > backend/requirements.lock   # commit this lock

# 2. Frontend dev environment
cd frontend && npm install && npx playwright install chromium && cd ..

# 3. Vendor the OCSF schema (once; commit the files)
#    From the OCSF schema browser (schema.ocsf.io), at the latest stable version, export the JSON Schema for
#    the classes network_activity, detection_finding and authentication. Save them as
#    backend/app/ocsf/schema/{network_activity,detection_finding,authentication}.json, and write the version
#    string (for example 1.3.0) to backend/app/ocsf/schema/VERSION.
make check-ocsf                          # fails if a file is missing or not valid JSON Schema

# 4. Build images and pull the local model (the model is stored in the ollama-models volume)
docker compose build
docker compose --profile llm up -d ollama
docker compose exec ollama ollama pull qwen2.5:3b

# 5. Prebake the AI parser draft (needs Ollama running; writes backend/app/aiparser/cache/acme-edge.json)
make prebake
git add backend/app/aiparser/cache/acme-edge.json

# 6. First run
./demo.sh                                # stack up, reset, ready at http://localhost:8080/overview
```

**Day-to-day development:**
- backend with reload: `make dev-api` (Uvicorn on :8000, data in `./data`)
- frontend: `make dev-ui` (Vite on :5173, which proxies `/api` to :8000)
- tests: `make test`

## 4. Container and script files (complete)

### 4.1 `docker-compose.yml`

```yaml
name: anvaya
services:
  api:
    build:
      context: .                      # repo root, so the image can include generator/
      dockerfile: backend/Dockerfile
      args:
        PYTHON_IMAGE: ${PYTHON_IMAGE:-python:3-slim}
    image: anvaya-api
    environment:
      ANVAYA_DEMO_MODE: ${ANVAYA_DEMO_MODE:-1}
      ANVAYA_DATA_DIR: /data
      ANVAYA_SYSLOG_UDP_PORT: "5514"
      OLLAMA_URL: http://ollama:11434
      OLLAMA_MODEL: qwen2.5:3b
      DISPLAY_TZ: Asia/Kolkata
      SOURCE_TZ_DEFAULT: Asia/Kolkata
    ports:
      - "8000:8000"
      - "5514:5514/udp"
    volumes:
      - anvaya-data:/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/api/health', timeout=2).status == 200 else 1)"]
      interval: 3s
      timeout: 3s
      retries: 40
    restart: unless-stopped

  ui:
    build:
      context: ./frontend
      args:
        NODE_IMAGE: ${NODE_IMAGE:-node:lts-slim}
        NGINX_IMAGE: ${NGINX_IMAGE:-nginx:stable-alpine}
    image: anvaya-ui
    ports:
      - "8080:80"
    depends_on:
      api:
        condition: service_healthy
    restart: unless-stopped

  ollama:
    image: ${OLLAMA_IMAGE:-ollama/ollama}
    profiles: ["llm"]
    ports:
      - "11434:11434"
    volumes:
      - ollama-models:/root/.ollama
    restart: unless-stopped

volumes:
  anvaya-data:
  ollama-models:
```

The services share Compose's default network, so `api` reaches `ollama` by name. When the `llm` profile isn't running, `OLLAMA_URL` doesn't resolve, the LLM status is `fallback`, and search uses templates.

### 4.2 `backend/Dockerfile`

Built from the repo root (see the `api` build block above), so it can copy both `backend/app` and `generator/`.

```dockerfile
ARG PYTHON_IMAGE=python:3-slim
FROM ${PYTHON_IMAGE}
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /srv
COPY backend/requirements.lock backend/pyproject.toml ./
RUN pip install --no-cache-dir -r requirements.lock
COPY backend/app ./app
COPY generator ./generator
EXPOSE 8000 5514/udp
CMD ["uvicorn", "app.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
```

### 4.3 `frontend/Dockerfile` and `frontend/nginx.conf`

```dockerfile
ARG NODE_IMAGE=node:lts-slim
ARG NGINX_IMAGE=nginx:stable-alpine
FROM ${NODE_IMAGE} AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run lint && npm run build

FROM ${NGINX_IMAGE}
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
```

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  location /api/stream {
    proxy_pass http://api:8000/api/stream;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 1h;
  }
  location /api/ {
    proxy_pass http://api:8000/api/;
    proxy_http_version 1.1;
  }
  location / {
    try_files $uri /index.html;
  }
}
```

### 4.4 `demo.sh`

```bash
#!/usr/bin/env bash
# Reset ANVAYA to the first frame of the demo script. Usage: ./demo.sh [--speed 10|20|40] [--no-llm]
set -euo pipefail
SPEED=20
USE_LLM=1
while [[ $# -gt 0 ]]; do
  case "$1" in
    --speed) SPEED="$2"; shift 2 ;;
    --no-llm) USE_LLM=0; shift ;;
    *) echo "Unknown option: $1"; exit 2 ;;
  esac
done
if [[ "$SPEED" != "10" && "$SPEED" != "20" && "$SPEED" != "40" ]]; then
  echo "Speed must be 10, 20 or 40"; exit 2
fi

API="http://localhost:8000/api"
json() { python3 -c "import sys,json; d=json.load(sys.stdin); print($1)"; }

running() { docker compose ps --status running --services 2>/dev/null | grep -qx "$1"; }

# 1. If the stack isn't running, check the ports are free and start it.
if ! running api; then
  for p in 8000 8080; do
    if (exec 3<>/dev/tcp/127.0.0.1/$p) 2>/dev/null; then
      echo "Port $p is already in use by another program. Free it and run ./demo.sh again."; exit 1
    fi
  done
  if [[ "$USE_LLM" == "1" ]]; then
    docker compose --profile llm up -d
  else
    docker compose up -d
  fi
elif [[ "$USE_LLM" == "1" ]] && ! running ollama; then
  docker compose --profile llm up -d ollama
fi

# 2. Wait for the API to be healthy (60 s max).
for i in $(seq 1 60); do
  if curl -fsS "$API/health" >/dev/null 2>&1; then break; fi
  sleep 1
  if [[ "$i" == "60" ]]; then echo "API did not become healthy. Run: docker compose logs api"; exit 1; fi
done

# 3. Reset everything and set the speed.
curl -fsS -X POST "$API/demo/reset" >/dev/null || { echo "Reset failed. Run: docker compose restart api"; exit 1; }
curl -fsS -X POST "$API/demo/speed" -H 'Content-Type: application/json' -d "{\"speed\": $SPEED}" >/dev/null

# 4. Confirm the clean state.
for i in $(seq 1 30); do
  RAW=$(curl -fsS "$API/stats" | json "d['raw_total']")
  [[ "$RAW" == "0" ]] && break
  sleep 0.5
done
if [[ "$RAW" != "0" ]]; then echo "State not clean (raw_total=$RAW). Run ./demo.sh again."; exit 1; fi

# 5. Report the health chips the video depends on.
HEALTH=$(curl -fsS "$API/health")
LLM=$(echo "$HEALTH" | json "d['components']['llm']['status']")
NET=$(echo "$HEALTH" | json "d['components']['network']['status']")
echo "Local LLM: $LLM   Network: $NET   Speed: ${SPEED}x"
[[ "$LLM" != "ready" ]] && echo "Note: Search will use query templates — use the alternate narration lines."
[[ "$NET" == "external_route" ]] && echo "Warning: this machine has an external route. Turn Wi-Fi off for the offline shots."
echo "ANVAYA ready · open http://localhost:8080/overview"
```

### 4.5 `Makefile`

```make
.PHONY: dev-api dev-ui up up-llm down demo test test-backend test-frontend e2e bench prebake offline-test check-ocsf lint

dev-api:        ; ANVAYA_DEMO_MODE=1 ANVAYA_DATA_DIR=./data uvicorn app.main:create_app --factory --reload --app-dir backend --port 8000
dev-ui:         ; cd frontend && npm run dev
up:             ; docker compose up -d
up-llm:         ; docker compose --profile llm up -d
down:           ; docker compose --profile llm down
demo:           ; ./demo.sh
test: test-backend test-frontend
test-backend:   ; cd backend && pytest -q
test-frontend:  ; cd frontend && npm run lint && npx vitest run
e2e:            ; ./demo.sh --speed 40 && cd frontend && npx playwright test e2e/golden-path.spec.ts
bench:          ; python bench/load_test.py --api http://localhost:8000 --out docs/BENCHMARKS.md
prebake:        ; docker compose exec api python -m app.aiparser.prebake && docker compose cp api:/srv/app/aiparser/cache/acme-edge.json backend/app/aiparser/cache/acme-edge.json
offline-test:   ; docker run --rm --network none -e ANVAYA_DEMO_MODE=1 -e ANVAYA_DATA_DIR=/tmp/anvaya anvaya-api python -m app.selfcheck
check-ocsf:     ; python -m app.ocsf.validate --check-vendored --app-dir backend
lint:           ; cd backend && ruff check . && cd ../frontend && npm run lint
```

### 4.6 `bench/load_test.py` (behaviour)

1. Reset via `/api/demo/reset`.
2. Pre-generate 50,000 steady-phase lines (§5.3 of BACKEND_SCHEMA.md) in memory.
3. Send them over UDP as fast as possible from one socket for 30 s.
4. Every second, poll `/api/stats` for `raw_total` and `normalized_total`.
5. Report sustained raw events/s (the median of per-second deltas after the first 5 s), normalized events/s, the UDP datagrams sent minus received (loss), and the p95 time from send to `normalized_total` catching up.
6. Append a dated table row with the CPU model (`/proc/cpuinfo` or `platform.processor()`), core count, RAM and OS.
7. Print the row.

It never writes estimates; if the run fails, it writes nothing.

## 5. Team roles

| Role | Owns | Reviews |
|---|---|---|
| **R1** Tech lead / backend | FastAPI app, interfaces, SQLite repo, ingest, vault, ledger, witnesses, API, SSE | R2, R5 PRs |
| **R2** Parsers & schema | Syslog header, pack format, engine, extractors (incl. span-aware JSON), 3 packs + acme-edge, OCSF mapper/validator, lineage, round-trip, confidence, quarantine, replay | R1, R3 PRs |
| **R3** AI/ML | Ollama client, NL-to-SQL, SQL guard, templates, AI draft service, prebake, correlation, silence | R2 PRs |
| **R4** Frontend | Tokens, shell, all 8 screens, stores, stream client, presenter controls, accessibility | R6 reviews UX |
| **R5** DevOps | Compose, Dockerfiles, nginx, demo controller and clock, demo.sh, Makefile, Parquet / SIEM / report, netprobe, selfcheck, bench | R1 PRs |
| **R6** Data / QA / pitch | Generator, golden samples, backend tests, Playwright e2e, golden-path checklist, README, ARCHITECTURE.md, 5-slide technical deck, recording | Everyone's tests |

Rules: one owner per module; every PR is reviewed by one other role; `main` is always demo-ready (CI green); each role updates `docs/PROGRESS.md` at the end of each day.

## 6. Phased plan (14 days)

Each phase's definition of done is checked by **running the app**, not by reading code.

### Phase 0 · Foundations (Day 1)

| Task | Owner |
|---|---|
| Repo tree from §2; `pyproject.toml` with deps; Vite + TS + Tailwind + shadcn init; CI (lint + tests) | R1, R4, R5 |
| `core/interfaces.py`, `core/models.py`, `lib/types.ts` exactly from TRD.md §4 and BACKEND_SCHEMA.md §6 | R1, R4 |
| `tokens.css`, Tailwind mapping, `no-raw-hex` lint, fonts via @fontsource | R4 |
| Vendor the OCSF schema (§3 step 3); pull the Ollama model | R2, R3 |
| Generator skeleton with the §5.2 formulas and `--dry-run` | R6 |
| Compose, Dockerfiles, nginx, Makefile, demo.sh stubs (§4) | R5 |

**DoD:** `docker compose up -d` serves `http://localhost:8080` (the shell renders with tokens) and `http://localhost:8000/api/health` returns 200. `python -m generator.scenario --dry-run | wc -l` prints `207`.

### Phase 1 · Core pipeline (Days 2–4)

| Task | Owner |
|---|---|
| UDP + HTTP ingest, vetting, `SegmentRawStore`, `raw_events` | R1 |
| Merkle, ledger sealer, keys, witnesses, `verify_all`, `verify_record` | R1 |
| Syslog header split, pack registry and detect, engine, extractors (regex, json-span, kv, delimited) | R2 |
| cisco-asa, suricata-eve, fortigate-kv packs; mapper, validator, lineage, round-trip, confidence | R2 |
| Golden sample files from BACKEND_SCHEMA.md §4.1–4.3; `test_parsers.py`, `test_roundtrip.py`, `test_generator.py` | R6 |
| `ScenarioClock`, `DemoController.start/reset/speed`, `/api/demo/*` stubs returning real state | R5 |
| App shell, top bar, nav, health chips, Overview counters wired to `/api/stats` | R4 |
| Ollama client with warm-up and timeout; SQL guard with tests | R3 |

**DoD:**
- `curl -X POST localhost:8000/api/demo/scenario/start` produces 207 `raw_events` and 207 `events` (check `/api/stats`).
- `curl -X POST localhost:8000/api/ledger/verify` returns `verified`.
- `curl localhost:8000/api/events/e-899f4edf0030` returns the envelope in BACKEND_SCHEMA.md §4.1 (differences allowed only in `received_at`).
- Overview counters move during the run.

### Phase 2 · Intelligence and core screens (Days 5–7)

| Task | Owner |
|---|---|
| All read APIs (events, sources, incidents, ledger batches, witnesses) + SSE bus with replay buffer | R1 |
| Quarantine with shape signature, replay | R2 |
| Correlation engine; silence monitor; NL search service with templates and the full prompt | R3 |
| Live Feed (virtualised, coalesced), Event Detail lineage view, Incidents + toast, Search | R4 |
| Demo silence, tamper and reseal endpoints; presenter keys backend contract | R5 |
| `test_correlation.py`, `test_silence.py`, `test_search_templates.py`, `test_sql_guard.py`, `test_ledger.py` (tamper, reseal) | R6 |

**DoD:**
- Running the scenario from the UI shows the feed, opens `inc-0001` (risk 95) with the toast, and Event Detail hover highlights bytes 166–178 for `evidences[0].src_endpoint.ip`.
- Search Q1 returns 17 rows with Ollama running **and** with `docker compose stop ollama`.

### Phase 3 · Integrity, onboarding and delivery (Days 8–10)

| Task | Owner |
|---|---|
| Ledger verify / tamper / reseal APIs final; record-verify proof | R1 |
| acme-edge pack; draft test gates (held-out, all, round-trip, OCSF); approve → registry reload → replay | R2 |
| Draft service with Cached/Llm sources; `make prebake` | R3 |
| Integrity screen, Sources & parsers (tabs, dialog), Delivery screen, presenter panel and keys | R4 |
| Parquet export (DuckDB), SIEM file sink, retention report, netprobe, `selfcheck.py`, `make offline-test` | R5 |
| Playwright `golden-path.spec.ts`; `test_parser_drafts.py`, `test_parquet.py`, `test_api_contract.py` | R6 |

**DoD:**
- The full golden path (§9) runs end to end once by hand.
- `make offline-test` exits 0.
- `make e2e` passes.

### Phase 4 · Demo packaging and polish (Days 11–12)

| Task | Owner |
|---|---|
| `demo.sh` final; port checks; health report lines | R5 |
| `make bench` run on the recording laptop; BENCHMARKS.md | R5, R6 |
| Camera pass at 1920×1080 and 1440×900 for every screen; all empty / loading / error states checked by forcing them (stop api, reset, 404s) | R4, R6 |
| Copy pass: every string matches UI_UX_BRIEF.md; no console errors; reduced-motion check | R4 |
| Performance: feed at 40× stays smooth (no dropped frames visible); verify < 100 ms for 300 records | R1, R4 |
| README with the a–k table and run instructions; ARCHITECTURE.md (2 pages) | R6, R1 |

**DoD:** the golden-path checklist passes 3 times in a row from `./demo.sh --speed 10` and from `./demo.sh`.

### Phase 5 · Rehearsal and recording (Days 13–14)

| Task | Owner |
|---|---|
| 10 consecutive golden-path runs; log any deviation in PROGRESS.md and fix it | R6 + owners |
| Record the takes in DEMO_SCRIPT.md §5 (3:50 main, 2:00 compliance, 1:00 short) | R6 (presenter), R4 (screen) |
| Narration recording and edit at shot boundaries | R6 |
| 5-slide technical deck from the spec (problem, architecture, differentiators, results measured, roadmap) | R6, R1 |
| Final repo tidy: tag `v1.0.0-demo`, freeze dependencies | R1 |

**DoD:** both video cuts are exported, the repo is tagged, and the README links the video.

## 7. Behind schedule: cut in this order

Cut from the top. Never cut the items marked **keep**.

1. Delivery **screen** (keep `POST /api/export/parquet` and mention it in narration over the Overview; drop shot 3:20–3:32 from the main cut).
2. Retention report and SIEM forwarder cards (keep the API endpoints if already built).
3. Insider re-seal (SCN-02b) and its shot (keep witness co-signing and witness status in verify).
4. The 60-second cut.
5. Rate sparklines on Overview and Sources (show the numbers only).
6. TCP syslog listener (UDP and HTTP are enough).
7. Presenter **panel** UI (keep the keys).

**Keep:** lineage view with real spans; round-trip check; raw vault + Merkle ledger + tamper detection; correlation incident; NL search with the template fallback; quarantine → draft gates → approve → replay; silence alert; offline self-check.

## 8. Tests that matter

### 8.1 Backend (`backend/tests/`, pytest)

| File | What it proves |
|---|---|
| `test_generator.py` | 207 scripted lines; per-device counts 104/33/70; seq checkpoints 40, 100, 164, 180 with their `raw_id`s; no `random` import; steady lines never contain 198.51.100.9 or 10.2.4.18 |
| `test_vault.py` | Append/read byte-exact; SHA-256 over exact bytes; segment rotation; `overwrite_byte_for_demo` changes exactly 1 byte |
| `test_merkle.py` | RFC 6962 roots for n = 1…17; audit paths verify; a modified leaf fails |
| `test_ledger.py` | Count-triggered seals put seq 164 in batch 11; verify → `verified`; tamper → exactly 1 failed record at offset 153 (`'8'` → `'9'`, field `dst_endpoint.ip`); reseal → `witness_mismatch` from batch 11 |
| `test_witness.py` | Rejects bad signatures and prev-head gaps; never overwrites a batch |
| `test_syslog_header.py` | The three header patterns and their spans |
| `test_parsers.py` | Every sample line in `tests/samples/*/*.log` produces its golden `*.json` envelope (ignoring `received_at`) |
| `test_ocsf_validation.py` / `test_ocsf_enums.py` | Every golden `ocsf` object validates against the vendored schema; the enum ids used match the schema |
| `test_roundtrip.py` | All golden samples: `value_check` true, coverage 1.0, dropped 0; a deliberately broken pack (wrong span) fails |
| `test_confidence.py` | Golden scores 91 / 90 / 89 / 95 |
| `test_correlation.py` | The scripted scenario → exactly 1 incident, risk 95, 56 events; noise-only → 0; replaying the same events twice → still 1 |
| `test_silence.py` | With a fake clock: fw01 silent after 15 s; br01 burst never monitored; recovery clears status |
| `test_sql_guard.py` | Rejects DROP, UPDATE, ATTACH, PRAGMA, multiple statements, other tables, unknown columns; enforces LIMIT 200 |
| `test_search_templates.py` | Q1 → 17 rows, Q2 → (40, 2), Q3 → 12 rows, Q4 → 0 before / 20 after approval, using the template engine |
| `test_parser_drafts.py` | The cached draft passes held-out 6/6, all 20/20, round-trip 20/20, OCSF 20/20; approve replays 20 and empties quarantine; approve without tests → 409 |
| `test_parquet.py` | Export row count equals DuckDB read-back count; no network access needed |
| `test_api_contract.py` | Each endpoint in TRD.md §16 returns the documented shape (Pydantic response models) and the documented errors |

### 8.2 Frontend (Vitest)

- `lineage.test.ts`: `spanToPath` picks the narrowest span; `fieldOrder` puts `unmapped` last.
- `format.test.ts`: `formatEventSummary` for each row of UI_UX_BRIEF.md §4.4; "—" for nulls.
- `feedStore.test.ts`: coalescing (≤ 12 rows per flush); duplicate `event_id` ignored.

### 8.3 End-to-end (Playwright, `frontend/e2e/golden-path.spec.ts`)

At 40× speed, the test:
1. Starts the simulation and waits for the incident toast (≤ 20 s).
2. Opens the incident and checks "Risk 95".
3. On Search, clicks Q1 and expects "17 rows".
4. On Integrity: verify → green; tamper → verify → "1 record failed verification" and "byte 153"; reseal → verify → "witness nodes disagree".
5. Injects br01 → drafts → tests → approves → expects "20 records replayed".
6. Silences fw01 and expects the "Silent" badge within 20 s.
7. Throughout, asserts `page.on('console')` saw no errors.

### 8.4 Offline self-check

`make offline-test` (TRD.md §15).

## 9. Golden-path manual checklist

Run this from `./demo.sh --speed 10`, then from `./demo.sh`. Tick every line; any miss means the take doesn't count.

| # | Check | Expected |
|---|---|---|
| 1 | Overview at load | All counters 0; parse coverage "—"; chips Vault ✓, Ledger ✓, Witnesses 2/2, Local LLM ready, No external route; "Synthetic lab data" pill visible |
| 2 | Presenter panel (`` ` `` twice) | Opens and closes; shows Phase idle and the selected speed |
| 3 | Start attack simulation | Navigates to Live Feed within 1 s; rows from fw01, ids01, vpn01 with three format chips |
| 4 | First ids01 alert row | Opens Event Detail; hovering `evidences[0].src_endpoint.ip` highlights bytes 166–178 with a connector |
| 5 | Lineage legend and chips | Derived fields marked ◇; unmapped fields ○; "Lossless ✓ 100% of bytes accounted for" |
| 6 | Verify this record | "Verified · sealed in batch 3 · co-signed by 2 witness nodes" |
| 7 | VPN burst on Live Feed | 40 failures cascade readably (no jank) |
| 8 | Incident toast | Appears once, at scenario 10:07:12, "High-risk incident · 198.51.100.9 → 10.2.4.18 · 3 sources" |
| 9 | Incident detail | Risk 95 · High; three stages with the headlines from TRD.md API 21; 56 events |
| 10 | Search Q1 | Engine label shown; 17 rows; first row 10:08:30 fw01 |
| 11 | Search Q2 | 198.51.100.9 = 40, 203.0.113.60 = 2 |
| 12 | Verify ledger | Green banner; both witnesses agree |
| 13 | Key `4` then Verify | Red banner; 1 record; "byte 153 changed '8' → '9' (dst_endpoint.ip)"; Open event shows the red byte |
| 14 | Key `5` then Verify | Amber banner "witness nodes disagree on batch 11 onward" |
| 15 | Key `3` (silent) | No toast at press time |
| 16 | Key `2` | Quarantine (20) group card with shape "12 pipe-delimited fields starting with EDGE" |
| 17 | Draft parser | Origin label "Drafted by local LLM (cached run)" (or "Hand-authored draft"; then use the alternate line) |
| 18 | Run tests | 6/6, 20/20, 20/20, 20/20; Approve enabled |
| 19 | Approve & replay | Progress to 20/20; toast; quarantine count 0 |
| 20 | Search Q4 | 20 rows |
| 21 | Devices tab | fw01 "Silent · last event {n} s ago" (n ≥ 15) |
| 22 | Delivery | Export shows matching row and read-back counts; SIEM card shows the last payload; retention statement exact |
| 23 | Overview close | 4 formats detected; parse coverage 100% |
| 24 | Browser console | No errors or warnings during the whole run |
| 25 | `./demo.sh` again | Returns to check 1 in under 15 s |
