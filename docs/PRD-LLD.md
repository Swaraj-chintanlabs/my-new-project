# PageData Capture: PRD + LLD

Chrome extension that captures a full web page, extracts **structured records** (products, leads, invoices) with an LLM, lets the user review them, and pushes them to a cloud store so any project can read them through an API instead of hand-built Excel sheets.

---

# Part 1: PRD

## 1. Problem
Collecting data from websites means copy-pasting into spreadsheets, which is slow, error-prone and not reusable across projects.

## 2. Goals
- G1. One click (or auto on allowed sites) turns a page into clean structured JSON.
- G2. Data lands in a central cloud store, queryable by any project via API.
- G3. Works on any layout, including image-heavy and canvas pages, because extraction is vision-based and not tied to a DOM scraper.
- G4. Human-in-the-loop review before data is stored.

## 3. Non-goals (v1)
Crawling many pages unattended, bypassing logins or paywalls, team and role management, a mobile app, Firefox/Safari.

## 4. Users
Solo operator or small team who collects product, lead and invoice data from the web and feeds it into other projects.

## 5. User stories
| ID | Story | Priority |
|---|---|---|
| US1 | Sign in once in the extension | P0 |
| US2 | Open a page, pick a type (Product / Lead / Invoice) or Auto, and click Capture | P0 |
| US3 | See extracted records in an editable table and fix values | P0 |
| US4 | Approve and push to cloud; duplicates update instead of duplicating | P0 |
| US5 | Capture works on long pages (full page, not only the visible area) | P0 |
| US6 | If offline or the API fails, the capture is queued and retried | P1 |
| US7 | Auto-run on allowlisted domains | P1 |
| US8 | Browse and export stored records (CSV, JSON) | P1 |
| US9 | Remember the chosen type per domain | P2 |

## 6. Functional requirements
- FR1. Full-page screenshot with no user region selection, including pages taller than one viewport.
- FR2. DOM context (text, tables, JSON-LD, meta) captured alongside the screenshot to raise accuracy.
- FR3. Server-side LLM extraction into a fixed JSON schema per type; the API key never reaches the client.
- FR4. Review UI: editable grid, delete row, confidence flags on low-confidence fields.
- FR5. Upsert by deterministic `dedupe_key`.
- FR6. Auth with per-user data isolation.
- FR7. Settings: allowlist, auto-run, default type, max page height.

## 7. Non-functional requirements
| Area | Target |
|---|---|
| Latency | Capture to review shown in under 20 s (p95) for a page of up to 20k px |
| Accuracy | At least 95% field-level on a labelled set of 100 pages per type (measured in M3) |
| Privacy | Per-user row-level security; no data kept by the LLM provider beyond the request; screenshots optional |
| Reliability | Offline queue with retry and exponential backoff |
| Cost | Under $0.05 per page on average; daily per-user cap |
| Security | MV3 with minimal permissions; token in `chrome.storage.session` |

## 8. Success metrics
Pages captured per week, percentage of captures approved without edits (target at least 80%), time per record vs. manual (target 10x faster), cost per page.

## 9. Risks
| Risk | Mitigation |
|---|---|
| Debugger banner annoys users | Documented; stitch fallback in v2 |
| Very tall pages blow up size or cost | Tile cap (8), max height setting |
| LLM hallucinates values | Constrain to visible data, per-field confidence, mandatory review |
| Site terms of service | Manual-trigger default; user is responsible; allowlist for auto-run |
| Personal data in leads | Consent notice, RLS, delete endpoint |

## 10. Milestones
| M | Scope | Output |
|---|---|---|
| M1 | Supabase project, tables, RLS, auth | DB migration applied |
| M2 | Extension capture, tiling, and DOM extraction | Local capture of any page |
| M3 | `extract` Edge Function + review UI + push | End-to-end Product flow, accuracy eval |
| M4 | Lead and Invoice schemas, dedupe, offline queue | P0 and most P1 |
| M5 | Allowlist and auto-run, records browser, CSV export | v1 release |

---

# Part 2: LLD

## 1. Architecture
```
┌──────────────── Chrome Extension (MV3) ────────────────┐
│ Service Worker ── debugger capture, tiling, queue, API │
│ Content Script ── DOM context (text, tables, JSON-LD)  │
│ Side Panel (React) ── type picker, review grid, push   │
└───────────────┬────────────────────────────────────────┘
                │ HTTPS, Supabase JWT
┌───────────────▼───────────── Supabase ─────────────────┐
│ Auth │ Edge Function `extract` ──► Claude API (vision) │
│ Postgres (RLS): records, profiles │ Storage (optional) │
└────────────────────────────────────────────────────────┘
   Other projects ──► PostgREST / supabase-js ──► records
```
Push to `records` goes directly from the extension via PostgREST (RLS enforces the owner), so there is only **one** custom server function.

## 2. Repository layout
```
extension/
  manifest.json
  src/
    background.ts      # capture, tiling, queue, messaging
    content.ts         # DOM context
    sidepanel/         # React app: App.tsx, ReviewGrid.tsx, api.ts
    shared/types.ts    # Record types, schemas, message types
  vite.config.ts       # @crxjs/vite-plugin
supabase/
  migrations/0001_init.sql
  functions/extract/index.ts
docs/PRD-LLD.md
```

## 3. Manifest (key parts)
```json
{
  "manifest_version": 3,
  "permissions": ["debugger", "activeTab", "scripting", "storage", "sidePanel", "alarms"],
  "host_permissions": ["https://<project>.supabase.co/*"],
  "optional_host_permissions": ["<all_urls>"],
  "background": { "service_worker": "src/background.ts", "type": "module" },
  "side_panel": { "default_path": "src/sidepanel/index.html" },
  "action": { "default_title": "Capture page" }
}
```
`activeTab` covers manual capture; `<all_urls>` is requested only when the user enables auto-run on a domain.

## 4. Capture algorithm (background.ts)
1. `chrome.debugger.attach({tabId}, "1.3")`.
2. Optional pre-scroll to the bottom and back (once) to trigger lazy-load; wait for `Network.idle` (about 800 ms with no requests, 5 s max).
3. `Page.getLayoutMetrics` → `cssContentSize {width, height}`; `H = min(height, MAX_H)` with `MAX_H = 30000`.
4. Tile plan: `TILE_H = 1600` CSS px, `OVERLAP = 80`, `scale = min(1, 1568 / width)` so the long edge stays within model limits; `N = ceil((H - OVERLAP) / (TILE_H - OVERLAP))`, capped at 8 (if exceeded, increase `TILE_H`).
5. For each tile: `Page.captureScreenshot {format:"jpeg", quality:80, captureBeyondViewport:true, clip:{x:0,y:i*(TILE_H-OVERLAP),width,height:TILE_H,scale}}` → base64.
6. `chrome.debugger.detach`. Always detach in `finally`.
7. Ask the content script for the DOM context (section 5) and build the payload.

## 5. DOM context (content.ts)
```ts
type DomContext = {
  url: string; title: string; lang: string;
  jsonld: unknown[];            // all <script type="application/ld+json">
  meta: Record<string,string>;  // og:*, product:*, twitter:*
  text: string;                 // document.body.innerText, collapsed, first 30k chars
  tables: string[][][];         // up to 10 tables, 200 rows each
}
```
JSON-LD gives near-perfect Product and Invoice-like data on many sites and is cross-checked against the screenshot.

## 6. Message contracts
```ts
// SidePanel → SW
{ type: "CAPTURE", kind: "product"|"lead"|"invoice"|"auto" }
// SW → SidePanel (progress)
{ type: "PROGRESS", stage: "capturing"|"extracting"|"done"|"error", pct: number }
// SW → SidePanel (result)
{ type: "RESULT", kind, records: ExtractedRecord[], page: {url,title} }
// SidePanel → SW
{ type: "PUSH", kind, records: ExtractedRecord[], page }
```

## 7. `extract` Edge Function
`POST /functions/v1/extract` with header `Authorization: Bearer <supabase jwt>`

Request:
```json
{ "kind": "product|lead|invoice|auto",
  "page": { "url": "...", "title": "..." },
  "dom": { "...DomContext" },
  "images": ["<base64 jpeg>", "..."] }
```
Response:
```json
{ "kind": "product",
  "records": [ { "data": { "...fields" }, "confidence": { "price": 0.98 } } ] }
```
Steps: verify JWT → enforce daily quota (count in `usage`) → validate size (≤ 8 images, ≤ 6 MB) → call Claude with images + trimmed DOM and a **forced tool call** whose `input_schema` is the JSON Schema for `kind` → validate output with Zod → return.

Prompt rules: extract only values visible on the page or present in the DOM; use `null` for unknown fields; never infer or invent; normalise currency to ISO 4217 and dates to ISO 8601; return one record per item for list pages; if `kind=auto`, classify first, then extract.

Model: `claude-sonnet-5-5` (vision + tool use). Keep the prompt prefix stable to benefit from prompt caching.

## 8. Schemas (shared/types.ts, mirrored in the function)
```ts
Product { name, sku?, brand?, price?, currency?, availability?, image_url?, url, description? }
Lead    { full_name, company?, title?, email?, phone?, linkedin?, location?, source_url }
Invoice { vendor, invoice_no, issue_date, due_date?, currency, subtotal?, tax?, total,
          bill_to?, line_items: { description, qty, unit_price, amount }[] }
```
Dedupe key (computed in the extension and recomputed by a DB trigger):
- product: `sha1("product|" + host + "|" + (sku ?? normalizedUrl))`
- lead: `sha1("lead|" + (email ?? linkedin ?? full_name + "|" + company))`
- invoice: `sha1("invoice|" + vendor.lower + "|" + invoice_no)`

## 9. Database (migrations/0001_init.sql)
```sql
create table public.records (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null default auth.uid() references auth.users on delete cascade,
  kind          text not null check (kind in ('product','lead','invoice')),
  data          jsonb not null,
  confidence    jsonb,
  source_url    text not null,
  page_title    text,
  dedupe_key    text not null,
  captured_at   timestamptz not null default now(),
  updated_at    timestamptz not null default now(),
  unique (user_id, dedupe_key)
);
create index records_kind_idx on public.records (user_id, kind, captured_at desc);
create index records_data_gin on public.records using gin (data jsonb_path_ops);

create table public.usage (
  user_id uuid references auth.users on delete cascade,
  day date default current_date,
  calls int not null default 0,
  primary key (user_id, day)
);

alter table public.records enable row level security;
alter table public.usage   enable row level security;
create policy records_owner on public.records
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy usage_owner on public.usage for select using (user_id = auth.uid());
```
The `usage` table is written only by the Edge Function using the service role.

Push (upsert) from the extension:
```ts
supabase.from("records").upsert(rows, { onConflict: "user_id,dedupe_key" })
```
Typed views per kind can be added later, e.g. `create view products as select data->>'name' as name, (data->>'price')::numeric as price, ... from records where kind='product'`.

## 10. Auth
Supabase email magic link or Google via `chrome.identity.launchWebAuthFlow`. Session stored in `chrome.storage.local`; the access token is refreshed in the service worker. The anon key in the extension is public by design; RLS is the protection.

## 11. Offline queue
Pending pushes are stored in `chrome.storage.local` under `queue`. A `chrome.alarms` job every minute retries with exponential backoff (1, 2, 4, 8 min, max 30 min). Items are removed only after a 2xx response, and upserts are idempotent thanks to `dedupe_key`.

## 12. Review UI (sidepanel)
- Type picker (Product / Lead / Invoice / Auto) and **Capture** button; progress bar from `PROGRESS` messages.
- Grid with one row per record and one column per field; inline edit, row delete, and an amber highlight on fields with confidence below 0.8.
- Invoice rows expand to show line items.
- **Push all** and **Copy JSON**; toast with the count of new vs. updated records.

## 13. Error handling
| Case | Behaviour |
|---|---|
| `chrome://` or Web Store page | Disable Capture, show reason |
| Debugger attach fails (DevTools already open) | Retry once, then show a clear message |
| Page above `MAX_H` | Capture the top `MAX_H` and warn |
| Extract 4xx/5xx | Show the error and keep the captured payload for retry |
| Quota exceeded | Show remaining quota and reset time |
| Push failure | Queue and retry in the background |

## 14. Security and privacy
- Claude key only in Edge Function secrets.
- Strict CSP; no remote code; no `eval`.
- Screenshots are not stored by default (optional toggle saves to a private Storage bucket).
- Delete-my-data action removes all `records` for the user.
- Logs exclude page content.

## 15. Testing
- **Unit:** tiling math, dedupe keys, schema validation (Vitest).
- **Integration:** Edge Function against 10 saved fixtures per type (screenshot + DOM), checking exact-match fields.
- **E2E:** Playwright with the extension loaded against local fixture pages (long page, lazy-load, table list).
- **Accuracy eval (M3):** a labelled set of 100 pages per type with per-field accuracy reporting, run before each release.

## 16. Build order
1. `0001_init.sql`, Supabase auth and project config.
2. Manifest, Vite setup, empty side panel opened from the toolbar.
3. `content.ts` DOM context, then `background.ts` capture and tiling; verify on a 20k px page.
4. `extract` function with the Product schema; wire the CAPTURE → RESULT flow.
5. Review grid and push; verify upsert and dedupe.
6. Add Lead and Invoice schemas and prompts; run the accuracy eval.
7. Offline queue, settings, allowlist and auto-run, records browser and CSV export.
8. Package and publish (private/unlisted first).

## 17. Open decisions
- Allow screenshot storage by default? (Recommended: no.)
- Auto-run default: off, allowlist only. (Recommended.)
- Daily quota value (suggest 200 pages per user per day).
