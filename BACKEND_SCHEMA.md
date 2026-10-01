# BACKEND_SCHEMA.md: ANVAYA Data Model, Samples and Generator

| Attribute | Detail |
|---|---|
| Version | 1.0.0 |
| Storage | SQLite (WAL) file `/data/anvaya.db`; raw vault segment files under `/data/vault/` |
| Depends on | TRD.md (rules and API), APP_FLOW.md (flows) |
| Sample timestamps | Shown for the date 2026-09-30. The generator always writes **today's** date. |

---

## 1. Conventions

- Times are `INTEGER` epoch milliseconds, UTC. Display is in `Asia/Kolkata` (IST).
- Hashes are lowercase hex `TEXT` (64 chars for SHA-256). Signatures are hex `TEXT` (128 chars for Ed25519).
- **IDs**
  - `raw_id` = `"r-" + sha256(seq_be64 ‖ raw_sha256_bytes)[:12]`
  - `event_id` = `"e-" +` the same 12 hex as its `raw_id`
  - `incident_id` = `"inc-" + 4 digits`
  - `draft_id` = `"dr-" + 4 digits`
  - `replay_id` = `"rp-" + 4 digits`
  - `group_id` = `"qg-" + sha256(shape_signature)[:8]`
- JSON columns are `TEXT` holding compact JSON (`json.dumps(..., separators=(',',':'))`).
- `PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA foreign_keys=ON;` are set at connect.

## 2. SQLite schema (`backend/app/storage/schema.sql`)

### 2.1 `raw_events`: one row per received record

```sql
CREATE TABLE raw_events (
  seq              INTEGER PRIMARY KEY,                -- monotonic from 1
  raw_id           TEXT    NOT NULL UNIQUE,
  raw_sha256       TEXT    NOT NULL,                   -- over the exact received bytes
  size             INTEGER NOT NULL CHECK (size >= 0),
  segment          TEXT    NOT NULL,                   -- 'seg-000001.log'
  offset           INTEGER NOT NULL,                   -- byte offset of the u32 length prefix
  encoding         TEXT    NOT NULL CHECK (encoding IN ('utf-8','binary')),
  flags            TEXT    NOT NULL DEFAULT '[]',      -- JSON array of vetting flags
  transport        TEXT    NOT NULL CHECK (transport IN ('udp','tcp','http','file')),
  peer_ip          TEXT    NOT NULL,
  received_at      INTEGER NOT NULL,                   -- Clock.now_ms() at receipt
  wall_received_at INTEGER NOT NULL,                   -- real wall clock (for silence detection)
  status           TEXT    NOT NULL CHECK (status IN ('pending','parsed','quarantined')),
  batch_id         INTEGER REFERENCES ledger_batches(batch_id)
);
CREATE INDEX ix_raw_status ON raw_events(status);
CREATE INDEX ix_raw_batch  ON raw_events(batch_id);
```

### 2.2 `events`: one row per normalized OCSF event (flat columns + full envelope)

```sql
CREATE TABLE events (
  event_id          TEXT    PRIMARY KEY,
  raw_id            TEXT    NOT NULL UNIQUE REFERENCES raw_events(raw_id),
  seq               INTEGER NOT NULL,
  time              INTEGER NOT NULL,                  -- OCSF time (epoch ms UTC)
  device            TEXT    NOT NULL,                  -- fw01 | ids01 | vpn01 | br01
  vendor            TEXT    NOT NULL,                  -- Cisco | OISF | Fortinet | Acme
  product           TEXT    NOT NULL,                  -- ASA | Suricata | FortiGate | EdgeRouter
  format            TEXT    NOT NULL,                  -- cisco-asa-syslog | suricata-eve-json | fortigate-kv | acme-edge-pipe
  class_uid         INTEGER NOT NULL CHECK (class_uid IN (4001,2004,3002)),
  class_name        TEXT    NOT NULL,
  activity_name     TEXT    NOT NULL,                  -- Open | Close | Refuse | Traffic | Create | Logon
  src_ip            TEXT,                              -- 4001/3002: src_endpoint.ip · 2004: evidences[0].src_endpoint.ip
  src_port          INTEGER,
  dst_ip            TEXT,
  dst_port          INTEGER,
  protocol          TEXT,                              -- connection_info.protocol_name (lower case)
  direction         TEXT,                              -- Inbound | Outbound | NULL
  user_name         TEXT,
  action            TEXT CHECK (action IN ('Allowed','Denied') OR action IS NULL),
  status            TEXT CHECK (status IN ('Success','Failure') OR status IS NULL),
  severity          TEXT    NOT NULL,                  -- Informational | Low | Medium | High | Critical
  title             TEXT,                              -- 2004: finding_info.title · else NULL
  parse_confidence  INTEGER NOT NULL CHECK (parse_confidence BETWEEN 0 AND 100),
  parser            TEXT    NOT NULL,                  -- 'cisco-asa@1.0.0'
  rule_id           TEXT    NOT NULL,
  roundtrip_ok      INTEGER NOT NULL CHECK (roundtrip_ok IN (0,1)),
  roundtrip_coverage REAL   NOT NULL,
  ocsf_json         TEXT    NOT NULL,                  -- the OCSF object (§4)
  anvaya_json       TEXT    NOT NULL                   -- the anvaya object (§4)
);
CREATE INDEX ix_events_time   ON events(time DESC);
CREATE INDEX ix_events_src    ON events(src_ip);
CREATE INDEX ix_events_dst    ON events(dst_ip);
CREATE INDEX ix_events_device ON events(device, time DESC);
CREATE INDEX ix_events_class  ON events(class_uid, status, action);
```

NL search (TRD.md §12) may read only these columns: `time, device, vendor, class_uid, class_name, src_ip, src_port, dst_ip, dst_port, user_name, action, status, severity, title`.

### 2.3 `ledger_batches`

```sql
CREATE TABLE ledger_batches (
  batch_id     INTEGER PRIMARY KEY,                    -- 1, 2, 3 …
  first_seq    INTEGER NOT NULL,
  last_seq     INTEGER NOT NULL,
  leaf_count   INTEGER NOT NULL CHECK (leaf_count BETWEEN 1 AND 16),
  merkle_root  TEXT    NOT NULL,
  prev_head    TEXT    NOT NULL,                       -- 64 zeros for batch 1
  head         TEXT    NOT NULL,
  signature    TEXT    NOT NULL,                       -- Ed25519 over head
  pubkey_fp    TEXT    NOT NULL,                       -- primary key fingerprint (16 hex)
  sealed_at    INTEGER NOT NULL,
  resealed     INTEGER NOT NULL DEFAULT 0 CHECK (resealed IN (0,1)),   -- set by demo reseal only
  CHECK (last_seq >= first_seq)
);
```

### 2.4 `witness_signatures` (primary-side record of co-signatures)

```sql
CREATE TABLE witness_signatures (
  batch_id     INTEGER NOT NULL REFERENCES ledger_batches(batch_id),
  witness      TEXT    NOT NULL CHECK (witness IN ('witness-ntro','witness-certin')),
  status       TEXT    NOT NULL CHECK (status IN ('cosigned','rejected')),
  head         TEXT    NOT NULL,                       -- head as sent at co-sign time
  cosignature  TEXT,                                   -- NULL if rejected
  reason       TEXT,                                   -- rejection reason
  cosigned_at  INTEGER NOT NULL,
  PRIMARY KEY (batch_id, witness)
);
```

Each witness also keeps its own store, `/data/witness/{witness}.db`, with table `heads(batch_id INTEGER PRIMARY KEY, head TEXT NOT NULL, prev_head TEXT NOT NULL, primary_sig TEXT NOT NULL, cosig TEXT NOT NULL, received_at INTEGER NOT NULL)`. Verification compares the primary `ledger_batches.head` with **the witness's own store**, not with `witness_signatures`.

### 2.5 `sources`

```sql
CREATE TABLE sources (
  device          TEXT    PRIMARY KEY,
  vendor          TEXT    NOT NULL,
  product         TEXT    NOT NULL,
  format          TEXT    NOT NULL,
  parser          TEXT    NOT NULL,
  first_seen      INTEGER NOT NULL,                    -- Clock ms
  last_seen       INTEGER NOT NULL,                    -- Clock ms
  first_seen_wall INTEGER NOT NULL,                    -- real ms
  last_seen_wall  INTEGER NOT NULL,                    -- real ms
  events_total    INTEGER NOT NULL DEFAULT 0,
  expected_gap_s  REAL,                                -- median of the last 20 real gaps
  status          TEXT    NOT NULL DEFAULT 'learning' CHECK (status IN ('learning','active','silent'))
);
```

The silence monitor keeps its gap ring buffer in memory and persists only `expected_gap_s` and `status`.

### 2.6 `parsers` and `parser_drafts`

```sql
CREATE TABLE parsers (
  name          TEXT NOT NULL,
  version       TEXT NOT NULL,
  status        TEXT NOT NULL CHECK (status IN ('active','retired')),
  origin        TEXT NOT NULL CHECK (origin IN ('builtin','ai_draft','user')),
  format        TEXT NOT NULL,
  priority      INTEGER NOT NULL,
  yaml          TEXT NOT NULL,
  yaml_sha256   TEXT NOT NULL,
  approved_by   TEXT,
  approved_at   INTEGER,
  PRIMARY KEY (name, version)
);

CREATE TABLE parser_drafts (
  draft_id         TEXT PRIMARY KEY,
  group_id         TEXT NOT NULL,
  status           TEXT NOT NULL CHECK (status IN ('drafted','tested','approved','rejected')),
  origin           TEXT NOT NULL CHECK (origin IN ('llm_live','llm_cached','hand_authored')),
  model            TEXT,
  generated_at     INTEGER,
  yaml             TEXT NOT NULL,
  holdout_indexes  TEXT NOT NULL,                      -- JSON array, e.g. [2,5,8,11,14,17]
  test_result      TEXT,                               -- JSON (TRD.md API 17)
  promotable       INTEGER NOT NULL DEFAULT 0 CHECK (promotable IN (0,1)),
  decided_by       TEXT,
  decided_at       INTEGER,
  reject_reason    TEXT,
  created_at       INTEGER NOT NULL,
  UNIQUE (group_id, status)                            -- one open draft per group in practice
);
```

### 2.7 `quarantine`

```sql
CREATE TABLE quarantine (
  raw_id          TEXT PRIMARY KEY REFERENCES raw_events(raw_id),
  group_id        TEXT NOT NULL,
  shape_signature TEXT NOT NULL,                       -- 'EDGE|x12'
  reason          TEXT NOT NULL CHECK (reason IN ('no_fingerprint','no_rule','parse_error','schema_invalid','roundtrip_fail')),
  detail          TEXT,
  first_seen      INTEGER NOT NULL
);
CREATE INDEX ix_quarantine_group ON quarantine(group_id);
```

Rows are deleted when a replay parses the record successfully. `roundtrip_fail` rows coexist with an `events` row (the event is stored but flagged).

### 2.8 `incidents` and `incident_events`

```sql
CREATE TABLE incidents (
  incident_id    TEXT PRIMARY KEY,
  attacker_ip    TEXT NOT NULL,
  target_ip      TEXT,
  window_start   INTEGER NOT NULL,
  title          TEXT NOT NULL,
  stages         TEXT NOT NULL,                        -- JSON array in order: ["recon","credential","access"]
  devices        TEXT NOT NULL,                        -- JSON array
  risk           INTEGER NOT NULL CHECK (risk BETWEEN 0 AND 100),
  level          TEXT NOT NULL CHECK (level IN ('Low','Medium','High')),
  risk_breakdown TEXT NOT NULL,                        -- JSON {"stages":75,"devices":10,"ordered":10}
  first_time     INTEGER NOT NULL,
  last_time      INTEGER NOT NULL,
  status         TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','closed')),
  UNIQUE (attacker_ip, window_start)
);

CREATE TABLE incident_events (
  incident_id TEXT NOT NULL REFERENCES incidents(incident_id),
  event_id    TEXT NOT NULL REFERENCES events(event_id),
  stage       TEXT NOT NULL CHECK (stage IN ('recon','credential','access')),
  PRIMARY KEY (incident_id, event_id)
);
```

### 2.9 `audit_log` and `meta`

```sql
CREATE TABLE audit_log (
  id      INTEGER PRIMARY KEY AUTOINCREMENT,
  at      INTEGER NOT NULL,
  actor   TEXT    NOT NULL,                            -- 'system' | 'presenter' | 'analyst@ntro-demo'
  action  TEXT    NOT NULL,                            -- 'parser.approve' | 'parser.reject' | 'search.ask' | 'demo.tamper' | 'demo.reseal' | 'demo.reset' | 'export.parquet'
  object  TEXT,
  detail  TEXT    NOT NULL DEFAULT '{}'
);
CREATE TRIGGER audit_no_update BEFORE UPDATE ON audit_log BEGIN SELECT RAISE(ABORT,'audit_log is append-only'); END;
CREATE TRIGGER audit_no_delete BEFORE DELETE ON audit_log BEGIN SELECT RAISE(ABORT,'audit_log is append-only'); END;

CREATE TABLE meta (
  key   TEXT PRIMARY KEY,                              -- 'last_verify' | 'ocsf_version' | 'schema_version' | 'demo_state'
  value TEXT NOT NULL
);
```

`demo reset` deletes the database file rather than updating `audit_log`, so the triggers never block a reset.

## 3. Data directory and vault layout

```
/data
├── anvaya.db  anvaya.db-wal  anvaya.db-shm
├── vault/
│   ├── seg-000001.log            # append-only; rotates at 64 MiB
│   └── seg-000002.log
├── keys/                         # mode 0700; files 0600; kept on demo reset
│   ├── primary.ed25519           # 32-byte raw seed
│   ├── witness-ntro.ed25519
│   └── witness-certin.ed25519
├── witness/
│   ├── witness-ntro.db
│   └── witness-certin.db
├── packs/                        # approved non-builtin packs (acme-edge.yaml after SCN-04)
├── outbox/siem-forward.ndjson    # SIEM file sink
└── lake/events-<from>-<to>.parquet
```

**Segment record framing:** `[u32 big-endian length N][N bytes exactly as received]`, back to back, with no separators. `raw_events.offset` points at the length prefix, so the payload starts at `offset + 4`. Reading `read(raw_id)` = seek, read 4, read N. Rotation starts a new segment when `current_size + 4 + N > 64 MiB`.

## 4. Formats: sample lines, packs and complete OCSF envelopes

`GET /api/events/{event_id}` returns this envelope: `{event_id, raw, ocsf, anvaya}`. `ocsf` is the pure OCSF object (validated against the vendored schema). `anvaya` holds ANVAYA's own metadata.

> **Schema note:** examples were authored against OCSF 1.3.0. At build time the vendored schema version is written to `metadata.version`. Field names marked "verify" in TRD.md §7.3 are checked by `test_ocsf_validation.py`; if the vendored schema disagrees, follow the schema and update the golden files.

### 4.1 Cisco ASA syslog (fw01): scripted access event, seq 164

**Raw line** (174 bytes, UTF-8, no trailing newline):

```
<166>Sep 30 2026 10:07:12 fw01 : %ASA-6-302013: Built inbound TCP connection 77801 for outside:198.51.100.9/40133 (198.51.100.9/40133) to inside:10.2.4.18/443 (10.2.4.18/443)
```

`raw_sha256` = `d12173d4509247b2d5e571a3922472f9262482d1ef617ef33b6ba152f74f4251`, `raw_id` = `r-899f4edf0030`, batch 11.

**Complete envelope:**

```json
{
  "event_id": "e-899f4edf0030",
  "raw": {
    "raw_id": "r-899f4edf0030", "seq": 164, "size": 174, "encoding": "utf-8", "flags": [],
    "transport": "udp", "peer_ip": "127.0.0.1", "received_at": 1790743032000,
    "raw_sha256": "d12173d4509247b2d5e571a3922472f9262482d1ef617ef33b6ba152f74f4251", "batch_id": 11,
    "text": "<166>Sep 30 2026 10:07:12 fw01 : %ASA-6-302013: Built inbound TCP connection 77801 for outside:198.51.100.9/40133 (198.51.100.9/40133) to inside:10.2.4.18/443 (10.2.4.18/443)"
  },
  "ocsf": {
    "class_uid": 4001, "class_name": "Network Activity",
    "category_uid": 4, "category_name": "Network Activity",
    "activity_id": 1, "activity_name": "Open",
    "type_uid": 400101, "type_name": "Network Activity: Open",
    "severity_id": 1, "severity": "Informational",
    "action_id": 1, "action": "Allowed",
    "disposition_id": 1, "disposition": "Allowed",
    "time": 1790743032000, "time_dt": "2026-09-30T04:37:12Z",
    "device": { "hostname": "fw01" },
    "connection_info": { "protocol_name": "tcp", "direction_id": 1, "direction": "Inbound", "uid": "77801" },
    "src_endpoint": { "ip": "198.51.100.9", "port": 40133 },
    "dst_endpoint": { "ip": "10.2.4.18", "port": 443 },
    "metadata": {
      "version": "1.3.0", "uid": "e-899f4edf0030", "log_name": "syslog",
      "original_time": "Sep 30 2026 10:07:12",
      "product": { "vendor_name": "Cisco", "name": "ASA" }
    },
    "observables": [
      { "name": "src_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "198.51.100.9" },
      { "name": "dst_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "10.2.4.18" }
    ],
    "raw_data": "<166>Sep 30 2026 10:07:12 fw01 : %ASA-6-302013: Built inbound TCP connection 77801 for outside:198.51.100.9/40133 (198.51.100.9/40133) to inside:10.2.4.18/443 (10.2.4.18/443)",
    "unmapped": {
      "syslog_pri": "166", "asa_severity": "6", "asa_message_id": "302013",
      "src_zone": "outside", "dst_zone": "inside",
      "src_mapped": "198.51.100.9/40133", "dst_mapped": "10.2.4.18/443"
    }
  },
  "anvaya": {
    "format": "cisco-asa-syslog", "vendor": "Cisco", "product": "ASA", "device": "fw01",
    "parser": "cisco-asa@1.0.0", "rule_id": "asa-302013-inbound",
    "parse_confidence": 91, "confidence_breakdown": { "required": 1.0, "typed": 1.0, "mapped": 0.5625 },
    "source_timezone": "Asia/Kolkata", "device_time_raw": "Sep 30 2026 10:07:12", "clock_skew_s": 0,
    "roundtrip": { "ok": true, "value_check": true, "coverage": 1.0, "dropped": 0 },
    "lineage": [
      { "path": "time", "spans": [[5, 25]], "derived": true },
      { "path": "metadata.original_time", "spans": [[5, 25]], "derived": false },
      { "path": "device.hostname", "spans": [[26, 30]], "derived": false },
      { "path": "activity_name", "spans": [[40, 46]], "derived": true },
      { "path": "action", "spans": [[40, 46]], "derived": true },
      { "path": "disposition", "spans": [[40, 46]], "derived": true },
      { "path": "severity", "spans": [[40, 46]], "derived": true },
      { "path": "connection_info.direction", "spans": [[54, 61]], "derived": true },
      { "path": "connection_info.protocol_name", "spans": [[62, 65]], "derived": true },
      { "path": "connection_info.uid", "spans": [[77, 82]], "derived": false },
      { "path": "src_endpoint.ip", "spans": [[95, 107]], "derived": false },
      { "path": "src_endpoint.port", "spans": [[108, 113]], "derived": false },
      { "path": "dst_endpoint.ip", "spans": [[145, 154]], "derived": false },
      { "path": "dst_endpoint.port", "spans": [[155, 158]], "derived": false },
      { "path": "unmapped.syslog_pri", "spans": [[1, 4]], "derived": false },
      { "path": "unmapped.asa_severity", "spans": [[38, 39]], "derived": false },
      { "path": "unmapped.asa_message_id", "spans": [[40, 46]], "derived": false },
      { "path": "unmapped.src_zone", "spans": [[87, 94]], "derived": false },
      { "path": "unmapped.src_mapped", "spans": [[115, 133]], "derived": false },
      { "path": "unmapped.dst_zone", "spans": [[138, 144]], "derived": false },
      { "path": "unmapped.dst_mapped", "spans": [[160, 173]], "derived": false }
    ]
  }
}
```

The tamper target byte is offset **153** (the final `8` of `10.2.4.18`, inside `dst_endpoint.ip` span 145–154).

### 4.2 Suricata EVE JSON (ids01): first scripted scan alert, seq 40

**Raw line** (422 bytes): EVE JSON carried in a BSD syslog wrapper.

```
<134>Sep 30 10:02:05 ids01 suricata[2211]: {"timestamp":"2026-09-30T10:02:05.000000+0530","flow_id":1500000000000900,"in_iface":"eth1","event_type":"alert","src_ip":"198.51.100.9","src_port":40100,"dest_ip":"10.2.4.18","dest_port":22,"proto":"TCP","alert":{"action":"allowed","gid":1,"signature_id":9000001,"rev":1,"signature":"ANVAYA-LAB SCAN Inbound TCP port sweep","category":"Attempted Information Leak","severity":2}}
```

`raw_sha256` = `9411734615b92fb98885e675ad1412802146e7efce2f550290c4eff32b479b0d`, `raw_id` = `r-64f70aaee53f`, batch 3.

**Pack** `backend/app/parsers/packs/suricata-eve.yaml`:

```yaml
name: suricata-eve
version: 1.0.0
origin: builtin
priority: 10
vendor: OISF
product: Suricata
format: suricata-eve-json
source_timezone: Asia/Kolkata
detect:
  header: bsd3164
  header_app: suricata
  body_json_has: [event_type]
header_fields:
  pri: { to: unmapped.syslog_pri }
  ts: { to: unmapped.syslog_time }
  host: { to: device.hostname }
  app: { to: unmapped.app }
  pid: { to: unmapped.pid }
extractor: { type: json }
rules:
  - id: eve-alert
    when: { json_equals: { event_type: alert } }
    ocsf: { class_uid: 2004, activity: Create }
    map:
      timestamp: { to: time, transform: { timestamp: '%Y-%m-%dT%H:%M:%S.%f%z' }, also: metadata.original_time }
      src_ip: { to: 'evidences[0].src_endpoint.ip', type: ip }
      src_port: { to: 'evidences[0].src_endpoint.port', type: int }
      dest_ip: { to: 'evidences[0].dst_endpoint.ip', type: ip }
      dest_port: { to: 'evidences[0].dst_endpoint.port', type: int }
      proto: { to: 'evidences[0].connection_info.protocol_name', transform: lower }
      alert.signature_id: { to: finding_info.uid }
      alert.signature: { to: finding_info.title }
      alert.category: { to: 'finding_info.types[0]' }
      alert.severity: { to: severity, transform: { enum: { '1': High, '2': Medium, '3': Low } } }
      flow_id: { to: unmapped.flow_id }
      in_iface: { to: unmapped.in_iface }
      event_type: { to: unmapped.event_type }
      alert.action: { to: unmapped.alert.action }
      alert.gid: { to: unmapped.alert.gid }
      alert.rev: { to: unmapped.alert.rev }
required: [time, finding_info.uid, finding_info.title]
tests:
  - input: '<134>Sep 30 10:02:05 ids01 suricata[2211]: {"timestamp":"2026-09-30T10:02:05.000000+0530","flow_id":1500000000000900,"in_iface":"eth1","event_type":"alert","src_ip":"198.51.100.9","src_port":40100,"dest_ip":"10.2.4.18","dest_port":22,"proto":"TCP","alert":{"action":"allowed","gid":1,"signature_id":9000001,"rev":1,"signature":"ANVAYA-LAB SCAN Inbound TCP port sweep","category":"Attempted Information Leak","severity":2}}'
    expect: { class_uid: 2004, severity: Medium, finding_info.uid: '9000001', 'evidences[0].dst_endpoint.port': 22 }
```

**Complete envelope:**

```json
{
  "event_id": "e-64f70aaee53f",
  "raw": {
    "raw_id": "r-64f70aaee53f", "seq": 40, "size": 422, "encoding": "utf-8", "flags": [],
    "transport": "udp", "peer_ip": "127.0.0.1", "received_at": 1790742725000,
    "raw_sha256": "9411734615b92fb98885e675ad1412802146e7efce2f550290c4eff32b479b0d", "batch_id": 3,
    "text": "<134>Sep 30 10:02:05 ids01 suricata[2211]: {\"timestamp\":\"2026-09-30T10:02:05.000000+0530\",\"flow_id\":1500000000000900,\"in_iface\":\"eth1\",\"event_type\":\"alert\",\"src_ip\":\"198.51.100.9\",\"src_port\":40100,\"dest_ip\":\"10.2.4.18\",\"dest_port\":22,\"proto\":\"TCP\",\"alert\":{\"action\":\"allowed\",\"gid\":1,\"signature_id\":9000001,\"rev\":1,\"signature\":\"ANVAYA-LAB SCAN Inbound TCP port sweep\",\"category\":\"Attempted Information Leak\",\"severity\":2}}"
  },
  "ocsf": {
    "class_uid": 2004, "class_name": "Detection Finding",
    "category_uid": 2, "category_name": "Findings",
    "activity_id": 1, "activity_name": "Create",
    "type_uid": 200401, "type_name": "Detection Finding: Create",
    "severity_id": 3, "severity": "Medium",
    "time": 1790742725000, "time_dt": "2026-09-30T04:32:05Z",
    "device": { "hostname": "ids01" },
    "finding_info": { "uid": "9000001", "title": "ANVAYA-LAB SCAN Inbound TCP port sweep", "types": ["Attempted Information Leak"] },
    "evidences": [
      {
        "src_endpoint": { "ip": "198.51.100.9", "port": 40100 },
        "dst_endpoint": { "ip": "10.2.4.18", "port": 22 },
        "connection_info": { "protocol_name": "tcp" }
      }
    ],
    "metadata": {
      "version": "1.3.0", "uid": "e-64f70aaee53f", "log_name": "eve-json",
      "original_time": "2026-09-30T10:02:05.000000+0530",
      "product": { "vendor_name": "OISF", "name": "Suricata" }
    },
    "observables": [
      { "name": "evidences[0].src_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "198.51.100.9" },
      { "name": "evidences[0].dst_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "10.2.4.18" }
    ],
    "raw_data": "<134>Sep 30 10:02:05 ids01 suricata[2211]: {\"timestamp\":\"2026-09-30T10:02:05.000000+0530\",\"flow_id\":1500000000000900,\"in_iface\":\"eth1\",\"event_type\":\"alert\",\"src_ip\":\"198.51.100.9\",\"src_port\":40100,\"dest_ip\":\"10.2.4.18\",\"dest_port\":22,\"proto\":\"TCP\",\"alert\":{\"action\":\"allowed\",\"gid\":1,\"signature_id\":9000001,\"rev\":1,\"signature\":\"ANVAYA-LAB SCAN Inbound TCP port sweep\",\"category\":\"Attempted Information Leak\",\"severity\":2}}",
    "unmapped": {
      "syslog_pri": "134", "syslog_time": "Sep 30 10:02:05", "app": "suricata", "pid": "2211",
      "flow_id": "1500000000000900", "in_iface": "eth1", "event_type": "alert",
      "alert": { "action": "allowed", "gid": "1", "rev": "1" }
    }
  },
  "anvaya": {
    "format": "suricata-eve-json", "vendor": "OISF", "product": "Suricata", "device": "ids01",
    "parser": "suricata-eve@1.0.0", "rule_id": "eve-alert",
    "parse_confidence": 90, "confidence_breakdown": { "required": 1.0, "typed": 1.0, "mapped": 0.5238 },
    "source_timezone": "Asia/Kolkata", "device_time_raw": "2026-09-30T10:02:05.000000+0530", "clock_skew_s": 0,
    "roundtrip": { "ok": true, "value_check": true, "coverage": 1.0, "dropped": 0 },
    "lineage": [
      { "path": "time", "spans": [[57, 88]], "derived": true },
      { "path": "metadata.original_time", "spans": [[57, 88]], "derived": false },
      { "path": "device.hostname", "spans": [[21, 26]], "derived": false },
      { "path": "evidences[0].src_endpoint.ip", "spans": [[166, 178]], "derived": false },
      { "path": "evidences[0].src_endpoint.port", "spans": [[191, 196]], "derived": false },
      { "path": "evidences[0].dst_endpoint.ip", "spans": [[208, 217]], "derived": false },
      { "path": "evidences[0].dst_endpoint.port", "spans": [[231, 233]], "derived": false },
      { "path": "evidences[0].connection_info.protocol_name", "spans": [[243, 246]], "derived": true },
      { "path": "finding_info.uid", "spans": [[299, 306]], "derived": false },
      { "path": "finding_info.title", "spans": [[328, 366]], "derived": false },
      { "path": "finding_info.types[0]", "spans": [[380, 406]], "derived": false },
      { "path": "severity", "spans": [[419, 420]], "derived": true },
      { "path": "activity_name", "spans": [[149, 154]], "derived": true },
      { "path": "unmapped.syslog_pri", "spans": [[1, 4]], "derived": false },
      { "path": "unmapped.syslog_time", "spans": [[5, 20]], "derived": false },
      { "path": "unmapped.app", "spans": [[27, 35]], "derived": false },
      { "path": "unmapped.pid", "spans": [[36, 40]], "derived": false },
      { "path": "unmapped.flow_id", "spans": [[100, 116]], "derived": false },
      { "path": "unmapped.in_iface", "spans": [[129, 133]], "derived": false },
      { "path": "unmapped.event_type", "spans": [[149, 154]], "derived": false },
      { "path": "unmapped.alert.action", "spans": [[267, 274]], "derived": false },
      { "path": "unmapped.alert.gid", "spans": [[282, 283]], "derived": false },
      { "path": "unmapped.alert.rev", "spans": [[313, 314]], "derived": false }
    ]
  }
}
```

The flat `events` row stores `src_ip = 198.51.100.9`, `dst_ip = 10.2.4.18`, `dst_port = 22`, `protocol = 'tcp'`, `title = 'ANVAYA-LAB SCAN Inbound TCP port sweep'`, `severity = 'Medium'`, `action = NULL`.

### 4.3 FortiGate key=value (vpn01): first scripted login failure, seq 100

**Raw line** (323 bytes):

```
<185>date=2026-09-30 time=10:05:00 devname="vpn01" devid="FGVM01TM00000001" logid="0101039426" type="event" subtype="vpn" level="alert" vd="root" logdesc="SSL VPN login fail" action="ssl-login-fail" tunneltype="ssl-web" remip=198.51.100.9 user="admin" reason="sslvpn_login_permission_denied" msg="SSL VPN user login failed"
```

`raw_sha256` = `632ad16d39e829111ce7101d17d716afebe6773d675a2498f66c64abf93b7f84`, `raw_id` = `r-3db59309ddaa`, batch 7. The line is FortiGate-style synthetic data: field names follow the vendor's key=value convention; the `devid` and `logid` values are invented for the lab.

**Pack** `backend/app/parsers/packs/fortigate-kv.yaml`:

```yaml
name: fortigate-kv
version: 1.0.0
origin: builtin
priority: 20
vendor: Fortinet
product: FortiGate
format: fortigate-kv
source_timezone: Asia/Kolkata
detect:
  header: pri_only
  body_regex: '^date=\d{4}-\d{2}-\d{2} time=\d{2}:\d{2}:\d{2} '
  body_contains: [' devname=', ' logid=']
header_fields:
  pri: { to: unmapped.syslog_pri }
extractor: { type: kv }
rules:
  - id: fgt-ssl-login-fail
    when: { kv_equals: { subtype: vpn, action: ssl-login-fail } }
    ocsf: { class_uid: 3002, activity: Logon, status: Failure, severity: Medium }
    map:
      date+time: { to: time, transform: { timestamp: '%Y-%m-%d %H:%M:%S' }, also: metadata.original_time }
      devname: { to: device.hostname }
      remip: { to: src_endpoint.ip, type: ip }
      user: { to: user.name }
      action: { to: status, transform: { enum: { ssl-login-fail: Failure } } }
      reason: { to: status_detail }
      msg: { to: message }
      devid: { to: unmapped.devid }
      logid: { to: unmapped.logid }
      type: { to: unmapped.type }
      subtype: { to: unmapped.subtype }
      level: { to: unmapped.level }
      vd: { to: unmapped.vd }
      logdesc: { to: unmapped.logdesc }
      tunneltype: { to: unmapped.tunneltype }
  - id: fgt-tunnel-up
    when: { kv_equals: { subtype: vpn, action: tunnel-up } }
    ocsf: { class_uid: 3002, activity: Logon, status: Success, severity: Informational }
    map:
      date+time: { to: time, transform: { timestamp: '%Y-%m-%d %H:%M:%S' }, also: metadata.original_time }
      devname: { to: device.hostname }
      remip: { to: src_endpoint.ip, type: ip }
      user: { to: user.name }
      action: { to: status, transform: { enum: { tunnel-up: Success } } }
      reason: { to: status_detail }
      msg: { to: message }
      devid: { to: unmapped.devid }
      logid: { to: unmapped.logid }
      type: { to: unmapped.type }
      subtype: { to: unmapped.subtype }
      level: { to: unmapped.level }
      vd: { to: unmapped.vd }
      logdesc: { to: unmapped.logdesc }
      tunneltype: { to: unmapped.tunneltype }
required: [time, user.name, src_endpoint.ip]
tests:
  - input: '<185>date=2026-09-30 time=10:05:00 devname="vpn01" devid="FGVM01TM00000001" logid="0101039426" type="event" subtype="vpn" level="alert" vd="root" logdesc="SSL VPN login fail" action="ssl-login-fail" tunneltype="ssl-web" remip=198.51.100.9 user="admin" reason="sslvpn_login_permission_denied" msg="SSL VPN user login failed"'
    expect: { class_uid: 3002, status: Failure, user.name: admin, src_endpoint.ip: 198.51.100.9 }
```

`date+time` means two extracted values combined into one target. The lineage gets both spans and `derived: true`; both count as mapped values.

**Complete envelope:**

```json
{
  "event_id": "e-3db59309ddaa",
  "raw": {
    "raw_id": "r-3db59309ddaa", "seq": 100, "size": 323, "encoding": "utf-8", "flags": [],
    "transport": "udp", "peer_ip": "127.0.0.1", "received_at": 1790742900000,
    "raw_sha256": "632ad16d39e829111ce7101d17d716afebe6773d675a2498f66c64abf93b7f84", "batch_id": 7,
    "text": "<185>date=2026-09-30 time=10:05:00 devname=\"vpn01\" devid=\"FGVM01TM00000001\" logid=\"0101039426\" type=\"event\" subtype=\"vpn\" level=\"alert\" vd=\"root\" logdesc=\"SSL VPN login fail\" action=\"ssl-login-fail\" tunneltype=\"ssl-web\" remip=198.51.100.9 user=\"admin\" reason=\"sslvpn_login_permission_denied\" msg=\"SSL VPN user login failed\""
  },
  "ocsf": {
    "class_uid": 3002, "class_name": "Authentication",
    "category_uid": 3, "category_name": "Identity & Access Management",
    "activity_id": 1, "activity_name": "Logon",
    "type_uid": 300201, "type_name": "Authentication: Logon",
    "severity_id": 3, "severity": "Medium",
    "status_id": 2, "status": "Failure", "status_detail": "sslvpn_login_permission_denied",
    "message": "SSL VPN user login failed",
    "time": 1790742900000, "time_dt": "2026-09-30T04:35:00Z",
    "device": { "hostname": "vpn01" },
    "user": { "name": "admin" },
    "src_endpoint": { "ip": "198.51.100.9" },
    "metadata": {
      "version": "1.3.0", "uid": "e-3db59309ddaa", "log_name": "fortigate-kv",
      "original_time": "2026-09-30 10:05:00",
      "product": { "vendor_name": "Fortinet", "name": "FortiGate" }
    },
    "observables": [
      { "name": "src_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "198.51.100.9" },
      { "name": "user.name", "type_id": 4, "type": "User Name", "value": "admin" }
    ],
    "raw_data": "<185>date=2026-09-30 time=10:05:00 devname=\"vpn01\" devid=\"FGVM01TM00000001\" logid=\"0101039426\" type=\"event\" subtype=\"vpn\" level=\"alert\" vd=\"root\" logdesc=\"SSL VPN login fail\" action=\"ssl-login-fail\" tunneltype=\"ssl-web\" remip=198.51.100.9 user=\"admin\" reason=\"sslvpn_login_permission_denied\" msg=\"SSL VPN user login failed\"",
    "unmapped": {
      "syslog_pri": "185", "devid": "FGVM01TM00000001", "logid": "0101039426", "type": "event",
      "subtype": "vpn", "level": "alert", "vd": "root", "logdesc": "SSL VPN login fail", "tunneltype": "ssl-web"
    }
  },
  "anvaya": {
    "format": "fortigate-kv", "vendor": "Fortinet", "product": "FortiGate", "device": "vpn01",
    "parser": "fortigate-kv@1.0.0", "rule_id": "fgt-ssl-login-fail",
    "parse_confidence": 89, "confidence_breakdown": { "required": 1.0, "typed": 1.0, "mapped": 0.4706 },
    "source_timezone": "Asia/Kolkata", "device_time_raw": "2026-09-30 10:05:00", "clock_skew_s": 0,
    "roundtrip": { "ok": true, "value_check": true, "coverage": 1.0, "dropped": 0 },
    "lineage": [
      { "path": "time", "spans": [[10, 20], [26, 34]], "derived": true },
      { "path": "metadata.original_time", "spans": [[10, 20], [26, 34]], "derived": true },
      { "path": "device.hostname", "spans": [[44, 49]], "derived": false },
      { "path": "status", "spans": [[183, 197]], "derived": true },
      { "path": "activity_name", "spans": [[183, 197]], "derived": true },
      { "path": "severity", "spans": [[183, 197]], "derived": true },
      { "path": "src_endpoint.ip", "spans": [[226, 238]], "derived": false },
      { "path": "user.name", "spans": [[245, 250]], "derived": false },
      { "path": "status_detail", "spans": [[260, 290]], "derived": false },
      { "path": "message", "spans": [[297, 322]], "derived": false },
      { "path": "unmapped.syslog_pri", "spans": [[1, 4]], "derived": false },
      { "path": "unmapped.devid", "spans": [[58, 74]], "derived": false },
      { "path": "unmapped.logid", "spans": [[83, 93]], "derived": false },
      { "path": "unmapped.type", "spans": [[101, 106]], "derived": false },
      { "path": "unmapped.subtype", "spans": [[117, 120]], "derived": false },
      { "path": "unmapped.level", "spans": [[129, 134]], "derived": false },
      { "path": "unmapped.vd", "spans": [[140, 144]], "derived": false },
      { "path": "unmapped.logdesc", "spans": [[155, 173]], "derived": false },
      { "path": "unmapped.tunneltype", "spans": [[211, 218]], "derived": false }
    ]
  }
}
```

### 4.4 Acme EdgeRouter pipe format (br01): the unknown format for SCN-04

Acme is a fictional vendor. Before approval these lines are quarantined (`no_fingerprint`, shape `EDGE|x12`). After approval they parse with the pack below.

**Raw line**, index 0 of the 20 injected (102 bytes). The example timestamp assumes injection at scenario 10:14:03:

```
EDGE|2026-09-30 10:13:06|br01|SESSION|ALLOW|tcp|10.2.4.31|52300|203.0.113.12|443|bytes=1200|rule=R-110
```

`raw_sha256` = `4f29c17441c6daeac27707f2a196a77a710375b1982f87c049499a091ba1738a`. The `raw_id` depends on the seq at injection time (for seq 1342 it would be `r-b487689bfcd6`).

**Pack** `/data/packs/acme-edge.yaml` (the content of the cached draft; approved as 1.0.0):

```yaml
name: acme-edge
version: 1.0.0
origin: ai_draft
priority: 40
vendor: Acme
product: EdgeRouter
format: acme-edge-pipe
source_timezone: Asia/Kolkata
detect:
  header: none
  body_regex: '^EDGE\|'
  delimited_field_count: { delimiter: '|', count: 12 }
extractor:
  type: delimited
  delimiter: '|'
  columns: [marker, ts, host, record_type, verdict, proto, src_ip, src_port, dst_ip, dst_port, bytes_kv, rule_kv]
rules:
  - id: edge-session-allow
    when: { column_equals: { record_type: SESSION, verdict: ALLOW } }
    ocsf: { class_uid: 4001, activity: Traffic, action: Allowed, disposition: Allowed, severity: Informational }
    map: &edge_map
      ts: { to: time, transform: { timestamp: '%Y-%m-%d %H:%M:%S' }, also: metadata.original_time }
      host: { to: device.hostname }
      verdict: { to: action, transform: { enum: { ALLOW: Allowed, DENY: Denied } } }
      proto: { to: connection_info.protocol_name, transform: lower }
      src_ip: { to: src_endpoint.ip, type: ip }
      src_port: { to: src_endpoint.port, type: int }
      dst_ip: { to: dst_endpoint.ip, type: ip }
      dst_port: { to: dst_endpoint.port, type: int }
      bytes_kv: { to: traffic.bytes, type: int, transform: { kv_value: true } }
      marker: { to: unmapped.marker }
      record_type: { to: unmapped.record_type }
      rule_kv: { to: unmapped.rule, transform: { kv_value: true } }
  - id: edge-session-deny
    when: { column_equals: { record_type: SESSION, verdict: DENY } }
    ocsf: { class_uid: 4001, activity: Refuse, action: Denied, disposition: Blocked, severity: Low }
    map: *edge_map
required: [time, src_endpoint.ip, dst_endpoint.ip]
tests:
  - input: 'EDGE|2026-09-30 10:13:06|br01|SESSION|ALLOW|tcp|10.2.4.31|52300|203.0.113.12|443|bytes=1200|rule=R-110'
    expect: { class_uid: 4001, action: Allowed, dst_endpoint.port: 443, traffic.bytes: 1200 }
```

**Complete envelope** (seq 1342 used for illustration):

```json
{
  "event_id": "e-b487689bfcd6",
  "raw": {
    "raw_id": "r-b487689bfcd6", "seq": 1342, "size": 102, "encoding": "utf-8", "flags": [],
    "transport": "http", "peer_ip": "127.0.0.1", "received_at": 1790743443000,
    "raw_sha256": "4f29c17441c6daeac27707f2a196a77a710375b1982f87c049499a091ba1738a", "batch_id": 84,
    "text": "EDGE|2026-09-30 10:13:06|br01|SESSION|ALLOW|tcp|10.2.4.31|52300|203.0.113.12|443|bytes=1200|rule=R-110"
  },
  "ocsf": {
    "class_uid": 4001, "class_name": "Network Activity",
    "category_uid": 4, "category_name": "Network Activity",
    "activity_id": 6, "activity_name": "Traffic",
    "type_uid": 400106, "type_name": "Network Activity: Traffic",
    "severity_id": 1, "severity": "Informational",
    "action_id": 1, "action": "Allowed",
    "disposition_id": 1, "disposition": "Allowed",
    "time": 1790743386000, "time_dt": "2026-09-30T04:43:06Z",
    "device": { "hostname": "br01" },
    "connection_info": { "protocol_name": "tcp" },
    "src_endpoint": { "ip": "10.2.4.31", "port": 52300 },
    "dst_endpoint": { "ip": "203.0.113.12", "port": 443 },
    "traffic": { "bytes": 1200 },
    "metadata": {
      "version": "1.3.0", "uid": "e-b487689bfcd6", "log_name": "edge-pipe",
      "original_time": "2026-09-30 10:13:06",
      "product": { "vendor_name": "Acme", "name": "EdgeRouter" }
    },
    "observables": [
      { "name": "src_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "10.2.4.31" },
      { "name": "dst_endpoint.ip", "type_id": 2, "type": "IP Address", "value": "203.0.113.12" }
    ],
    "raw_data": "EDGE|2026-09-30 10:13:06|br01|SESSION|ALLOW|tcp|10.2.4.31|52300|203.0.113.12|443|bytes=1200|rule=R-110",
    "unmapped": { "marker": "EDGE", "record_type": "SESSION", "rule": "R-110" }
  },
  "anvaya": {
    "format": "acme-edge-pipe", "vendor": "Acme", "product": "EdgeRouter", "device": "br01",
    "parser": "acme-edge@1.0.0", "rule_id": "edge-session-allow",
    "parse_confidence": 95, "confidence_breakdown": { "required": 1.0, "typed": 1.0, "mapped": 0.75 },
    "source_timezone": "Asia/Kolkata", "device_time_raw": "2026-09-30 10:13:06", "clock_skew_s": 57,
    "roundtrip": { "ok": true, "value_check": true, "coverage": 1.0, "dropped": 0 },
    "lineage": [
      { "path": "time", "spans": [[5, 24]], "derived": true },
      { "path": "metadata.original_time", "spans": [[5, 24]], "derived": false },
      { "path": "device.hostname", "spans": [[25, 29]], "derived": false },
      { "path": "action", "spans": [[38, 43]], "derived": true },
      { "path": "activity_name", "spans": [[38, 43]], "derived": true },
      { "path": "connection_info.protocol_name", "spans": [[44, 47]], "derived": true },
      { "path": "src_endpoint.ip", "spans": [[48, 57]], "derived": false },
      { "path": "src_endpoint.port", "spans": [[58, 63]], "derived": false },
      { "path": "dst_endpoint.ip", "spans": [[64, 76]], "derived": false },
      { "path": "dst_endpoint.port", "spans": [[77, 80]], "derived": false },
      { "path": "traffic.bytes", "spans": [[87, 91]], "derived": false },
      { "path": "unmapped.marker", "spans": [[0, 4]], "derived": false },
      { "path": "unmapped.record_type", "spans": [[30, 37]], "derived": false },
      { "path": "unmapped.rule", "spans": [[97, 102]], "derived": false }
    ]
  }
}
```

`batch_id` 84 is illustrative; it depends on when the lines are injected. `clock_skew_s` 57 reflects that line 0 is stamped 57 s before the injection moment (below the 300 s skew flag).

### 4.5 AI parser draft: cache file and drafting prompt

**Cache file** `backend/app/aiparser/cache/acme-edge.json` (committed after `make prebake`) has these fields:

| Field | Type | Value |
|---|---|---|
| `group_shape` | string | `"EDGE\|x12"` |
| `origin` | string | `"llm_cached"` if the model output passed every gate, else `"hand_authored"` |
| `model` | string or null | `"qwen2.5:3b"`, or `null` when hand-authored |
| `generated_at` | integer | Epoch ms when prebake ran |
| `prompt_sha256` | string | SHA-256 hex of the exact prompt text sent to the model |
| `yaml` | string | The complete pack text (for the hand-authored fallback, exactly the YAML in §4.4) |
| `gate_report` | object | `{holdout, all, roundtrip, ocsf_valid}` counts from the prebake run |

`make prebake` (`python -m app.aiparser.prebake`):
1. Generates the 20 br01 lines (§5.4).
2. Takes the 14 training lines.
3. Calls Ollama with the prompt below (temperature 0).
4. Validates the result with the same gates as `POST …/test`.
5. Writes `origin: "llm_cached"` with the real model output if every gate passes. Otherwise it writes the hand-authored YAML from §4.4 with `origin: "hand_authored"` and `model: null`, and prints which gate failed.

**Drafting prompt** (used by `LlmDraftSource` and `prebake`):

```
You write ANVAYA parser packs in YAML. Output ONLY the YAML document, no prose, no fences.

Pack format (follow it exactly):
{CISCO_ASA_PACK}

Allowed OCSF targets: time, metadata.original_time, device.hostname, action, connection_info.protocol_name,
src_endpoint.ip, src_endpoint.port, dst_endpoint.ip, dst_endpoint.port, traffic.bytes, user.name, status,
unmapped.<name>.
Allowed extractor types: regex, json, kv, delimited. Allowed transforms: lower, upper, enum, timestamp, kv_value.
Rules:
- Every field present in the lines must be mapped to a target or to unmapped.<name>. Drop nothing.
- Name the pack after the vendor marker you see; version 1.0.0; origin ai_draft; priority 40.
- Use class_uid 4001 for network sessions. Use activity Traffic for allowed sessions and Refuse for denied ones.
- Device timestamps without an offset use source_timezone Asia/Kolkata.

Sample lines (14):
{TRAIN_LINES}
```

`{CISCO_ASA_PACK}` is replaced with the full text of `backend/app/parsers/packs/cisco-asa.yaml` (TRD.md §7.1), and `{TRAIN_LINES}` with the 14 training lines, one per line.

## 5. Generator specification (`generator/`)

### 5.1 Conventions

- **Date:** the scenario date `D` is today in IST. All lines use `D`. ASA uses `%b %d %Y %H:%M:%S`; BSD syslog uses `%b %e %H:%M:%S` (day padded with a space); FortiGate uses `date=%Y-%m-%d time=%H:%M:%S`; EVE uses `%Y-%m-%dT%H:%M:%S.000000+0530`.
- **No randomness.** Every value is a function of the line index. The `random` module is not imported.
- **Order:** sort by `(scenario_time, device_order)` with fw01 = 0, ids01 = 1, vpn01 = 2. Ties within a device keep generation order (the order of the lists below).
- **Transport:** every scripted and steady line is sent as one UDP datagram to `127.0.0.1:5514`, sequentially from one socket. br01 lines go via `POST /api/ingest`.
- **IP ranges:** documentation and private only: 198.51.100.0/24, 203.0.113.0/24, 10.2.4.0/24.

**Line templates**

| Id | Template |
|---|---|
| ASA-302013-out | `<166>{ts} fw01 : %ASA-6-302013: Built outbound TCP connection {cid} for outside:{ext}/{eport} ({ext}/{eport}) to inside:{int}/{iport} ({int}/{iport})` |
| ASA-302013-in | `<166>{ts} fw01 : %ASA-6-302013: Built inbound TCP connection {cid} for outside:{ext}/{eport} ({ext}/{eport}) to inside:{int}/{iport} ({int}/{iport})` |
| ASA-302014 | `<166>{ts} fw01 : %ASA-6-302014: Teardown TCP connection {cid} for outside:{ext}/{eport} to inside:{int}/{iport} duration 0:00:12 bytes {bytes} TCP FINs` |
| ASA-106023 | `<164>{ts} fw01 : %ASA-4-106023: Deny tcp src outside:{ext}/{eport} dst inside:{int}/{iport} by access-group "OUTSIDE_IN" [0x0, 0x0]` |
| EVE-alert | `<134>{bsd_ts} ids01 suricata[2211]: {json}`, where `json` is compact (`separators=(',',':')`) with keys in this order: `timestamp, flow_id, in_iface ("eth1"), event_type ("alert"), src_ip, src_port, dest_ip, dest_port, proto, alert{action ("allowed"), gid (1), signature_id, rev (1), signature, category, severity}` |
| FGT | `<{pri}>date={d} time={t} devname="vpn01" devid="FGVM01TM00000001" logid="{logid}" type="event" subtype="vpn" level="{level}" vd="root" logdesc="{logdesc}" action="{action}" tunneltype="ssl-web" remip={ip} user="{user}" reason="{reason}" msg="{msg}"` |

FortiGate variants:
- **fail:** `pri=185`, `logid=0101039426`, `level=alert`, `logdesc="SSL VPN login fail"`, `action=ssl-login-fail`, `reason=sslvpn_login_permission_denied`, `msg="SSL VPN user login failed"`
- **success:** `pri=190`, `logid=0101039947`, `level=information`, `logdesc="SSL VPN tunnel up"`, `action=tunnel-up`, `reason=login_successfully`, `msg="SSL VPN tunnel established"`

### 5.2 Scripted phase: 207 lines from 10:00:00 to 10:11:48 IST

| Group | Count | Formula (i is the index) |
|---|---|---|
| fw01 outbound connections | 60 | i = 0…59: t = 10:00:00 + 10·i s; ASA-302013-out; cid = 77000 + i; int = 10.2.4.(30 + i mod 10); iport = 51000 + i; ext = 203.0.113.(10 + (3·i) mod 10); eport = 80 if i mod 4 = 0 else 443 |
| fw01 teardowns | 24 | i = 0…23: t = 10:00:00 + 10·i + 12 s; ASA-302014 with the same cid/int/iport/ext/eport as connection i; bytes = 4096 + 64·i |
| fw01 noise denies | 6 | 203.0.113.50 → 10.2.4.20:23 at 10:01:10, 10:04:10, 10:08:10 (eport 45000, 45001, 45002); 203.0.113.51 → 10.2.4.21:3389 at 10:01:40, 10:06:40, 10:10:40 (eport 45000, 45001, 45002); ASA-106023 |
| fw01 **recon denies** | 12 | p = 0…11: t = 10:02:00 + 3·p s; ext = 198.51.100.9; eport = 40101 + p; int = 10.2.4.18; iport = [21, 22, 23, 25, 53, 80, 110, 139, 445, 1433, 3389, 8080][p]; ASA-106023 |
| fw01 **access** | 1 | t = 10:07:12; ASA-302013-in; cid 77801; ext 198.51.100.9; eport 40133; int 10.2.4.18; iport 443 |
| fw01 **follow-on** | 1 | t = 10:08:30; ASA-302013-out; cid 77802; ext 203.0.113.7; eport 443; int 10.2.4.18; iport 51522 |
| ids01 noise | 30 | j = 0…29: t = 10:00:12 + 24·j s; src 10.2.4.(30 + j mod 10):(53000 + j) → 203.0.113.53:53 UDP; flow_id 1500000000000100 + j; signature_id 9000101 "ANVAYA-LAB POLICY DNS query to non-approved resolver"; category "Potential Corporate Privacy Violation"; severity 3 |
| ids01 **recon alerts** | 3 | k = 0…2: t = 10:02:05 / 10:02:20 / 10:02:35; src 198.51.100.9:(40100 / 40115 / 40130) → 10.2.4.18:(22 / 445 / 3389) TCP; flow_id 1500000000000900 + k; signature_id 9000001 "ANVAYA-LAB SCAN Inbound TCP port sweep"; category "Attempted Information Leak"; severity 2 |
| vpn01 noise successes | 28 | k = 0…27: t = 10:00:20 + 25·k s; success variant; ip 203.0.113.(70 + k mod 8); user = [a.sharma, r.verma, k.iyer, p.nair][k mod 4] |
| vpn01 noise failures | 2 | t = 10:03:10 and 10:09:50; fail variant; ip 203.0.113.60; user "guest" |
| vpn01 **credential attack** | 40 | f = 0…39: t = 10:05:00 + f s; fail variant; ip 198.51.100.9; user = [admin, root, vpnuser, administrator][f mod 4] |

**Totals:** fw01 104, ids01 33, vpn01 70, all 207.

**Seq checkpoints** (verified by `test_generator.py`):

| Line | Seq | raw_id |
|---|---|---|
| First ids01 recon alert | 40 | `r-64f70aaee53f` |
| First attacker VPN failure | 100 | `r-3db59309ddaa` |
| fw01 access | 164 | `r-899f4edf0030` |
| fw01 follow-on | 180 | `r-cc9f90d79b5a` |

### 5.3 Steady phase (scenario ≥ 10:12:00, 1×)

These run indefinitely, at real 1× pace, until reset or silence:

| Device | Cadence | Formula (n counts from 0 for each device) |
|---|---|---|
| fw01 | every 1 s (t = 10:12:00 + n s) | Even n: ASA-302013-out with cid 78000 + n, int 10.2.4.(30 + (n/2) mod 10), iport 52000 + n, ext 203.0.113.(10 + n mod 10), eport 443. Odd n: ASA-302014 teardown of connection n − 1, bytes 2048 + 32·n. |
| ids01 | every 4 s (t = 10:12:02 + 4n s) | DNS policy alert as in the noise group, with j = 30 + n |
| vpn01 | every 4 s (t = 10:12:03 + 4n s) | Success variant with k = 28 + n |

Steady traffic never uses 198.51.100.9 or 10.2.4.18, never denies, and never fails a login. That keeps the incident count and the SCN-03 results stable.

### 5.4 br01 injection (SCN-04)

At injection time `T` (scenario clock), for n = 0…19:
- `t = T − (19 − n) · 3 s`
- `verdict = DENY` if n ∈ {4, 11, 17}, else `ALLOW`
- `dport = [443, 443, 80, 53, 443][n mod 5]`; `proto = udp` if dport = 53, else `tcp`
- `src = 10.2.4.(31 + n mod 5)`; `sport = 52300 + n`; `dst = 203.0.113.(12 + n mod 3)`
- `bytes = 1200 + 37·n`; `rule = R-900` if DENY, else `R-110`

Line: `EDGE|{t:%Y-%m-%d %H:%M:%S}|br01|SESSION|{verdict}|{proto}|{src}|{sport}|{dst}|{dport}|bytes={bytes}|rule={rule}`.

Held-out indexes: {2, 5, 8, 11, 14, 17} (including DENY lines 11 and 17). Training: the other 14 (including DENY line 4).

## 6. Types

### 6.1 Pydantic (`backend/app/core/models.py`)

```python
from typing import Literal
from pydantic import BaseModel

DeviceId = Literal["fw01", "ids01", "vpn01", "br01"]
FormatId = Literal["cisco-asa-syslog", "suricata-eve-json", "fortigate-kv", "acme-edge-pipe"]
Severity = Literal["Informational", "Low", "Medium", "High", "Critical"]
QuarantineReason = Literal["no_fingerprint", "no_rule", "parse_error", "schema_invalid", "roundtrip_fail"]

class RawMeta(BaseModel):
    transport: Literal["udp", "tcp", "http", "file"]; peer_ip: str; flags: list[str]; encoding: Literal["utf-8", "binary"]

class RawRecord(BaseModel):
    raw_id: str; seq: int; raw_sha256: str; size: int; segment: str; offset: int
    received_at: int; wall_received_at: int; meta: RawMeta; batch_id: int | None = None

class LineageEntry(BaseModel):
    path: str; spans: list[tuple[int, int]]; derived: bool

class RoundTrip(BaseModel):
    ok: bool; value_check: bool; coverage: float; dropped: int

class AnvayaMeta(BaseModel):
    format: FormatId; vendor: str; product: str; device: str; parser: str; rule_id: str
    parse_confidence: int; confidence_breakdown: dict[str, float]
    source_timezone: str; device_time_raw: str; clock_skew_s: int
    roundtrip: RoundTrip; lineage: list[LineageEntry]

class NormalizedEvent(BaseModel):
    event_id: str; raw_id: str; seq: int; ocsf: dict; anvaya: AnvayaMeta
    # flat columns are derived from ocsf + anvaya by storage.flatten()

class EventSummary(BaseModel):
    event_id: str; raw_id: str; seq: int; time: int; device: str; vendor: str; format: str
    class_uid: int; class_name: str; activity_name: str
    src_ip: str | None; src_port: int | None; dst_ip: str | None; dst_port: int | None
    protocol: str | None; direction: str | None; user_name: str | None
    action: str | None; status: str | None; severity: Severity; title: str | None
    parse_confidence: int; batch_id: int | None; flags: list[str]

class FailedRecord(BaseModel):
    raw_id: str; event_id: str | None; seq: int; batch_id: int; device: str | None; time: int | None
    first_diff_offset: int | None; changed: dict[str, str] | None; field: str | None

class LedgerVerifyResult(BaseModel):
    verdict: Literal["verified", "tampered", "witness_mismatch"]
    checked_records: int; checked_batches: int; took_ms: int
    failed_records: list[FailedRecord]
    failed_batches: list[dict]          # {"batch_id": int, "reason": "leaf_mismatch"|"root_mismatch"|"chain_break"|"bad_signature"}
    witness_status: list[dict]          # {"name": str, "agree": bool, "mismatched_batches": list[int]}

class IncidentSummary(BaseModel):
    incident_id: str; title: str; attacker_ip: str; target_ip: str | None; risk: int
    level: Literal["Low", "Medium", "High"]; stages: list[Literal["recon", "credential", "access"]]
    devices: list[str]; event_count: int; first_time: int; last_time: int; status: Literal["open", "closed"]

class SearchResult(BaseModel):
    question: str; engine: Literal["llm", "template"]; model: str | None; sql: str; params: dict
    clock_caption: str; columns: list[str]; rows: list[list]; row_count: int; took_ms: int
```

### 6.2 TypeScript (`frontend/src/lib/types.ts`)

```ts
export type DeviceId = 'fw01' | 'ids01' | 'vpn01' | 'br01';
export type FormatId = 'cisco-asa-syslog' | 'suricata-eve-json' | 'fortigate-kv' | 'acme-edge-pipe';
export type Severity = 'Informational' | 'Low' | 'Medium' | 'High' | 'Critical';
export type DemoPhase = 'idle' | 'scripted' | 'steady';
export type QuarantineReason = 'no_fingerprint' | 'no_rule' | 'parse_error' | 'schema_invalid' | 'roundtrip_fail';

export interface EventSummary {
  event_id: string; raw_id: string; seq: number; time: number; device: DeviceId; vendor: string; format: FormatId;
  class_uid: 4001 | 2004 | 3002; class_name: string; activity_name: string;
  src_ip: string | null; src_port: number | null; dst_ip: string | null; dst_port: number | null;
  protocol: string | null; direction: string | null; user_name: string | null;
  action: 'Allowed' | 'Denied' | null; status: 'Success' | 'Failure' | null; severity: Severity; title: string | null;
  parse_confidence: number; batch_id: number | null; flags: string[];
}
export interface LineageEntry { path: string; spans: [number, number][]; derived: boolean; }
export interface EventDetail {
  event_id: string;
  raw: { raw_id: string; seq: number; size: number; encoding: 'utf-8' | 'binary'; flags: string[]; transport: string;
         peer_ip: string; received_at: number; raw_sha256: string; batch_id: number | null; text: string };
  ocsf: Record<string, unknown>;
  anvaya: { format: FormatId; vendor: string; product: string; device: DeviceId; parser: string; rule_id: string;
            parse_confidence: number; confidence_breakdown: { required: number; typed: number; mapped: number };
            source_timezone: string; device_time_raw: string; clock_skew_s: number;
            roundtrip: { ok: boolean; value_check: boolean; coverage: number; dropped: number }; lineage: LineageEntry[] };
}
export interface RecordVerifyResult {
  raw_id: string; ok: boolean; seq: number; batch_id: number; leaf_index: number; sha256_stored: string; sha256_now: string;
  audit_path: string[]; root_ok: boolean; signature_ok: boolean; witnesses: { name: string; agree: boolean }[];
  first_diff_offset: number | null; changed: { from: string; to: string } | null; field: string | null;
}
export interface FailedRecord { raw_id: string; event_id: string | null; seq: number; batch_id: number; device: DeviceId | null;
  time: number | null; first_diff_offset: number | null; changed: { from: string; to: string } | null; field: string | null; }
export interface WitnessStatus { name: string; agree: boolean; mismatched_batches: number[]; }
export interface LedgerVerifyResult { verdict: 'verified' | 'tampered' | 'witness_mismatch'; checked_records: number; checked_batches: number;
  took_ms: number; failed_records: FailedRecord[]; failed_batches: { batch_id: number; reason: string }[]; witness_status: WitnessStatus[]; }
export interface LedgerBatch { batch_id: number; first_seq: number; last_seq: number; leaf_count: number; merkle_root: string;
  prev_head: string; head: string; signature: string; sealed_at: number; witnesses: { name: string; cosigned: boolean }[]; }
export interface WitnessInfo { name: string; label: string; pubkey_fingerprint: string; last_cosigned_batch: number; status: string; }
export interface SourceInfo { device: DeviceId; vendor: string; product: string; format: FormatId; parser: string; first_seen: number;
  last_seen: number; events_total: number; rate_epm: number; expected_gap_s: number | null; status: 'learning' | 'active' | 'silent'; silent_for_s: number; }
export interface QuarantineGroup { group_id: string; shape_signature: string; shape_label: string; reason: QuarantineReason; count: number;
  first_seen: number; last_seen: number; transport: string; peer_ip: string; sample_lines: string[]; }
export interface DraftTestResult { draft_id: string; promotable: boolean; holdout: { passed: number; total: number };
  all: { passed: number; total: number }; roundtrip: { passed: number; total: number; min_coverage: number };
  ocsf_valid: { passed: number; total: number }; failures: { index: number; check: string; detail: string }[];
  samples: { index: number; raw: string; ocsf: Record<string, unknown>; lineage: LineageEntry[] }[]; }
export interface IncidentSummary { incident_id: string; title: string; attacker_ip: string; target_ip: string | null; risk: number;
  level: 'Low' | 'Medium' | 'High'; stages: ('recon' | 'credential' | 'access')[]; devices: DeviceId[]; event_count: number;
  first_time: number; last_time: number; status: 'open' | 'closed'; }
export interface IncidentStage { stage: 'recon' | 'credential' | 'access'; label: string; first_time: number; last_time: number;
  devices: DeviceId[]; event_count: number; headline: string; event_ids: string[]; }
export interface Stats { raw_total: number; normalized_total: number; quarantined_total: number; formats_detected: number;
  parse_coverage_pct: number | null; ingest_rate_eps: number; sealed_records: number; sealed_batches: number; cosigned_batches: number;
  open_incidents: number; sources_total: number; sources_silent: number;
  last_bench: { eps: number; machine: string; measured_at: number } | null;
  clock: { now_ms: number; label: string; display_tz: string }; }
export interface SearchResult { question: string; engine: 'llm' | 'template'; model: string | null; sql: string; params: { now_ms: number };
  clock_caption: string; columns: string[]; rows: (string | number | null)[][]; row_count: number; took_ms: number; }
export interface DeliveryStatus { siem: { mode: 'file_sink'; target: string; forwarded: number; last_forwarded: number | null; last_payload: string | null; status: string };
  lake: { exports: { file: string; rows: number; bytes: number; created: number }[] }; retention: { available: boolean }; }
export interface ParquetExport { file: string; rows: number; duckdb_rows: number; bytes: number; took_ms: number; }
export interface TamperInfo { raw_id: string; offset: number; from: string; to: string; field: string; }
```

`parse_coverage_pct` is `null` before any event arrives; the UI shows "—".

## 7. Mapping to the production storage design (for the architecture document)

| MVP element | Production element | Notes |
|---|---|---|
| `raw_events` + segment files | Object store (MinIO / S3): immutable objects per segment, with the index in PostgreSQL | Same framing; object lock (WORM) for retention |
| `events` (SQLite) | PostgreSQL `events` partitioned by day, with `ocsf JSONB`; read-only role for search | Flat columns stay; GIN index on `ocsf` |
| Flat event columns | Parquet lake partitioned `date=/hour=/device=` | The same flatten function feeds both |
| `ledger_batches`, `witness_signatures` | PostgreSQL tables, plus witness services run by independent organisations over HTTPS | One ledger per stream partition; heads published to the witnesses |
| `/data/witness/*.db` | Separate witness databases on separate infrastructure | |
| `quarantine`, `parser_drafts`, `parsers` | PostgreSQL; parser packs as signed files in a git-backed repository | Ed25519-signed pack files |
| `incidents`, `incident_events` | PostgreSQL, fed by a streaming correlation job | Windowed per external IP |
| `audit_log` (append-only triggers) | PostgreSQL with `INSERT`-only grants, mirrored to the vault | |
| In-process `asyncio.Queue` | Kafka-compatible topic partitioned by device | `EventQueue` interface |
