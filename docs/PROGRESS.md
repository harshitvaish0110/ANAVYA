# PROGRESS

## Done

| Module | What | Check (run and passing) |
|---|---|---|
| M00 | Scaffold, `pyproject.toml` (no pins), `interfaces.py`, `models.py`, `config.py`, `schema.sql` | `test_schema.py`, `test_config.py` |
| M01 | Deterministic generator: 207 scripted lines, steady phase, br01 injection, UDP sink, `--dry-run` | line 164 = BACKEND_SCHEMA §4.1; seq 40/100/164/180 raw_ids |
| M02 | Vetting, segment vault, `raw_events`, UDP + HTTP ingest, `GET /api/raw` | `test_vault.py`; live UDP stores `d12173d4…4251` |
| M03 | RFC 6962 Merkle (incl. CT known answers), Ed25519 heads, 2 witnesses, verify, record proof, tamper, re-seal | `test_merkle.py`, `test_witness.py`, `test_ledger.py` |
| M04 | Syslog header split, pack registry + detection, regex / span-aware JSON / kv / delimited extractors, shape | `test_syslog_header.py`, `test_extractors.py` |
| M05 | OCSF mapper, lineage, round-trip, confidence, flatten; OCSF **1.9.0 vendored** | `test_parsers.py`, `test_ocsf_validation.py`, `test_ocsf_enums.py`, `test_roundtrip.py`, `test_confidence.py` (91/90/89/95) |
| M06 | Pipeline worker, quarantine, replay, event bus, stats | `test_pipeline.py` |
| M07 | Correlation and silence detection | `test_correlation.py` (inc-0001, 95, 56 events; opened at the 10th failure at 70), `test_silence.py` |
| M08 | NL search: Ollama client, exact prompt, T1–T5, sqlglot guard, read-only execution | `test_sql_guard.py`, `test_search_templates.py` (Q1 17 · Q2 40/2 · Q3 12 · Q4 0→20) |
| M09 | AI parser builder: cached/LLM sources, 4 gates, approve → reload → replay; prebake | `test_parser_drafts.py`; cache written by prebake |
| M10 | 35 routes with response models + error envelope, SSE, scenario clock, demo controller, supervisor | `test_api_contract.py`; live `curl -N /api/stream`: normalized → opened (70) → updated (95) |
| M11 | Parquet (DuckDB read-back), SIEM file sink, retention report, netprobe, `selfcheck.py` | `test_parquet.py`, `test_retention.py`; `make offline-test` → **PASS** in `--network none` |
| M12 | Vite + React 19 + TS + Tailwind 4 + shadcn/ui, tokens, no-raw-hex lint, types, typed API client, SSE client, Zustand slices, shell, presenter panel/keys | `npm run lint`, Vitest |
| M13–M19 | Overview, Live Feed, Event Detail (lineage view), Incidents + toast, Search, Integrity, Sources & parsers, Delivery | screenshot walkthrough at 1920×1080 and 1440×900; axe WCAG 2.1 AA: 0 violations on all 8 screens |
| M20 | Compose, Dockerfiles, nginx, `demo.sh`, Makefile, bench, Playwright golden path | `docker compose build`; `./demo.sh` (reset 1.5 s); `make e2e` passes on the compose stack; `make bench` row in BENCHMARKS.md |

Totals: backend `pytest` **166 passed**; frontend Vitest **21 passed**; `ruff` and `npm run lint` clean; Playwright
golden path passes against both `npm run dev` and the Docker stack with **zero console errors or warnings**.

## Next

- Record the takes (DEMO_SCRIPT.md §5) on the recording laptop with Wi-Fi off; run the §9 checklist 3× then 10×.
- Re-run `make bench` on the recording laptop (see BENCHMARKS.md for Docker Desktop).

## Decisions

- **Repo root is `anvaya-v2/`**; the spec files stay at the root (not `docs/spec/`).
- **OCSF 1.9.0** (latest stable) is vendored with the profiles ANVAYA uses: `security_control` (action/disposition),
  `datetime` (`time_dt`), `host` (`device` on 4001/3002). JSON Schema exports carry enum ids without captions, so the
  same version's native definitions (`schema/api/*.json`) are vendored for captions. `schema/PROFILES` records both URLs.
- **Golden files updated for 1.9.0** (spec rule: follow the schema): `metadata.version` 1.3.0 → 1.9.0;
  `device.type_id/type` added (packs declare `device_type`: ASA Firewall, Suricata IDS, FortiGate Firewall, Acme Router);
  `connection_info.direction_id` 0 (Unknown) where the source has no direction (Suricata evidence, acme);
  FortiGate gains `dst_endpoint.hostname` = devname (Authentication needs `dst_endpoint` or `service`), lineage [44,49].
- **acme-edge golden lineage** gains `disposition` and `severity` ([38,43], derived): TRD §7.2 gives every rule constant
  lineage and the §4.1 ASA golden follows it; §4.4 omitted them.
- **Lineage order** is canonical; tests compare lineage as a set (the UI orders fields itself).
- **Tamper target is "seq 164"**, not a fixed raw_id (`r-899f4edf0030` is only that id on 2026-09-30).
- **Silence "monitored" rule:** ≥ 3 gaps and the source has been *sending* for ≥ 60 s, evaluated on arrival (the literal
  reading would make br01's one-off burst raise a false alert a minute later).
- **`inject-unknown`** ingests in-process (transport `http`) instead of a loopback POST.
- **Queue and replay manager** live for the container's lifetime; replay ids restart at `rp-0001` after reset.
- **API 21** additionally returns `events` (summaries of the linked events) so evidence chips need no extra calls.
- **OCSF validation cache** keyed by object shape (keys, leaf types, enum/const id values); safe because the vendored
  schema constrains values only via enum/const on `*_id`/`*_uid` (checked at load; the cache turns itself off otherwise).
  Together with per-segment read handles and skipping benign events in correlation: ~77 → ~1,000 normalized events/s.
- **Tailwind v4**: tokens are mapped in `globals.css` `@theme inline` (v4's replacement for `tailwind.config.ts`); the
  default palette is removed so only tokens exist. `cn()` knows the brief's type scale (tailwind-merge would otherwise
  drop colours/sizes).
- **Accessibility over literal token pairs:** `SeverityBadge` text is ink on the severity tint with a severity dot
  (`--sev-medium` on its tint is 4.42:1); text under 15 px uses `--graphite`, not `--muted` (as the brief requires).
- **Zustand slices** consolidated: `state/feed.ts`, `state/demo.ts`, the rest in `state/slices.ts`.
- **Air-gap check**: `npm run build` fails on any URL in `dist/` outside an allow-list (localhost, XML namespaces, inert
  library error-doc strings).
- **`make bench`** sends for the full 30 s, reports pipeline catch-up against records received, and writes nothing on a
  zero-throughput run. On Docker Desktop it is run inside the api container (its UDP proxy drops most datagrams).
- **Ruff** pinned to `E,F,W` (minus line length); **`opentelemetry-api`** is a FastAPI dependency (API only, no exporter).
- **compose** mounts `./docs` read-only so `/api/stats.last_bench` shows the latest measured bench row.
- **`demo.sh`** picks the first Python that actually runs (Windows ships a `python3` Store stub).

## Assumptions

- `parse_coverage_pct` = events passing round-trip ÷ (those + quarantine rows), 1 decimal; `null` before any event.
- `witnesses.ok_count` in health = witnesses whose last co-signed batch equals the primary's last batch (after an
  insider re-seal the witnesses refuse the rewritten chain, so it reads 0/2 — intended).
- Record verify `ok` requires the hash, root, signature and every witness to agree.
- `sources.rate_epm` = events received in the last 60 real seconds.
