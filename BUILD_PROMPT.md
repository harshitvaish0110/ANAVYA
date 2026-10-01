# BUILD_PROMPT.md: Paste-Ready Prompts for Building the ANVAYA MVP

**Tool:** Claude Pro chats, one module per session. The prompts also work in Claude Code or Cursor.

**How to use this file:**
1. Start every new session by pasting the **Context header** (§1).
2. Attach the spec files that module lists.
3. Paste that module's prompt.
4. Run what it gives you, paste results or errors back, and move on only when its **Check** passes.
5. After each module, paste the agent's report into `docs/PROGRESS.md`.

The spec files live in `docs/spec/`: DEMO_SCRIPT.md, PRD.md, TRD.md, APP_FLOW.md, UI_UX_BRIEF.md, BACKEND_SCHEMA.md, IMPLEMENTATION_PLAN.md.

---

## 1. Context header (paste at the start of every session)

```
You are a senior engineer on a 6-person student team building ANVAYA, the MVP for Smart India Hackathon 2026,
problem statement SIH26156 "Universal Log Pre-processing Framework" (NTRO, Blockchain & Cybersecurity, Software).
Tagline: "Every Perimeter Log, One Verifiable Truth". Use only the name ANVAYA.

What ANVAYA does: ingests logs from perimeter devices (firewall, IDS, VPN, router) in any format, stores every
original byte untouched and hashed, seals them in Ed25519-signed Merkle batches co-signed by witness stores,
parses them with YAML parser packs into OCSF with byte-level lineage and a lossless round-trip proof,
and then correlates attacks across devices, alerts on log silence, answers plain-English questions with a local
LLM writing guarded read-only SQL, and exports Parquet. Everything runs offline in Docker.

The MVP exists to make a 3-4 minute demo video (plus a 2:00 cut) that runs identically every take.
The spec files I attach are the source of truth. If they disagree, priority is:
BACKEND_SCHEMA.md > TRD.md > APP_FLOW.md > UI_UX_BRIEF.md > PRD.md > DEMO_SCRIPT.md, and you tell me about the conflict.

Non-negotiables (never break these):
1. Storage, queue, clock, LLM, witness, draft source and SIEM sink go through the interfaces in TRD.md §4.
2. One DEMO_MODE flag (ANVAYA_DEMO_MODE=1) switches the scenario clock, /api/demo/* routes and cached drafts.
   The frontend talks only to the typed API client in src/lib/api.ts.
3. Frontend colours, fonts, radii, shadows and durations come only from tokens (UI_UX_BRIEF.md §2). No hex outside tokens.css.
4. Only real sample logs from BACKEND_SCHEMA.md and the generator. No lorem ipsum, no mock API responses, no invented numbers.
5. No console errors. No runtime network calls (no CDNs, no telemetry). Fonts via @fontsource.
6. The LLM is never in the live ingest path. Search falls back to templates silently; drafts are cached in DEMO_MODE.
7. Use the latest stable versions of dependencies at install time; don't pin versions from memory. I will lock them.
8. Write tests with the code. Every module ends with a command I can run and a concrete check.

Working style: don't ask questions unless truly blocked. Make a sensible assumption, state it in your report,
and continue. When you finish a module, STOP and report: files created/changed, commands to run, expected
output, assumptions made, and anything left undone.
```

## 2. Goal and quality bar

- **Goal:** a demo-video MVP whose golden path (DEMO_SCRIPT.md §2) runs identically 10 times in a row after `./demo.sh`, works with the network disconnected and with Ollama stopped, and looks like a finished product in every frame.
- **Quality bar:**
  - the core pipeline (hashing, parsing, lineage, verify, tamper detection) is genuinely real and tested
  - every screen has designed empty, loading, error and success states
  - copy matches UI_UX_BRIEF.md exactly
  - text sizes meet the camera rules
- **Do not build:** a Kafka-compatible stream or worker fleet, PostgreSQL, MinIO, a Wazuh/OpenSearch integration, RBAC, signed plugins, an offline install bundle, formats beyond the four in BACKEND_SCHEMA.md §4, authentication, multi-tenancy, mobile layouts, dark mode, i18n.

## 3. Module prompts

Run them in this order. Each prompt assumes the context header has been pasted and the listed files are attached.

---

### M00 · Scaffold and contracts
**Attach:** IMPLEMENTATION_PLAN.md, TRD.md, BACKEND_SCHEMA.md

```
Build module M00: the repository scaffold and shared contracts.
1. Create the repo tree from IMPLEMENTATION_PLAN.md §2 (empty modules may contain only a docstring).
2. backend/pyproject.toml declaring: fastapi, uvicorn[standard], pydantic, pyyaml, cryptography, jsonschema,
   sqlglot, httpx, duckdb, pyarrow, tzdata, regex; dev extras pytest, pytest-asyncio, ruff. No version pins.
3. backend/app/core/interfaces.py exactly as TRD.md §4; backend/app/core/models.py exactly as BACKEND_SCHEMA.md §6.1;
   backend/app/core/config.py reading every environment variable in the TRD.md §4 table with those defaults.
4. backend/app/storage/schema.sql exactly as BACKEND_SCHEMA.md §2 (all tables, indexes, triggers).
5. backend/app/main.py with create_app() exposing GET /api/health returning the TRD.md API 1 shape
   (components may report "not_checked" for now).
6. .gitignore from IMPLEMENTATION_PLAN.md §2; docs/PROGRESS.md with sections Done / Next / Decisions / Assumptions.
Tests: tests/test_schema.py creates the schema in a temp SQLite file and asserts every table exists;
tests/test_config.py checks the defaults.
Run: pip install -e "backend[dev]" && cd backend && pytest -q && uvicorn app.main:create_app --factory --port 8000
Check: curl -s localhost:8000/api/health returns JSON with "status" and "components".
Stop and report.
```

### M01 · Deterministic generator
**Attach:** BACKEND_SCHEMA.md, TRD.md

```
Build module M01: generator/ (scenario.py, formats.py, br01.py, sinks.py).
Implement BACKEND_SCHEMA.md §5 exactly: line templates (§5.1), the 207 scripted lines (§5.2), steady phase (§5.3)
and br01 injection (§5.4). No random module. Lines take a date parameter (default today in Asia/Kolkata).
Provide:
- build_scripted_lines(date) -> list[(scenario_time_ms, device, text)], sorted by (time, device_order fw01<ids01<vpn01)
- steady_lines(start_ms) -> iterator, per §5.3
- build_br01_lines(now_ms) -> list[str] of 20 lines
- async run(clock, sink, stop_flags) that sleeps until each line's scenario time using clock and sends via sink
- UdpSink(host, port) sending one datagram per line from one socket
- CLI: python -m generator.scenario --dry-run [--date 2026-09-30] prints the 207 lines
Tests (backend/tests/test_generator.py): counts 207 and fw01 104 / ids01 33 / vpn01 70; with date 2026-09-30,
line 164 equals the raw line in BACKEND_SCHEMA.md §4.1, line 40 equals §4.2, line 100 equals §4.3;
br01 line 0 with injection at 2026-09-30 10:14:03 IST equals §4.4; no 'random' import; steady lines never contain
198.51.100.9 or 10.2.4.18 or 'Deny' or 'ssl-login-fail'.
Run: cd backend && pytest -q tests/test_generator.py && python -m generator.scenario --dry-run --date 2026-09-30 | sed -n '164p'
Check: the printed line is exactly the §4.1 raw line.
Stop and report.
```

### M02 · Ingest, vetting and raw vault
**Attach:** TRD.md, APP_FLOW.md, BACKEND_SCHEMA.md

```
Build module M02: ingest/ (udp.py, http.py, vetting.py), vault/store.py, core/ids.py, core/queue.py, and the
raw_events part of storage/sqlite_repo.py.
Follow APP_FLOW.md FL-01 steps 1–6 and BACKEND_SCHEMA.md §1–§3:
- raw_id = "r-" + sha256(seq_be64 || raw_sha256)[:12]
- segment framing [u32 BE length][bytes]; rotation at 64 MiB
- fsync every 16 records or 250 ms
- vetting flags exactly as PRD.md EP-02
- POST /api/ingest per TRD.md API 3; UDP listener on ANVAYA_SYSLOG_UDP_PORT
- GET /api/raw/{raw_id} per API 6
- SegmentRawStore.overwrite_byte_for_demo(raw_id, offset, value) (only callable when DEMO_MODE)
Tests (test_vault.py): byte-exact read-back including non-UTF-8 bytes; sha256 over exact bytes; rotation;
overwrite changes exactly one byte; raw_id for seq 164 + the §4.1 line equals r-899f4edf0030.
Run: cd backend && pytest -q tests/test_vault.py; then start the app and send:
  printf '%s' '<166>Sep 30 2026 10:07:12 fw01 : %ASA-6-302013: Built inbound TCP connection 77801 for outside:198.51.100.9/40133 (198.51.100.9/40133) to inside:10.2.4.18/443 (10.2.4.18/443)' | nc -u -w1 127.0.0.1 5514
Check: sqlite3 data/anvaya.db "select seq, raw_sha256 from raw_events" shows d12173d4…4251.
Stop and report.
```

### M03 · Merkle ledger and witnesses
**Attach:** TRD.md, BACKEND_SCHEMA.md, APP_FLOW.md

```
Build module M03: vault/merkle.py, vault/ledger.py, vault/keys.py, vault/witness.py.
Implement TRD.md §8.1–§8.8 exactly: leaf/node/RFC 6962 root, head with "ANVAYA-HEAD-v1", Ed25519 signing,
sealing at 16 records or by age (300 scenario s in the scripted phase, 5 real s otherwise; tick every 250 ms),
InProcessWitness ×2 (witness-ntro, witness-certin) with their own keys and SQLite files, verify_all,
verify_record (audit path), tamper target logic (raw_id r-899f4edf0030, offset 153, '8'->'9'),
reseal_from(batch_id). Keys in /data/keys (0600), kept across reset.
Tests: test_merkle.py (roots for n=1..17 against a reference implementation in the test, audit paths, modified leaf fails);
test_witness.py (bad signature rejected, prev-head gap rejected, no overwrite);
test_ledger.py (with the generator's 207 lines ingested in-process: seq 164 in batch 11, seq 40 in batch 3;
verify → verified; tamper → exactly one failed record with first_diff_offset 153, changed {'from':'8','to':'9'};
reseal → verdict witness_mismatch with mismatched_batches starting at 11).
Run: cd backend && pytest -q tests/test_merkle.py tests/test_witness.py tests/test_ledger.py
Check: all pass. Print the verify result JSON for the tamper case in your report.
Stop and report.
```

### M04 · Syslog header, pack registry and extractors
**Attach:** TRD.md, BACKEND_SCHEMA.md

```
Build module M04: parsers/syslog_header.py, registry.py, engine.py, extractors.py, jsonspan.py, shape.py.
Implement TRD.md §6 (header patterns with spans, detection order, shape signature) and §7.1–§7.2
(pack schema, extractors regex/json/kv/delimited, every extracted value carries its byte span in the raw).
The JSON extractor must record the span of each scalar's content (inside quotes for strings) and flatten keys
with dots. kv must strip quotes from spans. delimited must support kv_value narrowing. Packs load with
yaml.safe_load from backend/app/parsers/packs and /data/packs; PackRegistry.reload() hot-reloads.
Add the three built-in packs exactly as TRD.md §7.1 and BACKEND_SCHEMA.md §4.2–§4.3.
Tests: test_syslog_header.py (spans for the three sample lines: ASA ts (5,25), host (26,30); Suricata host (21,26);
FortiGate pri (1,4)); an extractor test asserting the spans listed in the lineage arrays of BACKEND_SCHEMA.md §4.1–§4.3
(e.g. ASA dst ip (145,154), Suricata signature (328,366), FortiGate user (245,250)).
Run: cd backend && pytest -q tests/test_syslog_header.py tests/test_extractors.py
Check: all pass.
Stop and report.
```

### M05 · OCSF mapping, validation, lineage, round-trip and confidence
**Attach:** TRD.md, BACKEND_SCHEMA.md

```
Build module M05: ocsf/mapper.py, enums.py, validate.py, lineage.py, roundtrip.py, confidence.py, and
storage/flatten.py.
Implement TRD.md §7.3–§7.5 and BACKEND_SCHEMA.md §4: enum ids read from the vendored schema in
backend/app/ocsf/schema (fail fast with a clear message if missing), time/time_dt/original_time handling with
source_timezone, raw_data, metadata, observables, unmapped (nothing dropped), the envelope {event_id, raw, ocsf, anvaya},
the confidence formula with round-half-up, and the three-part round-trip check.
flatten() produces the events table columns of BACKEND_SCHEMA.md §2.2 (for class 2004 use evidences[0]).
Golden files: create backend/tests/samples/{cisco-asa,suricata-eve,fortigate-kv,acme-edge}/ with the raw line (.log)
and the complete envelope (.json) from BACKEND_SCHEMA.md §4.1–§4.4.
Tests: test_parsers.py (each .log → envelope equal to its .json, ignoring raw.received_at and raw.batch_id),
test_ocsf_validation.py, test_ocsf_enums.py, test_roundtrip.py (golden ok; a broken pack with a wrong span fails),
test_confidence.py (91, 90, 89, 95).
Run: cd backend && pytest -q tests/test_parsers.py tests/test_ocsf_validation.py tests/test_ocsf_enums.py tests/test_roundtrip.py tests/test_confidence.py
Check: all pass. If the vendored OCSF schema disagrees with an example field or enum, follow the schema,
update the golden JSON, and list every change in your report.
Stop and report.
```

### M06 · Pipeline worker, storage repository, quarantine and replay
**Attach:** APP_FLOW.md, BACKEND_SCHEMA.md, TRD.md

```
Build module M06: pipeline/worker.py, pipeline/replay.py, the remaining SqliteRepository methods, and the stats task.
Follow APP_FLOW.md FL-01 steps 7–16 and FL-08 step 6: fingerprint → parse → map → validate → round-trip → score →
insert events (batched ≤ 32 or 100 ms) → sources upsert → publish to the event bus (stream/bus.py: in-process
pub/sub with monotonic ids and a 2,000-message replay buffer). Quarantine reasons and shape groups as TRD.md §6.3.
Replay re-queues raw_ids without touching vault or ledger, emits replay.progress/completed, deletes quarantine rows
on success. Stats per TRD.md API 2 (ingest_rate_eps = raw records received in the last 10 s / 10, on real time).
Tests: an integration test that runs the generator's 207 lines through the pipeline in-process and asserts
207 events, 0 quarantined, and the stats counters; inject the 20 br01 lines → 20 quarantined in one group "EDGE|x12".
Run: cd backend && pytest -q tests/test_pipeline.py
Check: all pass.
Stop and report.
```

### M07 · Correlation and silence detection
**Attach:** TRD.md, APP_FLOW.md

```
Build module M07: intel/correlate.py and intel/silence.py.
Implement TRD.md §10 exactly (stage tests, ≥2 stages opens, risk formula, levels, upsert key, idempotency,
incident.opened/updated events, title, headlines for API 21: "Port sweep of 12 ports on 10.2.4.18; 3 IDS scan alerts",
"40 failed VPN logins from 198.51.100.9 (admin, root, vpnuser, administrator)",
"Firewall allowed 198.51.100.9 → 10.2.4.18:443") and TRD.md §11 (monitored rule, median gap, threshold,
1 s tick, status change events). Silence uses real wall time; inject a clock for tests.
Tests: test_correlation.py (scenario → exactly 1 incident inc-0001, risk 95, High, 56 events, opened at the 10th
failure with risk 70; noise-only → 0; replay twice → still 1); test_silence.py (fw01 silent after 15 s, recovers;
br01 never monitored).
Run: cd backend && pytest -q tests/test_correlation.py tests/test_silence.py
Check: all pass.
Stop and report.
```

### M08 · Natural-language search
**Attach:** TRD.md (§12), BACKEND_SCHEMA.md (§2.2)

```
Build module M08: search/llm.py (OllamaClient: complete, ready, warmup), search/prompt.py (the exact system prompt of
TRD.md §12.2), search/templates.py (T1–T5), search/sql_guard.py (§12.3 with sqlglot), search/nl2sql.py
(SearchService.answer per §12.1), and the read-only execution in the repository (mode=ro, query_only, 2 s progress
abort, :now_ms bound from the Clock).
Tests: test_sql_guard.py (rejects DROP/UPDATE/ATTACH/PRAGMA/multi-statement/other tables/unknown columns; adds
LIMIT 200); test_search_templates.py (with the scenario loaded and Ollama absent: Q1 17 rows in the exact order of
TRD.md §12.5, Q2 (40, 2), Q3 12 rows, Q4 0 rows then 20 after the br01 pack is active).
Run: cd backend && pytest -q tests/test_sql_guard.py tests/test_search_templates.py
Check: all pass; also run one live Q1 against Ollama if available and paste the SQL it produced.
Stop and report.
```

### M09 · AI parser builder (draft, gates, approve, prebake)
**Attach:** TRD.md, APP_FLOW.md (FL-08), BACKEND_SCHEMA.md (§4.4, §4.5)

```
Build module M09: aiparser/sources.py (CachedDraftSource, LlmDraftSource implementing DraftSource),
aiparser/drafts.py (create, test, approve, reject per TRD.md API 15–19), aiparser/prompt.py (the drafting prompt),
aiparser/prebake.py (make prebake behaviour of BACKEND_SCHEMA.md §4.5), and the hand-authored fallback YAML file
equal to the acme-edge pack in §4.4.
Held-out indexes {2,5,8,11,14,17}. promotable only if held-out, all, round-trip and OCSF are all 100%.
Approve writes /data/packs/acme-edge.yaml, reloads the registry, writes audit_log, starts replay of the group.
Tests: test_parser_drafts.py (with the hand-authored cache: 6/6, 20/20, 20/20, 20/20, promotable; approve → 20
replayed, quarantine 0; approve before test → 409; a deliberately broken YAML → not promotable with a failure detail).
Run: cd backend && pytest -q tests/test_parser_drafts.py; then (with Ollama up) python -m app.aiparser.prebake
Check: tests pass; prebake prints origin llm_cached or hand_authored and the gate counts.
Stop and report.
```

### M10 · API routes, SSE, scenario clock and demo controller
**Attach:** TRD.md (§9, §16), APP_FLOW.md (FL-03, FL-06, FL-09, FL-11, FL-12), UI_UX_BRIEF.md (§8)

```
Build module M10: every route in TRD.md §16 (api/routes_*.py) with Pydantic response models and the error envelope;
GET /api/stream SSE with ids, retry 1500, 15 s ping, Last-Event-ID replay and the resync event; core/clock.py
(ScenarioClock and WallClock per TRD.md §9, sleep_until); demo/controller.py (start, inject_unknown, silence,
tamper, reseal, speed, reset per APP_FLOW.md) — all /api/demo/* return 403 demo_disabled unless DEMO_MODE, except
GET /api/demo/state; the background-task supervisor in main.py; LLM warm-up on startup and reset.
SSE event names and payloads exactly as UI_UX_BRIEF.md §8.
Tests: test_api_contract.py hitting each endpoint with httpx.AsyncClient in DEMO_MODE and asserting the shapes and
documented errors; a test that reset returns to raw_total 0 in under 15 s.
Run: cd backend && pytest -q tests/test_api_contract.py; then uvicorn …; curl -N localhost:8000/api/stream while
calling POST /api/demo/scenario/start in another terminal.
Check: SSE shows event.normalized messages, then incident.opened, then incident.updated with risk 95.
Stop and report.
```

### M11 · Delivery, network probe and offline self-check
**Attach:** TRD.md (§13, §15), APP_FLOW.md (FL-10)

```
Build module M11: delivery/parquet.py (DuckDB COPY from a registered pyarrow table, read-back count; no DuckDB
extensions that download), delivery/siem.py (FileSink + SiemForwarder.on_batch, RFC 5424 line with JSON payload),
delivery/retention.py (report with the exact statement), demo/netprobe.py (DEMO_MODE only, TCP connect to
1.1.1.1:443 and 9.9.9.9:443, 300 ms, every 10 s), and app/selfcheck.py implementing TRD.md §15 item 4 end to end.
Tests: test_parquet.py (export count == read-back count); test_retention.py (statement exact; never contains
"certified" or "compliant").
Run: cd backend && pytest -q tests/test_parquet.py tests/test_retention.py; docker compose build api && make offline-test
Check: offline-test prints PASS and exits 0.
Stop and report.
```

### M12 · Frontend foundations
**Attach:** UI_UX_BRIEF.md (§2, §3, §6–§9), BACKEND_SCHEMA.md (§6.2), TRD.md (§16)

```
Build module M12: Vite + React + TypeScript + Tailwind + shadcn/ui setup; @fontsource IBM Plex Sans/Serif/Mono;
src/styles/tokens.css exactly as UI_UX_BRIEF.md §2.9 and the Tailwind mapping; scripts/no-raw-hex.mjs wired into
npm run lint; src/lib/types.ts exactly as BACKEND_SCHEMA.md §6.2; src/lib/api.ts (typed client for every endpoint in
TRD.md §16, base path /api); src/lib/stream.ts (useStreamConnection with the dispatch table of UI_UX_BRIEF.md §8);
all Zustand slices of §7 with reset() and resetAllStores(); routes.tsx with lazy routes and skeleton fallbacks;
AppShell, TopBar, NavRail, HealthChip, ScenarioClock, SyntheticDataPill, StreamDot; PresenterPanel and
usePresenterKeys exactly as §9; common EmptyState, ErrorState, RouteErrorBoundary; the badges of §4.3.
Vite dev proxy /api → http://localhost:8000.
Tests: Vitest for feedStore coalescing and duplicate protection; lint passes.
Run: cd frontend && npm run lint && npx vitest run && npm run dev
Check: the shell renders at http://localhost:5173 with live health chips from the backend; the presenter panel
toggles with the backquote key; no console errors.
Stop and report.
```

### M13 · Overview and Live Feed screens
**Attach:** UI_UX_BRIEF.md (§4, §5.1, §5.2), DEMO_SCRIPT.md

```
Build module M13: features/overview and features/feed exactly as UI_UX_BRIEF.md §5.1 and §5.2 — wireframes,
props, local state, four states, motion and exact copy. Feed rows are virtualised, coalesced every 250 ms
(≤ 12 rows per flush), newest on top, with device left rules; formatEventSummary per §4.4 with Vitest tests.
Start attack simulation navigates to /feed after 600 ms.
Run: npm run dev with the backend in DEMO_MODE; click Start attack simulation.
Check: counters tween; the feed shows three devices and three format chips; the 40-failure burst cascades smoothly;
the empty, loading and error states render (stop the api container to see the error state).
Stop and report.
```

### M14 · Event Detail and the lineage view
**Attach:** UI_UX_BRIEF.md (§5.3), BACKEND_SCHEMA.md (§4)

```
Build module M14: features/event exactly as UI_UX_BRIEF.md §5.3. This is the signature element; build it with care.
RawBytesPane renders the raw text in mono 16 px wrapped at 64 characters with a byte ruler; highlights use the
lineage colours; hovering raw text resolves the narrowest span (lib/lineage.ts spanToPath, with Vitest tests);
LineageConnector draws a cubic Bézier between the field row and the first span (SVG overlay, recomputed on
resize/scroll, stroke-dashoffset draw over --m-connector); keyboard Tab/Enter/Esc behaviour; tamper mode
(red byte, inline tag, the note under the pane); EvidenceFooter with HashBlock, RoundTripChip,
VerifyRecordButton (409 → "Waiting for seal…" retries), IntegrityBadge.
Run: open /events/e-64f70aaee53f after starting the scenario.
Check: hovering evidences[0].src_endpoint.ip highlights exactly bytes 166–178; finding_info.title highlights 328–366;
Verify this record shows "Verified · sealed in batch 3 · co-signed by 2 witness nodes"; works at 1440×900.
Stop and report.
```

### M15 · Incidents and the incident toast
**Attach:** UI_UX_BRIEF.md (§5.4), TRD.md (API 20–21)

```
Build module M15: features/incidents exactly as UI_UX_BRIEF.md §5.4, including IncidentToast mounted in AppShell
(shown once per incident when risk ≥ 90, persistent until dismissed), the timeline with StageGroup and EvidenceChip
(+N expands), RiskBreakdown copy.
Check: during the scenario the toast appears once at scenario 10:07:12; Open incident shows Risk 95 · High and
the three stage headlines; clicking an evidence chip opens its Event Detail.
Stop and report.
```

### M16 · Search
**Attach:** UI_UX_BRIEF.md (§5.5), TRD.md (§12.5, API 22–23)

```
Build module M16: features/search exactly as UI_UX_BRIEF.md §5.5: SearchBox, SuggestedQuestions (from API 23),
EngineLabel, SqlBlock with the small token highlighter in lib/sqlHighlight.ts (keywords --seal, strings --dev-vpn01,
numbers --dev-fw01), ResultsTable (time column formatted HH:MM:SS IST), states including the Q4-before-approval empty
state with "Open quarantine".
Check: Q1 shows 17 rows with first row 10:08:30 fw01; Q2 shows 40 and 2; with `docker compose stop ollama` the
label switches to "Generated by query templates · read-only" and results are identical.
Stop and report.
```

### M17 · Integrity
**Attach:** UI_UX_BRIEF.md (§5.6), TRD.md (API 7–10, 33–34)

```
Build module M17: features/integrity exactly as UI_UX_BRIEF.md §5.6: VerifyLedgerButton, DemoIntegrityControls,
VerifySummary with the three verdict banners and the not-run state, LedgerTable with auto-expanded TamperDiff and
Open event, WitnessPanel. Toast copy as specified.
Check: Verify → green; key 4 then Verify → red, "byte 153 changed '8' → '9' (dst_endpoint.ip)", Open event shows the
red byte; key 5 then Verify → amber "witness nodes disagree on batch 11 onward".
Stop and report.
```

### M18 · Sources & parsers
**Attach:** UI_UX_BRIEF.md (§5.7), TRD.md (API 11–19), APP_FLOW.md (FL-08, FL-09)

```
Build module M18: features/sources exactly as UI_UX_BRIEF.md §5.7: URL-synced tabs, SourcesTable with SilenceBadge
and RateSpark, QuarantineGroupCard, ParserDraftDialog (DraftOriginLabel, YamlView, CheckList resolving 150 ms apart,
CompactLineage, ReplayProgress from SSE, Approve disabled until promotable with the tooltip), ParsersTable,
the silence toast.
Check: key 3 shows no toast; key 2 shows the 20-record group; Draft → Run tests → 6/6, 20/20, 20/20, 20/20 →
Approve & replay → 20/20 and quarantine 0; the fw01 Silent badge appears 15–16 s after key 3.
Stop and report.
```

### M19 · Delivery screen
**Attach:** UI_UX_BRIEF.md (§5.8), TRD.md (API 24–26)

```
Build module M19: features/delivery exactly as UI_UX_BRIEF.md §5.8 (three cards, the retention report sheet,
Download JSON, all states).
Check: Export to Parquet shows equal exported and read-back counts; the SIEM card shows the last payload; the
retention statement is exact.
Stop and report.
```

### M20 · Compose, demo.sh, Makefile, bench and end-to-end test
**Attach:** IMPLEMENTATION_PLAN.md (§3, §4, §8.3, §9)

```
Build module M20: docker-compose.yml, backend/Dockerfile, frontend/Dockerfile, frontend/nginx.conf, demo.sh and
Makefile exactly as IMPLEMENTATION_PLAN.md §4 (fix anything that doesn't run and tell me), bench/load_test.py per
§4.6 (measured numbers only), and frontend/e2e/golden-path.spec.ts per §8.3 (fails on any console error).
Run: docker compose build && ./demo.sh --speed 40 && make e2e && make offline-test && make bench
Check: demo.sh prints "ANVAYA ready · open http://localhost:8080/overview"; e2e passes; offline-test prints PASS;
docs/BENCHMARKS.md gains a dated row with the machine details.
Stop and report.
```

---

## 4. Polish pass (run after M20)

**Attach:** UI_UX_BRIEF.md, DEMO_SCRIPT.md

```
Polish pass. Go screen by screen through UI_UX_BRIEF.md §5 and fix every deviation. Specifically:
1. Copy: compare every visible string with the "Exact copy" sections; fix mismatches; make toasts reuse the button verb.
2. Tokens: npm run lint must pass no-raw-hex; replace any Tailwind arbitrary colour or shadow with token classes.
3. Typography: body ≥ 15 px, key demo text ≥ 18 px, mono ≥ 14 px (16 px in lineage); tabular numerals for numbers.
4. States: force empty, loading (throttle the network in devtools), error (stop api) and success for every screen;
   each must match its spec; skeletons, not spinners.
5. Motion: one primary motion at a time (feed animation suspended during toasts and banners); no layout shift;
   prefers-reduced-motion respected.
6. Accessibility: visible --seal focus rings, keyboard path through every golden-path action, AA contrast,
   ARIA roles of UI_UX_BRIEF.md §11.
7. Remove every console.log; zero console errors or warnings during the golden path.
Report a table: screen | issue found | fix applied.
```

## 5. Recording-readiness pass (run before the first take)

**Attach:** DEMO_SCRIPT.md, IMPLEMENTATION_PLAN.md (§9)

```
Recording-readiness pass. Using DEMO_SCRIPT.md §2 (3:50 at 10×) and §3 (2:00 at 20×):
1. Walk each shot list against the running app and confirm every on-screen action, label and number exists
   exactly as written, at the stated time offsets (APP_FLOW.md FL-03 timing table). Note any shot that is early or late.
2. Run the IMPLEMENTATION_PLAN.md §9 golden-path checklist three times from ./demo.sh --speed 10 and three times
   from ./demo.sh; record pass/fail per item.
3. Verify the honesty rules: "Synthetic lab data" visible; witness labels say "simulated, local"; the draft origin
   label is shown; the SIEM card says "file sink (demo)"; the retention statement says "supports" and never
   "certified"/"compliant"; no benchmark number appears unless docs/BENCHMARKS.md has a measured row.
4. Check the layout at 1920×1080 and 1440×900 at 100% zoom; no scrollbars on the golden path except in the feed,
   ledger table and YAML view.
5. With Wi-Fi off, confirm the network chip reads "No external route"; with `docker compose stop ollama`, confirm
   the Search fallback label and identical results.
Report: a table of shot | status | fix needed, the checklist results, and a final READY or NOT READY with reasons.
```
