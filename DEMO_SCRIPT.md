# DEMO_SCRIPT.md: ANVAYA Demo Video Script

| Attribute | Detail |
|---|---|
| Product | ANVAYA: *Every Perimeter Log, One Verifiable Truth* |
| Problem statement | SIH26156, Universal Log Pre-processing Framework (NTRO · Blockchain & Cybersecurity · Software) |
| Document version | 1.0.0 |
| Video length | 3:50 main cut (3–4 minute target) · 2:00 compliance cut · 1:00 short cut |
| Recording resolution | 1920×1080, browser zoom 100% |
| Depends on | PRD.md, UI_UX_BRIEF.md, APP_FLOW.md, BACKEND_SCHEMA.md |

This script is the source of truth for the MVP. Everything the build contains exists because a shot below needs it.

---

## 1. Conventions used in this script

- **Scenario clock.** In `DEMO_MODE` every component reads time from the scenario clock (TRD.md §9). When the attack simulation starts, the clock jumps to **10:00:00 IST today** and runs at the playback speed (10× for the main cut, 20× for the 2:00 cut, 40× for the 60 s cut) until 10:12:00, then at 1×. `./demo.sh --speed N` sets the starting speed. Times in the narration (10:02, 10:05, 10:07) are scenario-clock times, and the top bar labels the clock "Scenario clock".
- **Real-time offsets.** Real seconds after Start = scenario seconds since 10:00:00 ÷ speed. At 10×: 10:02:05 → 12.5 s, 10:05:00 → 30.0 s, 10:07:12 → 43.2 s. At 20×: 6.3 s, 15.0 s, 21.6 s. Each shot list is choreographed around its speed.
- **Presenter keys** work only in `DEMO_MODE`, and only when focus is not in a text field. They are defined in UI_UX_BRIEF.md §9.

| Key | Action | Endpoint |
|---|---|---|
| `` ` `` | Show or hide the presenter panel | (client only) |
| `1` | Start attack simulation (SCN-01) | `POST /api/demo/scenario/start` |
| `2` | Inject unknown br01 format (SCN-04) | `POST /api/demo/inject-unknown` |
| `3` | Silence / resume fw01 (SCN-05) | `POST /api/demo/silence` |
| `4` | Simulate tamper of one raw record (SCN-02) | `POST /api/demo/tamper` |
| `5` | Insider re-seal of the ledger (SCN-02b) | `POST /api/demo/reseal` |
| `6` | Cycle playback speed 10× → 20× → 40× → 10× | `POST /api/demo/speed` |
| `0` then `0` within 2 s | Reset everything to the first frame | `POST /api/demo/reset` |

- **Scenario IDs** are used identically in every file: `SCN-01` attack correlation, `SCN-02` tamper detection (with `SCN-02b` insider re-seal), `SCN-03` NL search, `SCN-04` unknown format → AI parser → replay, `SCN-05` log silence.

---

## 2. Main cut: 3 minutes 50 seconds

This is the primary video (target 3–4 minutes).

> **Length rule:** the official SIH26156 brief lists "Demo Video (Max 2 Minutes)" as an evaluation deliverable. Where that limit applies, submit the 2:00 compliance cut (§3), which is recorded from the same build. Use this 3:50 cut wherever a 3–4 minute video is requested.

**Pre-roll state:** run `./demo.sh --speed 10`. At **10×**, scenario 10:02:05 arrives 12.5 s after you press Start, 10:05:00 after 30.0 s, 10:07:12 after 43.2 s, and 10:12:00 after 72.0 s. The browser is on `/overview` with all counters at 0. Wi-Fi is off and the health chip reads **"No external route"**. A terminal window with the completed `make offline-test` output is open behind the browser for the 3:32 shot.

| Time | Screen | On-screen action | Exact narration | Key |
|---|---|---|---|---|
| 0:00–0:14 | Overview (idle) | Static. The hero reads "Perimeter evidence overview"; counters are at 0; the three-layer strip shows L1 · L2 · L3. | "NTRO's perimeter is made of firewalls, intrusion sensors and VPN gateways from different vendors — and every one of them writes logs in its own dialect. Before an analyst can use a single line, an engineer has to hand-write a parser for it." | — |
| 0:14–0:25 | Overview (idle) | Slowly hover the three layer labels in the strip; then hover the "No external route" health chip (its tooltip opens). | "ANVAYA is a universal pre-processing framework. It seals every original byte, normalises each event into the OCSF standard, and turns the result into intelligence — entirely offline. This laptop has no network connection." | — |
| 0:25–0:37 | Overview → Live Feed | Click **Start attack simulation**; the app moves to Live Feed after 600 ms. Rows stream from fw01, ids01 and vpn01. Hover the three format chips one after another. | "Here is lab traffic from three devices in three formats: Cisco ASA syslog, Suricata JSON, and FortiGate key-value. Each row is parsed, mapped and given a confidence score as it arrives." | `1` |
| 0:37–0:54 | Event Detail (`/events/e-64f70aaee53f`) | At 0:38 click the first purple ids01 row, "ANVAYA-LAB SCAN Inbound TCP port sweep". Hover `evidences[0].src_endpoint.ip` (bytes 166–178 highlight, connector draws), then `finding_info.title` (bytes 328–366), then `severity` (marked "derived"), then the `unmapped.flow_id` row. Point at the "Lossless ✓ 100% of bytes accounted for" chip. Click **Verify this record** → "Verified · sealed in batch 3 · co-signed by 2 witness nodes". | "This is the heart of ANVAYA. Every field in the standard record points to the exact bytes it came from. Derived values say so. Fields with no standard home are kept, not dropped — so the round-trip is provably lossless. And the original is hashed, sealed in a signed batch, and co-signed by witness nodes." | — |
| 0:54–1:08 | Live Feed | Go back to Live Feed. The vpn01 failure burst is streaming (0:55–0:59). Click the vpn01 device chip to filter, then clear the filter. | "Back on the feed, the VPN is suddenly flooded with failed logins. On its own, that is noise. ANVAYA keeps watching across devices." | — |
| 1:08–1:35 | Incidents (`/incidents/inc-0001`) | At about 1:08 the toast appears: "High-risk incident · 198.51.100.9 → 10.2.4.18 · 3 sources". Click **Open incident**. Walk the timeline: Reconnaissance 10:02 (fw01 + ids01) → Credential attack 10:05 (vpn01, 40 failures) → Access 10:07 (fw01 allowed). Hover the risk breakdown "3 stages +75 · 3 devices +10 · in attack order +10". | "And there it is. A port sweep at 10:02, seen by both the firewall and the IDS. Forty failed VPN logins at 10:05. Then, at 10:07, the firewall lets the very same address in. No single device sees this story. Because every log now speaks one schema, ANVAYA joins them into one incident — risk 95, high — with each step traceable to its evidence." | — |
| 1:35–2:00 | Search (`/search`) | Click the chip "What did 10.2.4.18 do in the last hour?" → SQL with "Generated by local LLM · qwen2.5:3b · read-only" → 17 rows. Then click "Which IPs had failed VPN logins in the last 15 minutes?" → 198.51.100.9: 40, 203.0.113.60: 2. | "Analysts don't need to learn four vendors' field names. They ask. A local language model writes the SQL — shown in full, checked against a read-only allow-list — and the answer comes back in seconds. Nothing ever leaves the network." | — |
| 2:00–2:08 | Integrity (`/integrity`) | Click **Verify ledger** → green banner "All {records} records verified across {batches} batches · both witness nodes agree" (live counts, not read aloud). | "Now, trust. Every batch of records has a Merkle root, chained and signed. Everything verifies." | — |
| 2:08–2:20 | Integrity | Press `4` → toast "Raw record r-899f4edf0030 altered on disk (demo)". Click **Verify ledger** → red banner "1 record failed verification". Expand the row: "fw01 · 10:07:12 · byte 153 changed: '8' → '9' (dst_endpoint.ip)". Click **Open event** → Event Detail shows the byte in `--lin-tampered` red inside `dst_endpoint.ip`. Click **Back to Integrity**. | "Suppose an insider edits the one log that proves the attacker got in — changing a single byte of the target address. Verification names the record, the byte, and the field it belonged to." | `4` |
| 2:20–2:35 | Integrity | Press `5` → toast "Ledger re-signed with the primary key from batch 11 (demo insider)". Click **Verify ledger** → amber banner "Primary ledger consistent · witness nodes disagree on batch 11 onward — tamper suspected". | "A smarter insider re-signs the whole ledger with the primary key. On its own, it would now look clean. But two witness nodes — simulated here on this laptop, with their own keys — still hold the original heads. The rewrite is caught." | `5` |
| 2:35–2:42 | Sources & Parsers (`?tab=quarantine`) | Press `3` silently on arrival (silences fw01; the badge appears about 16 s later). Then press `2`: the group card appears, "Unrecognised source · 20 records · 12 pipe-delimited fields starting with EDGE". | "Next, a brand-new branch router comes online with a format ANVAYA has never seen. Nothing is dropped. All twenty records are sealed and held in quarantine." | `3`, then `2` |
| 2:42–3:05 | Parser draft dialog | Click **Draft parser** → YAML with "Drafted by local LLM (cached run)". Click **Run tests** → the four checks fill in: Held-out 6/6 · All lines 20/20 · Lossless 20/20 · Valid OCSF 20/20. Click **Approve & replay** → progress bar 20/20 → toast "Parser acme-edge@1.0.0 approved · 20 records replayed". The quarantine count becomes 0. | "ANVAYA's local model drafted a parser from the samples. But AI output never goes straight to production. The draft must parse every held-out line, pass the lossless round-trip check, and produce valid OCSF — and then a human approves it. Approved, and the twenty quarantined records are replayed." | — |
| 3:05–3:12 | Search | Click "Show events from br01" → 20 rows. | "The new router is now searchable like everything else." | — |
| 3:12–3:20 | Sources & Parsers (`?tab=devices`) | Devices tab: fw01 shows the amber badge "Silent · last event {n} s ago". | "One more thing. The firewall has gone quiet — often the first move of an attacker covering their tracks. ANVAYA noticed." | — |
| 3:20–3:32 | Delivery (`/delivery`) | Click **Export to Parquet** → "Exported {rows} events · verified by reading back {rows} rows". Point at the SIEM forwarder card (file sink, demo) and the retention report card's statement. | "Normalised data goes out to the data lake as Parquet and to any SIEM. The retention report shows how ANVAYA's tamper-evident vault supports CERT-In's 180-day log retention rule." | — |
| 3:32–3:40 | Terminal (insert) | Show the finished `make offline-test` output: `docker run --rm --network none …` → `selfcheck: 207 raw · 1 incident (risk 95) · verify ok · tamper caught · reseal caught · Q1 17 rows · br01 20 replayed · PASS`. | "Everything you saw also passes as an automated self-check inside a container with networking disabled." | — |
| 3:40–3:50 | Overview | Click Overview. Counters show events ingested, live ingest rate (measured), 4 formats detected, parse coverage 100%, sealed and co-signed records. | "Lossless. Traceable. Verifiable. Fully offline. ANVAYA — every perimeter log, one verifiable truth." | — |

**Narration word count:** 518 words over 230 s, about 2.25 words per second, which leaves room for pauses.

**Alternate lines (use if the UI shows a different state):**
- Parser dialog origin reads **"Hand-authored draft"**: at 2:42 say *"ANVAYA proposed a parser from the samples. But no draft ever goes straight to production…"* (rest unchanged).
- Search label reads **"Generated by query templates"**: at 1:35 say *"…They ask. ANVAYA writes the SQL — shown in full, checked against a read-only allow-list…"* (rest unchanged).

---

## 3. Compliance cut: 2 minutes

Pre-roll state: `./demo.sh` (default speed 20×) has just run. This is the cut to submit where a 2-minute limit applies. The browser is on `/overview`, all counters read 0, Wi-Fi is off, and the health chip reads **"No external route"** (green).

| Time | Screen | On-screen action | Exact narration | Key |
|---|---|---|---|---|
| 0:00–0:09 | Overview (`/overview`) | Static. Counters at 0. Top bar shows "Synthetic lab data", "Scenario clock", and health chips Vault ✓ · Ledger ✓ · Witnesses 2/2 · Local LLM Ready · No external route. | "Every firewall, IDS and VPN on NTRO's perimeter writes logs in its own dialect. Before anyone can analyse them, an engineer hand-writes a parser for each one." | — |
| 0:09–0:14 | Overview | Click **Start attack simulation**. Counters begin ticking. The app auto-navigates to Live Feed after 600 ms. | "ANVAYA turns any perimeter log into one verifiable record — running on this laptop, with the network disconnected." | `1` |
| 0:14–0:21 | Live Feed (`/feed`) | Rows stream from fw01 (Cisco ASA syslog), ids01 (Suricata EVE JSON) and vpn01 (FortiGate key=value), each with a device chip, format chip and confidence pill. | "Three devices, three formats — Cisco syslog, Suricata JSON, FortiGate key-value — arriving live." | — |
| 0:21–0:35 | Event Detail (`/events/e-64f70aaee53f`) | At 0:21 click the first purple ids01 row, "ANVAYA-LAB SCAN Inbound TCP port sweep". In the lineage view, hover `evidences[0].src_endpoint.ip`: bytes 166–178 of the raw log light up and a connector links them. Hover `finding_info.title`: bytes 328–366 light up. Click **Verify this record** → "Verified · sealed in batch 3 · co-signed by 2 witness nodes". | "Every field in the standard OCSF record points back to the exact bytes it came from. The original is kept untouched — hashed, sealed and verifiable." | — |
| 0:35–0:49 | Incidents (`/incidents/inc-0001`) | At about 0:36 the incident toast appears: "High-risk incident · 198.51.100.9 → 10.2.4.18 · 3 sources". Click **Open incident**. The timeline shows Recon 10:02 (ids01 + fw01) → Credential attack 10:05 (vpn01, 40 failures) → Access 10:07 (fw01 allowed). Risk 95 · High. | "Seconds later ANVAYA connects what no single device can see: a port scan, forty failed VPN logins, then the firewall letting the same address in. One incident. High risk." | — |
| 0:49–1:01 | Search (`/search`) | Click the suggestion chip "What did 10.2.4.18 do in the last hour?". The generated SQL appears with the label "Generated by local LLM · qwen2.5:3b · read-only". The results table shows 17 rows. | "Analysts simply ask. A local model writes the SQL — shown in full, read-only — and nothing ever leaves the network." | — |
| 1:01–1:07 | Integrity (`/integrity`) | Click **Verify ledger** → green banner "All {records} records verified across {batches} batches · both witness nodes agree", with live counts filled in (they depend on the moment, so don't read them aloud). | "Now an insider edits the one log that proves the attacker got in." | — |
| 1:07–1:13 | Integrity | Press `4`. The toast reads "Raw record r-899f4edf0030 altered on disk (demo)". Click **Verify ledger** → red banner "1 record failed verification". The failing row expands: "fw01 · 10:07:12 · byte 153 changed: '8' → '9' (dst_endpoint.ip)". | "Verification pinpoints the record — and the exact byte that changed." | `4` |
| 1:13–1:21 | Integrity | Press `5` (insider re-seal). Click **Verify ledger** → amber banner "Primary ledger consistent · witness nodes disagree on batch 11 — tamper suspected". The witness panel shows witness-ntro ✗ and witness-certin ✗ against the primary. | "Even if they re-sign the whole ledger, two witness nodes — simulated here on this laptop — still hold the original. The rewrite is caught." | `5` |
| 1:21–1:24 | Sources & Parsers (`/sources?tab=quarantine`) | Press `3` silently on arrival (silences fw01; the badge appears about 16 s later). Then press `2`: the quarantine group appears, "Unrecognised source · 20 records · 12 pipe-delimited fields starting with EDGE". | "A brand-new router format arrives. Nothing is dropped — it waits in quarantine." | `3`, then `2` |
| 1:24–1:40 | Sources & Parsers → Parser draft dialog | Click **Draft parser**. The dialog shows the YAML with the origin label "Drafted by local LLM (cached run)". Click **Run tests** → "Held-out 6/6 passed · Lossless 20/20 · OCSF valid 20/20". Click **Approve & replay** → progress bar 20/20 → toast "Parser acme-edge@1.0.0 approved · 20 records replayed". The quarantine count drops to 0. | "ANVAYA's local model drafted a parser — but it only goes live after passing every held-out line, losslessly, and a human approves it. Twenty records, replayed." | — |
| 1:40–1:49 | Sources & Parsers (`?tab=devices`) | Switch to the Devices tab. fw01 shows the amber badge "Silent · last event 17 s ago". | "And when a firewall suddenly goes quiet — often an attacker's first move — ANVAYA raises it." | — |
| 1:49–2:00 | Overview | Click Overview. Counters show events ingested, live ingest rate (measured), 4 formats detected, parse coverage 100%, sealed records. The "No external route" chip is still green. | "Lossless. Traceable. Verifiable. Fully offline. ANVAYA — every perimeter log, one verifiable truth." | — |

**Narration word count:** 268 words, about 2.2 words per second, which leaves room for pauses.

**Alternate lines (use if the UI shows a different state):**

- If the parser draft dialog shows origin **"Hand-authored draft"** (because `make prebake` ran without Ollama), replace the 1:24 line with: *"ANVAYA proposed a parser — but it only goes live after passing every held-out line, losslessly, and a human approves it. Twenty records, replayed."*
- If the Search label reads **"Generated by query templates"** (Ollama stopped or slower than 5 s), replace the 0:49 line with: *"Analysts simply ask. ANVAYA writes the SQL — shown in full, read-only — and nothing ever leaves the network."*

---

## 4. Short cut: 60 seconds

Pre-roll: run `./demo.sh --speed 40`. At 40×, scenario 10:02:05 arrives at +3.1 s and 10:07:12 at +10.8 s.

| Time | Screen | On-screen action | Exact narration | Key |
|---|---|---|---|---|
| 0:00–0:06 | Overview | Static, counters at 0 | "Every perimeter device writes logs in its own dialect, and each one needs a hand-written parser." | — |
| 0:06–0:10 | Overview → Live Feed | Start attack simulation | "ANVAYA turns them all into one verifiable record — fully offline." | `1` |
| 0:10–0:22 | Event Detail | Click the first ids01 alert; hover `evidences[0].src_endpoint.ip`, then `finding_info.title`; click Verify this record | "Every standard field points to the exact bytes it came from, and the original is sealed and verifiable." | — |
| 0:22–0:34 | Incidents | Toast → Open incident → timeline | "A scan, forty failed VPN logins, then a firewall allow — ANVAYA links them into one high-risk incident." | — |
| 0:34–0:43 | Search | Click "What did 10.2.4.18 do in the last hour?" → SQL + 17 rows | "Ask in plain English; a local model writes read-only SQL." | — |
| 0:43–0:54 | Integrity | Verify (green) → `4` → Verify (red, byte 153) | "Change one byte on disk, and verification names the record and the byte." | `4` |
| 0:54–1:00 | Overview | Counters | "Plus AI-drafted parsers and log-silence alerts. ANVAYA — every perimeter log, one verifiable truth." | — |

---

## 5. Recording checklist

**Machine and browser**

1. Use a dedicated Chrome profile named "ANVAYA demo" with no extensions, no bookmarks bar and no signed-in account.
2. Set browser zoom to 100% and the window to exactly 1920×1080 (use full screen with `F11`, or a window-sizing tool). Check the in-app footer readout "Viewport 1920×1080" (shown in the presenter panel).
3. Turn on OS "Do Not Disturb", quit chat apps, and hide the dock or taskbar.
4. Set the system clock to automatic time; the scenario clock still overrides it in `DEMO_MODE`.
5. Turn Wi-Fi off and unplug Ethernet **before** `./demo.sh`, so the Network chip reads "No external route".
6. Ollama must be running locally with `qwen2.5:3b` already pulled (`docker compose --profile llm up -d`). If you decide not to use it, use the alternate narration lines in §2 and §3.

**State reset before every take**

1. Run `./demo.sh` (it resets the vault, database, witnesses, outbox, lake and UI state, then waits for health).
2. Wait for the terminal line `ANVAYA ready · open http://localhost:8080/overview`.
3. Reload the browser tab once (`Ctrl+R`) and confirm the counters read 0 and Live Feed is empty.
4. Press `` ` `` to confirm the presenter panel opens and shows Phase: idle, Speed: 20×. Press `` ` `` again to hide it.
5. Run the golden-path checklist (IMPLEMENTATION_PLAN.md §9) three times in a row before the first real take.

**Takes plan**

| Take | Purpose |
|---|---|
| 1–2 | Full 3:50 main cut at 10× (`./demo.sh --speed 10`), screen-only, for timing. Record the narration separately afterwards. |
| 3 | Full 3:50 main cut with live narration, as a backup |
| 4 | 2:00 compliance cut at 20× (`./demo.sh`) |
| 5 | 60-second cut at 40× (`./demo.sh --speed 40`) |
| 6 | Insert shots: `make offline-test` terminal output, a lineage hover close-up, and the tamper byte diff |

Edit only at shot boundaries in the tables above. Never cut inside a running animation.

---

## 6. Judge Q&A preparation

**Q1. How is this different from Logstash, Fluentd or a SIEM?**
Logstash and Fluentd are pipelines: you still hand-write a Grok or regex pattern for every source, and once a log is transformed, nothing proves it matches the original. A SIEM is a downstream consumer with its own (often closed and licensed) parsers. ANVAYA sits before the SIEM and adds four things those tools don't give you together:
1. every normalized field carries the exact byte span it came from
2. every raw byte is sealed in a signed Merkle ledger with witness co-signatures
3. new formats are onboarded by an offline AI draft that must pass held-out tests before a human approves it
4. correlation across vendors becomes possible because everything shares one OCSF schema

It feeds any SIEM, rather than replacing one.

**Q2. What if the LLM hallucinates a parser?**
It can't reach production. The model never runs in the live ingest path; it only drafts a YAML parser offline. A draft is promotable only if it passes:
- **100%** of held-out sample lines
- the round-trip losslessness check (every extracted value equals its source bytes, and at least 95% of each line is accounted for)
- OCSF schema validation

Even then, a human must click Approve, and the approval is written to the audit log. A wrong draft fails the tests, and the raw records stay safely in quarantine.

**Q3. How does it reach billions of events per day?**
The MVP runs on one laptop, so we only claim what we measured there (the BENCHMARKS.md number, labelled with the machine). The design scales horizontally:
- The ingest queue interface is built to be swapped for a Kafka-compatible partitioned stream.
- Parser workers are stateless, so throughput grows by adding workers per partition.
- Normalized data lands in a columnar Parquet lake.
- Signing cost stays low because one signature covers a whole Merkle batch, not each event.

That path is on our roadmap slide. We have not built or benchmarked it in the MVP.

**Q4. What happens to logs you can't parse?**
They are never dropped. Every raw byte is stored and sealed **before** parsing. A line that can't be fingerprinted or parsed goes to quarantine with a reason, grouped by its structural shape. When a parser is approved, quarantined records are replayed through it; the raw bytes and ledger are untouched.

**Q5. Why should NTRO trust the hash chain, and who holds the keys?**
Each event's SHA-256 is a leaf in a Merkle tree. Each batch root is chained to the previous batch and signed with Ed25519. That alone catches edits, deletions and reordering. Because whoever holds the primary key could re-sign history, each signed batch head is also co-signed by witness nodes holding their own keys. In production, those would be operated independently, for example by NTRO and CERT-In. In the MVP, the two witnesses are simulated as separate local stores with separate keys, and we show that a re-signed ledger is caught by witness disagreement. Keys are generated locally at first boot and never leave the machine.

**Q6. Does it really run with no internet?**
Yes. The video was recorded with Wi-Fi off, and the health chip checks for an external route live. The model weights, fonts and OCSF schema files are all local. `make offline-test` runs the full pipeline self-check inside a container started with `--network none`.

**Q7. What did you measure, and on what data?**
- The live ingest rate on the Overview is measured from real ingest timestamps during the demo.
- The `make bench` result is the sustained events per second on the named laptop, written to BENCHMARKS.md with CPU, RAM and date.
- Parser correctness is measured by golden tests on sample lines modelled on vendor documentation, plus the round-trip check.

All demo traffic is **synthetic lab data** from our deterministic generator, and the UI says so on every screen.

### What is real and what is simulated

| Capability | In the MVP | What is actually happening |
|---|---|---|
| Log generation | Real (synthetic data) | The deterministic generator sends real UDP syslog and HTTP to the real ingest |
| Ingest, vetting, raw vault, SHA-256 | Real | Bytes are written to disk and hashed on receipt |
| Merkle batches, Ed25519 signatures, verify, tamper detection | Real | The tamper button really changes one byte on disk |
| Fingerprinting, 3 parsers, OCSF, byte lineage, round-trip check | Real | Every row on screen went through the pipeline |
| Quarantine and replay | Real | |
| Cross-device correlation | Real | The rule runs on stored events |
| Log-silence detection | Real | The generator really stops sending fw01 |
| NL search | Real (local LLM with a template fallback) | The engine label says which one produced the SQL |
| Parquet export | Real | Written by DuckDB to `data/lake/` |
| Live ingest rate, `make bench` | Real (measured) | |
| Offline operation | Real | Wi-Fi off; `--network none` self-check |
| AI parser builder | **Simulated draft step** | The YAML draft is cached output from an earlier local-LLM run (or hand-authored, labelled as such). The tests, approval and replay run live. |
| Witness nodes | **Simulated** | Two local stores with their own keys. The co-signing and comparison are real cryptography. |
| SIEM delivery | **Simulated target** | A real RFC 5424/JSON payload is written to a file sink, not sent to a SIEM |
| CERT-In retention report | **Simulated template** | Filled from real counts and worded "supports", never "certifies" |
| Kafka-compatible scale-out, Postgres, MinIO, Wazuh/OpenSearch, RBAC, signed plugins | Cut | Roadmap slide only |
