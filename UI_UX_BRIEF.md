# UI_UX_BRIEF.md: ANVAYA Analyst Console

| Attribute | Detail |
|---|---|
| Version | 1.0.0 |
| Depends on | DEMO_SCRIPT.md (shots), TRD.md §16 (API), APP_FLOW.md (flows), BACKEND_SCHEMA.md §6 (types) |
| Stack | React + Vite + TypeScript, Tailwind CSS, shadcn/ui on Radix, Framer Motion, Zustand, TanStack Query, lucide-react, Recharts, `@fontsource` IBM Plex Sans / Serif / Mono |
| Design target | 1920×1080 at 100% zoom; verified at 1440×900 |

---

## 1. Design intent

**The feeling.** An evidence-grade forensic console for a national agency: calm authority, nothing to hide. It should feel like a well-kept evidence register, not a hacker movie. Light "paper and ink" surfaces, ruled panels, institutional typography, and colour that always means something.

**What we avoid:** near-black backgrounds with neon green; matrix rain; glitch effects; identical rounded cards with the same soft shadow; gradient washes; ALL-CAPS eyebrow labels over every heading; fade-in on every section.

**The one signature element:** the **raw ↔ OCSF lineage view** on Event Detail (§5.3). Hovering any OCSF field highlights the exact bytes it came from in the original log, draws a fine connector curve between the two panes, and keeps the SHA-256 and verification state beside it. All the visual boldness goes here; every other screen stays quiet so this one stands out.

**Colour rule:** colour encodes one of four things: device, severity, integrity state, or a lineage field. It is never decoration.

---

## 2. Design tokens

All tokens are CSS custom properties in `frontend/src/styles/tokens.css`, mapped into Tailwind in `tailwind.config.ts` (§2.9). **No hex value may appear anywhere else in the codebase.** The lint rule `no-raw-hex` (§2.9) fails the build if one does.

### 2.1 Base colours

| Token | Hex | Role |
|---|---|---|
| `--paper` | `#F7F6F2` | App background (warm off-white, like register paper) |
| `--surface` | `#FFFFFF` | Panels, tables, dialogs |
| `--ink` | `#14213D` | Primary text, headings, nav rail background |
| `--graphite` | `#4B5563` | Secondary text, table headers |
| `--rule` | `#DAD7CF` | Borders, dividers, table rules |
| `--seal` | `#9A3B1E` | Brand accent (wax-seal terracotta): primary buttons, focus ring, selected nav item marker |

Derived neutrals: `--muted: #6B7280` (tertiary text, used at 15 px and above only), `--paper-sunk: #EFEDE6` (code and raw-log wells), `--ink-soft: #24345A` (nav hover), `--seal-hover: #82301A`, `--seal-tint: #F6E7E1`.

### 2.2 Semantic tokens: severity (OCSF `severity`)

| Token | Text / dot | Tint background | Used for |
|---|---|---|---|
| `--sev-info` | `#64748B` | `#F1F5F9` | Informational |
| `--sev-low` | `#2563EB` | `#EFF6FF` | Low |
| `--sev-medium` | `#A16207` | `#FEF3C7` | Medium |
| `--sev-high` | `#C2410C` | `#FFEDD5` | High |
| `--sev-critical` | `#9F1239` | `#FFE4E6` | Critical |

### 2.3 Semantic tokens: integrity and status

| Token | Text / icon | Tint background | Meaning |
|---|---|---|---|
| `--int-verified` | `#1E7B4F` | `#E6F4EC` | Hash, root, signature and witnesses all agree |
| `--int-tampered` | `#B42318` | `#FDECEA` | A record or primary-ledger check failed |
| `--int-mismatch` | `#A15C07` | `#FEF0D9` | Primary consistent, witnesses disagree |
| `--int-pending` | `#6B7280` | `#F3F4F6` | Not yet sealed or not yet verified |
| `--st-live` | `#1E7B4F` | — | Stream live, source active |
| `--st-warn` | `#A16207` | `#FEF3C7` | Reconnecting, source silent, LLM fallback, external route |
| `--st-down` | `#B42318` | `#FDECEA` | Offline, component error |

### 2.4 Semantic tokens: devices and formats

Each device has one colour, used for its dot, chip outline and the feed row's left rule. Formats use an outlined mono chip in their own colour.

| Device token | Hex | Device · vendor · product |
|---|---|---|
| `--dev-fw01` | `#1D5FA7` | fw01 · Cisco · ASA (firewall) |
| `--dev-ids01` | `#6D3FA8` | ids01 · OISF · Suricata (IDS) |
| `--dev-vpn01` | `#0E7C74` | vpn01 · Fortinet · FortiGate (VPN) |
| `--dev-br01` | `#8A6D1F` | br01 · Acme · EdgeRouter (branch router, synthetic vendor) |

| Format token | Hex | Format id · chip label |
|---|---|---|
| `--fmt-asa` | `#3F5873` | `cisco-asa-syslog` · "Syslog · ASA" |
| `--fmt-eve` | `#7A5C99` | `suricata-eve-json` · "JSON · EVE" |
| `--fmt-fgt` | `#2F7A6F` | `fortigate-kv` · "key=value" |
| `--fmt-edge` | `#7A6A3A` | `acme-edge-pipe` · "pipe-delimited" |
| `--fmt-unknown` | `#6B7280` | (quarantined) · "unrecognised" |

### 2.5 Semantic tokens: lineage palette

Fields in the lineage view are coloured in rotating order by their position in the OCSF tree (excluding `unmapped`, which always uses `--lin-unmapped`).

| Token | Stroke / text | Highlight background (18% tint) |
|---|---|---|
| `--lin-1` | `#1D5FA7` | `#D6E3F1` |
| `--lin-2` | `#0E7C74` | `#D2E9E7` |
| `--lin-3` | `#9A3B1E` | `#EDDAD3` |
| `--lin-4` | `#6D3FA8` | `#E3D9EF` |
| `--lin-5` | `#A16207` | `#F1E3CC` |
| `--lin-6` | `#3F6212` | `#DCE4D1` |
| `--lin-unmapped` | `#6B7280` | `#E5E7EB` |
| `--lin-tampered` | `#B42318` | `#F9D3CF` (with a 2 px underline) |

### 2.6 Typography

| Family | Token | Role |
|---|---|---|
| IBM Plex Serif 600 | `--font-display` | Page titles (h1) and incident titles only. Gives the "evidence document" voice. |
| IBM Plex Sans 400/500/600 | `--font-ui` | All UI text, labels, tables, buttons |
| IBM Plex Mono 400/500 | `--font-mono` | Raw logs, OCSF JSON, hashes, IDs, SQL, IPs in tables |

All three are imported from `@fontsource/ibm-plex-*` in `src/main.tsx`, so they are bundled with no network calls. The fallback stack is `ui-sans-serif, system-ui` / `ui-serif, Georgia` / `ui-monospace, SFMono-Regular, Menlo`.

| Step | Token | Size / line height | Weight | Family | Used for |
|---|---|---|---|---|---|
| Display | `--text-display` | 32 / 40 px | 600 | Serif | Overview hero title |
| H1 | `--text-h1` | 26 / 34 px | 600 | Serif | Page titles, incident title |
| H2 | `--text-h2` | 20 / 28 px | 600 | Sans | Panel titles |
| H3 | `--text-h3` | 17 / 24 px | 600 | Sans | Card titles, table group headers |
| Body-lg | `--text-body-lg` | 18 / 28 px | 400 | Sans | **Key demo text**: banners, toasts, incident stage headlines, search question |
| Body | `--text-body` | 15 / 24 px | 400 | Sans | Default text, table cells |
| Label | `--text-label` | 14 / 20 px | 500 | Sans | Chip labels, button text (buttons use 15 px), metadata. Never used for key demo text. |
| Mono | `--text-mono` | 14 / 22 px | 400 | Mono | Table IPs, hashes, SQL |
| Mono-lg | `--text-mono-lg` | 16 / 26 px | 400 | Mono | Raw bytes and OCSF JSON in the lineage view |
| KPI | `--text-kpi` | 40 / 44 px | 600 | Sans, tabular numerals | Overview counters |

All numbers use `font-variant-numeric: tabular-nums`. Labels are sentence case; there are no uppercase eyebrows anywhere.

### 2.7 Spacing, radius, elevation

| Spacing token | px | Radius token | px | Used for |
|---|---|---|---|---|
| `--s-1` | 4 | `--r-chip` | 4 | Chips, badges, inline code |
| `--s-2` | 8 | `--r-control` | 6 | Buttons, inputs, table rows on hover |
| `--s-3` | 12 | `--r-panel` | 8 | Panels, cards, toasts |
| `--s-4` | 16 | `--r-dialog` | 12 | Dialogs, the presenter panel |
| `--s-5` | 20 | `--r-pill` | 999 | Health chips, status pills |
| `--s-6` | 24 | | | |
| `--s-8` | 32 | | | |
| `--s-10` | 40 | | | |
| `--s-14` | 56 | | | |

Page gutter is 32 px. Panel padding is 20 px. Table row height is 44 px; Live Feed rows are 48 px.

| Elevation token | Value | Used for |
|---|---|---|
| `--e-0` | `none`, with a `1px solid var(--rule)` border | All panels, cards and tables (the default) |
| `--e-1` | `0 1px 2px rgba(20,33,61,0.06)` | Sticky table headers, the top bar |
| `--e-2` | `0 8px 24px rgba(20,33,61,0.14)` | Toasts, popovers, the incident toast, dialogs |

### 2.8 Motion

| Token | Duration | Easing | Used for |
|---|---|---|---|
| `--m-fast` | 120 ms | `cubic-bezier(0.2, 0, 0, 1)` | Hover, focus, chip state changes |
| `--m-base` | 200 ms | `cubic-bezier(0.2, 0, 0, 1)` | Tab switch, panel expand, highlight on/off |
| `--m-row` | 160 ms | `cubic-bezier(0.05, 0.7, 0.1, 1)` | Feed row insertion (height 0 → 48 px plus 4 px left-rule flash) |
| `--m-emph` | 320 ms | `cubic-bezier(0.05, 0.7, 0.1, 1)` | Incident toast entrance, verify banner entrance |
| `--m-connector` | 240 ms | `cubic-bezier(0.2, 0, 0, 1)` | Lineage connector path draw (stroke-dashoffset) |
| `--m-count` | 600 ms | `cubic-bezier(0.2, 0, 0, 1)` | KPI number tween |
| `--m-exit` | 160 ms | `cubic-bezier(0.3, 0, 1, 1)` | Toast and dialog exit |

Rules:
- **One primary motion at a time.** When the incident toast or a verify banner animates, the feed suspends row animation for 400 ms (rows still update, but appear without animation).
- **No layout shift.** Every animated element has its final size reserved. The feed uses a fixed-height virtualised list.
- **Reduced motion.** With `prefers-reduced-motion: reduce`, all durations become 0 ms, except opacity transitions, which become 80 ms. The lineage connector appears without being drawn.

### 2.9 Implementation

`tokens.css`:

```css
:root{
  --paper:#F7F6F2; --surface:#FFFFFF; --ink:#14213D; --graphite:#4B5563; --rule:#DAD7CF; --seal:#9A3B1E;
  --muted:#6B7280; --paper-sunk:#EFEDE6; --ink-soft:#24345A; --seal-hover:#82301A; --seal-tint:#F6E7E1;
  --sev-info:#64748B; --sev-info-bg:#F1F5F9; --sev-low:#2563EB; --sev-low-bg:#EFF6FF;
  --sev-medium:#A16207; --sev-medium-bg:#FEF3C7; --sev-high:#C2410C; --sev-high-bg:#FFEDD5;
  --sev-critical:#9F1239; --sev-critical-bg:#FFE4E6;
  --int-verified:#1E7B4F; --int-verified-bg:#E6F4EC; --int-tampered:#B42318; --int-tampered-bg:#FDECEA;
  --int-mismatch:#A15C07; --int-mismatch-bg:#FEF0D9; --int-pending:#6B7280; --int-pending-bg:#F3F4F6;
  --st-live:#1E7B4F; --st-warn:#A16207; --st-warn-bg:#FEF3C7; --st-down:#B42318; --st-down-bg:#FDECEA;
  --dev-fw01:#1D5FA7; --dev-ids01:#6D3FA8; --dev-vpn01:#0E7C74; --dev-br01:#8A6D1F;
  --fmt-asa:#3F5873; --fmt-eve:#7A5C99; --fmt-fgt:#2F7A6F; --fmt-edge:#7A6A3A; --fmt-unknown:#6B7280;
  --lin-1:#1D5FA7; --lin-1-bg:#D6E3F1; --lin-2:#0E7C74; --lin-2-bg:#D2E9E7; --lin-3:#9A3B1E; --lin-3-bg:#EDDAD3;
  --lin-4:#6D3FA8; --lin-4-bg:#E3D9EF; --lin-5:#A16207; --lin-5-bg:#F1E3CC; --lin-6:#3F6212; --lin-6-bg:#DCE4D1;
  --lin-unmapped:#6B7280; --lin-unmapped-bg:#E5E7EB; --lin-tampered:#B42318; --lin-tampered-bg:#F9D3CF;
  --font-display:"IBM Plex Serif",ui-serif,Georgia,serif;
  --font-ui:"IBM Plex Sans",ui-sans-serif,system-ui,sans-serif;
  --font-mono:"IBM Plex Mono",ui-monospace,SFMono-Regular,Menlo,monospace;
  --r-chip:4px; --r-control:6px; --r-panel:8px; --r-dialog:12px; --r-pill:999px;
  --e-1:0 1px 2px rgba(20,33,61,0.06); --e-2:0 8px 24px rgba(20,33,61,0.14);
  --m-fast:120ms; --m-base:200ms; --m-row:160ms; --m-emph:320ms; --m-connector:240ms; --m-count:600ms; --m-exit:160ms;
  --ease-standard:cubic-bezier(0.2,0,0,1); --ease-decel:cubic-bezier(0.05,0.7,0.1,1); --ease-exit:cubic-bezier(0.3,0,1,1);
}
@media (prefers-reduced-motion: reduce){
  :root{ --m-fast:0ms; --m-base:0ms; --m-row:0ms; --m-emph:0ms; --m-connector:0ms; --m-count:0ms; --m-exit:0ms; }
}
```

- `tailwind.config.ts` maps every token: `colors: { paper: 'var(--paper)', ink: 'var(--ink)', seal: 'var(--seal)', sev: { info: 'var(--sev-info)', … }, dev: { fw01: 'var(--dev-fw01)', … } }`, plus `fontFamily`, `borderRadius`, `boxShadow` and `transitionDuration`.
- shadcn/ui's CSS variables (`--background`, `--primary`, …) are set **from** these tokens in `tokens.css`, never with their own values.
- **Lint:** `scripts/no-raw-hex.mjs` fails if `/#[0-9a-fA-F]{3,8}\b/` appears in `src/**` outside `styles/tokens.css`. It runs in `npm run lint` and in CI.
- `src/lib/tokens.ts` exports typed lookups:

  ```ts
  deviceColor(device: DeviceId): string        // 'var(--dev-fw01)'
  formatChip(format: FormatId): {label: string; color: string}
  severityTokens(sev: Severity): {fg: string; bg: string}
  lineageColor(index: number): {fg: string; bg: string}
  ```

### 2.10 Self-review against this brief

| First plan (generic default) | Problem | Revised to |
|---|---|---|
| Dark slate background with a cyan accent | Reads as "hacker console", the look the brief rules out; also worse on recorded video | Light paper `#F7F6F2` with ink `#14213D` and the seal accent `#9A3B1E` |
| Inter everywhere | Generic; no institutional voice | IBM Plex Sans for UI, Plex Serif for page and incident titles, Plex Mono for evidence |
| Rounded-xl cards with the same soft shadow on every panel | The "identical cards" pattern | Ruled panels (`--e-0` plus a 1 px border); shadow only on floating layers (`--e-2`) |
| Uppercase eyebrow labels ("LIVE FEED", "INTEGRITY") over headings | Explicitly excluded | Sentence-case titles; no eyebrows |
| Fade-in animation on every section on load | Explicitly excluded; competes with the signature moment | Motion only when data arrives or the user acts (row insert, toast, banner, connector) |
| Gradient header band on Overview | Gradient wash | Flat paper; the hero is typography plus counters |
| Severity colours reused for device identity | Ambiguous colour meaning | Separate device palette (§2.4); severity appears only in badges |
| A green "matrix" style raw-log well | Cliché | Neutral `--paper-sunk` well; colour appears only on hover-highlighted spans |
| A spinning loader on every fetch | Hanging-spinner risk on camera | Skeletons that match the final layout; a spinner only inside buttons, with a label |

---

## 3. App shell and navigation

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│ ▣ ANVAYA  Every perimeter log, one verifiable truth   [Synthetic lab data]   Scenario clock 10:07:12 IST  ● Live │
│                         ( Vault ✓ ) ( Ledger ✓ ) ( Witnesses 2/2 ) ( Local LLM ready ) ( No external route ) │
├────────┬─────────────────────────────────────────────────────────────────────────────────────────┤
│ ◉ Over │                                                                                         │
│ ▤ Feed │   <route content, max-width 1760 px, gutter 32 px>                                     │
│ ⚑ Inc 1│                                                                                         │
│ ⌕ Srch │                                                                                         │
│ ⛨ Integ│                                                                                         │
│ ⧉ Src 20│                                                                                        │
│ ➤ Deliv│                                                                                         │
└────────┴─────────────────────────────────────────────────────────────────────────────────────────┘
```

- **Top bar** (`TopBar`, 64 px, `--surface`, `--e-1`). Left: the product mark (the word "ANVAYA" in Plex Sans 600 at 18 px, letter-spacing 0.04em, with a 14 px seal-coloured square glyph) and the tagline (body, `--graphite`; hidden below 1600 px). Centre-right: `SyntheticDataPill` (DEMO_MODE only), `ScenarioClock`, `StreamDot`. Second row (40 px): `HealthChip` ×5. At 1440×900, both rows merge into one 64 px bar and the tagline is hidden.
- **Nav rail** (`NavRail`, 112 px wide, `--ink` background): icon (24 px, lucide) plus a 13 px label under it, in white at 80% opacity. The selected item has 100% opacity and a 3 px `--seal` left marker. Badges: a numeric badge on Incidents (open count) and on Sources (quarantined count), in `--seal` with white text.
- **Content area:** `--paper` background, max width 1760 px.

| Nav item | Icon (lucide) | Route | Badge |
|---|---|---|---|
| Overview | `Gauge` | `/overview` | — |
| Live feed | `Activity` | `/feed` | — |
| Incidents | `Siren` | `/incidents` | open incidents |
| Search | `Search` | `/search` | — |
| Integrity | `ShieldCheck` | `/integrity` | ✗ if the last verify ≠ verified |
| Sources & parsers | `Network` | `/sources` | quarantined records |
| Delivery | `Send` | `/delivery` | — |

### 3.1 Routes

| Route | Screen | Data dependencies |
|---|---|---|
| `/` | Redirects to `/overview` | — |
| `/overview` | Overview (§5.1) | `GET /api/stats` (poll 1 s), `GET /api/health` (poll 5 s), SSE `stats.tick` |
| `/feed` | Live Feed (§5.2) | `GET /api/events?limit=50`, SSE `event.normalized`, `event.quarantined`, `source.status_changed` |
| `/events/:eventId` | Event Detail (§5.3) | `GET /api/events/{id}`, `POST /api/ledger/verify/{raw_id}` |
| `/incidents` | Incidents list (§5.4) | `GET /api/incidents`, SSE `incident.*` |
| `/incidents/:incidentId` | Incident detail (§5.4) | `GET /api/incidents/{id}` |
| `/search` | Search (§5.5) | `GET /api/search/suggestions`, `POST /api/search` |
| `/integrity` | Integrity (§5.6) | `GET /api/ledger/batches`, `GET /api/witnesses`, `POST /api/ledger/verify`, SSE `ledger.sealed`, `verify.completed` |
| `/sources` | Sources & parsers (§5.7), `?tab=devices\|quarantine\|parsers` (default `devices`) | `GET /api/sources`, `/api/quarantine`, `/api/parsers`, drafts endpoints, SSE `replay.*`, `parser.approved` |
| `/delivery` | Delivery (§5.8) | `GET /api/delivery/status`, `POST /api/export/parquet`, `GET /api/reports/retention` |
| `*` | Not found | Copy: "There's nothing at this address." Button: "Go to Overview" |

Each route is lazy-loaded (`React.lazy`) with a route-level `Suspense` fallback that is that screen's skeleton (not a spinner), and is wrapped in a `RouteErrorBoundary`.

---

## 4. Global patterns

### 4.1 The four states

Every screen and data panel implements four states:

| State | Pattern | Rule |
|---|---|---|
| **Empty** | Centred within the panel: a 32 px lucide icon in `--muted`, an H3 line, one body line explaining what will appear and how, and at most one action button | Written for the analyst, never "No data" |
| **Loading** | A skeleton matching the final layout (rows, counters, panes) in `--paper-sunk`, with a 1.2 s shimmer (static under reduced motion). Shown only after 150 ms, so fast loads never flash. | No full-screen spinners. A button spinner always comes with a verb ("Verifying…"). |
| **Error** | Panel border `--st-down`, an icon, an H3 stating what failed in plain words, one line of what to do, and a **Retry** button | Never show a stack trace or raw error code (the code goes in the tooltip on the error icon) |
| **Success** | The designed screen | The golden path must never show an empty or error state unless the script calls for it |

### 4.2 Toasts

- shadcn `Sonner`, at bottom-right, 420 px wide, `--e-2`, `--r-panel`; body-lg title plus an optional body line.
- Duration 5,000 ms; the incident toast stays until dismissed or clicked.
- **Copy rule:** a toast repeats the verb of the button that caused it ("Verify ledger" → "Ledger verified …"; "Approve & replay" → "Parser … approved · 20 records replayed").

### 4.3 Chips and badges

| Component | Look |
|---|---|
| `DeviceChip` | 8 px dot in the device colour, then the device name in mono 14 px, then the product in label `--graphite`. Example: "● fw01 Cisco ASA" |
| `FormatChip` | Outlined 1 px in the format colour, `--r-chip`, mono 13 px label in the same colour. Example: "Syslog · ASA" |
| `ConfidencePill` | Number in mono 14 px. ≥ 85: `--int-verified` text on `--int-verified-bg`; 60–84: `--sev-medium`; < 60: `--sev-high`. Tooltip shows the breakdown "Required 100% · Typed 100% · Mapped 52%". |
| `SeverityBadge` | Tinted background plus a text label (never colour alone) |
| `IntegrityBadge` | Icon (`ShieldCheck`, `ShieldAlert`, `ShieldQuestion`, `Clock`) plus a label, in the integrity tokens |
| `SilenceBadge` | `--st-warn` on `--st-warn-bg`, `BellOff` icon. Copy: "Silent · last event {n} s ago" |
| `HealthChip` | Pill; a 6 px status dot plus label; tooltip with detail |

### 4.4 Event summary text (`formatEventSummary`)

`src/lib/format.ts → formatEventSummary(e: EventSummary): string` generates the "Event" column text:

| Condition | Text |
|---|---|
| class 4001, activity Open, action Allowed | "Connection opened · {direction lower} {protocol upper}", from the `direction` and `protocol` fields, e.g. "Connection opened · inbound TCP" |
| class 4001, activity Close | "Connection closed" |
| class 4001, action Denied | "Connection refused" |
| class 4001, activity Traffic (br01) | "Session {action lower}" |
| class 2004 | `title`, e.g. "ANVAYA-LAB SCAN Inbound TCP port sweep" |
| class 3002, status Failure | "Login failed · {user_name}" |
| class 3002, status Success | "Login succeeded · {user_name}" |

---

## 5. Screens

Wireframes are drawn for 1920×1080 at 100% zoom. Column widths are in px unless noted.

### 5.1 Overview (`/overview`)

**Purpose:** the opening and closing frame. States the problem and the value in one glance and shows live, measured counters.

**Demo moments:** main cut 0:00–0:25 (idle, problem + layers + offline chip) and 3:40–3:50 (closing counters); 2:00 cut 0:00–0:14 and 1:49–2:00.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Perimeter evidence overview                                  (Serif 32)                  │
│ Every perimeter log, one verifiable truth.                   (body-lg, graphite)         │
│                                                                                          │
│ ┌──────────────┐┌──────────────┐┌──────────────┐┌──────────────┐┌──────────────────────┐ │
│ │ 0            ││ 0.0 /s       ││ 0            ││ —            ││ 0                    │ │
│ │ Events       ││ Live ingest  ││ Formats      ││ Parse        ││ Sealed & co-signed   │ │
│ │ ingested     ││ rate (meas.) ││ detected     ││ coverage     ││ records              │ │
│ └──────────────┘└──────────────┘└──────────────┘└──────────────┘└──────────────────────┘ │
│                                                                                          │
│ ┌─ How ANVAYA works ──────────────────────────────────┐ ┌─ Live activity ──────────────┐ │
│ │ L1 Secure ingestion & forensic vault  ───▶          │ │ events/s sparkline (60 s)     │ │
│ │ L2 Detection, parsing & normalization ───▶          │ │                              │ │
│ │ L3 Intelligence, delivery & governance              │ │ Open incidents: 0            │ │
│ └─────────────────────────────────────────────────────┘ │ Silent sources: 0            │ │
│                                                         └──────────────────────────────┘ │
│ ┌──────────────────────────────────────────────────────────────────────────────────────┐ │
│ │ ▶ Start attack simulation      Lab scenario: scan → VPN brute force → firewall allow │ │
│ └──────────────────────────────────────────────────────────────────────────────────────┘ │
│ Benchmark on this machine: {eps} events/s sustained · {machine} · {date}   (if present) │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Components:** `PageHeader`, `KpiCounter` ×5, `LayerStrip`, `LiveActivityPanel` (`Sparkline`, `MiniStat` ×2), `StartSimulationBar` (DEMO_MODE only), `BenchLine`.

```ts
interface KpiCounterProps { label: string; value: number | null; unit?: string; format: 'int' | 'rate' | 'pct'; hint?: string; }
interface LayerStripProps { layers: { id: 'L1' | 'L2' | 'L3'; title: string; detail: string }[]; }
interface SparklineProps { points: { t: number; eps: number }[]; height: number; }
interface StartSimulationBarProps { phase: DemoPhase; onStart: () => void; pending: boolean; }
interface BenchLineProps { bench: { eps: number; machine: string; measured_at: number } | null; }
```

**Local state:** `startPending: boolean`. Counters come from `useStatsStore`.

**States**
- **Empty** (idle, all zeros): the design shown; parse coverage shows "—" with the hint "Appears once events arrive". This is the designed opening frame.
- **Loading:** five counter skeletons.
- **Error:** if `/api/stats` fails, the counters panel shows "Couldn't reach the ANVAYA service. Check that the api container is running." with **Retry**.
- **Success:** live counters.

**Motion:**
- `KpiCounter` tweens numbers over `--m-count` and never animates on first paint.
- The sparkline extends by one point per second, with no transition.
- **Start attack simulation:** press state, then the button turns into "Simulation running · {speed}×" (disabled), and the app navigates to `/feed` after 600 ms.

**Exact copy**
- Counter labels: "Events ingested", "Live ingest rate (measured)", "Formats detected", "Parse coverage", "Sealed & co-signed records".
- Layer strip:
  - L1 "Secure ingestion & forensic vault": "Every original byte stored, hashed and sealed"
  - L2 "Detection, parsing & normalization": "Any format to OCSF, with byte-level lineage"
  - L3 "Intelligence, delivery & governance": "Cross-device incidents, plain-English search, export"
- Live activity: "Open incidents", "Silent sources".
- Start bar: button "Start attack simulation"; caption "Lab scenario: scan → VPN brute force → firewall allow · synthetic data".
- Bench line: "Benchmark on this machine: {eps} events/s sustained · {machine} · measured {date}".

### 5.2 Live Feed (`/feed`)

**Purpose:** show heterogeneous logs arriving and being normalized live.

**Demo moments:** main cut 0:25–0:37 and 0:54–1:08; 2:00 cut 0:14–0:21.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Live feed                                       [● fw01 Cisco ASA] [● ids01 Suricata]      │
│ 207 events · 3 sources · 3 formats              [● vpn01 FortiGate]   [Pause feed]         │
├──────────┬──────────┬───────────────┬──────────────────────────────┬──────────────────────┬────────┬──────┤
│ Time IST │ Device   │ Format        │ Event                        │ Source → Destination │ Action │ Conf │
├──────────┼──────────┼───────────────┼──────────────────────────────┼──────────────────────┼────────┼──────┤
│▌10:07:12 │ ● fw01   │ [Syslog·ASA]  │ Connection opened · inbound… │ 198.51.100.9 → 10.2.4.18:443 │ Allowed │ 91 │
│▌10:05:39 │ ● vpn01  │ [key=value]   │ Login failed · administrator │ 198.51.100.9 → vpn01  │ —      │ 89  │
│▌10:02:05 │ ● ids01  │ [JSON · EVE]  │ ANVAYA-LAB SCAN Inbound TCP… │ 198.51.100.9 → 10.2.4.18:22 │ High… │ 90  │
│ …  (virtualised, 48 px rows, newest on top, max 500 kept in memory)                       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

Column widths: Time 120, Device 150, Format 160, Event flexible (min 360), Source → Destination 360, Action 110, Confidence 80. The left 4 px rule is in the device colour. The Action column shows `action` for 4001 and the severity badge for 2004 ("Medium"/"High"); "—" otherwise.

**Components:** `PageHeader`, `DeviceFilterChips`, `PauseButton`, `LiveFeedTable` (TanStack Virtual), `FeedRow`.

```ts
interface LiveFeedTableProps { rows: EventSummary[]; paused: boolean; filterDevice: DeviceId | null; onSelect: (eventId: string) => void; }
interface FeedRowProps { e: EventSummary; isNew: boolean; onSelect: (eventId: string) => void; }
interface DeviceFilterChipsProps { sources: SourceInfo[]; selected: DeviceId | null; onToggle: (d: DeviceId) => void; }
```

**Local state:** `filterDevice: DeviceId | null` (null); `paused: boolean` (false; while paused, new rows queue in the store and "{n} new events" appears as a button that resumes).

**Pacing:** the store coalesces `event.normalized` messages and flushes to the table every 250 ms, at most 12 rows per flush. The rest flush on the next tick, so a burst of 40 VPN failures plays as a readable cascade over about 1 s. Quarantined records appear as rows with `FormatChip` "unrecognised", Event "Held in quarantine · no matching parser", and no confidence.

**States**
- **Empty** (idle): icon `Activity`, "Waiting for the first log", "Start the attack simulation on Overview, or send syslog to UDP port 5514." Button: "Go to Overview".
- **Loading:** 10 skeleton rows.
- **Error:** "The live stream is disconnected. Reconnecting automatically…" (shown after 3 s offline) with **Retry now**.
- **Success:** streaming rows.

**Motion:** row insertion `--m-row` (height and opacity) plus a left-rule flash (device colour at 100% → 60% over 600 ms). Suspended for 400 ms during toasts (§2.8).

**Exact copy:** title "Live feed"; subtitle "{n} events · {s} sources · {f} formats"; "Pause feed" / "Resume feed"; queued "{n} new events"; column headers as above, with "Conf." tooltip "Parse confidence (0–100)".

### 5.3 Event Detail and lineage view (`/events/:eventId`): the signature element

**Purpose:** prove "lossless and traceable" visually: every OCSF field maps to exact raw bytes, and the record is sealed.

**Demo moments:** main cut 0:37–0:54 and the tamper view at 2:08–2:20; 2:00 cut 0:21–0:35.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ← Live feed                                                                              │
│ ANVAYA-LAB SCAN Inbound TCP port sweep            (Serif 26)                             │
│ ● ids01 Suricata  [JSON · EVE]  Detection Finding (2004)  Severity: Medium  Conf 90      │
├───────────────────────────────────────────┬──────────────────────────────────────────────┤
│ Original bytes, as received  (422 bytes)  │ Normalized record · OCSF 1.3.0               │
│ 0         10        20        30          │ class_uid ............ 2004                   │
│ <134>Sep 30 10:02:05 ids01 suricata[2211]:│ time ................. 2026-09-30T04:32:05Z ◇│
│ {"timestamp":"2026-09-30T10:02:05.000000+ │ finding_info.title ... ANVAYA-LAB SCAN …    ●│
│ 0530","flow_id":1500000000000900,"in_ifac │ finding_info.uid ..... 9000001              ●│
│ e":"eth1","event_type":"alert","src_ip":"█│ evidences[0].src_endpoint.ip 198.51.100.9 ◀┐│
│ ███████████","src_port":40100,"dest_ip":… │ evidences[0].src_endpoint.port 40100       ││
│      ▲ highlighted span 166–178 ──────────┼────────────────────────────────────────────┘│
│ (mono 16 px, wrapped at 64 chars, ruler   │ severity ............. Medium            ◇  │
│  every 10 bytes)                          │ unmapped.flow_id ..... 1500000000000900  ○  │
├───────────────────────────────────────────┴──────────────────────────────────────────────┤
│ SHA-256 9411 7346 15b9 2fb9 … 479b 0d [copy]   raw_id r-64f70aaee53f   seq 40   batch 3 │
│ (Lossless ✓ 100% of bytes accounted for)   [Verify this record]  ⛨ Verified · sealed in │
│                                             batch 3 · co-signed by 2 witness nodes       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Legend:** ● direct mapping, ◇ derived (transformed or computed), ○ unmapped (kept as-is). The panes split 50/50 with a 24 px gutter, where the connector is drawn. The raw pane shows the text in mono 16 px, wrapped at 64 characters per line, with a byte ruler (mono 12 px, `--muted`) above the first line.

**Components:** `EventHeader`, `LineageView` → `RawBytesPane`, `OcsfFieldList`, `LineageConnector`, `LineageLegend`; `EvidenceFooter` → `HashBlock`, `RoundTripChip`, `VerifyRecordButton`, `IntegrityBadge`.

```ts
interface LineageEntry { path: string; spans: [number, number][]; derived: boolean; }
interface LineageViewProps {
  rawText: string; rawBytesLength: number;
  ocsfFields: { path: string; value: string | number | boolean; colorIndex: number; kind: 'direct' | 'derived' | 'unmapped' }[];
  lineage: LineageEntry[];
  tamper?: { offset: number; from: string; to: string; field: string } | null;   // from the last verify result
}
interface RawBytesPaneProps { text: string; highlights: { spans: [number, number][]; color: string; bg: string }[]; tamperOffset?: number | null; onHoverOffset: (offset: number | null) => void; wrapAt: number; }
interface OcsfFieldListProps { fields: LineageViewProps['ocsfFields']; hoveredPath: string | null; pinnedPath: string | null; onHover: (p: string | null) => void; onPin: (p: string | null) => void; }
interface LineageConnectorProps { fromRect: DOMRect | null; toRect: DOMRect | null; color: string; }
interface VerifyRecordButtonProps { rawId: string; onResult: (r: RecordVerifyResult) => void; }
interface IntegrityBadgeProps { state: 'verified' | 'tampered' | 'mismatch' | 'pending' | 'unknown'; label: string; }
```

**Local state:** `hoveredPath` and `pinnedPath` (in `useLineageStore`, so the keyboard and both panes share them); `verifyResult: RecordVerifyResult | null`.

**Behaviour**
- Hovering or focusing (Tab) a field sets `hoveredPath`. Every span of that path is highlighted in its lineage colour (background tint plus a 1 px outline). The connector is drawn from the field row's left edge to the first span's line box.
- Hovering raw text resolves the byte offset to the narrowest containing span, then highlights the corresponding field row and scrolls it into view (`block: 'nearest'`).
- Click pins; `Esc` unpins.
- If `tamper` is set: the byte at `tamper.offset` is shown in `--lin-tampered` with a 2 px underline, and the owning field row shows an inline tag "Altered on disk: '8' → '9'". The raw pane shows **current vault bytes**; a note under the pane reads "Showing bytes currently in the vault. The copy captured at ingest differs at byte 153."
- Fields are listed in OCSF order: the class fields, `time`, the endpoint fields, the class-specific fields, `metadata.*` (collapsed by default, with the toggle "Show metadata"), then `unmapped.*`.

**States**
- **Empty:** not applicable (always a single record).
- **Loading:** two-pane skeleton.
- **Error:** 404, "This event isn't in the store. It may have been cleared by a demo reset." Button "Back to Live feed". Network error, "Couldn't load this event." with **Retry**.
- **Success:** as drawn.
- **Verify button states:**
  - idle: "Verify this record"
  - pending: "Verifying…"
  - verified: `IntegrityBadge` "Verified · sealed in batch {b} · co-signed by {w} witness nodes"
  - tampered: "Altered on disk · byte {o} changed '{from}' → '{to}' ({field})"
  - not sealed: "Waiting for seal…", retried every 1 s

**Motion:**
- Highlight on/off `--m-base`.
- The connector draws over `--m-connector` via stroke-dashoffset on hover change, as a cubic Bézier of 1.5 px stroke in the field's colour.
- The `IntegrityBadge` enters with opacity plus a 0.96 → 1 scale over `--m-base`.
- No other motion on this screen.

**Exact copy:** back link "← Live feed"; pane titles "Original bytes, as received ({n} bytes)" and "Normalized record · OCSF {version}"; helper under the panes "Hover any field to see the exact bytes it came from. Click to pin."; legend "● mapped directly · ◇ derived · ○ kept unmapped"; round-trip chip "Lossless ✓ {pct}% of bytes accounted for" (or "Round-trip check failed · {reason}" in `--int-tampered`); hash label "SHA-256".

### 5.4 Incidents (`/incidents`, `/incidents/:incidentId`) and the incident toast

**Purpose:** show the cross-device story as one incident with traceable evidence.

**Demo moments:** main cut 1:08–1:35; 2:00 cut 0:35–0:49.

**Incident toast** (global, rendered in `AppShell`):

```
┌──────────────────────────────────────────────────┐
│ ⚑ High-risk incident                    Risk 95  │
│ 198.51.100.9 → 10.2.4.18 · 3 sources             │
│ Recon → Credential attack → Access               │
│                     [Dismiss]  [Open incident]   │
└──────────────────────────────────────────────────┘
```

`IncidentToast` appears when an `incident.updated` or `incident.opened` payload has `risk ≥ 90` and has not been shown before. Entrance: slide up 16 px plus fade over `--m-emph`. A 4 px left rule in `--sev-high`.

**Incident detail:**

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ← Incidents                                                                              │
│ Coordinated intrusion attempt: 198.51.100.9 → 10.2.4.18     (Serif 26)                   │
│ [High] Risk 95   ● fw01 ● ids01 ● vpn01   10:02:00–10:07:12 IST   56 events             │
│ Risk breakdown: 3 stages +75 · 3 devices +10 · in attack order +10                       │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ ○ 10:02  Reconnaissance                                  ● fw01 ● ids01    15 events     │
│ │        Port sweep of 12 ports on 10.2.4.18; 3 IDS scan alerts   (body-lg)               │
│ │        [10:02:05 ids01 ANVAYA-LAB SCAN …] [10:02:00 fw01 Connection refused :21] +13    │
│ ○ 10:05  Credential attack                               ● vpn01           40 events     │
│ │        40 failed VPN logins from 198.51.100.9 (admin, root, vpnuser, administrator)    │
│ │        [10:05:00 vpn01 Login failed · admin] [10:05:01 …] +38                           │
│ ● 10:07  Access                                          ● fw01             1 event      │
│          Firewall allowed 198.51.100.9 → 10.2.4.18:443                                   │
│          [10:07:12 fw01 Connection opened · inbound TCP]                                  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Components:** `IncidentList` (`IncidentCard` rows), `IncidentHeader`, `RiskBreakdown`, `IncidentTimeline` → `StageGroup` → `EvidenceChip`.

```ts
interface IncidentCardProps { incident: IncidentSummary; onOpen: (id: string) => void; }
interface IncidentTimelineProps { stages: IncidentStage[]; onOpenEvent: (eventId: string) => void; }
interface StageGroupProps { stage: IncidentStage; isLast: boolean; onOpenEvent: (eventId: string) => void; }
interface EvidenceChipProps { e: EventSummary; onOpen: (eventId: string) => void; }
interface IncidentToastProps { incident: IncidentSummary; onOpen: () => void; onDismiss: () => void; }
```

**Local state:** `expandedStage: StageId | null`; "+13" expands the stage to show all its evidence chips.

**States**
- **Empty** (list): icon `Siren`, "No incidents yet", "ANVAYA opens an incident when the same external address shows two or more attack stages across your devices."
- **Loading:** card skeletons.
- **Error:** "Couldn't load incidents." with **Retry**.
- **Success:** as drawn. The list shows one `IncidentCard` with the same header line.

**Motion:** the toast as above; stage rows render without entrance animation; the timeline rail is static.

**Exact copy:**
- Stage labels: "Reconnaissance", "Credential attack", "Access"; headlines come from the API (`stages_detail[].headline`).
- Breakdown: "Risk breakdown: {n} stages +{a} · {d} devices +{b} · in attack order +{c}".
- Buttons "Open incident", "Dismiss".
- List title "Incidents", subtitle "Cross-device stories, built from normalized evidence".

### 5.5 Search (`/search`)

**Purpose:** plain-English questions over normalized data, with the SQL always visible.

**Demo moments:** main cut 1:35–2:00 and 3:05–3:12; 2:00 cut 0:49–1:01.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Ask the evidence                                     (Serif 26)                          │
│ ┌──────────────────────────────────────────────────────────────────────────┐ [Ask]       │
│ │ Ask a question about your perimeter logs…                    (body-lg)    │             │
│ └──────────────────────────────────────────────────────────────────────────┘             │
│ Try: (What did 10.2.4.18 do in the last hour?) (Which IPs had failed VPN logins in the  │
│      last 15 minutes?) (Show denied connections to 10.2.4.18) (Show events from br01)   │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ Generated by local LLM · qwen2.5:3b · read-only          now = 10:12:40 IST (Scenario clock) │
│ ┌────────────────────────────────────────────────────────────────────────────── [copy]─┐ │
│ │ SELECT time, device, class_name, src_ip, src_port, dst_ip, dst_port, action          │ │
│ │ FROM events WHERE (src_ip = '10.2.4.18' OR dst_ip = '10.2.4.18')                    │ │
│ │ AND time >= :now_ms - 3600000 ORDER BY time DESC LIMIT 200                           │ │
│ └──────────────────────────────────────────────────────────────────────────────────────┘ │
│ 17 rows · 1.8 s                                                                          │
│ ┌ Time IST │ Device │ Class │ Source │ Src port │ Destination │ Dst port │ Action ─────┐ │
│ │ 10:08:30 │ fw01   │ Network Activity │ 10.2.4.18 │ 51522 │ 203.0.113.7 │ 443 │ Allowed │ │
│ │ …                                                                                    │ │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

SQL keywords are shown in `--seal`, identifiers in `--ink`, strings in `--dev-vpn01`, numbers in `--dev-fw01`. The highlighting is done by a tiny tokenizer in `src/lib/sqlHighlight.ts`, not a third-party theme. Columns named `time` render as HH:MM:SS in IST; IPs and ports in mono.

**Components:** `SearchBox`, `SuggestedQuestions`, `EngineLabel`, `SqlBlock`, `ResultsMeta`, `ResultsTable`.

```ts
interface SearchBoxProps { value: string; onChange: (v: string) => void; onSubmit: () => void; pending: boolean; }
interface SuggestedQuestionsProps { items: string[]; onPick: (q: string) => void; disabled: boolean; }
interface SqlBlockProps { sql: string; engine: 'llm' | 'template'; model: string | null; clockCaption: string; }
interface ResultsTableProps { columns: string[]; rows: (string | number | null)[][]; }
```

**Local state:** `question` (string), `lastResult: SearchResult | null`. Clicking a suggestion sets the question and submits immediately.

**States**
- **Empty:** before the first question, the suggestions are emphasised, with the line "Answers come from normalized events only. Queries are read-only and run on this machine."
- **Loading:** after 150 ms, the SQL block shows the skeleton "Writing SQL…", and the Ask button shows "Asking…".
- **Error** (422 unanswerable): "I can answer questions about addresses, devices, failed logins and denied connections." plus the four chips.
- **Success:** as drawn. A zero-row result shows "No matching events." For Q4 before approval: "No normalized events from br01 yet. 20 raw records from this source are waiting in quarantine." with the link "Open quarantine".

**Motion:** the results table fades in over `--m-base`; the SQL block types nothing (shown whole).

**Exact copy:**
- Title "Ask the evidence"; placeholder "Ask a question about your perimeter logs…"; button "Ask"; "Try:" before the chips.
- Engine labels: "Generated by local LLM · {model} · read-only" or "Generated by query templates · read-only".
- Meta "{n} rows · {s} s".

### 5.6 Integrity (`/integrity`)

**Purpose:** show that the vault is tamper-evident and that witness nodes catch a re-signed ledger.

**Demo moments:** main cut 2:00–2:35; 2:00 cut 1:01–1:21.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Evidence integrity                                  (Serif 26)                           │
│ Every record is hashed, sealed in a Merkle batch, chained, signed, and co-signed.        │
│ [Verify ledger]   Demo: [Simulate tamper] [Simulate insider re-seal]                     │
│ ┌──────────────────────────────────────────────────────────────────────────────────────┐ │
│ │ ⛨ All 231 records verified across 15 batches · both witness nodes agree  (body-lg)   │ │
│ │   Checked in 38 ms · primary key 7f3a…c21e                                           │ │
│ └──────────────────────────────────────────────────────────────────────────────────────┘ │
│ ┌─ Ledger batches ─────────────────────────────────────────────┐ ┌─ Witness nodes ─────┐ │
│ │ Batch │ Records  │ Merkle root │ Signed │ ntro │ certin │ St │ │ witness-ntro        │ │
│ │ 15    │ 225–231  │ 4b1e…09aa   │ ✓      │ ✓    │ ✓      │ ✓  │ │ NTRO witness ·      │ │
│ │ 11    │ 161–176  │ 0c7d…ae12   │ ✓      │ ✓    │ ✓      │ ✗  │ │ simulated, local    │ │
│ │  └ r-899f4edf0030 · fw01 · 10:07:12 · byte 153 changed '8' → '9' (dst_endpoint.ip)   │ │
│ │                                                  [Open event] │ │ key 2c9e…0b14 · ✓   │ │
│ │ …                                                             │ │ witness-certin …    │ │
│ └───────────────────────────────────────────────────────────────┘ └─────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Components:** `PageHeader`, `VerifyLedgerButton`, `DemoIntegrityControls` (DEMO_MODE only), `VerifySummary`, `LedgerTable` → `TamperDiff`, `WitnessPanel` → `WitnessCard`.

```ts
interface VerifySummaryProps { result: LedgerVerifyResult | null; pending: boolean; }
interface LedgerTableProps { batches: LedgerBatch[]; result: LedgerVerifyResult | null; onOpenEvent: (eventId: string) => void; }
interface TamperDiffProps { failure: FailedRecord; onOpenEvent: () => void; }
interface WitnessPanelProps { witnesses: WitnessInfo[]; status: WitnessStatus[] | null; }
interface DemoIntegrityControlsProps { tampered: boolean; resealed: boolean; onTamper: () => void; onReseal: () => void; }
```

**Local state:** `expandedBatch: number | null` (auto-expands the first failing batch after verify).

**States**
- **Empty:** no batches yet: icon `ShieldCheck`, "Nothing sealed yet", "Records are sealed in batches of 16 as they arrive."
- **Loading:** table skeleton; the verify button shows "Verifying…".
- **Error:** "Verification couldn't run." with **Retry**.
- **Success — verdict banners:**
  - verified (`--int-verified`): "All {records} records verified across {batches} batches · both witness nodes agree"
  - tampered (`--int-tampered`): "{k} record failed verification" (plural: "records failed")
  - witness_mismatch (`--int-mismatch`): "Primary ledger consistent · witness nodes disagree on batch {b} onward — tamper suspected"
  - not yet run: neutral "Not verified yet. Run Verify ledger to check every record against its sealed hash."

**Motion:** the banner replaces its predecessor with a crossfade over `--m-emph`; the failing row expands over `--m-base`; there is no motion while verifying, besides the button label.

**Exact copy:**
- Buttons "Verify ledger", "Simulate tamper", "Simulate insider re-seal".
- Demo group label "Demo controls (synthetic)".
- Toasts: "Raw record r-899f4edf0030 altered on disk (demo)", "Ledger re-signed with the primary key from batch 11 (demo insider)", "Ledger verified · {verdict text}".
- Witness card: "{label}", "Key {fingerprint}", "Last co-signed batch {n}", status "Agrees with primary" / "Disagrees from batch {b}".

### 5.7 Sources & parsers (`/sources`)

**Purpose:** device health (silence), quarantine, and the AI parser builder.

**Demo moments:** main cut 2:35–3:05 and 3:12–3:20; 2:00 cut 1:21–1:49.

Tabs (shadcn `Tabs`, URL-synced `?tab=`): "Devices", "Quarantine ({n})", "Parsers".

```
Devices tab
┌───────┬────────────────────┬───────────────┬──────────────┬──────────┬───────────┬───────────────────────────┐
│ Device│ Vendor · product   │ Format        │ Parser       │ Events   │ Rate      │ Status                    │
│ ● fw01│ Cisco · ASA        │ [Syslog·ASA]  │ cisco-asa@1.0.0 │ 312   │ 60 /min ▁▂▃ │ (Silent · last event 37 s ago) │
│ ● ids01│ OISF · Suricata   │ [JSON · EVE]  │ suricata-eve@1.0.0 │ 52 │ 15 /min    │ Active                    │
│ ● vpn01│ Fortinet · FortiGate │ [key=value]│ fortigate-kv@1.0.0 │ 88 │ 15 /min   │ Active                    │
│ ● br01│ Acme · EdgeRouter  │ [pipe-delim.] │ acme-edge@1.0.0 │ 20     │ —         │ Learning                  │
└───────┴────────────────────┴───────────────┴──────────────┴──────────┴───────────┴───────────────────────────┘

Quarantine tab
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ⚠ Unrecognised source · 20 records                                       [Draft parser]  │
│   12 pipe-delimited fields starting with EDGE · via HTTP from 127.0.0.1                  │
│   EDGE|2026-09-30 10:13:06|br01|SESSION|ALLOW|tcp|10.2.4.31|52300|203.0.113.12|443|…    │
│   Raw bytes are sealed. Nothing is dropped.                                              │
└──────────────────────────────────────────────────────────────────────────────────────────┘

Parser draft dialog (900 px wide)
┌──────────────────────────────────────────────────────────────────────────────┐
│ Parser draft for EDGE|x12                                                   ✕ │
│ Drafted by local LLM (cached run) · qwen2.5:3b · generated 29 Sep 2026 21:40  │
│ ┌ YAML (read-only, mono 14) ───────────────────┐ ┌ Checks ─────────────────┐ │
│ │ name: acme-edge                              │ │ Held-out lines parsed 6/6 ✓ │ │
│ │ version: 1.0.0                               │ │ All lines parsed   20/20 ✓ │ │
│ │ extractor: { type: delimited, delimiter: '|' }│ │ Lossless round-trip 20/20 ✓│ │
│ │ …                                            │ │ Valid OCSF          20/20 ✓ │ │
│ └──────────────────────────────────────────────┘ └─────────────────────────┘ │
│ Sample: raw line ↔ OCSF (first held-out line, compact lineage view)          │
│ Approving activates acme-edge@1.0.0 and replays 20 quarantined records.       │
│ This is recorded in the audit log.                                           │
│                         [Reject draft]  [Run tests]  [Approve & replay]      │
│ Replaying ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░ 16/20                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Components:** `SourcesTabs`, `SourcesTable` (`SilenceBadge`, `RateSpark`), `QuarantineGroupCard`, `ParserDraftDialog` → `DraftOriginLabel`, `YamlView`, `CheckList`, `CompactLineage`, `ReplayProgress`; `ParsersTable`.

```ts
interface SourcesTableProps { sources: SourceInfo[]; }
interface QuarantineGroupCardProps { group: QuarantineGroup; onDraft: (groupId: string) => void; draftPending: boolean; }
interface ParserDraftDialogProps { draftId: string; open: boolean; onOpenChange: (o: boolean) => void; }
interface CheckListProps { result: DraftTestResult | null; running: boolean; }
interface ReplayProgressProps { done: number; total: number; status: 'idle' | 'running' | 'done'; }
interface DraftOriginLabelProps { origin: 'llm_live' | 'llm_cached' | 'hand_authored'; model: string | null; generatedAt: number | null; }
```

**Local state (dialog):** `phase: 'drafted' | 'testing' | 'tested' | 'approving' | 'replaying' | 'done'`. **Approve & replay** is disabled until `result.promotable`; its tooltip when disabled is "All four checks must pass 100% before approval."

**States**
- **Empty:**
  - Devices: "No devices have sent logs yet."
  - Quarantine: icon `Inbox`, "Quarantine is empty", "Logs ANVAYA can't parse yet are sealed and held here — never dropped."
  - Parsers: always has the 3 built-ins.
- **Loading:** table or card skeletons; dialog check rows show "Checking…".
- **Error:** draft unavailable: "No cached draft found. Run `make prebake` before recording." Test failure: the check row shows ✗ with "{passed}/{total}" and the first failing line in mono.
- **Success:** as drawn.

**Motion:**
- The check rows resolve one after another, 150 ms apart (a ✓ fades in).
- The replay bar fills from `replay.progress` events.
- On completion, the group card collapses over `--m-base`, and the "Quarantine (20)" tab label counts down to "(0)".

**Exact copy:**
- Origin labels: "Drafted by local LLM (live) · {model}", "Drafted by local LLM (cached run) · {model} · generated {date}", "Hand-authored draft".
- Check labels: "Held-out lines parsed", "All lines parsed", "Lossless round-trip", "Valid OCSF".
- Buttons "Draft parser", "Run tests", "Approve & replay", "Reject draft".
- Toast "Parser acme-edge@1.0.0 approved · 20 records replayed".
- Silence toast: "fw01 has gone silent — no events for {n} s (usually every {g} s)".
- Status labels "Active", "Silent · last event {n} s ago", "Learning".

### 5.8 Delivery (`/delivery`)

**Purpose:** show outputs to the SIEM, the data lake and auditors.

**Demo moment:** main cut 3:20–3:32 (not in the 2:00 cut).

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Delivery                                             (Serif 26)                          │
│ ┌─ SIEM forwarder · file sink (demo) ─┐ ┌─ Data lake export · Parquet ─┐ ┌─ Retention report ─┐ │
│ │ Forwarded 227 events                │ │ [Export to Parquet]           │ │ ANVAYA's tamper-   │ │
│ │ Target /data/outbox/siem-forward…   │ │ Exported 227 events ·         │ │ evident retention  │ │
│ │ Last payload (mono, 3 lines):       │ │ verified by reading back 227  │ │ supports CERT-In's │ │
│ │ <134>1 2026-09-30T04:45:00Z anvaya… │ │ rows · 47 KB · 120 ms         │ │ 180-day log retention requirement. │ │
│ └─────────────────────────────────────┘ └───────────────────────────────┘ │ [View report] [Download JSON] │ │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Components:** `SiemForwarderCard`, `ParquetExportCard`, `RetentionReportCard`, `RetentionReportSheet` (a side sheet with the per-source table).

```ts
interface SiemForwarderCardProps { status: DeliveryStatus['siem']; }
interface ParquetExportCardProps { lastExport: ParquetExport | null; onExport: () => void; pending: boolean; }
interface RetentionReportCardProps { onView: () => void; onDownload: () => void; }
```

**States:** empty ("Nothing forwarded yet."; "No exports yet. Export writes every normalized event to a Parquet file in /data/lake."); loading (card skeletons; "Exporting…"); error ("Couldn't write to the data lake folder. Check that /data/lake is writable."); success.

**Motion:** none beyond button states.

**Exact copy:** card titles as drawn; the retention statement is exactly "ANVAYA's tamper-evident retention supports CERT-In's 180-day log retention requirement."; report footnote "Synthetic lab data. Coverage reflects records held in this vault."

---

## 6. Component inventory

All paths are under `frontend/src/`. Primitives come from shadcn/ui in `components/ui/` (Button, Tabs, Dialog, Sheet, Tooltip, Sonner, Table, Skeleton, ScrollArea, Separator).

| Component | Path | Screen(s) |
|---|---|---|
| `AppShell`, `TopBar`, `NavRail`, `HealthChip`, `ScenarioClock`, `SyntheticDataPill`, `StreamDot` | `components/shell/` | all |
| `PresenterPanel`, `usePresenterKeys` | `components/presenter/` | all (DEMO_MODE) |
| `PageHeader`, `EmptyState`, `ErrorState`, `RouteErrorBoundary` | `components/common/` | all |
| `DeviceChip`, `FormatChip`, `ConfidencePill`, `SeverityBadge`, `IntegrityBadge`, `SilenceBadge` | `components/badges/` | many |
| `KpiCounter`, `LayerStrip`, `LiveActivityPanel`, `Sparkline`, `StartSimulationBar`, `BenchLine` | `features/overview/` | Overview |
| `LiveFeedTable`, `FeedRow`, `DeviceFilterChips`, `PauseButton` | `features/feed/` | Live Feed |
| `EventHeader`, `LineageView`, `RawBytesPane`, `OcsfFieldList`, `LineageConnector`, `LineageLegend`, `EvidenceFooter`, `HashBlock`, `RoundTripChip`, `VerifyRecordButton` | `features/event/` | Event Detail |
| `IncidentToast`, `IncidentList`, `IncidentCard`, `IncidentHeader`, `RiskBreakdown`, `IncidentTimeline`, `StageGroup`, `EvidenceChip` | `features/incidents/` | Incidents |
| `SearchBox`, `SuggestedQuestions`, `EngineLabel`, `SqlBlock`, `ResultsMeta`, `ResultsTable` | `features/search/` | Search |
| `VerifyLedgerButton`, `DemoIntegrityControls`, `VerifySummary`, `LedgerTable`, `TamperDiff`, `WitnessPanel`, `WitnessCard` | `features/integrity/` | Integrity |
| `SourcesTabs`, `SourcesTable`, `RateSpark`, `QuarantineGroupCard`, `ParserDraftDialog`, `DraftOriginLabel`, `YamlView`, `CheckList`, `CompactLineage`, `ReplayProgress`, `ParsersTable` | `features/sources/` | Sources & parsers |
| `SiemForwarderCard`, `ParquetExportCard`, `RetentionReportCard`, `RetentionReportSheet` | `features/delivery/` | Delivery |
| `api` (typed client), `queryKeys`, `useStreamConnection` | `lib/api.ts`, `lib/queryKeys.ts`, `lib/stream.ts` | — |
| `tokens.ts`, `format.ts`, `sqlHighlight.ts`, `lineage.ts` (`spanToPath`, `fieldOrder`) | `lib/` | — |

`CompactLineage` is a smaller `LineageView` (mono 14 px, no connector) used inside the parser dialog.

## 7. Client state (Zustand slices)

Server data is fetched with TanStack Query (keys in `lib/queryKeys.ts`). Zustand holds live-stream state and UI state. Each slice exposes `reset()`, which `resetAllStores()` calls on `demo.state {reset:true}`.

| Slice (file) | State | Actions | Fed by |
|---|---|---|---|
| `useStreamStore` (`state/stream.ts`) | `status: 'connecting'\|'live'\|'reconnecting'\|'offline'`, `lastEventId: number` | `setStatus`, `setLastId`, `reset` | `useStreamConnection` |
| `useFeedStore` (`state/feed.ts`) | `rows: EventSummary[]` (max 500, newest first), `ids: Set<string>`, `pending: EventSummary[]`, `paused: boolean`, `suspendAnimUntil: number` | `enqueue(e)`, `flush()` (every 250 ms, ≤ 12 rows), `setPaused`, `suspendAnimation(ms)`, `reset` | SSE `event.normalized`, `event.quarantined` |
| `useStatsStore` (`state/stats.ts`) | `stats: Stats \| null`, `epsSeries: {t, eps}[]` (last 60) | `setStats`, `reset` | SSE `stats.tick`, `GET /api/stats` |
| `useIncidentStore` (`state/incidents.ts`) | `byId: Record<string, IncidentSummary>`, `toasted: Set<string>`, `toastQueue: string[]` | `upsert(i)`, `markToasted(id)`, `reset` | SSE `incident.opened`, `incident.updated` |
| `useSourcesStore` (`state/sources.ts`) | `byDevice: Record<string, SourceInfo>` | `setAll`, `setStatus(device, status, silentFor)`, `reset` | SSE `source.status_changed`, `GET /api/sources` |
| `useIntegrityStore` (`state/integrity.ts`) | `lastVerify: LedgerVerifyResult \| null`, `tamperInfo: TamperInfo \| null`, `sealedBatches: number` | `setVerify`, `setTamper`, `incSealed`, `reset` | SSE `verify.completed`, `ledger.sealed`; tamper response |
| `useReplayStore` (`state/replay.ts`) | `active: {replayId, done, total} \| null`, `quarantineTotal: number` | `progress`, `complete`, `setQuarantineTotal`, `reset` | SSE `replay.*`, `event.quarantined` |
| `useDemoStore` (`state/demo.ts`) | `demoMode: boolean`, `phase: 'idle'\|'scripted'\|'steady'`, `speed: 10\|20\|40`, `silenced: string[]`, `tampered`, `resealed`, `unknownInjected`, `panelOpen: boolean`, `resetArmedUntil: number` | `hydrate(state)`, `togglePanel`, `startScenario`, `injectUnknown`, `toggleSilence(device)`, `tamper`, `reseal`, `cycleSpeed`, `armOrReset`, `reset` | SSE `demo.state`, `GET /api/demo/state` |
| `useLineageStore` (`state/lineage.ts`) | `hoveredPath: string \| null`, `pinnedPath: string \| null` | `setHover`, `setPin`, `clear` | `LineageView` |

## 8. Real-time stream (SSE)

The connection is `GET /api/stream` (TRD.md API 27). Message format: `id: <int>\nevent: <type>\ndata: <json>\n\n`.

| `event` | Payload (TypeScript) | Store action | Emitted by |
|---|---|---|---|
| `event.normalized` | `EventSummary` | `useFeedStore.enqueue` | FL-01 step 16 |
| `event.quarantined` | `{ raw_id: string; seq: number; time: number; group_id: string; reason: QuarantineReason; preview: string }` | `useFeedStore.enqueue` (as a quarantine row), `useReplayStore.setQuarantineTotal` | FL-01 step 9/10/12 |
| `stats.tick` | `Stats` (same as API 2), every 1 s | `useStatsStore.setStats` | stats task |
| `ledger.sealed` | `{ batch_id: number; first_seq: number; last_seq: number; leaf_count: number; cosigned: number }` | `useIntegrityStore.incSealed`; invalidate `['ledger','batches']` | FL-02 |
| `incident.opened` | `IncidentSummary` | `useIncidentStore.upsert` | FL-04 |
| `incident.updated` | `IncidentSummary` | `useIncidentStore.upsert` (toast if `risk ≥ 90` and not toasted) | FL-04 |
| `source.status_changed` | `{ device: string; status: 'learning'\|'active'\|'silent'; silent_for_s: number; expected_gap_s: number }` | `useSourcesStore.setStatus`; toast on `silent` | FL-09 |
| `verify.completed` | `LedgerVerifyResult` (same as API 9) | `useIntegrityStore.setVerify` | FL-06 |
| `parser.approved` | `{ name: string; version: string; group_id: string; replay_id: string }` | invalidate `['parsers']`, `['quarantine']` | FL-08 |
| `replay.progress` | `{ replay_id: string; done: number; total: number }` | `useReplayStore.progress` | FL-08 |
| `replay.completed` | `{ replay_id: string; parsed: number; still_quarantined: number }` | `useReplayStore.complete`; toast | FL-08 |
| `demo.state` | `{ phase: DemoPhase; speed: 10\|20\|40; silenced: string[]; tampered: boolean; resealed: boolean; unknown_injected: boolean; reset?: boolean }` | `useDemoStore.hydrate`; if `reset`, `resetAllStores()`, `queryClient.clear()`, navigate to `/overview` | FL-03, FL-06, FL-11 |
| `resync` | `{}` | refetch stats, events, incidents, sources | FL-12 |

The server sends `retry: 1500` once, and a `: ping` comment every 15 s.

## 9. Presenter controls (DEMO_MODE only)

`usePresenterKeys()` is mounted in `AppShell`. It listens on `keydown` and ignores events when `document.activeElement` is an `input`, `textarea` or `[contenteditable]`, or when any modifier (Ctrl, Alt, Meta) is held.

| Key | Action | Feedback |
|---|---|---|
| `` ` `` | Toggle `PresenterPanel` | Panel slides in from the bottom-left |
| `1` | `startScenario()` | Same as the Overview button |
| `2` | `injectUnknown()` | Toast "20 records from an unknown source received" |
| `3` | `toggleSilence('fw01')` | **No toast** (silent on purpose; the silence alert itself is the moment). The panel shows "fw01: silenced". |
| `4` | `tamper()` | Toast "Raw record r-899f4edf0030 altered on disk (demo)" |
| `5` | `reseal()` | Toast "Ledger re-signed with the primary key from batch 11 (demo insider)" |
| `6` | `cycleSpeed()` 10 → 20 → 40 → 10 | Panel shows "Speed {n}×" |
| `0` then `0` within 2 s | `armOrReset()`: first press arms (panel shows "Press 0 again to reset"), second resets | Toast "Demo reset · ready" |

**`PresenterPanel`** (bottom-left, 320 px, `--r-dialog`, `--e-2`, hidden by default and never visible during recording unless toggled):
- Phase
- Speed
- Scenario clock
- Stream status
- `send_errors`
- Viewport size ("Viewport 1920×1080")
- the key list
- the state of tamper, re-seal, silenced and injected

## 10. Camera-readiness checklist

1. Design at 1920×1080 at 100% zoom; check every screen at 1440×900, where nothing may be clipped and the top bar collapses to one row.
2. Body text ≥ 15 px; key demo text (banners, toasts, stage headlines, the search question, verify results) ≥ 18 px; mono logs ≥ 14 px (16 px in the lineage view).
3. Contrast meets WCAG AA. Every text token on its background is ≥ 4.5:1; `--muted` is used only at ≥ 15 px.
4. One primary motion at a time; the feed pauses row animation during toasts and banners.
5. No layout shift: reserved heights for the banner, toast area, counters and dialog check rows.
6. Hover states are visible (a `--paper-sunk` row background; a 1 px `--ink` outline on chips); click targets are ≥ 40×40 px.
7. No browser-default focus outlines: use a 2 px `--seal` ring with a 2 px offset.
8. No lorem ipsum, no "TODO", no `undefined`/`NaN`/`null` rendered. Formatters return "—" for missing values.
9. The cursor is visible and not hidden by toasts in shot areas (toasts sit bottom-right; the lineage panes are mid-screen).
10. A scrollbar never appears in the golden path at 1920×1080 except inside the feed, the ledger table and the YAML view.
11. Synthetic data is labelled ("Synthetic lab data" pill; the scenario caption on Overview; the retention-report footnote).
12. Zero console errors and warnings during the golden path (checked by the Playwright smoke test, IMPLEMENTATION_PLAN.md §8).

## 11. Accessibility floor

- Visible keyboard focus everywhere (§10.7); every action is reachable by Tab.
- Lineage works with the keyboard: Tab to a field highlights its bytes, and Enter pins it.
- `prefers-reduced-motion` respected (§2.8).
- WCAG AA contrast; colour is never the only signal (badges carry text; tampered bytes are also underlined).
- ARIA:
  - the feed table has `aria-live="off"` (to avoid flooding screen readers), with a polite live region announcing "{n} new events" every 10 s
  - toasts are `role="status"`
  - the verify banner is `role="alert"` only for tampered or mismatch verdicts
- Dialogs trap focus and return it to the trigger on close.
