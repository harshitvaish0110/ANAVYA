# PRD.md: ANVAYA Product Requirements (Demo-Video MVP)

## 1. Document metadata

| Attribute | Detail |
|---|---|
| Document | ANVAYA: Product Requirements Document |
| Version | 1.0.0 |
| Status | Approved for MVP build |
| Problem statement | SIH26156, Universal Log Pre-processing Framework |
| Sponsor organisation | National Technical Research Organisation (NTRO) |
| Theme / category | Blockchain & Cybersecurity · Software |
| Team | 6 members, IIIT Lucknow (roles R1–R6, see IMPLEMENTATION_PLAN.md §5) |
| Build window | 2 weeks (14 days) |
| Demo video | 3:50 main cut (3–4 minute target), plus a 2:00 compliance cut for the brief's "max 2 minutes" deliverable, and a 1:00 short cut |
| Coding tool | Claude Pro chats, one module per session (BUILD_PROMPT.md) |
| Companion files | DEMO_SCRIPT.md, TRD.md, APP_FLOW.md, UI_UX_BRIEF.md, BACKEND_SCHEMA.md, IMPLEMENTATION_PLAN.md, BUILD_PROMPT.md |

### Revision history

| Version | Date | Change |
|---|---|---|
| 0.9.0 | Day 0 | Draft from the idea deck and project plan |
| 1.0.0 | Day 0 | Scoped to the demo-video MVP; tiers assigned; scenarios SCN-01 to SCN-05 fixed |

### Glossary

| Term | Meaning |
|---|---|
| Perimeter device | A firewall, IDS/IPS, VPN gateway, WAF or router at the network edge |
| Raw record | The exact bytes received for one log event, stored untouched in the vault, identified by `raw_id` |
| Normalized event | The OCSF record derived from a raw record, identified by `event_id` |
| OCSF | Open Cybersecurity Schema Framework, the common event taxonomy used for output |
| Parser pack | A YAML file that tells the engine how to detect, extract and map one format |
| Byte lineage | For every extracted value, the byte span `[start, end)` in the raw record it came from |
| Round-trip check | Proof that every extracted value equals its source bytes and that the line is fully accounted for |
| Batch | A group of raw records sealed together under one Merkle root |
| Head | `SHA256("ANVAYA-HEAD-v1" ‖ batch_id ‖ merkle_root ‖ prev_head)`, signed with Ed25519 |
| Witness node | An independent store that co-signs each head, so a rewritten ledger can be detected |
| Quarantine | The holding area for raw records that could not be fingerprinted or parsed |
| Scenario clock | The demo-mode clock that starts at 10:00:00 IST today and runs at the playback speed |
| Tier | REAL, SIM or CUT (see §7) |

---

## 2. Problem statement

NTRO's perimeter is made up of devices from many vendors, and each one writes logs in its own format: Cisco ASA syslog, Suricata EVE JSON, FortiGate key=value, CEF, LEEF and proprietary layouts. Before those logs are usable in a SIEM, data lake or ML pipeline, an engineer must hand-write and maintain a parser per source. Three things go wrong:

1. **Effort and delay.** Every new device means a new parser before its events can be analysed.
2. **Loss and doubt.** Once a log is transformed, nothing proves the normalized record is faithful to the original. That matters for forensics and compliance; CERT-In directions require logs to be retained for 180 days.
3. **Blind spots.** Because each device speaks its own dialect, a multi-stage attack that touches the IDS, the VPN and the firewall is never seen as one story.

The brief also demands air-gapped deployment, container packaging and a credible path to Big Data scale.

## 3. Solution

**Vision.** ANVAYA turns any perimeter log into one standardized, lossless, verifiable record, fully offline.

**How.** Three layers:

| Layer | What it does |
|---|---|
| L1 · Secure Ingestion & Forensic Vault | Receives syslog (UDP/TCP), HTTP and file input; vets input; stores every original byte; hashes it with SHA-256; seals batches into Merkle roots signed with Ed25519; witness nodes co-sign each head |
| L2 · Detection, Parsing & Normalization | Fingerprints format and vendor; parses with YAML parser packs; maps to OCSF with UTC time, `raw_id` link, byte lineage, round-trip proof and a parse-confidence score; quarantines anything unparsed; onboards unseen formats via an AI-drafted, auto-tested, human-approved parser |
| L3 · Intelligence, Delivery & Governance | Correlates attacks across devices; raises log-silence alerts; answers natural-language questions with a local LLM writing guarded SQL; exports Parquet; forwards to SIEM; reports retention |

**Differentiators** (the video leads with these):
1. Cross-device attack correlation
2. Offline natural-language search
3. Witness-signed Merkle ledger
4. Self-verifying AI parser builder
5. Log-silence alerts
6. The signature visual: the byte-lineage view

**Table stakes** (must be solid, not the pitch): raw preservation with SHA-256, OCSF output, quarantine, YAML onboarding, local-LLM parser drafts, Docker running offline.

## 4. Users

| Persona | Context | Needs | In the demo |
|---|---|---|---|
| **Tier-2 SOC analyst** | Investigates alerts across many devices; time-poor; not a parser engineer | One schema for every device; fast questions without learning each vendor's fields; confidence that a record is faithful | Live Feed, Event Detail, Incidents, Search |
| **NTRO platform & integrity operator** | Onboards devices, owns evidence integrity and retention | Onboarding without code; proof nothing was altered; alerts when a source goes quiet; exports for SIEM, lake and auditors | Integrity, Sources & Parsers, Delivery |

**Anti-persona:** a commercial MSSP wanting multi-tenant SaaS billing. Out of scope.

## 5. MVP goals and success criteria

| # | Goal | Success criterion (measurable) |
|---|---|---|
| G1 | Every frame looks finished | 0 console errors and 0 unhandled promise rejections during 10 consecutive golden-path runs; no placeholder text; every screen has designed empty, loading, error and success states |
| G2 | The golden path is deterministic | 10 consecutive runs of DEMO_SCRIPT.md §2 after `./demo.sh` each produce exactly 1 incident (`inc-0001`, risk 95, High), exactly 1 failed record after tamper (`r-899f4edf0030`, byte 153), 17 rows for SCN-03 Q1, and 20 replayed records for SCN-04 |
| G3 | Works offline and without the model | The golden path passes with Wi-Fi off; it also passes with Ollama stopped (Search shows the template engine label); `make offline-test` passes inside `--network none` |
| G4 | One-command reset | `./demo.sh` returns to the first frame in under 15 s |
| G5 | A judge understands in 15 s | The Overview headline plus the first narration line state the problem; all five differentiators appear within 3:20 of the 3:50 main cut (and within 1:49 of the 2:00 compliance cut) |
| G6 | The core is genuinely real | Hashing, parsing, lineage, verify and tamper detection are real code paths with tests (§6 epics tagged REAL) |
| G7 | Credible scaling story | The code uses the storage, queue and clock interfaces from TRD.md §4, matching the production design; the deviations are documented in TRD.md §14 |

## 6. Expected outcomes a–k: coverage table

This table is copied into the README.

| # | Outcome (from the brief) | ANVAYA feature | Tier | Proof (command or test) |
|---|---|---|---|---|
| a | Preserve complete raw data without information loss | Append-only raw vault; SHA-256 per record; raw stored before parsing | REAL | `pytest backend/tests/test_vault.py::test_bytes_roundtrip` · `curl -s localhost:8000/api/raw/r-899f4edf0030 \| sha256sum` matches `raw_sha256` |
| b | Extract and parse source-specific attributes | Parser packs `cisco-asa`, `suricata-eve`, `fortigate-kv`; vendor fields kept in `unmapped` | REAL | `pytest backend/tests/test_parsers.py` (golden files per sample line) |
| c | Normalize fields into a common event taxonomy | OCSF mapping to classes 4001, 2004, 3002; validated against the vendored schema | REAL | `pytest backend/tests/test_ocsf_validation.py` |
| d | Maintain traceability between normalized and original events | `raw_id` + `raw_sha256` on every event; byte lineage per field; inclusion proof per record | REAL | `pytest backend/tests/test_roundtrip.py` · `curl -s -X POST localhost:8000/api/ledger/verify/r-899f4edf0030` |
| e | Plug-and-play onboarding of new log sources | YAML packs hot-loaded; AI parser builder flow (draft → tests → approve → replay) | REAL (packs) · SIM (draft step) | SCN-04 in the video · `pytest backend/tests/test_parser_drafts.py` |
| f | Unified visibility across enterprise environments | One console and one schema across vendors; cross-device incident timeline | REAL | SCN-01 · `pytest backend/tests/test_correlation.py::test_scenario_yields_one_high_incident` |
| g | Efficient SIEM and data lake integration | Parquet export via DuckDB; SIEM forwarder writing RFC 5424/JSON payloads | REAL (Parquet) · SIM (SIEM target) | `curl -s -X POST localhost:8000/api/export/parquet` · `cat data/outbox/siem-forward.ndjson \| head -1` |
| h | AI/ML-ready security and operational analytics | Typed, flattened OCSF columns in Parquet; NL search over normalized data | REAL | `python -c "import duckdb;print(duckdb.sql(\"select count(*) from 'data/lake/*.parquet'\"))"` |
| i | Reduced parser development effort | Declarative packs; AI-drafted parser gated by held-out tests and round-trip check | REAL (gates) · SIM (draft) | SCN-04: held-out 6/6, lossless 20/20 |
| j | Deployable in an air-gapped network | No runtime network calls; bundled fonts, model and schema; live network-route check | REAL | `make offline-test` (runs in `docker run --network none`) |
| k | Packaged in a container, platform independent | Docker Compose for `api`, `ui` and optional `ollama` | REAL | `docker compose up -d && curl -s localhost:8000/api/health` |

## 7. Scope tiers

**REAL** features are the actual production code path. **SIM** features are scripted or partly hard-coded, behind the same interface a real version would use, and labelled honestly in the UI. **CUT** features appear on the roadmap slide only.

| Feature | Tier | Justification |
|---|---|---|
| Deterministic generator (3 formats + scripted attack + steady-state noise) | REAL | All demo data must flow through the real pipeline |
| HTTP + UDP syslog ingest; input vetting | REAL | NTRO judges will probe ingestion |
| Raw vault + SHA-256 | REAL | Outcome a |
| Merkle batches + Ed25519-signed, chained heads; `/verify`; tamper detection | REAL | Core trust story |
| Fingerprinting; 3 parsers to OCSF; `raw_id`, `raw_sha256`, confidence | REAL | Outcomes b, c |
| Byte lineage + round-trip check | REAL | The signature visual must be real data, not decoration |
| Quarantine + replay | REAL | Outcome a/e |
| Correlation rule | REAL | Differentiator 1 |
| Log-silence detection | REAL | Differentiator 5 |
| NL search via Ollama with template fallback | REAL | Differentiator 2 |
| Parquet export via DuckDB | REAL | Outcome g/h |
| Measured live ingest rate; `make bench` | REAL | Honesty rule: only measured numbers |
| Offline run (`--network none` self-check, Wi-Fi off) | REAL | Outcome j |
| AI parser builder | SIM | The LLM draft is cached from an earlier real run (or hand-authored, labelled). Held-out tests, approval and replay run live. Live drafting on a laptop CPU is too slow and variable for a recording. |
| Witness nodes | SIM | Two local witness stores with separate keys; real co-signing and comparison. Independent operators can't be shown on one laptop. |
| SIEM delivery | SIM | Real payload written to a file sink; no SIEM container in the MVP |
| CERT-In 180-day retention report | SIM | Template filled from real counts; worded "supports", never "certifies" |
| Kafka-compatible stream + worker scale-out + scaling chart | CUT | Roadmap |
| PostgreSQL, MinIO | CUT | Interfaces exist; SQLite and filesystem in the MVP |
| Wazuh / OpenSearch integration | CUT | Roadmap |
| RBAC, signed parser plugins | CUT | Roadmap |
| Offline install bundle | CUT | Roadmap (`docker save` script) |
| Formats beyond Cisco ASA, Suricata EVE, FortiGate kv and the br01 demo format | CUT | Roadmap |

## 8. Epics

Priority uses MoSCoW: M = must, S = should, C = could.

### EP-01 · Deterministic scenario generator · REAL · M
- **US-01.1** As the presenter, I want the attack simulation to produce exactly the same logs every take, so the video is repeatable.
- **US-01.2** As a judge, I want the logs to look like real vendor output, so the parsing is credible.
- **Acceptance criteria**
  1. `python -m generator.scenario --dry-run` prints exactly 207 scripted lines (fw01 104, ids01 33, vpn01 70) in the order given in BACKEND_SCHEMA.md §5.
  2. Lines are sent over UDP to `127.0.0.1:5514` in strict order, paced by the scenario clock at the current speed.
  3. After scenario 10:12:00 the generator switches to steady-state noise at 1× (fw01 every 1 s; ids01 and vpn01 every 4 s), which never involves 198.51.100.9, 10.2.4.18, denies or authentication failures.
  4. `--stop-device fw01` (and `POST /api/demo/silence`) stops and resumes fw01 emissions.
  5. No `random` module is used in demo mode; every value is derived from the line index.

### EP-02 · Multi-transport ingest and input vetting · REAL · M
- **US-02.1** As an operator, I want devices to send standard syslog, so no agent is needed on the device.
- **US-02.2** As an operator, I want malformed or hostile input flagged, not silently altered.
- **Acceptance criteria**
  1. UDP `5514` accepts one event per datagram; `POST /api/ingest` accepts `text/plain` (one per line) or JSON `{"lines": [...]}` and returns `202` with `raw_ids`.
  2. Events larger than 65,536 bytes are stored in full, flagged `oversized`, and parsed from the first 65,536 bytes only.
  3. Embedded CR/LF in a datagram, NUL bytes and other control characters are flagged (`embedded_newline`, `nul_byte`, `control_chars`) and never stripped from the vault.
  4. Invalid UTF-8 is stored as received, with `encoding = "binary"` and the flag `invalid_utf8`.

### EP-03 · Raw forensic vault · REAL · M
- **US-03.1** As an auditor, I want every original byte kept unchanged, so evidence is defensible.
- **Acceptance criteria**
  1. Records are appended to `data/vault/seg-000001.log` with `u32` big-endian length framing; segments rotate at 64 MiB.
  2. `raw_sha256` is computed over the exact received bytes before any decoding.
  3. The application never writes to an existing record, except `POST /api/demo/tamper` in `DEMO_MODE`.
  4. `GET /api/raw/{raw_id}` returns bytes whose SHA-256 equals `raw_sha256`, until tampered.

### EP-04 · Merkle ledger and verification · REAL · M
- **US-04.1** As an auditor, I want to verify that no record was changed, deleted or reordered.
- **US-04.2** As an analyst, I want a one-click proof for a single record.
- **Acceptance criteria**
  1. A batch seals at 16 records, or when the oldest unsealed record is too old: 300 s of **scenario** time during the scripted phase, 5 s of real time otherwise. The longest 16-record span in the scripted data is 128 scenario seconds, so every scripted seal is count-triggered at any speed, which makes batch numbers deterministic (`r-64f70aaee53f` lands in batch 3, `r-899f4edf0030` in batch 11).
  2. Leaf, node and head construction follow TRD.md §8; heads chain from a 32-zero-byte genesis and are signed with Ed25519.
  3. `POST /api/ledger/verify` re-reads every record from disk and returns `verdict` `verified`, `tampered` or `witness_mismatch`, with per-record and per-batch failures.
  4. After `POST /api/demo/tamper`, verify reports exactly one failed record, `r-899f4edf0030`, `first_diff_offset = 153`.
  5. `POST /api/ledger/verify/{raw_id}` returns the inclusion proof and its validity.

### EP-05 · Witness co-signing · SIM · S
- **US-05.1** As NTRO, I want independent parties to co-sign the ledger, so even a key holder can't rewrite history unnoticed.
- **Acceptance criteria**
  1. Two witness stores, `witness-ntro` and `witness-certin`, each with its own Ed25519 key and SQLite file, co-sign every sealed head after verifying the primary signature and `prev_head` continuity.
  2. After `POST /api/demo/tamper` then `POST /api/demo/reseal`, primary-only checks pass but verify returns `verdict = "witness_mismatch"`, listing batch 11 for both witnesses.
  3. The UI labels both witnesses "simulated, local".

### EP-06 · Fingerprinting and parser packs · REAL · M
- **US-06.1** As an operator, I want new sources recognised automatically.
- **Acceptance criteria**
  1. Each pack declares a `detect` block; the engine evaluates packs in priority order (TRD.md §6) and assigns `format`, `vendor`, `product` and `device`.
  2. `cisco-asa@1.0.0`, `suricata-eve@1.0.0` and `fortigate-kv@1.0.0` ship built in; packs hot-load from `backend/app/parsers/packs/`.
  3. Every sample line in `backend/tests/samples/` parses to its golden OCSF JSON exactly.

### EP-07 · OCSF normalization, byte lineage and round-trip proof · REAL · M
- **US-07.1** As an analyst, I want one schema for every device.
- **US-07.2** As an auditor, I want to see exactly which bytes each field came from.
- **Acceptance criteria**
  1. Records validate against the vendored OCSF schema for classes 4001, 2004 and 3002; invalid ones go to quarantine with `schema_invalid`.
  2. Every extracted value has a lineage entry `{path, spans, derived}` whose spans index the raw bytes.
  3. The round-trip check passes for every golden sample (values equal, coverage ≥ 0.95, no dropped values).
  4. `parse_confidence` follows the formula in TRD.md §7.4; for the golden samples it is 91 (ASA access), 90 (Suricata alert), 89 (FortiGate failure) and 95 (acme-edge).

### EP-08 · Quarantine and replay · REAL · M
- **US-08.1** As an operator, I want unparsed logs held, grouped and replayable.
- **Acceptance criteria**
  1. Quarantined records are grouped by `shape_signature` (TRD.md §6.3).
  2. `POST /api/replay` re-runs a group through the current packs without touching the vault or ledger, emitting `replay.progress` and `replay.completed` events.

### EP-09 · AI parser builder · SIM (draft step) / REAL (gates) · M
- **US-09.1** As an operator, I want a parser drafted for an unseen format, so onboarding takes minutes.
- **US-09.2** As NTRO, I want no AI output to reach production without tests and human approval.
- **Acceptance criteria**
  1. `POST /api/parsers/drafts {group_id}` returns a YAML draft. In `DEMO_MODE` it comes from `backend/app/aiparser/cache/acme-edge.json`, with `origin` set to `llm_cached` (produced by `make prebake` with Ollama) or `hand_authored` (fallback).
  2. `POST /api/parsers/drafts/{draft_id}/test` runs the draft live on the held-out lines (6 of 20) and on all 20, plus the round-trip check and OCSF validation; `promotable` is true only if every check passes 100%.
  3. `POST /api/parsers/drafts/{draft_id}/approve` returns `409 not_promotable` unless `promotable`; on success it activates `acme-edge@1.0.0`, writes the audit log, and replays the quarantine group (20 records).

### EP-10 · Cross-device correlation · REAL · M
- **US-10.1** As an analyst, I want related events from different devices joined into one incident.
- **Acceptance criteria**
  1. The rule in TRD.md §10 runs after each normalized batch and is idempotent.
  2. The scripted scenario yields exactly one incident, `inc-0001`: attacker 198.51.100.9, target 10.2.4.18, stages recon → credential → access, 3 device types, risk 95, level High.
  3. Noise and steady-state traffic yield no incident.

### EP-11 · Log-silence detection · REAL · M
- **US-11.1** As an operator, I want to know when a steady source stops sending.
- **Acceptance criteria**
  1. The rule in TRD.md §11 marks a monitored source `silent`. In `DEMO_MODE`, with `SILENCE_MIN_SECONDS=15`, fw01 turns silent 15–17 s after `POST /api/demo/silence {device:"fw01", silent:true}`.
  2. One-off bursts (br01) are never monitored.
  3. The badge clears within 2 s of events resuming.

### EP-12 · Natural-language search · REAL · M
- **US-12.1** As an analyst, I want to ask questions in plain English and see the SQL.
- **Acceptance criteria**
  1. The four scripted questions (TRD.md §12.5) return the documented rows with Ollama running and with Ollama stopped.
  2. LLM SQL passes the guard (TRD.md §12.3) or is silently replaced by the template result; the engine label shows which engine was used.
  3. The LLM timeout is 5 s in `DEMO_MODE`; queries run read-only with a 2 s limit.

### EP-13 · Delivery: Parquet, SIEM forwarder, retention report · REAL / SIM · S
- **Acceptance criteria**
  1. `POST /api/export/parquet` writes `data/lake/events-<from>-<to>.parquet` via DuckDB and returns its row count, checked by DuckDB reading it back.
  2. The SIEM forwarder appends one RFC 5424 line with a JSON payload per normalized event to `data/outbox/siem-forward.ndjson`; the status card shows the count and last payload.
  3. `GET /api/reports/retention` returns the report described in TRD.md §13, including the sentence "ANVAYA's tamper-evident retention supports CERT-In's 180-day log retention requirement."

### EP-14 · Analyst console (8 screens) · REAL · M
- **Acceptance criteria**
  1. The screens in UI_UX_BRIEF.md §5 are built to the wireframes, with all four states.
  2. Design tokens only (no ad-hoc hex values); text sizes meet the camera rules (body ≥ 15 px, key demo text ≥ 18 px, mono ≥ 14 px).
  3. Lighthouse accessibility score ≥ 95 on `/overview` and `/events/:eventId`.

### EP-15 · Real-time stream · REAL · M
- **Acceptance criteria**
  1. `GET /api/stream` (SSE) emits the event types in UI_UX_BRIEF.md §8.
  2. The client reconnects with `Last-Event-ID` and no duplicate rows appear.

### EP-16 · Demo mode, presenter controls and reset · REAL · M
- **Acceptance criteria**
  1. `ANVAYA_DEMO_MODE=1` enables `/api/demo/*` and the scenario clock; otherwise those routes return `403 demo_disabled`.
  2. The presenter keys in DEMO_SCRIPT.md §1 work; `0 0` resets.
  3. `./demo.sh` resets the vault, database, witness stores, outbox, lake and UI state, then prints `ANVAYA ready · open http://localhost:8080/overview`.

### EP-17 · Packaging and offline proof · REAL · M
- **Acceptance criteria**
  1. `docker compose up -d` starts `api` and `ui`; `--profile llm` adds `ollama`.
  2. `make offline-test` runs `python -m app.selfcheck` in `docker run --network none` and exits 0.
  3. The health chip reports `air_gapped` when no external route exists (TRD.md §15).

### EP-18 · Measured throughput · REAL · S
- **Acceptance criteria**
  1. The Overview "Live ingest rate (measured)" is the count of raw records received in the last 10 s divided by 10.
  2. `make bench` writes the sustained events per second, CPU model, core count, RAM and date to `docs/BENCHMARKS.md`; the Overview shows the latest bench line with its machine label, or nothing if no bench has run.

## 9. Out of scope / roadmap

| Item | Why it is out | What bringing it in would need |
|---|---|---|
| Kafka-compatible partitioned stream, worker fleet, scaling chart | Not visible in a single-laptop video; needs multi-node measurement | A `RedpandaQueue` implementing the `EventQueue` interface; workers as separate containers |
| PostgreSQL and MinIO | SQLite and the filesystem meet MVP needs | `PostgresRepository` and `S3RawStore` behind existing interfaces |
| Wazuh / OpenSearch delivery | Several heavy containers for little on-screen value | Replace the file sink with an HTTP/syslog sink |
| RBAC, signed parser plugins | Governance hardening | Local users with argon2; Ed25519 signatures on pack files |
| Offline install bundle | Packaging step | `docker save` of the images plus model weights |
| More formats (Palo Alto, pfSense, generic CEF/LEEF) | MVP shows three plus one onboarded live | Additional packs with golden tests |
| Live LLM drafting during the video | Laptop CPU latency is unpredictable | Already implemented as `origin = "llm_live"` in full mode |

## 10. Assumptions

| # | Assumption | If wrong |
|---|---|---|
| A1 | The build window is 2 weeks (default; not specified). | Re-time IMPLEMENTATION_PLAN.md §6 and apply its cut list |
| A2 | The coding tool is Claude Pro chats, one module per session (default; not specified). | BUILD_PROMPT.md works unchanged in Claude Code or Cursor |
| A3 | The scripted attack is **against** 10.2.4.18, an internal RFC 1918 host, **from** 198.51.100.9. The brief wording "from 10.2.4.18" is read this way because the attacker must be external for the firewall "allow" stage to make sense, and the idea deck uses this pairing. | Swap the IPs in BACKEND_SCHEMA.md §5; correlation uses "external IP" as its key |
| A4 | Times are shown in IST (Asia/Kolkata); storage is UTC. | Change `DISPLAY_TZ` |
| A5 | OCSF field names and enum values are taken from the vendored schema at build time. The examples were authored against OCSF 1.3.0. | The build fails fast in `test_ocsf_validation.py`; fix the mapping |
| A6 | The recording laptop has at least 4 cores and 8 GB RAM, with Ollama able to answer a short prompt within 5 s after warm-up. | Use the template engine and its alternate narration line |
| A7 | All device names, sample lines and IPs are synthetic. Lines are modelled on public vendor formats; Acme EdgeRouter (br01) is a fictional vendor. | — |
| A8 | Suricata signature IDs 9000001 and 9000101 are local lab rules, not published rule-set IDs. | — |

## 11. Risks to the demo

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Ollama slow or not running on recording day | Medium | Search shows the template label | 5 s timeout with silent fallback; warm-up at startup; alternate narration line |
| Timing drift makes the incident toast appear mid-sentence | Medium | Awkward video cut | Count-based sealing, a deterministic generator and fixed speed; rehearse with the timing table; edit at shot boundaries |
| SSE disconnect during a take | Low | Feed freezes | Auto-reconnect with `Last-Event-ID`; the presenter panel shows stream status |
| A browser shortcut collides with a presenter key | Low | Wrong action | Single-key presenter keys, inactive in inputs; no Ctrl/Cmd combinations |
| Screen too dense for camera | Medium | Judges can't read | Camera rules in UI_UX_BRIEF.md §10; 1440×900 check |
| Docker port collision on the laptop | Low | Stack fails to start | Ports fixed and documented (8000, 8080, 5514/udp, 11434); `demo.sh` checks them first |
| Laptop has network during recording | Low | "External route detected" chip | Wi-Fi off before `demo.sh`; checklist item |
| Judges ask whether a simulated part is live | Medium | Credibility | Real/simulated table in DEMO_SCRIPT.md §5; honest UI labels |
