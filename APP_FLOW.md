# APP_FLOW.md: ANVAYA Application Flows

| Attribute | Detail |
|---|---|
| Version | 1.0.0 |
| Depends on | TRD.md (interfaces, rules, API numbers), BACKEND_SCHEMA.md (tables, samples), UI_UX_BRIEF.md (components, SSE events) |
| Notation | `component → function()` · **In:** data in · **Out:** data out · ⏱ typical time on a 4-core laptop (target, not a benchmark claim) |

Module paths refer to the repo tree in IMPLEMENTATION_PLAN.md §2.

## Flow index

| Flow | Name | Scenario |
|---|---|---|
| FL-01 | Ingest → vault → fingerprint → parse → OCSF → store | all |
| FL-02 | Ledger sealing and witness co-signing | all |
| FL-03 | Scenario start, clock and generator | SCN-01 |
| FL-04 | Cross-device attack correlation | SCN-01 |
| FL-05 | Event Detail lineage and single-record verify | SCN-01, SCN-02 |
| FL-06 | Ledger verify, tamper and insider re-seal | SCN-02, SCN-02b |
| FL-07 | Natural-language search with fallback | SCN-03 |
| FL-08 | Unknown format → quarantine → AI draft → tests → approve → replay | SCN-04 |
| FL-09 | Log-silence detection | SCN-05 |
| FL-10 | Parquet export, SIEM forward and retention report | Delivery screen |
| FL-11 | Demo reset | all |
| FL-12 | Real-time stream connect and reconnect | all |

---

## FL-01 · Ingest → vault → fingerprint → parse → OCSF → store

- **Trigger:** a UDP datagram arrives on 5514, or `POST /api/ingest`.
- **Actor:** generator (synthetic devices), or an operator using HTTP.
- **Preconditions:** the `api` container is healthy; the vault directory is writable; the pack registry is loaded.

**Steps**

1. `ingest/udp.py → UdpSyslogProtocol.datagram_received(data, addr)`. **In:** bytes, peer `(ip, port)`. **Out:** `RawIn(bytes, transport="udp", peer_ip)`. ⏱ < 0.1 ms.
   For HTTP: `ingest/http.py → ingest_route()` splits the body on `\n` (text) or reads `lines` (JSON), and produces one `RawIn` per line, encoded as UTF-8 bytes.
2. `ingest/vetting.py → vet(raw_in)`. **Out:** `flags: list[str]` from `oversized`, `embedded_newline`, `nul_byte`, `control_chars`, `invalid_utf8`, and `encoding` (`utf-8` or `binary`). Bytes are never modified. ⏱ < 0.1 ms.
3. `vault/store.py → SegmentRawStore.append(data, meta)`:
   - assign `seq = last_seq + 1` (in-memory counter, persisted in `raw_events`)
   - compute `raw_sha256 = sha256(data)`
   - compute `raw_id = "r-" + sha256(seq_be64 ‖ raw_sha256)[:12 hex]`
   - write the `u32` length and bytes to the current segment, then `flush` and `os.fsync` every 16 records or 250 ms

   **Out:** `RawRecord(raw_id, seq, raw_sha256, segment, offset, size, received_at_ms = Clock.now_ms(), wall_received_ms)`. ⏱ 0.2 ms (+ fsync amortised).
4. `storage/sqlite_repo.py → insert_raw(rec)` inserts the row into `raw_events` with `status='pending'`. ⏱ 0.2 ms.
5. `vault/ledger.py → LedgerSealer.add(rec)` appends to the unsealed list (FL-02).
6. `core/queue.py → InProcQueue.put(raw_id)`. HTTP replies `202` here with the `raw_ids`.
7. `pipeline/worker.py → PipelineWorker.run()` takes a `raw_id`, then `process_raw(raw_id)` reads the bytes from the vault (`RawStore.read`).
8. `parsers/syslog_header.py → split(text)`. **Out:** `HeaderSplit(pattern_id, fields, body_offset)` (TRD.md §6.1).
9. `parsers/registry.py → PackRegistry.detect(header_split, text)` evaluates packs by priority (TRD.md §6.2). **Out:** `Pack | None`.
   - **Branch: no pack.** `quarantine.add(raw_id, reason="no_fingerprint", shape=shape_signature(text))`; `raw_events.status='quarantined'`; publish SSE `event.quarantined`. End.
10. `parsers/engine.py → parse_with_pack(pack, text, header_split)` selects the first rule whose `when` matches and runs its extractor (TRD.md §7.2). **Out:** `list[Extracted]`, `rule_id`.
    - **Branch: no rule or regex no-match.** Quarantine with `no_rule` / `parse_error`. End.
11. `ocsf/mapper.py → map_to_ocsf(pack, rule, extracted, clock_received_ms)`:
    - builds the OCSF dict
    - sets `raw_data`, `metadata` and `observables`
    - converts the device time with `source_timezone` and computes `clock_skew_s = |device_time − received_at| / 1000` (flag `clock_skew` if > 300 s)

    **Out:** `(ocsf: dict, lineage: list[LineageEntry])`.
12. `ocsf/validate.py → validate_ocsf(ocsf)` with jsonschema.
    - **Branch: invalid.** Quarantine `schema_invalid` with the first error path. End.
13. `ocsf/roundtrip.py → check_roundtrip(raw_bytes, extracted, lineage, accounted_spans)`. **Out:** `{ok, value_check, coverage, dropped}`. If `ok` is false, add the flag `roundtrip_fail` (the event is still stored; it also appears in quarantine under reason `roundtrip_fail`).
14. `ocsf/confidence.py → score(pack, extracted, ocsf)`. **Out:** `parse_confidence`, `breakdown`.
15. `storage/sqlite_repo.py → insert_events([NormalizedEvent])`, batched per worker loop iteration (up to 32 events or 100 ms). `raw_events.status='parsed'`; `sources` upsert (`last_seen`, `events_total`). ⏱ 1–3 ms per event in total for steps 7–15.
16. After each batch insert:
    - `stream/bus.py → EventBus.publish("event.normalized", EventSummary)` per event (coalesced by the UI)
    - `intel/correlate.py → Correlator.on_batch(events)` (FL-04)
    - `intel/silence.py → SilenceMonitor.on_events(events)` (FL-09)
    - `delivery/siem.py → SiemForwarder.on_batch(events)` (FL-10)

**Error paths**

| Failure | Handling |
|---|---|
| Vault write fails (disk full) | HTTP returns `503 dependency_unavailable`; UDP datagrams are counted in `stats.overflow` and logged; `/api/health` shows `vault: error`; UI chip turns red |
| Worker exception on one record | Caught per record → quarantine `parse_error` with the exception class name; the worker continues |
| Queue backlog > 10,000 | `stats.overflow` increments; ingest still writes to the vault (nothing is lost); the worker drains the backlog |

**Post-conditions:** every received byte is in the vault, has a `raw_events` row, and will be sealed. It is either an `events` row or a `quarantine` row.

---

## FL-02 · Ledger sealing and witness co-signing

- **Trigger:** `LedgerSealer.add()` brings the unsealed count to 16, or the 250 ms sealer tick finds the oldest unsealed record past the age limit (300 scenario seconds in the scripted phase; 5 real seconds otherwise).
- **Preconditions:** keys loaded (`vault/keys.py → load_or_create_keys()` at startup).

**Steps**

1. `ledger.py → LedgerSealer.seal()` takes the first ≤ 16 unsealed records in seq order. **In:** `[(seq, raw_sha256)]`.
2. `merkle.py → leaf_hash(sha, seq)` for each, then `mth(leaves)`. **Out:** `merkle_root` (32 bytes). ⏱ < 0.2 ms.
3. `ledger.py → make_head(batch_id, root, prev_head)`, then `Ed25519PrivateKey.sign(head)`. **Out:** `head`, `signature`.
4. In one SQLite transaction:
   - insert `ledger_batches`
   - `UPDATE raw_events SET batch_id = ? WHERE seq BETWEEN ? AND ?`
   - for each witness in `[witness-ntro, witness-certin]`: `cosig = witness.cosign(batch_id, head, prev_head, signature, primary_pub)`, then insert `witness_signatures`
5. Commit, then publish SSE `ledger.sealed {batch_id, first_seq, last_seq, leaf_count, cosigned: 2}`. ⏱ 2–5 ms.

**Error paths**

| Failure | Handling |
|---|---|
| A witness rejects (bad signature or prev-head gap) | Insert `witness_signatures` with `status='rejected'` and the reason; the batch still seals; health shows `witnesses: degraded (1/2)`; the Integrity screen shows that witness ✗ |
| Transaction fails | Roll back; the records stay unsealed and are retried on the next tick |

**Post-conditions:** batch *n* covers seq `16n−15…16n` during the scripted phase (TRD.md §8.2).

---

## FL-03 · Scenario start, clock and generator

- **Trigger:** the "Start attack simulation" button on Overview, or presenter key `1`.
- **Preconditions:** `DEMO_MODE`; `demo.state.phase == "idle"`.

**Steps**

1. UI `useDemoStore.startScenario()` calls `api.demo.start({speed})` → `POST /api/demo/scenario/start`.
2. `demo/controller.py → DemoController.start(speed)`:
   - `ScenarioClock.anchor(today 10:00:00 IST, speed)`
   - `phase = "scripted"`
   - `asyncio.create_task(generator.scenario.run(clock, sink=UdpSink("127.0.0.1", 5514), stop_flags))`
   - returns `202 {phase, speed, scripted_lines: 207}`
3. `generator/scenario.py → run()`:
   - builds the 207 scripted lines with `build_scripted_lines(date=today)` (BACKEND_SCHEMA.md §5.2), each as `(scn_time_ms, device, text)`, sorted by `(scn_time, device_order)` with order fw01 = 0, ids01 = 1, vpn01 = 2
   - for each line: `await clock.sleep_until(scn_time_ms)` (converts to real time with the current speed), then unless `stop_flags[device]`, sends one UDP datagram
4. Publish SSE `demo.state {phase:"scripted", speed}`. The UI navigates to `/feed` after 600 ms.
5. When the clock passes 10:12:00 IST, `ScenarioClock` re-anchors at speed 1, `phase="steady"`, SSE `demo.state`. The generator switches to `steady_lines()` (BACKEND_SCHEMA.md §5.3): fw01 every 1 s, ids01 and vpn01 every 4 s, all benign.

**Timings:** real seconds after Start = scenario seconds since 10:00:00 ÷ speed.

| Scenario time | 10× (3:50 cut) | 20× (2:00 cut) | 40× (1:00 cut) |
|---|---|---|---|
| 10:02:00 | +12.0 s | +6.0 s | +3.0 s |
| 10:02:05 | +12.5 s | +6.25 s | +3.1 s |
| 10:05:00 | +30.0 s | +15.0 s | +7.5 s |
| 10:05:39 | +33.9 s | +16.95 s | +8.5 s |
| 10:07:12 | +43.2 s | +21.6 s | +10.8 s |
| 10:08:30 | +51.0 s | +25.5 s | +12.8 s |
| 10:12:00 | +72.0 s | +36.0 s | +18.0 s |

**Error paths**

| Failure | Handling |
|---|---|
| Called when not idle | `409 conflict`; UI toast "Simulation already running. Press 0 twice to reset." |
| UDP send fails | The generator retries once after 50 ms, then logs and continues; the line is counted in `demo.state.send_errors` (shown in the presenter panel) |

**Post-conditions:** 207 raw records exist after +36 s at 20×; steady noise continues.

---

## FL-04 · Cross-device attack correlation (SCN-01)

- **Trigger:** `Correlator.on_batch(events)` after each stored batch (FL-01 step 16).
- **Preconditions:** events stored with `src_ip`, `dst_ip`, `class_uid`, `action`, `status` and `title` populated.

**Steps**

1. `intel/correlate.py → on_batch(events)` collects external source IPs (non-RFC 1918) in the batch. For each `ip`: `windows = repo.events_for_attacker(ip, since = min(event.time) − 10 min)`.
2. `evaluate_window(events)` applies the stage tests (TRD.md §10). **Out:** `stages: dict[stage, list[event_id]]`, `devices`, `ordered`, `risk`, `level`, `target_ip`.
3. If `len(stages) ≥ 2`: `repo.upsert_incident(...)` (incident key `(attacker_ip, window_start)`); `insert or ignore` into `incident_events (incident_id, event_id, stage)`.
4. Publish SSE `incident.opened` (first time) or `incident.updated` (stages or risk changed), with the `IncidentSummary` payload.
5. UI `useIncidentStore.upsert(summary)`. If `summary.risk ≥ 90` and it hasn't been toasted yet, `IncidentToast` shows (UI_UX_BRIEF.md §5.4).

**Scenario timeline** (scenario clock → real time at 20×):

| Scenario time | Real at 20× (after start) | Event | Correlator result |
|---|---|---|---|
| 10:02:00–10:02:35 | +6.0–7.8 s | 12 denies + 3 IDS scan alerts from 198.51.100.9 | Window opens; 1 stage (recon), no incident |
| 10:05:09 | +15.5 s | 10th VPN failure | 2 stages → `incident.opened inc-0001`, risk 70, High |
| 10:05:39 | +16.95 s | 40th failure | No change in stages; incident not re-emitted |
| 10:07:12 | +21.6 s | fw01 302013 inbound Allowed | 3 stages → `incident.updated`, risk 95; **toast** |

**Error paths**

| Failure | Handling |
|---|---|
| Correlator exception | Logged; ingest is unaffected; the next batch retries |
| Duplicate processing (replay) | `insert or ignore` and the upsert key keep it to exactly 1 incident |

**Post-conditions:** exactly one incident `inc-0001` with risk 95 and 56 linked events.

---

## FL-05 · Event Detail lineage and single-record verify

- **Trigger:** clicking a Live Feed row (`FeedRow.onSelect`), or opening `/events/:eventId`.
- **Preconditions:** the event exists.

**Steps**

1. Router navigates to `/events/e-64f70aaee53f`. `useQuery(['event', id], api.events.get)` → `GET /api/events/{id}`. ⏱ < 30 ms.
2. Render `LineageView` with `raw.text`, `ocsf` and `anvaya.lineage` (UI_UX_BRIEF.md §5.3).
3. Hovering an OCSF field calls `useLineageStore.setHover(path)`:
   - `RawBytesPane` highlights the span(s) for that path
   - `LineageConnector` draws the curve from the field row to the first span
   - hovering the raw text does the reverse lookup (`spanToPath`)
4. Clicking **Verify this record** → `POST /api/ledger/verify/{raw_id}` → `Ledger.verify_record()` (TRD.md §8.6). ⏱ < 20 ms.
5. `IntegrityBadge` shows the result:
   - ok → "Verified · sealed in batch 3 · co-signed by 2 witness nodes"
   - not ok → "Altered on disk · byte 153 changed '8' → '9' (dst_endpoint.ip)"
   - not yet sealed (`409`) → "Waiting for seal…", retried automatically after 1 s, up to 6 times

**Error paths**

| Failure | Handling |
|---|---|
| 404 | Error state "This event isn't in the store. It may have been cleared by a demo reset." with a "Back to Live Feed" button |
| Network error | Error state with "Retry" |

---

## FL-06 · Ledger verify, tamper and insider re-seal (SCN-02, SCN-02b)

- **Trigger:** the Integrity buttons **Verify ledger**, **Simulate tamper** (key `4`), **Simulate insider re-seal** (key `5`).
- **Preconditions:** `DEMO_MODE` for tamper and re-seal; the target seq 164 is sealed (batch 11).

**Steps: verify**

1. `VerifyLedgerButton.onClick` → `useMutation(api.ledger.verify)` → `POST /api/ledger/verify`.
2. `ledger.py → verify_all()` (TRD.md §8.5): for each batch, read the bytes (`iter_seq_range`), recompute hashes, root, chain, signature and witness agreement. ⏱ < 100 ms for 300 records.
3. Store `meta.last_verify`; publish SSE `verify.completed {verdict, failed_records, witness_status}`.
4. `VerifySummary` renders a green, red or amber banner; `LedgerTable` marks the failing batch rows; `TamperDiff` expands for each failed record.

**Steps: tamper (key `4`)**

1. `POST /api/demo/tamper` → `DemoController.tamper()` → `SegmentRawStore.overwrite_byte_for_demo(raw_id="r-899f4edf0030", offset=153, value=0x39)`. Opens the segment `r+b`, seeks to `record_offset + 4 + 153`, writes 1 byte, `fsync`.
2. `demo.state.tampered = true`; audit log `demo.tamper`; SSE `demo.state`.
3. UI toast: "Raw record r-899f4edf0030 altered on disk (demo)". Nothing is re-verified automatically; the presenter clicks **Verify ledger**.
4. Verify → `verdict: "tampered"`, 1 failed record, offset 153, `'8' → '9'`, field `dst_endpoint.ip` (found from lineage: the span 145–154 contains offset 153).

**Steps: insider re-seal (key `5`)**

1. `POST /api/demo/reseal` → `DemoController.reseal()` → `Ledger.reseal_from(batch_id=11)` (TRD.md §8.8).
2. SSE `demo.state {resealed:true}`; toast "Ledger re-signed with the primary key from batch 11 (demo insider)".
3. Verify → `verdict: "witness_mismatch"`; `WitnessPanel` shows both witnesses ✗ for batches 11 onward.

**Error paths**

| Failure | Handling |
|---|---|
| Tamper before seq 164 is sealed | `409`; toast "Wait for the firewall allow event to be sealed, then try again." |
| Re-seal without tamper | `409`; toast "Simulate tamper first." |
| Verify while sealing | The verify reads committed batches only; unsealed records are listed as "pending seal" (count only) |

---

## FL-07 · Natural-language search with fallback (SCN-03)

- **Trigger:** submitting `SearchBox` (Enter) or clicking a `SuggestedQuestions` chip.

**Steps**

1. `useMutation(api.search.ask)` → `POST /api/search {question}`.
2. `search/nl2sql.py → SearchService.answer(q)`:
   1. `templates.match_template(q)` → `TemplateHit | None` (TRD.md §12.4).
   2. If `llm.ready()`: `sql = await OllamaClient.complete(SYSTEM, f"Q: {q}\nSQL:", timeout_s=LLM_TIMEOUT_S)`. ⏱ 0.8–4 s after warm-up.
   3. If `sql`: `g = sql_guard.check(sql)`. If `g.ok`: `cols, rows = repo.execute_readonly(g.normalized_sql, {"now_ms": clock.now_ms()}, 2.0)`; on success return `engine="llm"`.
   4. Otherwise, if a template hit exists: run the template SQL; return `engine="template"`.
   5. Otherwise raise `422 unanswerable` with suggestions.
3. Audit log `search.ask {question, engine, sql, row_count}`.
4. The UI renders `SqlBlock` (with the engine label and clock caption) and `ResultsTable`.

**Timings:** template path < 50 ms; LLM path ≤ 5 s (demo timeout); the UI shows the loading state "Writing SQL…" after 150 ms.

**Error paths**

| Failure | Handling |
|---|---|
| LLM timeout or error | Silent fallback to the template; label "Generated by query templates" |
| LLM SQL rejected by the guard | Silent fallback; the guard reason is logged (not shown) |
| Query exceeds 2 s | Silent fallback if a template exists; else error state "That question took too long. Try narrowing the time window." |
| No template and no LLM | `422` → empty state with 4 suggestion chips |

---

## FL-08 · Unknown format → quarantine → AI draft → tests → approve → replay (SCN-04)

- **Trigger:** presenter key `2` (`POST /api/demo/inject-unknown`), then the **Draft parser**, **Run tests** and **Approve & replay** buttons.
- **Preconditions:** `DEMO_MODE`; `acme-edge` is not yet registered; `backend/app/aiparser/cache/acme-edge.json` exists (created by `make prebake`).

**Steps**

1. `DemoController.inject_unknown()` builds 20 br01 lines with `generator/br01.py → build_lines(now_ms)` (BACKEND_SCHEMA.md §5.4) and `POST`s them to `http://127.0.0.1:8000/api/ingest` as text.
2. FL-01 runs for each line. Detection fails, so each goes to quarantine `no_fingerprint` with shape `EDGE|x12`, one group, `group_id = qg-…`. SSE `event.quarantined` ×20; the UI groups them into `QuarantineGroupCard`.
3. **Draft parser** → `POST /api/parsers/drafts {group_id}` → `aiparser/drafts.py → DraftService.create(group_id)`:
   - load the group's 20 lines in seq order
   - split: held-out = indexes `{2, 5, 8, 11, 14, 17}`; train = the other 14
   - `DraftSource.draft(group_id, train_lines)`: `CachedDraftSource` in `DEMO_MODE` loads `cache/acme-edge.json` (`{origin, model, generated_at, yaml}`); `LlmDraftSource` outside demo mode calls Ollama with the drafting prompt (BACKEND_SCHEMA.md §4.5)
   - insert `parser_drafts`; return 201
   - the UI opens `ParserDraftDialog` with the YAML and the origin label. ⏱ < 50 ms (cached).
4. **Run tests** → `POST /api/parsers/drafts/{id}/test` → `DraftService.test(id)`:
   - `PackRegistry.load_candidate(yaml)` (temporary; not registered)
   - for every line: `parse_with_pack`, `map_to_ocsf`, `validate_ocsf`, `check_roundtrip`
   - aggregate held-out, all, round-trip and OCSF counts; `promotable = all four are total`
   - store and return. ⏱ < 200 ms.
   - The UI animates the four result rows in sequence (UI_UX_BRIEF.md §5.7).
5. **Approve & replay** → `POST /api/parsers/drafts/{id}/approve {approved_by}`:
   - checks `promotable`, writes `/data/packs/acme-edge.yaml`
   - `PackRegistry.reload()` (the new pack has priority 40)
   - audit log `parser.approve`
   - SSE `parser.approved`, then `replay.start(group_id)`
6. `pipeline/replay.py → start_replay()` re-queues the 20 `raw_id`s. The worker runs FL-01 steps 7–16. On success it deletes the quarantine rows and sets `raw_events.status='parsed'`. SSE `replay.progress {done, total}` per record, then `replay.completed {parsed:20, still_quarantined:0}`. ⏱ < 1 s total. The UI's progress bar goes 0 → 20, then toast "Parser acme-edge@1.0.0 approved · 20 records replayed".

**Error paths**

| Failure | Handling |
|---|---|
| Cache file missing | `CachedDraftSource` raises `DraftUnavailable`; `503` with the message "No cached draft. Run `make prebake` before recording." (shown in the dialog error state) |
| Tests not promotable | Approve button disabled with the tooltip "All four checks must pass 100% before approval."; API returns `409 not_promotable` if called anyway |
| Replay leaves records quarantined | Toast "{n} records still need attention" and the group card stays with the remaining count |

**Post-conditions:** `parsers` includes `acme-edge@1.0.0` (origin `ai_draft`); 20 br01 events are stored; quarantine total is 0; Search Q4 returns 20 rows.

---

## FL-09 · Log-silence detection (SCN-05)

- **Trigger:** presenter key `3` → `POST /api/demo/silence {device:"fw01", silent:true}`. The monitor ticks every 1 s.
- **Preconditions:** steady phase; fw01 has been monitored for ≥ 60 s of real time (it has, since the scripted start).

**Steps**

1. `DemoController.silence("fw01", True)` sets `stop_flags["fw01"] = True`. The generator skips fw01 lines; scenario time keeps running.
2. `intel/silence.py → SilenceMonitor.tick()` every 1 s: for each monitored source, `if wall_now − last_seen_wall > threshold` → `status="silent"`, `repo.update_source_status`, SSE `source.status_changed {device:"fw01", status:"silent", silent_for_s}`.
3. The UI `useSourcesStore.setStatus()` shows the `SilenceBadge` on the Devices tab, in the Live Feed header chips, and a toast "fw01 has gone silent — no events for 15 s (usually every 1 s)". ⏱ the badge appears 15–16 s after the key press.
4. Pressing `3` again resumes fw01. The first new event triggers `on_events`, and the status flips to `active` with SSE. The badge disappears within 1 s of the next event.

**Error paths:** none expected. If the monitor task crashes, it is restarted by the supervisor in `main.py`, and `/api/health` reports `monitors: restarted`.

---

## FL-10 · Parquet export, SIEM forward and retention report

**Parquet export**
1. Delivery screen **Export to Parquet** → `POST /api/export/parquet {}`.
2. `delivery/parquet.py → export_parquet(from_ms, to_ms)`:
   - `repo.events_flat(...)` returns a list of dicts
   - `pa.Table.from_pylist(rows)`
   - `con = duckdb.connect()`; `con.register("t", table)`
   - `con.execute("COPY (SELECT * FROM t ORDER BY time) TO ? (FORMAT PARQUET)", [path])`
   - `n = con.execute("SELECT COUNT(*) FROM read_parquet(?)", [path]).fetchone()[0]`

   ⏱ < 300 ms for 1,000 rows.
3. The response populates `ParquetExportCard`: "Exported {rows} events · verified by reading back {duckdb_rows} rows".

**SIEM forward (SIM)**
1. `SiemForwarder.on_batch(events)` formats each event as an RFC 5424 line with a JSON body and calls `FileSink.send()`, which appends to `/data/outbox/siem-forward.ndjson`.
2. The UI polls `GET /api/delivery/status` every 5 s.

**Retention report (SIM)**
1. **View report** → `GET /api/reports/retention` → `delivery/retention.py → build_report()`.
2. The UI renders `RetentionReportCard`; **Download JSON** saves the response.

**Error path:** lake directory not writable → `503` → card error state "Couldn't write to the data lake folder. Check that /data/lake is writable."

---

## FL-11 · Demo reset

- **Trigger:** `./demo.sh`, presenter `0 0`, or `POST /api/demo/reset`.

**Steps**

1. `DemoController.reset()`:
   - cancel the generator task
   - `SegmentRawStore.close()`
   - delete `/data/vault/*`, `/data/anvaya.db*`, `/data/witness/*.db`, `/data/outbox/*`, `/data/lake/*`, `/data/packs/*`
   - recreate the schema (`storage/schema.sql`); reopen the stores
   - `PackRegistry.reload()` (built-in packs only)
   - `ScenarioClock.reset()` → `idle`
   - clear the in-memory monitors, sealer and SSE replay buffer (keep the SSE id counter increasing)
   - `OllamaClient.warmup()` in the background
2. Publish SSE `demo.state {phase:"idle", reset:true}`.
3. UI: `resetAllStores()` clears every Zustand slice; `queryClient.clear()`; navigate to `/overview`; toast "Demo reset · ready".
4. `demo.sh` polls `GET /api/health` until `status == "ok"` and `GET /api/stats` until `raw_total == 0`, then prints `ANVAYA ready · open http://localhost:8080/overview`. ⏱ 2–5 s (target < 15 s).

**Error path:** a file is locked → retry 3 times with 200 ms between; if it still fails, return `500` with the path, and `demo.sh` prints "Reset failed: {path}. Run `docker compose restart api` and try again."

---

## FL-12 · Real-time stream connect and reconnect

1. `App` mounts `useStreamConnection()`, which creates `new EventSource("/api/stream")`.
2. On `open`: `useStreamStore.setStatus("live")`.
3. Each message is dispatched by `event` type to the store actions (UI_UX_BRIEF.md §8). `lastEventId` is kept by the browser.
4. On error: status `reconnecting`. `EventSource` retries automatically (server `retry: 1500`); the browser sends `Last-Event-ID`, and the server replays from its 2,000-message buffer. If the requested id is older than the buffer, the server sends `event: resync`, and the client refetches `GET /api/stats`, `/api/events?limit=50`, `/api/incidents` and `/api/sources`.
5. Duplicate protection: `useFeedStore.add()` ignores an `event_id` it already holds.
6. The status shows as a small dot in the top bar: Live (green), Reconnecting (amber), Offline (red, after 10 s).
