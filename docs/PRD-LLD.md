# PageData Capture: PRD + LLD

Chrome extension that extracts **structured records** (products, leads, invoices) directly from a page's **DOM** with deterministic scripts, lets the user review them, and pushes them to the team's own cloud store so any project can read them through an API instead of hand-built Excel sheets.

**No third-party AI/LLM is used.** Page content never leaves the browser except to the team's own backend. This avoids the legal exposure of sharing page data with an external AI provider.

---

# Part 1: PRD

## 1. Problem
Collecting data from websites means copy-pasting into spreadsheets, which is slow, error-prone and not reusable across projects.

## 2. Goals
- G1. One click (or auto on allowed sites) turns a page into clean structured JSON.
- G2. Data lands in a central cloud store, queryable by any project via API.
- G3. Extraction is deterministic, repeatable and auditable: same page, same output.
- G4. No page data is sent to any third-party AI service.
- G5. Human-in-the-loop review before data is stored.

## 3. Non-goals (v1)
LLM or OCR extraction, screenshots, crawling many pages unattended, bypassing logins or paywalls, team and role management, Firefox/Safari.

## 4. Users
Solo operator or small team who collects product, lead and invoice data from the web and feeds it into other projects.

## 5. User stories
| ID | Story | Priority |
|---|---|---|
| US1 | Sign in once in the extension | P0 |
| US2 | Open a page, pick a type (Product / Lead / Invoice) or Auto, click Capture | P0 |
| US3 | See extracted records in an editable table and fix values | P0 |
| US4 | Approve and push to cloud; duplicates update instead of duplicating | P0 |
| US5 | Whole page is covered, including lazy-loaded lists | P0 |
| US6 | If offline or the API fails, the capture is queued and retried | P1 |
| US7 | Auto-run on allowlisted domains | P1 |
| US8 | Browse and export stored records (CSV, JSON) | P1 |
| US9 | Add support for a new site by point-and-click mapping (no code release) | P2 |

## 6. Functional requirements
- FR1. Extraction runs fully in the browser against the live DOM. It uses, in priority order: JSON-LD, Microdata/RDFa, OpenGraph/meta tags, site adapters (selector configs), then generic table parsing.
- FR2. Site adapters are declarative JSON (selectors, attributes, whitelisted transforms), stored in the backend and cached by the extension, so new sites need no extension release.
- FR3. Each field carries a `source` tag (`jsonld`, `microdata`, `adapter`, `table`) used as its confidence signal.
- FR4. Review UI: editable grid, delete row, highlight of fields that are missing or came from weaker sources.
- FR5. Upsert by deterministic `dedupe_key`.
- FR6. Auth with per-user data isolation.
- FR7. Settings: allowlist, auto-run, default type.
- FR8. When nothing structured is found, say so plainly and offer "create adapter" (P2) rather than guess.

## 7. Non-functional requirements
| Area | Target |
|---|---|
| Latency | Capture to review shown in under 3 s (p95) for typical pages |
| Accuracy | At least 98% field-level on supported sites (measured on saved fixtures); unsupported sites return nothing rather than wrong data |
| Privacy and legal | Page data goes only to the team's own Supabase project (region chosen to match legal requirements); no third-party processors; per-user RLS; delete-my-data action |
| Reliability | Offline queue with retry and exponential backoff |
| Cost | No per-page variable cost |
| Security | MV3 with minimal permissions; token in extension storage; no remote code |

## 8. Success metrics
Pages captured per week, percentage of captures approved without edits (target at least 90%), time per record vs. manual (target 10x faster), number of supported sites.

## 9. Risks
| Risk | Mitigation |
|---|---|
| Sites without structured markup | Site adapters; generic table parser; clear "nothing found" state |
| Site markup changes break adapters | Adapters are data, so fixing one is a DB update; fixtures run in CI/nightly to detect breakage |
| Site terms of service | Manual trigger by default; allowlist for auto-run; user is responsible for permitted use |
| Personal data in leads (GDPR, India DPDP, etc.) | Collect only needed fields, consent notice, RLS, retention setting, delete endpoint |
| Remote adapter config abused | Only selectors and whitelisted transforms; no eval, no code; adapters written by admins only |

## 10. Milestones
| M | Scope | Output |
|---|---|---|
| M1 | Supabase project, tables, RLS, auth | DB migration applied |
| M2 | Extension shell, side panel, content-script runner, generic extractors (JSON-LD, Microdata, meta, tables) | Product extracted from schema.org pages |
| M3 | Adapter engine, review UI, push and dedupe | End-to-end flow on 3 real sites |
| M4 | Lead and Invoice schemas, offline queue, fixture test suite | P0 and most P1 |
| M5 | Allowlist and auto-run, records browser, CSV export, adapter admin | v1 release |

---

# Part 2: LLD

## 1. Architecture
```
┌──────────────── Chrome Extension (MV3) ────────────────┐
│ Content Script ── extractors + adapter engine (DOM)    │
│ Service Worker ── adapter cache, queue, push           │
│ Side Panel (React) ── type picker, review grid         │
└───────────────┬────────────────────────────────────────┘
                │ HTTPS, Supabase JWT (PostgREST)
┌───────────────▼───────────── Supabase ─────────────────┐
│ Auth │ Postgres (RLS): records, adapters │ no AI calls  │
└────────────────────────────────────────────────────────┘
   Other projects ──► PostgREST / supabase-js ──► records
```
There is **no custom server code**. The extension talks to Postgres through PostgREST, and RLS enforces ownership.

## 2. Repository layout
```
extension/
  manifest.json
  src/
    content/
      index.ts          # message handler, orchestrates extraction
      extractors/       # jsonld.ts, microdata.ts, meta.ts, tables.ts
      adapter.ts        # declarative adapter engine
      normalize.ts      # price, date, currency, url helpers
    background.ts       # adapter cache, queue, push
    sidepanel/          # React: App.tsx, ReviewGrid.tsx
    shared/types.ts     # schemas (Zod), message types
  vite.config.ts        # @crxjs/vite-plugin
supabase/
  migrations/0001_init.sql
tests/
  fixtures/             # saved HTML per site
  extract.test.ts
docs/PRD-LLD.md
```

## 3. Manifest (key parts)
```json
{
  "manifest_version": 3,
  "permissions": ["activeTab", "scripting", "storage", "sidePanel", "alarms"],
  "host_permissions": ["https://<project>.supabase.co/*"],
  "optional_host_permissions": ["<all_urls>"],
  "background": { "service_worker": "src/background.ts", "type": "module" },
  "side_panel": { "default_path": "src/sidepanel/index.html" },
  "action": { "default_title": "Capture page" }
}
```
`activeTab` covers manual capture; `<all_urls>` is requested only when the user enables auto-run on a domain. The `debugger` permission is not needed.

## 4. Extraction pipeline (content script)
Input: `kind` (`product|lead|invoice|auto`). Output: `ExtractedRecord[]`.

1. **Ready:** wait for `document.readyState === "complete"`. For list pages, auto-scroll to the bottom in steps (max 15 s or until height stops growing) to load lazy content, then scroll back.
2. **Run extractors in priority order** and merge by field (higher-priority source wins, each field keeps its `source`):
   1. `jsonld`: parse every `<script type="application/ld+json">`, flatten `@graph`, map `Product`, `Offer`, `Person`, `Organization`, `Invoice`, `ItemList`.
   2. `microdata`: read `[itemscope][itemtype]` trees.
   3. `meta`: `og:*`, `product:*`, `twitter:*`, `<title>`, canonical URL.
   4. `adapter`: look up an adapter for the current host and path (section 5) and apply it.
   5. `table`: for Invoice and list data, parse `<table>` with a header row; map headers by synonym list (e.g. "Qty", "Quantity").
3. **Normalize:** prices to numbers and ISO 4217 currency, dates to ISO 8601, URLs to absolute, whitespace collapsed.
4. **Validate** with the Zod schema for the kind; drop records missing required fields and report them as warnings.
5. **Auto kind:** classify by what was found (JSON-LD `@type`, adapter `kind`, invoice-like table headers). If ambiguous, ask the user to choose.
6. Compute `dedupe_key` (section 7).

## 5. Site adapters (declarative, no code)
Stored in `adapters` table, fetched at start and cached in `chrome.storage.local` with a version number and refreshed periodically.
```json
{
  "id": "example-shop-product-list",
  "host": "www.example-shop.com",
  "path_regex": "^/category/",
  "kind": "product",
  "list": { "selector": "div.product-card" },
  "fields": {
    "name":  { "selector": "h2.title", "attr": "text" },
    "price": { "selector": ".price", "attr": "text", "transform": ["number"] },
    "url":   { "selector": "a", "attr": "href", "transform": ["absUrl"] },
    "image_url": { "selector": "img", "attr": "src", "transform": ["absUrl"] },
    "sku":   { "selector": ".sku", "attr": "text", "transform": ["regex:SKU[:\\s]*(\\w+)"] }
  },
  "detail": { "enabled": false }
}
```
- Without `list`, the adapter runs once against the whole document (single item page).
- Allowed `attr`: `text`, `html` (sanitized to text), `href`, `src`, `content`, `value`, or any `data-*`.
- Whitelisted `transform`: `trim`, `number`, `currency`, `date`, `absUrl`, `lower`, `regex:<pattern>`. No eval, no function strings.
- Adapters are authored by admins (service role) and read-only for users via RLS.

## 6. Message contracts
```ts
// SidePanel → Content (via SW)
{ type: "EXTRACT", kind: "product"|"lead"|"invoice"|"auto" }
// Content → SidePanel
{ type: "RESULT", kind, records: ExtractedRecord[], warnings: string[], page: {url,title} }
// SidePanel → SW
{ type: "PUSH", kind, records: ExtractedRecord[], page }
// ExtractedRecord
{ data: Record<string,unknown>, sources: Record<string,"jsonld"|"microdata"|"meta"|"adapter"|"table"> }
```

## 7. Schemas and dedupe
```ts
Product { name, sku?, brand?, price?, currency?, availability?, image_url?, url, description? }
Lead    { full_name, company?, title?, email?, phone?, linkedin?, location?, source_url }
Invoice { vendor, invoice_no, issue_date, due_date?, currency, subtotal?, tax?, total,
          bill_to?, line_items: { description, qty, unit_price, amount }[] }
```
Dedupe key (computed in the extension, enforced by the DB unique index):
- product: `sha1("product|" + host + "|" + (sku ?? normalizedUrl))`
- lead: `sha1("lead|" + (email ?? linkedin ?? full_name + "|" + company))`
- invoice: `sha1("invoice|" + vendor.lower + "|" + invoice_no)`

## 8. Database (migrations/0001_init.sql)
```sql
create table public.records (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null default auth.uid() references auth.users on delete cascade,
  kind         text not null check (kind in ('product','lead','invoice')),
  data         jsonb not null,
  sources      jsonb,
  source_url   text not null,
  page_title   text,
  dedupe_key   text not null,
  captured_at  timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  unique (user_id, dedupe_key)
);
create index records_kind_idx on public.records (user_id, kind, captured_at desc);
create index records_data_gin on public.records using gin (data jsonb_path_ops);

create table public.adapters (
  id         text primary key,
  host       text not null,
  kind       text not null,
  config     jsonb not null,
  version    int  not null default 1,
  enabled    boolean not null default true,
  updated_at timestamptz not null default now()
);
create index adapters_host_idx on public.adapters (host) where enabled;

alter table public.records  enable row level security;
alter table public.adapters enable row level security;
create policy records_owner on public.records
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy adapters_read on public.adapters
  for select using (auth.role() = 'authenticated' and enabled);
```
Push from the extension:
```ts
supabase.from("records").upsert(rows, { onConflict: "user_id,dedupe_key" })
```
Typed views per kind (e.g. `products`) can be added later for downstream projects.

## 9. Auth
Supabase email magic link or Google via `chrome.identity.launchWebAuthFlow`. Session in `chrome.storage.local`; token refreshed in the service worker. The anon key in the extension is public by design; RLS is the protection.

## 10. Offline queue
Pending pushes are stored in `chrome.storage.local` under `queue`. A `chrome.alarms` job every minute retries with exponential backoff (1, 2, 4, 8 min, max 30 min). Items are removed only after a 2xx response; upserts are idempotent thanks to `dedupe_key`.

## 11. Review UI (side panel)
- Type picker (Product / Lead / Invoice / Auto) and **Capture** button.
- Grid with one row per record, one column per field; inline edit and row delete. Missing required fields are red; fields from weaker sources (`table`, `meta`) are amber. A tooltip shows each field's source.
- Invoice rows expand to show line items.
- **Push all** and **Copy JSON**; toast with the count of new vs. updated records.
- Empty state: "No structured data found on this page" with a link to request or create an adapter.

## 12. Error handling
| Case | Behaviour |
|---|---|
| `chrome://` or Web Store page | Disable Capture, show reason |
| Nothing extracted | Empty state, no guessing |
| Validation failure on some records | Show valid ones, list warnings for the rest |
| Adapter fetch fails | Use the cached adapters; generic extractors still run |
| Push failure | Queue and retry in the background |
| Session expired | Prompt sign-in, keep captured data in the panel |

## 13. Security and privacy
- Page data goes only to the team's own Supabase project; pick the region to match legal requirements and sign a DPA if personal data is stored.
- No third-party AI, analytics or telemetry that includes page content.
- Strict CSP; no remote code; no `eval`; adapter transforms are a fixed whitelist.
- Delete-my-data action removes all `records` for the user; optional retention period setting.
- Logs exclude page content.

## 14. Testing
- **Unit:** each extractor, normalizers, dedupe keys, Zod schemas, adapter engine (Vitest + jsdom).
- **Fixtures:** saved HTML snapshots per supported site and kind, with expected JSON. A change in output fails CI.
- **Nightly check (P1):** fetch live pages for supported sites and compare against expected shape to catch markup changes early.
- **E2E:** Playwright with the extension loaded against local fixture pages (JSON-LD page, adapter list page, lazy-load page, invoice table).

## 15. Build order
1. `0001_init.sql`, Supabase auth and project config.
2. Manifest, Vite setup, empty side panel opened from the toolbar.
3. `normalize.ts`, JSON-LD, Microdata and meta extractors with fixtures; Product flow working end to end.
4. Review grid and push; verify upsert and dedupe.
5. Adapter engine, `adapters` table fetch and cache; add 3 real site adapters.
6. Lead and Invoice schemas and table extractor.
7. Offline queue, settings, allowlist and auto-run, records browser and CSV export.
8. P2: point-and-click adapter builder.
9. Package and publish (private/unlisted first).

## 16. Open decisions
- Supabase region and whether a DPA is required for the personal data in leads.
- Which sites to support first (this drives the first adapters).
- Auto-run default: off, allowlist only. (Recommended.)
- Retention period for lead data.
