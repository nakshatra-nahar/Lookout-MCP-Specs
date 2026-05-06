# Lookout MCP Server - Technical Specification

> **Author:** Nakshatra Nahar
> **Status:** v1 (build-ready)
> **Audience:** Engineer implementing the mock; future engineers extending it.
> **Purpose:** Define the schema, file layout, and tool surface for Lookout - a Tableau-inspired BI platform mock - in enough detail that another engineer can build it without follow-up questions.

---

## Table of Contents

1. [Overview & Design Principles](#1-overview--design-principles)
2. [Data Model & Schema](#2-data-model--schema)
3. [Filesystem Layout](#3-filesystem-layout)
4. [Global Conventions](#4-global-conventions) - time, pagination, filters, errors, naming, [response sizing & context budget](#46-response-sizing--context-budget)
5. [Tool Surface](#5-tool-surface)
   - 5.1 `list_datasources`
   - 5.2 `get_datasource`
   - 5.3 `list_projects`
   - 5.4 `list_workbooks`
   - 5.5 `get_workbook`
   - 5.6 `list_views`
   - 5.7 `get_view`
   - 5.8 `search_lookout`
   - 5.9 `query_datasource`
   - 5.10 `run_sql`
   - 5.11 `apply_view_filters`
   - 5.12 `render_view`
   - 5.13 `render_workbook`
   - 5.14 `export_view_data`
   - 5.15 `export_query_results`
   - 5.16 `refresh_datasource`
6. [Workflow → Tool Mapping](#6-workflow--tool-mapping)
7. [Assumptions](#7-assumptions)
8. [Positions on Open Questions](#8-positions-on-open-questions)
9. [Seed Data Strategy](#9-seed-data-strategy)
10. [Technical Implementation Notes](#10-technical-implementation-notes)
11. [Testing Strategy](#11-testing-strategy)
12. [Explicit Trade-offs](#12-explicit-trade-offs)
13. [Future Extensions](#13-future-extensions)

---

## 1. Overview & Design Principles

Lookout is a read-only BI platform mock. AI agents acting as analysts use it to discover datasources and workbooks, query data, render charts, and export results. The mock must be *contract-faithful* to a real BI tool - response shapes, ID formats, error codes, pagination semantics, and edge cases should all behave as a real product would. An agent should never be able to detect it's a mock during a normal session.

**Five principles drive every design choice in this document:**

1. **Focused tools, not kitchen sinks.** Each tool does one job and explains *when* and *why* the agent should use it. Where two responsibilities feel adjacent (e.g. exporting a view's data vs. exporting query results), I split them rather than overload one tool with conditional behavior.
2. **List + Get is the default discovery pattern.** `list_*` tools return small summaries with stable IDs; `get_*` tools return full detail for one entity. Agents drill in only when needed.
3. **Realism over convenience.** Two datasources backing the same workbook should have *different* shapes. Refresh times should vary. Numbers should be plausibly distributed (no flat $100s). One datasource is seeded as offline; one extract is seeded as stale. The agent should encounter the messiness of a real BI environment.
4. **Composition over special-purpose tools.** Where a workflow can be decomposed into 2–3 calls to general tools (e.g. period-over-period comparison = two `query_datasource` calls), I do not add a dedicated tool. Composition keeps the surface tight and the agent's reasoning more general.
5. **Failure is part of the contract.** Datasources can be offline, extracts can be stale, queries can time out, result sets can exceed limits. These are first-class error modes the agent must handle, not edge cases to hide.

**What Lookout does *not* do (explicit non-goals, restating the brief):**

- Real-time streaming, write-through to external systems, user auth, multi-tenant permissions, dashboard editing, custom calculated fields, geographic map rendering, story points, mobile layouts, or natural-language-to-SQL ("Ask Data" in real Tableau).

---

## 2. Data Model & Schema

### 2.1 ID Format

All IDs are prefixed strings of the form `<prefix>_<24 lowercase hex chars>`, e.g. `ds_9f2c1ab847e3d0b412a7f6c8`. The 24-hex format mirrors the example in the brief (`msg_9f2c1ab847e3d0b412a7f6c8`) and is realistic to ID-generation patterns at production BI platforms (Tableau itself uses LUIDs that are 36-char UUIDs; we use a slightly more compact form for the mock).

| Prefix | Entity | Example |
|--------|--------|---------|
| `ds_` | Datasource | `ds_9f2c1ab847e3d0b412a7f6c8` |
| `fld_` | Datasource field | `fld_3a8b2e17c4f9d1e60b5a8273` |
| `wb_` | Workbook | `wb_5c9d4f82e6a1b3d7c0f924e1` |
| `vw_` | View | `vw_2b8e7c1f9a3d6e0c4b5f8273` |
| `prj_` | Project | `prj_7d3f1a9c5b2e8f0d4c6a8e91` |
| `qry_` | Query | `qry_4e2c8a1f6b9d3e5c7f024918` |
| `exp_` | Export | `exp_8a4f2c1e7b3d9e5c0f482763` |
| `rnd_` | Render | `rnd_1c5e9a3b7f0d4e6c2a8f1294` |

**Why prefixed?** Two reasons: (a) The agent often passes an ID it received earlier from another tool. The prefix lets us validate the ID *belongs to the right entity* before hitting the database, returning a clean `INVALID_ID_FORMAT` instead of `WORKBOOK_NOT_FOUND` for a typo where a `vw_…` was passed where a `wb_…` was expected. (b) When the agent looks at an ID in conversation, the prefix tells it what the ID points to without a lookup.

**ID generation:** server-side, on insert. We use `<prefix>_<random 24 hex>` via `uuid4().hex[:24]` - collisions are astronomically unlikely at our scale.

### 2.2 Entity Relationships

```
project (1) ──< (n) workbook (1) ──< (n) view (n) >── (1) datasource
                                                            │
                                                            └──< (n) field
```

- A **project** organizes workbooks (e.g. "Sales Operations", "Marketing", "Customer Success"). Like Tableau projects, but we do not model nested projects.
- A **workbook** lives in exactly one project, contains 4–12 views, has a primary datasource for context, and stores its own metadata (title, description, owner, timestamps).
- A **view** belongs to exactly one workbook and references exactly one datasource. It carries a chart type (`bar`, `pie`, `treemap`, `line`, `histogram`), a config blob (axes, group-by, color encoding, aggregation), and zero or more *default filters* baked into the view definition.
- A **datasource** has many fields and a separate physical table holding its rows. Datasources have a `connection_type` (`live` vs `extract`) and a `status` (`online`, `offline`) plus a `last_refreshed_at` for extracts.
- A **field** describes one column on a datasource (technical name, display name, data type, semantic role, default aggregation, format string, description).

### 2.3 SQLite Schema

I use SQLite's foreign keys (with `PRAGMA foreign_keys = ON;`) and stick to portable SQL. JSON config blobs are stored as TEXT and parsed in application code.

```sql
-- ======================================================================
--  Metadata tables (catalog of BI entities)
-- ======================================================================

CREATE TABLE projects (
  id            TEXT PRIMARY KEY,                  -- prj_<24hex>
  name          TEXT NOT NULL,
  description   TEXT,
  created_at    TEXT NOT NULL,                     -- ISO-8601 UTC
  UNIQUE (name)
);

CREATE TABLE datasources (
  id                       TEXT PRIMARY KEY,       -- ds_<24hex>
  name                     TEXT NOT NULL,
  description              TEXT,
  connection_type          TEXT NOT NULL,          -- 'live' | 'extract'
  status                   TEXT NOT NULL,          -- 'online' | 'offline'
  last_refreshed_at        TEXT,                   -- NULL for live
  refresh_schedule         TEXT,                   -- 'daily' | 'weekly' | 'monthly' | NULL
  stale_after_minutes      INTEGER,                -- NULL = never stale; e.g. 1440 = 1 day
  row_count                INTEGER NOT NULL,
  data_table               TEXT NOT NULL,          -- physical SQLite table holding rows
  created_at               TEXT NOT NULL,
  UNIQUE (name),
  UNIQUE (data_table)
);

CREATE TABLE datasource_fields (
  id                    TEXT PRIMARY KEY,          -- fld_<24hex>
  datasource_id         TEXT NOT NULL,
  name                  TEXT NOT NULL,             -- physical column name
  display_name          TEXT NOT NULL,             -- human label
  data_type             TEXT NOT NULL,             -- 'integer'|'float'|'string'|'boolean'|'date'|'datetime'
  semantic_role         TEXT NOT NULL,             -- 'dimension' | 'measure'
  default_aggregation   TEXT,                      -- 'sum'|'avg'|'count'|'count_distinct'|'min'|'max'|NULL
  format_string         TEXT,                      -- e.g. '$#,##0.00', '0.00%', NULL
  description           TEXT,
  position              INTEGER NOT NULL,
  FOREIGN KEY (datasource_id) REFERENCES datasources(id) ON DELETE CASCADE,
  UNIQUE (datasource_id, name)
);

CREATE INDEX idx_fields_datasource ON datasource_fields(datasource_id);

CREATE TABLE workbooks (
  id                      TEXT PRIMARY KEY,        -- wb_<24hex>
  project_id              TEXT NOT NULL,
  name                    TEXT NOT NULL,
  description             TEXT,
  owner_name              TEXT NOT NULL,
  primary_datasource_id   TEXT,                    -- nullable; for context only
  created_at              TEXT NOT NULL,
  updated_at              TEXT NOT NULL,
  FOREIGN KEY (project_id)              REFERENCES projects(id),
  FOREIGN KEY (primary_datasource_id)   REFERENCES datasources(id)
);

CREATE INDEX idx_workbooks_project   ON workbooks(project_id);
CREATE INDEX idx_workbooks_updated   ON workbooks(updated_at DESC);

CREATE TABLE views (
  id              TEXT PRIMARY KEY,                -- vw_<24hex>
  workbook_id     TEXT NOT NULL,
  datasource_id   TEXT NOT NULL,
  name            TEXT NOT NULL,                   -- view title shown to user
  description     TEXT,
  chart_type      TEXT NOT NULL,                   -- 'bar'|'pie'|'treemap'|'line'|'histogram'
  config_json     TEXT NOT NULL,                   -- JSON: {x, y, color, group_by, aggregation, ...}
  position        INTEGER NOT NULL,                -- order within workbook
  FOREIGN KEY (workbook_id)   REFERENCES workbooks(id) ON DELETE CASCADE,
  FOREIGN KEY (datasource_id) REFERENCES datasources(id),
  UNIQUE (workbook_id, name)
);

CREATE INDEX idx_views_workbook   ON views(workbook_id);
CREATE INDEX idx_views_datasource ON views(datasource_id);

CREATE TABLE view_default_filters (
  id          TEXT PRIMARY KEY,
  view_id     TEXT NOT NULL,
  field_id    TEXT NOT NULL,
  operator    TEXT NOT NULL,                       -- see §4.3
  value_json  TEXT NOT NULL,                       -- JSON-encoded scalar or array
  position    INTEGER NOT NULL,
  FOREIGN KEY (view_id)  REFERENCES views(id)             ON DELETE CASCADE,
  FOREIGN KEY (field_id) REFERENCES datasource_fields(id)
);

-- ======================================================================
--  Side-effect tables (queries, exports, renders)
-- ======================================================================

CREATE TABLE queries (
  id              TEXT PRIMARY KEY,                -- qry_<24hex>
  datasource_id   TEXT NOT NULL,
  query_kind      TEXT NOT NULL,                   -- 'structured' | 'sql'
  query_json      TEXT NOT NULL,                   -- the structured query payload OR {"sql": "..."}
  row_count       INTEGER NOT NULL,                -- total rows in the result before truncation
  truncated       INTEGER NOT NULL DEFAULT 0,      -- 1 if hit the row cap
  ran_at          TEXT NOT NULL,
  duration_ms     INTEGER NOT NULL,
  FOREIGN KEY (datasource_id) REFERENCES datasources(id)
);

CREATE INDEX idx_queries_ran_at ON queries(ran_at DESC);

CREATE TABLE exports (
  id            TEXT PRIMARY KEY,                  -- exp_<24hex>
  kind          TEXT NOT NULL,                     -- 'view' | 'query'
  source_id     TEXT NOT NULL,                     -- view_id or query_id
  format        TEXT NOT NULL,                     -- 'csv' | 'json'
  file_path     TEXT NOT NULL,                     -- relative to fs root
  row_count     INTEGER NOT NULL,
  byte_size     INTEGER NOT NULL,
  created_at    TEXT NOT NULL,
  expires_at    TEXT NOT NULL                      -- created_at + 24h
);

CREATE INDEX idx_exports_created ON exports(created_at DESC);

CREATE TABLE renders (
  id            TEXT PRIMARY KEY,                  -- rnd_<24hex>
  kind          TEXT NOT NULL,                     -- 'view' | 'workbook'
  source_id     TEXT NOT NULL,                     -- view_id or workbook_id
  format        TEXT NOT NULL,                     -- 'png' | 'svg'
  width         INTEGER NOT NULL,
  height        INTEGER NOT NULL,
  file_path     TEXT NOT NULL,
  created_at    TEXT NOT NULL,
  expires_at    TEXT NOT NULL                      -- created_at + 24h
);

CREATE INDEX idx_renders_created ON renders(created_at DESC);
```

**Per-datasource data tables.** Each datasource owns one physical table, named `data_<datasource_short_id>` (e.g. `data_9f2c1ab8`), with columns matching its fields. This is critical: the physical schema *matches the published schema*, so `query_datasource` and `run_sql` translate cleanly into real SQL. The `datasources.data_table` column points at the table name. We do **not** use a single tall `(datasource_id, row_id, field_id, value)` table - that would prevent indexes on actual columns and produce slow queries on the 100k-row sets the brief allows.

**Why JSON columns for `config_json`, `query_json`, `value_json`?** SQLite's JSON1 extension is universally available and these blobs are read-most, write-once. Modeling 30+ chart-config fields as columns would be brittle. SQLite indexes on `json_extract(config_json, '$.x_axis')` are available if needed.

**Why no audit/event-log table for views?** v1 is read-only. The `queries`, `exports`, and `renders` tables exist because they produce side-effect artifacts the agent (or harness) may need to reference. Pure reads (e.g. `get_view`) leave no trace; this matches BI tools that don't audit every list call.

### 2.4 What is *not* in the data model and why

- **`users` / `permissions`:** explicit non-goal in the brief.
- **`dashboards` (separate from workbooks):** the brief flattens to Datasource → Workbook → View. A workbook with multiple views *is* the agent's "dashboard." Adding a third tier doubles UI complexity for negligible agent-facing value. (A workbook acts as a dashboard at render time via `render_workbook`.)
- **`calculated_fields`:** out of scope for v1 - adds a query-rewriting layer with little workflow benefit. Field schemas describe pre-computed columns only.
- **Tags / favorites / subscriptions:** Tableau has these; agents don't need them for the workflows in scope.
- **Connection credentials:** datasources are pre-loaded; no mock auth.

---

## 3. Filesystem Layout

The `fs` reference is rooted at the harness-mounted directory. Tools write under stable subpaths so the agent can refer to artifacts cleanly:

```
fs/
├── exports/
│   └── {export_id}.{csv|json}            # written by export_view_data, export_query_results
├── renders/
│   └── {render_id}.{png|svg}             # written by render_view, render_workbook
└── thumbnails/                           # seed-time, read-only
    ├── workbooks/{workbook_id}.png       # 320×200 preview, returned by get_workbook
    └── views/{view_id}.png               # 320×200 preview, returned by get_view
```

**Why per-id directories rather than per-entity directories?** Exports and renders are short-lived artifacts the agent passes around as paths. Flat `{id}.{ext}` filenames are simplest, and the ID is unique. We don't shard by date because at the seed scale (a few dozen artifacts per session), a flat directory is fine.

**Cleanup:** exports and renders have a 24-hour `expires_at`. The harness or a background task can sweep `fs/exports/` and `fs/renders/` of expired files. The mock itself does not implement cleanup - by design it's stateless across sessions where stateless is acceptable.

**Thumbnails are seed-time.** They're pre-rendered and shipped with the seed data so `get_workbook` / `get_view` can return a path immediately without invoking the renderer. This matches Tableau's behavior: workbook thumbnails are precomputed and updated on save.

---

## 4. Global Conventions

These conventions apply uniformly across every tool. Where a tool varies, it states so explicitly.

### 4.1 Time

- All timestamps in API responses are ISO-8601 UTC (`2026-04-22T14:32:07Z`). No timezone offsets.
- Where the agent supplies dates (filters, query inputs), it must use ISO-8601 (`YYYY-MM-DD` for dates, full ISO-8601 for datetimes). Strings that don't parse cleanly return `INVALID_FILTER_VALUE`.
- The mock has a notion of "now" supplied by the harness (a wall-clock or a fixed seed time). Staleness checks (`stale_after_minutes`) compare `last_refreshed_at` to "now". For deterministic testing, the harness can pin "now" to a fixed value.

### 4.2 Pagination

Every list-style tool uses **opaque cursor pagination**:

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `page_size` | integer | 25 | Max 100. Returning more than 100 in one call wastes context; the brief explicitly calls this out. |
| `cursor` | string | `null` | Opaque token returned as `next_cursor` from the previous page. First page = omit. |

Response shape:

```json
{
  "items": [ /* up to page_size items */ ],
  "next_cursor": "eyJvIjoyNX0=",          // null when no more pages
  "total_count": 137                       // when cheap to compute; otherwise omitted
}
```

**Cursor encoding.** Internally the cursor is a base64-encoded JSON object: `{"o": <offset>, "s": <sort_key>, "h": <hash>}`. The `h` is a hash of the query parameters; if it doesn't match on a follow-up request, we return `INVALID_CURSOR`. Agents shouldn't construct cursors themselves and shouldn't expect them to be stable across query-parameter changes.

**Why opaque cursors over offset/limit?** Opaqueness lets us evolve the implementation later (e.g., move to keyset pagination) without breaking clients, and it matches modern API conventions. For seed data scale, the cursor encodes an offset internally - implementation is trivial.

**`total_count` policy.** Cheap to compute over indexed columns (which is why it's included in `list_datasources`, `list_projects`, `list_workbooks`, `list_views`), but omitted from `query_datasource` and `run_sql` results unless the agent passes `include_total_count: true` (which forces a second `COUNT(*)` query - opt-in to avoid surprising the agent with slow queries).

### 4.3 Filter Semantics

A filter is a tuple of `{field, operator, value}`. Tools that accept filters take a list, AND-ed together. For OR or NOT logic, the agent uses `run_sql`.

```json
{ "field": "region", "operator": "in", "value": ["NA", "EMEA"] }
```

The `field` may be either the field's technical `name` (case-sensitive, as defined on the datasource) or its `id` (`fld_…`). The agent will usually use the technical name because that's what `get_datasource` exposes.

**Operators by data type:**

| Type | Allowed operators |
|------|-------------------|
| `integer`, `float` | `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `is_null`, `is_not_null`, `in`, `not_in` |
| `string` | `eq`, `neq`, `in`, `not_in`, `contains`, `starts_with`, `ends_with`, `is_null`, `is_not_null` |
| `date`, `datetime` | `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `is_null`, `is_not_null` |
| `boolean` | `eq`, `is_null`, `is_not_null` |

`between` takes a 2-element array `[lo, hi]` (inclusive). `in` / `not_in` take an array. `contains` / `starts_with` / `ends_with` are case-insensitive (translated to SQL `LIKE` with `LOWER()` on both sides). `is_null` / `is_not_null` ignore `value`.

**Validation order** (agent-friendly errors):
1. Field exists on the datasource → otherwise `FIELD_NOT_FOUND`.
2. Operator legal for that field's data type → otherwise `INVALID_FILTER_OPERATOR`.
3. Value parseable in the field's data type → otherwise `INVALID_FILTER_VALUE`.

### 4.4 Error Envelope

All errors return:

```json
{
  "error": {
    "code": "DATASOURCE_NOT_FOUND",
    "message": "No datasource exists with id 'ds_doesnotexist000000000000'.",
    "details": { "datasource_id": "ds_doesnotexist000000000000" }
  }
}
```

- `code` is a stable, uppercase-snake-case string.
- `message` is human-readable and may name the offending value.
- `details` is structured context the agent can branch on (e.g., for `INVALID_RECIPIENT` in the email example, the list of bad addresses).

**Common error codes** (specific tools list theirs in §5):

| Code | When |
|------|------|
| `INVALID_ID_FORMAT` | An ID doesn't match its expected prefix or length. |
| `DATASOURCE_NOT_FOUND` | A `ds_…` ID has no row. |
| `WORKBOOK_NOT_FOUND` | A `wb_…` ID has no row. |
| `VIEW_NOT_FOUND` | A `vw_…` ID has no row. |
| `FIELD_NOT_FOUND` | A field name or `fld_…` ID isn't on the referenced datasource. |
| `PROJECT_NOT_FOUND` | A `prj_…` ID has no row. |
| `DATASOURCE_OFFLINE` | The datasource's `status` is `offline`; queries and renders that touch it are rejected. |
| `EXTRACT_STALE` | Returned **as a warning, not an error** when an extract is past its staleness threshold (see §8). |
| `QUERY_TIMEOUT` | A structured query or SQL exceeded the 5-second budget. |
| `RESULT_TOO_LARGE` | Result row count exceeded 1,000,000 hard limit. |
| `INVALID_CURSOR` | Cursor doesn't match the supplied query params. |
| `INVALID_PAGE_SIZE` | `page_size` is `< 1` or `> 100`. |
| `INVALID_FILTER_OPERATOR` | Operator not legal for the field type. |
| `INVALID_FILTER_VALUE` | Value can't be parsed as the field type. |
| `INVALID_SQL` | `run_sql` SQL fails to parse, references unauthorized tables, or contains write statements. |
| `UNSUPPORTED_CHART_TYPE` | A chart type outside the v1 set (`bar`, `pie`, `treemap`, `line`, `histogram`). |
| `RENDER_FAILED` | Renderer raised; details include the underlying renderer message. |
| `EXPORT_TOO_LARGE` | Export would exceed 100MB. |

### 4.5 Naming, Casing, and Strings

- All tool names in `snake_case`.
- All field names in tool inputs/outputs in `snake_case` (technical names from the datasource are passed through as-is).
- Names of datasources/workbooks/views are arbitrary user strings (mixed case, spaces) and matched case-insensitively in fuzzy search.
- Emoji and Unicode in names are preserved unchanged.

### 4.6 Response Sizing & Context Budget

Every tool response is sized with the agent's context window in mind. The agent has finite tokens; bloated responses force pagination loops, token spend, and degraded reasoning. The numbers below are deliberate, not arbitrary.

**Per-call token budgets (rough targets, not hard caps):**

| Tool category | Typical | Max worst-case |
|---|---|---|
| `list_*` (default page) | ~1–4 KB / ~250–1000 tokens | ~16 KB / ~4000 tokens (page_size=100, large summaries) |
| `get_workbook`, `get_datasource` | ~3–8 KB | ~24 KB (60-field datasource, 12-view workbook) |
| `get_view` (with stats + 250-row data series) | ~6–15 KB | ~40 KB (250 rows × 6 columns) |
| `query_datasource` (default page=100) | ~5–20 KB | ~80 KB (1000-row max page) |
| `search_lookout` | ~1–3 KB | ~6 KB (limit=25) |
| `render_*`, `export_*` | <1 KB (just metadata + path) | <1 KB |
| Error responses | <1 KB | <2 KB |

The agent reads files (renders, exports) only when it explicitly needs them - file paths in responses are tiny but the file bytes are not in the response payload.

**Knobs that bound response size:**

1. **List page sizes default to 25, max 100.** Set deliberately so a typical task finishes in 1–3 paginated calls. Page size 100 only when the agent knows the entity count is small (e.g., projects).
2. **`query_datasource` page size defaults to 100, max 1000.** Higher than list tools because each row is small (~50–200 bytes). Limit defaults to 10,000 *post-aggregation* rows, hard cap 1,000,000.
3. **`get_view.data_series` is capped at 250 rows.** A view with more is rare for a chart (a bar chart with 250 bars is already unreadable); when it happens, the response sets `data_series_truncated: true` and the agent knows to use `apply_view_filters` for the full data.
4. **`search_lookout` is intentionally not paginated.** The agent should refine the query rather than walk a long ranked list. Default limit 10, max 25.
5. **No tool returns raw bulk data unprompted.** Bulk extraction always goes through `export_*`, which returns a path, not bytes. The brief calls this out: "Bulk data dumps... waste context. That's API behavior, not MCP behavior."

**What's deliberately *not* paginated, and why:**

- **Datasource fields in `get_datasource`.** A 60-field datasource is ~5 KB; partial schemas force an extra call before any query. The agent always wants the full schema.
- **Views inside `get_workbook`.** Workbooks have at most 12 views (per the brief); their summaries are ~150 bytes each, so the full list is ~2 KB.
- **Trend entries in `summary_stats`.** One trend per measure series; almost always 1–3 entries.

**Patterns that prevent pagination loops:**

- Fuzzy search by name first (`search_lookout`), drill in with `get_*` second.
- Filter aggressively before paginating: `query_datasource` with a date range and `group_by` returns 5–50 rows, not 50,000.
- Use `include_total_count` only when the agent will act on the count - it forces a second query.

The agent doesn't need to plan token budgets explicitly; the tool design absorbs that work. But the contract above is what each tool implementer must hold to.

---

## 5. Tool Surface

**Tool count:** 16. The set is grouped into Discovery (1–7), Search (8), Querying (9–10), View operations (11), Rendering (12–13), Export (14–15), and Operations (16).

**A note on tool descriptions.** The `**Description**` text under each tool is the *exact text the model sees in the tool list*. It's written to teach the agent (a) the one-line purpose, (b) when to use this tool versus its closest sibling, and (c) any non-obvious constraint (e.g., "fuzzy matching is supported", "this tool can return paths to files on disk"). It deliberately avoids prescriptive multi-step instructions - the agent should still reason about the workflow.

---

### 5.1 `list_datasources`

**Description:**

Lists published datasources available in this Lookout environment. Returns paginated summaries (id, name, description, connection type, status, row count, last refresh time). Use this to discover what data is queryable when you don't know which datasource you need; if you already have a datasource id, prefer `get_datasource` for the full schema. Datasources may be `offline` - those will still appear in this list, with `status: "offline"`, so you can tell the user a relevant source is unavailable.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `page_size` | `integer` | No | Default 25, max 100. |
| `cursor` | `string` | No | Opaque cursor from a previous page. |
| `name_query` | `string` | No | Substring; case-insensitive; matched against name and description. For richer matching across all entity types, use `search_lookout`. |
| `connection_type` | `"live" \| "extract"` | No | Filter by connection type. |
| `status` | `"online" \| "offline"` | No | Filter by health. Agents rarely need to filter on this. |
| `sort_by` | `"name" \| "row_count" \| "last_refreshed_at"` | No | Defaults to `name` ascending. |

**Output:**

```json
{
  "items": [
    {
      "id": "ds_9f2c1ab847e3d0b412a7f6c8",
      "name": "Sales Orders",
      "description": "Order-level fact table covering all retail and online channels since 2023.",
      "connection_type": "live",
      "status": "online",
      "last_refreshed_at": null,
      "row_count": 50412,
      "field_count": 12
    }
  ],
  "next_cursor": "eyJvIjoyNX0=",
  "total_count": 10
}
```

**Error modes:**

| Code | When |
|------|------|
| `INVALID_PAGE_SIZE` | `page_size` is out of range. |
| `INVALID_CURSOR` | Cursor is malformed or doesn't match the supplied query params. |

**Documentation notes:**

- `last_refreshed_at` is `null` for `connection_type: "live"` (live sources are never "refreshed" because they read at query time).
- Counts (`row_count`, `field_count`, `total_count`) are precomputed and cheap.
- Order is stable: `(sort_by, id)` so cursor pagination is deterministic.

**Example call:**

```json
{
  "tool": "list_datasources",
  "input": { "page_size": 25 }
}
```

**Example response:** as in **Output** above.

---

### 5.2 `get_datasource`

**Description:**

Returns full details for a single datasource, including the complete field schema (technical names, data types, semantic roles, default aggregations, format strings, descriptions). Use this whenever you're about to query a datasource and don't already know its fields, or when you need to tell a user what columns are available. Do not call this in a loop over the result of `list_datasources` - call it only for the source(s) you actually need.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `datasource_id` | `string` | Yes | Must be `ds_…` format. |

**Output:**

```json
{
  "id": "ds_9f2c1ab847e3d0b412a7f6c8",
  "name": "Sales Orders",
  "description": "Order-level fact table covering all retail and online channels since 2023.",
  "connection_type": "live",
  "status": "online",
  "last_refreshed_at": null,
  "stale_after_minutes": null,
  "is_stale": false,
  "row_count": 50412,
  "fields": [
    {
      "id": "fld_3a8b2e17c4f9d1e60b5a8273",
      "name": "order_id",
      "display_name": "Order ID",
      "data_type": "string",
      "semantic_role": "dimension",
      "default_aggregation": null,
      "format_string": null,
      "description": "Unique identifier per order. Format: ORD-YYYYMMDD-NNNNN."
    },
    {
      "id": "fld_5c9d4f82e6a1b3d7c0f924e1",
      "name": "gross_revenue",
      "display_name": "Gross Revenue",
      "data_type": "float",
      "semantic_role": "measure",
      "default_aggregation": "sum",
      "format_string": "$#,##0.00",
      "description": "Order subtotal before discount. USD."
    }
    /* ... */
  ]
}
```

**Error modes:**

| Code | When |
|------|------|
| `INVALID_ID_FORMAT` | `datasource_id` doesn't match `ds_<24hex>`. |
| `DATASOURCE_NOT_FOUND` | No datasource with that id. |

**Documentation notes:**

- This tool returns the **full** field list. If a datasource has 60 fields they are all returned - we do not paginate fields because the field list is the agent's primary input for query planning, and partial schemas would force unnecessary follow-ups.
- `is_stale` is `true` when `connection_type = "extract"` and `(now - last_refreshed_at) > stale_after_minutes`. A stale extract is still queryable; the staleness is a *warning*, surfaced again at query time. Agents may want to call `refresh_datasource` first if recency matters for the user's question.

**Example call:**

```json
{
  "tool": "get_datasource",
  "input": { "datasource_id": "ds_9f2c1ab847e3d0b412a7f6c8" }
}
```

---

### 5.3 `list_projects`

**Description:**

Lists projects (folders) that organize workbooks. Each Lookout environment typically has 4–10 projects covering business areas (e.g. "Sales Operations", "Marketing", "Finance"). Use this when you want to scope `list_workbooks` to a specific business area, or to give the user an overview of how the environment is organized.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `page_size` | `integer` | No | Default 25, max 100. |
| `cursor` | `string` | No | Opaque cursor from a previous page. |

**Output:**

```json
{
  "items": [
    {
      "id": "prj_7d3f1a9c5b2e8f0d4c6a8e91",
      "name": "Sales Operations",
      "description": "Pipeline, quota, and same-store sales analytics.",
      "workbook_count": 9
    }
  ],
  "next_cursor": null,
  "total_count": 7
}
```

**Error modes:** `INVALID_PAGE_SIZE`, `INVALID_CURSOR`.

**Documentation notes:** `workbook_count` is included for at-a-glance navigation. For workbook details, follow up with `list_workbooks` filtered by `project_id`.

**Example call:**

```json
{
  "tool": "list_projects",
  "input": { "page_size": 25 }
}
```

**Example success response:** as in **Output** above.

---

### 5.4 `list_workbooks`

**Description:**

Lists workbooks (collections of related views). Returns paginated summaries with id, title, description, owner, project, primary datasource, and last update time. Use this to discover dashboards and reports. Filter by `project_id` to scope to a business area, or by `name_query` to find by partial title. For richer cross-entity search by name, prefer `search_lookout`.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `page_size` | `integer` | No | Default 25, max 100. |
| `cursor` | `string` | No | Opaque cursor from a previous page. |
| `project_id` | `string` | No | `prj_…`. Restricts to one project. |
| `name_query` | `string` | No | Case-insensitive substring on workbook name and description. |
| `owner` | `string` | No | Exact, case-insensitive owner name. |
| `sort_by` | `"name" \| "updated_at" \| "created_at"` | No | Default `updated_at desc`. |

**Output:**

```json
{
  "items": [
    {
      "id": "wb_5c9d4f82e6a1b3d7c0f924e1",
      "name": "Executive Monday Review",
      "description": "Cross-functional snapshot for Monday leadership sync.",
      "owner_name": "Priya Iyer",
      "project": { "id": "prj_7d3f...", "name": "Executive" },
      "primary_datasource": { "id": "ds_9f2c...", "name": "Sales Orders" },
      "view_count": 8,
      "updated_at": "2026-05-04T16:21:00Z",
      "thumbnail_path": "fs/thumbnails/workbooks/wb_5c9d4f82e6a1b3d7c0f924e1.png"
    }
  ],
  "next_cursor": null,
  "total_count": 42
}
```

**Error modes:** `INVALID_PAGE_SIZE`, `INVALID_CURSOR`, `INVALID_ID_FORMAT`, `PROJECT_NOT_FOUND`.

**Documentation notes:**

- The `thumbnail_path` is a precomputed 320×200 PNG (read it via the `fs` reference if you want to show the user a preview without rendering).
- `primary_datasource` is informational; a workbook can contain views from multiple datasources. The full list is in `get_workbook`.

**Example call:**

```json
{
  "tool": "list_workbooks",
  "input": {
    "project_id": "prj_7d3f1a9c5b2e8f0d4c6a8e91",
    "name_query": "executive",
    "page_size": 10
  }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "PROJECT_NOT_FOUND",
    "message": "No project exists with id 'prj_doesnotexist00000000000'.",
    "details": { "project_id": "prj_doesnotexist00000000000" }
  }
}
```

---

### 5.5 `get_workbook`

**Description:**

Returns full details for a workbook: title, description, owner, project, every view's summary (id, name, chart type, datasource), timestamps, and the path to a precomputed thumbnail. Use this when the user asks about the contents of a specific workbook ("what's in the executive dashboard?") or you need to find a particular chart inside one. For a full chart definition with axis labels, default filters, and summary statistics, follow up with `get_view`.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `workbook_id` | `string` | Yes | `wb_…`. |

**Output:**

```json
{
  "id": "wb_5c9d4f82e6a1b3d7c0f924e1",
  "name": "Executive Monday Review",
  "description": "Cross-functional snapshot for Monday leadership sync.",
  "owner_name": "Priya Iyer",
  "project": { "id": "prj_7d3f...", "name": "Executive" },
  "primary_datasource": { "id": "ds_9f2c...", "name": "Sales Orders" },
  "created_at": "2025-11-14T08:00:00Z",
  "updated_at": "2026-05-04T16:21:00Z",
  "thumbnail_path": "fs/thumbnails/workbooks/wb_5c9d4f82e6a1b3d7c0f924e1.png",
  "views": [
    {
      "id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
      "name": "Revenue by Region",
      "chart_type": "bar",
      "datasource": { "id": "ds_9f2c...", "name": "Sales Orders" },
      "position": 1
    },
    {
      "id": "vw_4d8a1f5e2c9b3d6e7f0a8154",
      "name": "Pipeline Health by Stage",
      "chart_type": "treemap",
      "datasource": { "id": "ds_4e2c...", "name": "Pipeline Health" },
      "position": 2
    }
    /* ... up to 12 views ... */
  ]
}
```

**Error modes:** `INVALID_ID_FORMAT`, `WORKBOOK_NOT_FOUND`.

**Documentation notes:** The view list is *summary only*. Default filters, axis labels, and config blobs require `get_view`.

**Example call:**

```json
{
  "tool": "get_workbook",
  "input": { "workbook_id": "wb_5c9d4f82e6a1b3d7c0f924e1" }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "WORKBOOK_NOT_FOUND",
    "message": "No workbook exists with id 'wb_doesnotexist000000000000'.",
    "details": { "workbook_id": "wb_doesnotexist000000000000" }
  }
}
```

---

### 5.6 `list_views`

**Description:**

Lists views (charts) globally or scoped to a workbook, datasource, or chart type. Use this when you're hunting for a chart by partial name without knowing the workbook (e.g. "find the pipeline health chart"), or when you want every line chart in the environment. If you already know the workbook, `get_workbook` returns the same view summaries as part of its response - prefer that to avoid an extra call.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `page_size` | `integer` | No | Default 25, max 100. |
| `cursor` | `string` | No | |
| `workbook_id` | `string` | No | `wb_…`. Scope to one workbook. |
| `datasource_id` | `string` | No | `ds_…`. Find views built on a particular source. |
| `chart_type` | `"bar" \| "pie" \| "treemap" \| "line" \| "histogram"` | No | |
| `name_query` | `string` | No | Case-insensitive substring on view name. |
| `sort_by` | `"name" \| "workbook_name"` | No | Default `name`. |

**Output:**

```json
{
  "items": [
    {
      "id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
      "name": "Revenue by Region",
      "chart_type": "bar",
      "workbook": { "id": "wb_5c9d...", "name": "Executive Monday Review" },
      "datasource": { "id": "ds_9f2c...", "name": "Sales Orders" }
    }
  ],
  "next_cursor": null,
  "total_count": 312
}
```

**Error modes:** `INVALID_PAGE_SIZE`, `INVALID_CURSOR`, `INVALID_ID_FORMAT`, `WORKBOOK_NOT_FOUND`, `DATASOURCE_NOT_FOUND`.

**Documentation notes:** Multiple filters AND together. To find "all line charts in the Sales Operations project," combine `list_workbooks` (project_id) → for each workbook, `list_views` filtered by `chart_type=line` - though `search_lookout` may be faster for fuzzy matches.

**Example call:**

```json
{
  "tool": "list_views",
  "input": {
    "datasource_id": "ds_9f2c1ab847e3d0b412a7f6c8",
    "chart_type": "line",
    "page_size": 10
  }
}
```

**Example success response:** as in **Output** above.

---

### 5.7 `get_view`

**Description:**

Returns full details for a single view: title, description, chart type, axis labels, datasource reference, default filters, full chart configuration, the data series the chart plots (rows post-aggregation), and summary statistics including min, max, mean, median, stddev, and trend direction for time-series charts. Also returns the path to a precomputed thumbnail. Use this whenever you need to understand what a chart shows without rendering it, or before applying filter overrides via `apply_view_filters`. The data series and stats are computed at call time and reflect the view's default filters.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `view_id` | `string` | Yes | `vw_…`. |
| `include_summary_stats` | `boolean` | No | Default `true`. Set `false` to skip the stats query if you only need metadata. |
| `include_data_series` | `boolean` | No | Default `true`. Set `false` if you only need metadata + stats and don't need the actual data points. |
| `data_series_limit` | `integer` | No | Default 250, max 1000. If the view's natural row count exceeds this, the response sets `data_series_truncated: true`; use `apply_view_filters` for the full set. |

**Output:**

```json
{
  "id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
  "name": "Revenue by Region",
  "description": "Total gross revenue grouped by sales region for the trailing 12 weeks.",
  "chart_type": "bar",
  "workbook": { "id": "wb_5c9d...", "name": "Executive Monday Review" },
  "datasource": { "id": "ds_9f2c...", "name": "Sales Orders" },
  "config": {
    "x_axis": { "field": "region", "label": "Region" },
    "y_axis": { "field": "gross_revenue", "label": "Gross Revenue (USD)", "aggregation": "sum" },
    "color": null,
    "group_by": ["region"],
    "sort": { "field": "gross_revenue", "direction": "desc" },
    "limit": null
  },
  "default_filters": [
    {
      "field": "order_date",
      "operator": "between",
      "value": ["2026-02-09", "2026-05-04"]
    }
  ],
  "thumbnail_path": "fs/thumbnails/views/vw_2b8e7c1f9a3d6e0c4b5f8273.png",
  "data_series": {
    "columns": [
      { "name": "region",         "data_type": "string", "role": "x" },
      { "name": "gross_revenue",  "data_type": "float",  "role": "y", "aggregation": "sum" }
    ],
    "rows": [
      { "region": "NA",    "gross_revenue": 9842301.45 },
      { "region": "EMEA",  "gross_revenue": 6128094.70 },
      { "region": "LATAM", "gross_revenue": 4391280.60 },
      { "region": "APAC",  "gross_revenue": 3204189.85 },
      { "region": "MEA",   "gross_revenue": 1872342.10 }
    ],
    "row_count": 5,
    "data_series_truncated": false
  },
  "summary_stats": {
    "row_count": 5,
    "columns": [
      {
        "name": "gross_revenue",
        "aggregation": "sum",
        "min": 1872342.10,
        "max": 9842301.45,
        "mean": 5128109.22,
        "median": 4391280.60,
        "stddev": 3004112.53
      }
    ],
    "categorical": {
      "region": { "distinct_count": 5, "top": [["NA", 9842301.45], ["EMEA", 6128094.70]] }
    },
    "trends": []
  }
}
```

For a `line` chart over time, `summary_stats.trends` is populated:

```json
"trends": [
  {
    "series": "gross_revenue",
    "x_field": "order_date",
    "direction": "up",                       // 'up' | 'down' | 'flat'
    "slope_per_unit": 12842.15,              // OLS slope; units = y per x-unit (e.g. USD/day)
    "percent_change": 0.183,                 // (last - first) / first
    "first_value": 142891.20,
    "last_value": 169073.35,
    "first_x": "2026-02-09",
    "last_x": "2026-05-04"
  }
]
```

**Error modes:** `INVALID_ID_FORMAT`, `VIEW_NOT_FOUND`, `DATASOURCE_OFFLINE` (if the view's datasource is offline; data series and summary stats can't be computed). When the datasource is `EXTRACT_STALE`, the response is returned with a top-level `warnings: [{ "code": "EXTRACT_STALE", ... }]` array rather than an error - the agent can still see the (stale) data and stats.

**Documentation notes:**

- `data_series.rows` is the post-aggregation rows that would be plotted on the chart. For a bar chart, one row per bar. For a line chart, one row per (x, optional series) pair. For a pie chart, one row per slice. For a treemap, one row per leaf rectangle. For a histogram, one row per bin (with `bin_lo`, `bin_hi`, `count`).
- `data_series.columns` carries roles (`x`, `y`, `series`, `size`, `color`) so the agent knows what each column means without re-reading the config.
- `summary_stats.columns` lists stats for measure fields used in the view. `summary_stats.categorical` lists distinct counts and top-5 for dimension fields.
- `summary_stats.trends` is populated only for charts whose x-axis field is a `date` or `datetime` (typically `line` charts, occasionally `bar` charts with a date dimension). `direction` is `flat` when `|percent_change| < 0.02`. For multi-series charts, one trend entry per series.
- The view's `default_filters` are applied when computing data series and stats. To use different filters, call `apply_view_filters` (which returns the same data + stats payload but with filter overrides).

**Example call:**

```json
{
  "tool": "get_view",
  "input": { "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273" }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "VIEW_NOT_FOUND",
    "message": "No view exists with id 'vw_doesnotexist000000000000'.",
    "details": { "view_id": "vw_doesnotexist000000000000" }
  }
}
```

---

### 5.8 `search_lookout`

**Description:**

Cross-entity fuzzy search across datasources, workbooks, and views by name and description. Returns a single ranked list with each item tagged by `kind`. Use this as your first call when the user names a thing without identifying its type ("show me the pipeline health stuff", "find the Q1 revenue dashboard"). Use entity-specific list tools when you already know the type. Matching uses case-insensitive substring + token-prefix scoring; results are scored 0.0–1.0.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `query` | `string` | Yes | At least 2 characters. |
| `kinds` | `("datasource" \| "workbook" \| "view")[]` | No | Defaults to all three. |
| `limit` | `integer` | No | Default 10, max 25. Search is not paginated - refine the query if you don't see what you want. |

**Output:**

```json
{
  "results": [
    {
      "kind": "view",
      "id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
      "name": "Pipeline Health by Stage",
      "context": "in workbook 'Executive Monday Review'",
      "score": 0.94
    },
    {
      "kind": "workbook",
      "id": "wb_5c9d...",
      "name": "Pipeline Health Dashboard",
      "context": "in project 'Sales Operations'",
      "score": 0.91
    },
    {
      "kind": "datasource",
      "id": "ds_4e2c...",
      "name": "Pipeline Health",
      "context": "extract, last refreshed 2026-05-05T06:00:00Z",
      "score": 0.88
    }
  ]
}
```

**Error modes:** `INVALID_QUERY` (less than 2 characters).

**Documentation notes:**

- Score is a relative ranking signal, not an absolute confidence. Items with score ≥ 0.7 are usually the right hit; lower-scored items are worth surfacing as alternatives.
- Search is best-effort. If the user's exact intent isn't returned, fall back to the relevant `list_*` tool with `name_query`.
- Search does not match field names within datasources. To find "the column named `gross_revenue`," call `get_datasource` and inspect its fields.

**Example call:**

```json
{
  "tool": "search_lookout",
  "input": { "query": "pipeline health", "limit": 5 }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "INVALID_QUERY",
    "message": "Search query must be at least 2 characters.",
    "details": { "query": "p" }
  }
}
```

---

### 5.9 `query_datasource`

**Description:**

Runs a structured query against a datasource and returns paginated rows plus column metadata. This is the primary tool for data retrieval and aggregation: select fields, apply filters, group, aggregate, sort, and limit - all without writing SQL. Prefer this over `run_sql` whenever the question can be expressed structurally; it's safer, validates inputs against the schema, and produces clearer errors. Use `run_sql` only for queries that require expressions or joins this tool can't express.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `datasource_id` | `string` | Yes | `ds_…`. |
| `select` | `Select[]` | Yes | At least one item. See below. |
| `filters` | `Filter[]` | No | List of filters; AND-ed. See §4.3. |
| `group_by` | `string[]` | No | Field technical names. Required if `select` has any aggregated entries that aren't `count(*)`. |
| `order_by` | `OrderBy[]` | No | Each `{ field, direction }`; up to 3. |
| `limit` | `integer` | No | Max rows to return *after* aggregation. Default 10000, hard cap 1000000. |
| `page_size` | `integer` | No | Default 100, max 1000. |
| `cursor` | `string` | No | Opaque cursor from previous page. |
| `include_total_count` | `boolean` | No | Default `false`. If `true`, runs an extra `COUNT(*)` and includes `total_count` in the response. |

**`Select` shape:**

```json
{ "field": "gross_revenue", "aggregation": "sum", "alias": "total_revenue" }
```

- `field`: technical name from the datasource. `*` legal only for `count`.
- `aggregation`: one of `sum`, `avg`, `count`, `count_distinct`, `min`, `max`, or `null` (no aggregation, raw column).
- `alias`: optional output column name; defaults to the field name (or `<aggregation>_<field>` for aggregated columns).

**Output:**

```json
{
  "query_id": "qry_4e2c8a1f6b9d3e5c7f024918",
  "columns": [
    { "name": "region", "data_type": "string" },
    { "name": "total_revenue", "data_type": "float", "aggregation": "sum" }
  ],
  "rows": [
    { "region": "NA",   "total_revenue": 9842301.45 },
    { "region": "EMEA", "total_revenue": 6128094.70 },
    { "region": "APAC", "total_revenue": 4391280.60 }
  ],
  "row_count": 3,
  "total_count": 5,
  "next_cursor": "eyJvIjoxMDB9",
  "execution_time_ms": 42,
  "warnings": []
}
```

**Error modes:**

| Code | When |
|------|------|
| `INVALID_ID_FORMAT` | Bad ID. |
| `DATASOURCE_NOT_FOUND` | No such datasource. |
| `DATASOURCE_OFFLINE` | Datasource status is `offline`. |
| `FIELD_NOT_FOUND` | A name in `select`, `filters`, `group_by`, or `order_by` doesn't exist on the datasource. |
| `INVALID_AGGREGATION` | Aggregation not legal for the field's type (e.g. `sum` on a string column). |
| `MISSING_GROUP_BY` | `select` has both aggregated and non-aggregated columns and `group_by` is missing or doesn't include all non-aggregated columns. |
| `INVALID_FILTER_OPERATOR` / `INVALID_FILTER_VALUE` | See §4.3. |
| `QUERY_TIMEOUT` | Underlying SQLite query exceeded the 5-second budget. |
| `RESULT_TOO_LARGE` | Result row count exceeds `limit` (default 10000) - agent should add filters or aggregation. |
| `INVALID_PAGE_SIZE` / `INVALID_CURSOR` | See §4.2. |

**Documentation notes:**

- `query_id` is returned for reference/debugging. To export a full result set that's too large to page through, pass the same `structured_query` to `export_query_results` - it runs the query once and writes the output to disk in one call.
- `warnings` may contain `EXTRACT_STALE` (the data is older than the staleness threshold) and similar non-fatal advisories. The agent should surface this to the user when it materially affects the answer.
- For period-over-period comparisons, run two queries with the two date ranges and have your reasoning compare the results - there is no dedicated "compare" tool.
- Aggregation rules for `count`: `count(*)` counts rows, `count(field)` counts non-null values of that field, `count_distinct(field)` counts distinct non-null values.

**Example call:**

```json
{
  "tool": "query_datasource",
  "input": {
    "datasource_id": "ds_9f2c1ab847e3d0b412a7f6c8",
    "select": [
      { "field": "region" },
      { "field": "gross_revenue", "aggregation": "sum", "alias": "total_revenue" }
    ],
    "filters": [
      { "field": "order_date", "operator": "between", "value": ["2026-01-01", "2026-03-31"] }
    ],
    "group_by": ["region"],
    "order_by": [{ "field": "total_revenue", "direction": "desc" }],
    "limit": 10
  }
}
```

**Example success:** as in **Output** above.

**Example error:**

```json
{
  "error": {
    "code": "MISSING_GROUP_BY",
    "message": "Selected 'region' (no aggregation) alongside aggregated 'sum(gross_revenue)' but 'region' is not in group_by.",
    "details": { "missing_in_group_by": ["region"] }
  }
}
```

---

### 5.10 `run_sql`

**Description:**

Executes a read-only SQL `SELECT` against a single datasource's underlying table. Use this only when `query_datasource` cannot express the query - typically window functions, multi-step CTEs, or string-manipulation expressions. The query runs against a sandboxed read-only connection scoped to the datasource's data table; you cannot reference other tables or perform writes. This tool is more powerful than `query_datasource` but easier to misuse: errors are less specific, and queries that work in your head may not match the table's actual schema.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `datasource_id` | `string` | Yes | `ds_…`. |
| `sql` | `string` | Yes | A single SQL `SELECT` statement. The datasource's table is exposed under the alias `data`. |
| `page_size` | `integer` | No | Default 100, max 1000. |
| `cursor` | `string` | No | Opaque cursor from previous page. |

**Output:** Same shape as `query_datasource` (columns, rows, row_count, next_cursor, execution_time_ms, query_id, warnings).

**Error modes:**

| Code | When |
|------|------|
| `INVALID_SQL` | SQL fails to parse, contains writes (`INSERT`/`UPDATE`/`DELETE`/`DDL`/`PRAGMA`), uses multiple statements, or references tables other than `data`. `details.parser_message` carries the SQLite error. |
| `DATASOURCE_NOT_FOUND` / `DATASOURCE_OFFLINE` | As elsewhere. |
| `QUERY_TIMEOUT` | Exceeded 5-second budget. |
| `RESULT_TOO_LARGE` | Result has more than 1,000,000 rows. |

**Documentation notes:**

- The table is aliased to `data`. Example: `SELECT region, SUM(gross_revenue) FROM data GROUP BY region;` - *not* `FROM sales_orders`.
- Column names in `data` are the field technical names from `get_datasource`. Use those; aliases assigned on output do not change input column names.
- The connection is opened with `?mode=ro&immutable=1` semantics so writes are physically impossible; `INVALID_SQL` is the early signal.
- Window functions, `WITH` CTEs, `CASE`, and SQLite's date functions all work. `JOIN`s do not - `data` is the only table in scope.
- Don't use this to look up entity metadata. Use `list_*` and `get_*` tools instead - they read structured tables that are not exposed here.

**Example call:**

```json
{
  "tool": "run_sql",
  "input": {
    "datasource_id": "ds_9f2c1ab847e3d0b412a7f6c8",
    "sql": "SELECT region, SUM(gross_revenue) AS total_revenue, RANK() OVER (ORDER BY SUM(gross_revenue) DESC) AS rank FROM data WHERE order_date BETWEEN '2026-01-01' AND '2026-03-31' GROUP BY region;"
  }
}
```

---

### 5.11 `apply_view_filters`

**Description:**

Re-runs a view with overridden or additional filters and returns the resulting tabular data plus refreshed summary statistics. Use this whenever the user asks "what about Q1 only?" or "filter to NA region" against an existing chart - it's faster than reconstructing the view's query yourself. The view definition itself is not mutated; this is an ephemeral re-run. To render the filtered view as an image, follow up with `render_view` passing the same filter overrides.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `view_id` | `string` | Yes | `vw_…`. |
| `filter_overrides` | `Filter[]` | No | Filters to apply. By default these *replace* the view's default filters on the same fields and *add* filters on new fields. See `merge_strategy`. |
| `merge_strategy` | `"override" \| "merge_replace_per_field" \| "additive"` | No | Default `merge_replace_per_field`: per-field, overrides win. `override` discards all defaults. `additive` keeps all defaults and ANDs the overrides. |
| `page_size` | `integer` | No | Default 100, max 1000. |
| `cursor` | `string` | No | |

**Output:**

```json
{
  "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
  "applied_filters": [
    { "field": "order_date", "operator": "between", "value": ["2026-01-01", "2026-03-31"] },
    { "field": "region", "operator": "in", "value": ["NA"] }
  ],
  "columns": [ /* as in query_datasource */ ],
  "rows": [ /* ... */ ],
  "row_count": 1,
  "next_cursor": null,
  "summary_stats": { /* refreshed for these filters */ },
  "execution_time_ms": 38
}
```

**Error modes:** `INVALID_ID_FORMAT`, `VIEW_NOT_FOUND`, `DATASOURCE_OFFLINE`, `FIELD_NOT_FOUND`, `INVALID_FILTER_OPERATOR`, `INVALID_FILTER_VALUE`, `QUERY_TIMEOUT`, `RESULT_TOO_LARGE`.

**Documentation notes:**

- The default `merge_replace_per_field` is the most natural to reason about: if the view already filters `order_date` and the override sets a new `order_date` range, the new range wins. Filters on fields not previously filtered are *added* under AND.
- The returned `applied_filters` always reflects the merged result. The agent should inspect this field to confirm which filters actually ran.
- `summary_stats` is recomputed for the new filter set. Cost is similar to `get_view`'s stats.

**Example call:**

```json
{
  "tool": "apply_view_filters",
  "input": {
    "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
    "filter_overrides": [
      { "field": "order_date", "operator": "between", "value": ["2026-01-01", "2026-03-31"] },
      { "field": "region", "operator": "in", "value": ["NA"] }
    ]
  }
}
```

---

### 5.12 `render_view`

**Description:**

Generates an image of a single view (chart) and writes it to disk. Returns the file path - read it via the `fs` reference if you want to show or attach the image. Supports PNG and SVG, configurable width and height, and optional filter overrides applied at render time without mutating the view definition. Use this when the user wants to *see* the chart, not the underlying data; if they want the data, use `get_view` (with summary stats) or `apply_view_filters`.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `view_id` | `string` | Yes | `vw_…`. |
| `format` | `"png" \| "svg"` | No | Default `png`. |
| `width` | `integer` | No | Default 800; min 320; max 2400. |
| `height` | `integer` | No | Default 600; min 240; max 1800. |
| `filter_overrides` | `Filter[]` | No | Same as `apply_view_filters`; applied with `merge_replace_per_field` strategy. |
| `theme` | `"light" \| "dark"` | No | Default `light`. |

**Output:**

```json
{
  "render_id": "rnd_1c5e9a3b7f0d4e6c2a8f1294",
  "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
  "file_path": "fs/renders/rnd_1c5e9a3b7f0d4e6c2a8f1294.png",
  "format": "png",
  "width": 800,
  "height": 600,
  "byte_size": 142318,
  "expires_at": "2026-05-07T13:42:01Z",
  "applied_filters": [ /* the merged filter set actually used */ ],
  "warnings": []
}
```

**Error modes:** `INVALID_ID_FORMAT`, `VIEW_NOT_FOUND`, `DATASOURCE_OFFLINE`, `INVALID_FILTER_OPERATOR`, `INVALID_FILTER_VALUE`, `INVALID_DIMENSIONS` (width/height out of range), `RENDER_FAILED` (renderer error; details include the underlying message).

**Documentation notes:**

- The file at `file_path` is written before the response returns - the agent can read it immediately.
- Renders are cached in `fs/renders/` for 24h by `expires_at`.
- For workbook-level rendering ("show me the executive dashboard"), use `render_workbook`. Calling this tool repeatedly for each view of a workbook is allowed but slower and produces N images instead of one composite.

**Example call:**

```json
{
  "tool": "render_view",
  "input": {
    "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
    "format": "png",
    "width": 1024,
    "height": 768,
    "filter_overrides": [
      { "field": "region", "operator": "in", "value": ["NA", "EMEA"] }
    ],
    "theme": "light"
  }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "DATASOURCE_OFFLINE",
    "message": "Cannot render view: underlying datasource 'Employee Directory' is offline.",
    "details": {
      "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
      "datasource_id": "ds_offlinedatasource000000",
      "datasource_name": "Employee Directory"
    }
  }
}
```

---

### 5.13 `render_workbook`

**Description:**

Generates a single composite image of a workbook by rendering its views in a grid layout. Returns the file path. Use this when the user asks for a "dashboard" view that shows multiple charts together (e.g. "render the executive dashboard for the Monday review"). For one specific chart only, use `render_view`.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `workbook_id` | `string` | Yes | `wb_…`. |
| `format` | `"png" \| "svg"` | No | Default `png`. |
| `width` | `integer` | No | Default 1600; min 800; max 3200. |
| `height` | `integer` | No | Default 1200; min 600; max 2400. |
| `theme` | `"light" \| "dark"` | No | Default `light`. |
| `view_filter_overrides` | `Record<view_id, Filter[]>` | No | Per-view overrides (e.g. apply a date range to all charts). |

**Output:**

```json
{
  "render_id": "rnd_1c5e9a3b7f0d4e6c2a8f1294",
  "workbook_id": "wb_5c9d4f82e6a1b3d7c0f924e1",
  "file_path": "fs/renders/rnd_1c5e9a3b7f0d4e6c2a8f1294.png",
  "format": "png",
  "width": 1600,
  "height": 1200,
  "byte_size": 482171,
  "view_count": 8,
  "expires_at": "2026-05-07T13:42:01Z",
  "warnings": [
    {
      "view_id": "vw_4d8a1f5e2c9b3d6e7f0a8154",
      "code": "EXTRACT_STALE",
      "message": "Underlying datasource 'Pipeline Health' is 36 hours old (threshold 24h)."
    }
  ]
}
```

**Error modes:** `INVALID_ID_FORMAT`, `WORKBOOK_NOT_FOUND`, `DATASOURCE_OFFLINE` (one or more views' datasources are offline; details list which), `INVALID_DIMENSIONS`, `RENDER_FAILED`.

**Documentation notes:**

- Layout is a 2-column or 3-column grid (chosen automatically based on view count and the requested dimensions). View titles are rendered above each chart.
- If individual views fail to render, the composite still returns with placeholders for the failures and `warnings` describing each failure - the workbook isn't all-or-nothing.

**Example call:**

```json
{
  "tool": "render_workbook",
  "input": {
    "workbook_id": "wb_5c9d4f82e6a1b3d7c0f924e1",
    "format": "png",
    "width": 1600,
    "height": 1200
  }
}
```

**Example success response:** as in **Output** above.

---

### 5.14 `export_view_data`

**Description:**

Writes the data behind a view to a CSV or JSON file under `fs/exports/`, returns the file path and metadata. Use this when the user wants the raw rows ("export the data behind this chart") rather than the chart image. Filters work the same way as `apply_view_filters` - pass `filter_overrides` to scope the export. To export an arbitrary query that isn't tied to a view, use `export_query_results` instead.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `view_id` | `string` | Yes | `vw_…`. |
| `format` | `"csv" \| "json"` | No | Default `csv`. |
| `filter_overrides` | `Filter[]` | No | |
| `merge_strategy` | `"override" \| "merge_replace_per_field" \| "additive"` | No | Default `merge_replace_per_field`. |
| `column_labels` | `"display" \| "technical"` | No | Default `display` (uses field display names as headers). `technical` uses the raw column names - pick this if a downstream system expects machine-readable headers. |

**Output:**

```json
{
  "export_id": "exp_8a4f2c1e7b3d9e5c0f482763",
  "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
  "file_path": "fs/exports/exp_8a4f2c1e7b3d9e5c0f482763.csv",
  "format": "csv",
  "row_count": 5,
  "byte_size": 412,
  "expires_at": "2026-05-07T13:42:01Z"
}
```

**Error modes:** `INVALID_ID_FORMAT`, `VIEW_NOT_FOUND`, `DATASOURCE_OFFLINE`, `INVALID_FILTER_OPERATOR`, `INVALID_FILTER_VALUE`, `EXPORT_TOO_LARGE` (export would exceed 100MB; ask the agent to add filters).

**Example call:**

```json
{
  "tool": "export_view_data",
  "input": {
    "view_id": "vw_2b8e7c1f9a3d6e0c4b5f8273",
    "format": "csv",
    "filter_overrides": [
      { "field": "order_date", "operator": "between", "value": ["2026-01-01", "2026-03-31"] }
    ],
    "column_labels": "display"
  }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "EXPORT_TOO_LARGE",
    "message": "Export would produce 87.4 MB; the limit is 100 MB. Add filters or aggregate.",
    "details": { "estimated_byte_size": 91641032, "limit": 104857600 }
  }
}
```

**Documentation notes:**

- CSV uses RFC-4180 quoting, `\r\n` line endings, UTF-8 with no BOM.
- JSON output is an array of objects keyed by the chosen column header style. Numeric values are emitted as JSON numbers; nulls as `null`; dates and datetimes as ISO-8601 strings.
- Files persist for 24h under `expires_at`.

---

### 5.15 `export_query_results`

**Description:**

Runs a query (structured or SQL) and writes the full result set to a CSV or JSON file under `fs/exports/`. Use this for arbitrary data extraction that isn't tied to an existing view, or when a `query_datasource` result is too large to page through and you want the whole thing in one file. The query is the same shape as `query_datasource` or `run_sql`. Returns the file path.

**Inputs (one of `structured_query` or `sql_query` is required):**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `datasource_id` | `string` | Yes | `ds_…`. |
| `structured_query` | `StructuredQuery` | Cond. | Same shape as the inputs of `query_datasource` (without `page_size`/`cursor`). |
| `sql_query` | `string` | Cond. | A single read-only SELECT, same constraints as `run_sql`. |
| `format` | `"csv" \| "json"` | No | Default `csv`. |
| `column_labels` | `"display" \| "technical"` | No | Default `technical` here (since query selects often alias columns the agent picked). |

**Output:**

```json
{
  "export_id": "exp_8a4f2c1e7b3d9e5c0f482763",
  "query_id": "qry_4e2c8a1f6b9d3e5c7f024918",
  "file_path": "fs/exports/exp_8a4f2c1e7b3d9e5c0f482763.csv",
  "format": "csv",
  "row_count": 50412,
  "byte_size": 5142318,
  "expires_at": "2026-05-07T13:42:01Z"
}
```

**Error modes:** Union of `query_datasource`/`run_sql` errors (`FIELD_NOT_FOUND`, `INVALID_AGGREGATION`, `INVALID_SQL`, `QUERY_TIMEOUT`, `RESULT_TOO_LARGE`, `DATASOURCE_OFFLINE`) plus `EXPORT_TOO_LARGE` and `MISSING_QUERY_INPUT` (neither `structured_query` nor `sql_query` provided) and `CONFLICTING_QUERY_INPUT` (both provided).

**Documentation notes:** This tool *runs* the query (unlike a hypothetical export-by-query-id) - it does not require a previous `query_datasource` call. That keeps the contract self-contained and predictable.

**Example call:**

```json
{
  "tool": "export_query_results",
  "input": {
    "datasource_id": "ds_9f2c1ab847e3d0b412a7f6c8",
    "structured_query": {
      "select": [
        { "field": "store_id" },
        { "field": "gross_revenue", "aggregation": "sum", "alias": "total_revenue" }
      ],
      "filters": [
        { "field": "order_date", "operator": "between", "value": ["2026-04-01", "2026-04-30"] }
      ],
      "group_by": ["store_id"],
      "order_by": [{ "field": "total_revenue", "direction": "desc" }]
    },
    "format": "csv",
    "column_labels": "technical"
  }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "MISSING_QUERY_INPUT",
    "message": "Either 'structured_query' or 'sql_query' must be provided.",
    "details": {}
  }
}
```

---

### 5.16 `refresh_datasource`

**Description:**

Triggers a refresh of an extract datasource, updating its `last_refreshed_at` to "now" and clearing any stale-data warnings. Use this only when the user explicitly asks for fresh data, or when an `EXTRACT_STALE` warning would materially affect the answer (e.g. they're asking about today's pipeline and the extract is 48 hours old). Live-connection datasources don't need refreshing - calling this on one returns `INVALID_OPERATION` rather than silently succeeding, so you don't accidentally double-up steps.

**Inputs:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `datasource_id` | `string` | Yes | `ds_…`. |

**Output:**

```json
{
  "id": "ds_4e2c8a1f6b9d3e5c7f024918",
  "name": "Pipeline Health",
  "connection_type": "extract",
  "status": "online",
  "last_refreshed_at": "2026-05-06T13:42:01Z",
  "is_stale": false,
  "duration_ms": 1842,
  "rows_loaded": 4521
}
```

**Error modes:**

| Code | When |
|------|------|
| `INVALID_ID_FORMAT` / `DATASOURCE_NOT_FOUND` | Standard. |
| `INVALID_OPERATION` | Datasource is `connection_type: "live"` - refresh is not applicable. |
| `DATASOURCE_OFFLINE` | The datasource is offline; refresh cannot complete. For v1, an offline datasource stays offline regardless of refresh attempts (a future "force_online" operation could change this). |

**Documentation notes:**

- Refresh duration is roughly proportional to the row count (approximately `row_count / 5000` seconds, capped at 5s). The duration is reported back so the agent can mirror that detail to the user.
- For v1, the refresh updates only the `last_refreshed_at` timestamp; row-level changes between refreshes are not simulated (an explicit non-goal). A future version can swap in a "scenario" system that ages data forward on refresh.

**Example call:**

```json
{
  "tool": "refresh_datasource",
  "input": { "datasource_id": "ds_4e2c8a1f6b9d3e5c7f024918" }
}
```

**Example success response:** as in **Output** above.

**Example error response:**

```json
{
  "error": {
    "code": "INVALID_OPERATION",
    "message": "Datasource 'Sales Orders' is a live connection; refresh is not applicable.",
    "details": {
      "datasource_id": "ds_9f2c1ab847e3d0b412a7f6c8",
      "connection_type": "live"
    }
  }
}
```

---

## 6. Workflow → Tool Mapping

The brief defines eight workflows. Each maps to one or a chain of tool calls.

| # | Workflow | Tool sequence |
|---|----------|---------------|
| W1 | Explore Datasources - discover and inspect schema | `list_datasources` → `get_datasource` (per source of interest) |
| W2 | View Workbook Contents - browse workbooks, open one, list its views | `list_workbooks` (or `search_lookout`) → `get_workbook` |
| W3 | Apply Filters to Views | `get_view` (to learn fields/defaults) → `apply_view_filters` (for tabular result + stats) and/or `render_view` with `filter_overrides` (for image) |
| W4 | Query Datasource - raw data retrieval | Common path: `list_datasources` → `get_datasource` → `query_datasource`. Power path: `list_datasources` → `get_datasource` → `run_sql`. |
| W5 | Compare Time Periods | Two `query_datasource` calls (or two `run_sql`) with different date filters; agent compares results. For visual comparison, two `render_view` calls with different `filter_overrides`. No dedicated `compare_periods` tool - composition keeps the surface small and the pattern general. |
| W6 | Analyze View Details - chart type, axis labels, summary stats | `get_view` (with default `include_summary_stats: true`) |
| W7 | Generate View Images | `render_view` (single chart) or `render_workbook` (composite). `filter_overrides` are accepted at render time. |
| W8 | Export View Data | `export_view_data` (for view-tied exports) or `export_query_results` (for arbitrary queries). |

**Common entry points an agent will use:**

- "What data is loaded?" → `list_datasources`
- "Find the X dashboard / chart" → `search_lookout`
- "What's in workbook Y?" → `get_workbook`
- "Show me Z data" → `query_datasource`
- "Render Z" → `render_view` or `render_workbook`
- "Export Z as CSV" → `export_view_data` or `export_query_results`
- "Fresh data on Z" → `refresh_datasource` (then re-query)

---

## 7. Assumptions

Each row records an ambiguity in the brief, the choice I made, and why.

| Question | Decision | Why |
|---|---|---|
| Does Lookout distinguish *workbooks* from *dashboards*? | No - the schema is `Datasource → Workbook → View`. A workbook is the agent's "dashboard" for rendering purposes; `render_workbook` produces a composite image. | The brief says "30 to 80 dashboards, each with 4 to 12 charts" alongside a flat workbook→view model. A separate `dashboards` table doubles modeling complexity without changing what the agent can do. Tableau's distinction (workbook contains both sheets and dashboard layouts) is unnecessary for read-only mock workflows. |
| Are dashboards ever "stand-alone" (no parent workbook)? | No. Every view has exactly one parent workbook. | Same simplification; matches the "workbooks contain views" framing. |
| Is the query surface SQL, structured, or both? | Both. Structured `query_datasource` is the primary path; read-only sandboxed `run_sql` is the escape hatch. | W5 says "either as SQL or as a chart." Structured-only would lock out window functions and CTEs that real analyst workflows need. SQL-only would be unsafe (injection, unbounded surface area) and would produce worse error messages for the typical case. |
| What chart types ship in v1? | The five named in the brief (`bar`, `pie`, `treemap`, `line`, `histogram`). | The brief says "let's keep [these] to start with." Adding more chart types is a future extension; the design is explicitly extensible (see §13). |
| What is the canonical timezone? | UTC, in ISO-8601, everywhere. | Mixed timezones would be an endless source of edge cases in a static-seed mock. UTC is the only contract that's fully unambiguous. |
| Are there projects/folders, or is the workbook namespace flat? | Projects exist as a one-level grouping. No nested projects. | Realism: real BI tools organize by business area. A single project would make `list_workbooks` ergonomically poor at the 80-workbook scale. Nesting adds little. |
| What ID format? | `<prefix>_<24 lowercase hex>` (e.g. `ds_…`, `wb_…`). | Mirrors the brief's example (`msg_9f2c1ab847e3d0b412a7f6c8`); short enough to keep responses compact, long enough for collision-free generation; prefix discriminates entity types. |
| Pagination - cursor or offset? | Opaque cursor encoding offset internally. | Cursor is the modern convention and lets us swap to keyset pagination later without breaking clients. Internally it's a base64 of a small JSON object including a hash of the query params, so cursor reuse across changed queries fails fast. |
| Filter logic - AND/OR/NOT? | AND only at the structured layer. OR/NOT require `run_sql`. | A structured filter language with full boolean support is an order of magnitude more complex (parser, validator, error messages). AND covers ~95% of agent workflows and the SQL escape hatch covers the rest. The trade-off is documented per tool. |
| Default filters on a view - replace or merge with overrides? | Per-field replacement is the default; explicit `merge_strategy` switches the behavior. | Per-field replacement matches how a human user thinks ("change this filter, keep the others"). Making it the default is the principle of least surprise. |
| Where do exports/renders live? | `fs/exports/{id}.{ext}` and `fs/renders/{id}.{ext}`, with 24-hour TTL recorded in DB and cleaned by the harness. | Two flat directories are simpler than per-entity hierarchies at this scale. The TTL is a contract with the agent - if the file might be referenced later, the agent should re-export. |
| What does `count()` mean in `query_datasource`? | `count(*)` counts rows; `count(field)` counts non-nulls; `count_distinct(field)` counts distinct non-nulls. | Matches SQL's behavior. Aliasing it differently would surprise an agent that thinks in SQL. |
| Are field metadata calls paginated? | No - `get_datasource` returns *all* fields. | Agents need the full schema to plan a query. A 60-field datasource fits easily in one response (~5KB). Pagination here would force unnecessary follow-ups. |
| Is there a dedicated "compare periods" tool? | No - composition with two `query_datasource` calls is sufficient. | The brief is explicit: "the analyst querying the data... and analyzing the returned data themselves." A dedicated tool would be a kitchen sink (date ranges, granularities, percent-change semantics) for a workflow that's already cleanly composable. |
| How long are queries allowed to run? | 5-second wall clock budget per query, 1,000,000-row hard cap, 10,000-row default soft cap (configurable per call). | A mock should still feel like a real BI tool: queries that scan 100k rows can take seconds; runaway queries (full cross-product, etc.) should be rejected. |
| Are sums of float currencies kept as floats? | Yes - `data_type: "float"`, with a `format_string` of `'$#,##0.00'` to convey decimal places to the agent. | Decimal arithmetic in SQLite is non-trivial. For a mock the float rounding error is acceptable and documented. |
| Are exports authenticated or signed URLs? | No - they're plain filesystem paths under `fs/exports/`. | The brief specifies `fs` as a harness-mounted reference; the agent receives raw paths. No URL-signing model is defined. |
| Should `get_view`/`apply_view_filters` always include summary stats? | `get_view` includes them by default with an opt-out flag; `apply_view_filters` always includes them since the agent is asking about the filtered data. | Stats are the most common reason to look at a view. Making them opt-in by default would force a second tool call most of the time. |

---

## 8. Positions on Open Questions

The brief asks one open question:

> *Are there realistic read-only failure modes (source offline, cache stale) we should expose to agents so they learn to handle them?*

**Position: yes, four modes, surfaced consistently as either errors (when fatal) or warnings (when non-fatal).**

The brief warns that "if an agent could tell it was a mock during a typical session, it's designed wrong." Real BI users encounter these failures regularly: a database goes offline, an extract is two days stale, a query times out, a result blows past the row cap. Hiding these makes the mock easier to build but worse to learn from. Agents that never encounter them will never learn to handle them.

**The four modes I model:**

1. **`DATASOURCE_OFFLINE` (fatal error).** Datasource `status` is `offline`. Surfaced from `get_datasource` (in the response payload), `query_datasource`, `run_sql`, `apply_view_filters`, `render_view`, `render_workbook`, `export_view_data`, `export_query_results`. The agent's recovery path is to call `list_datasources` to find an alternative source, or surface the outage to the user. We seed exactly one datasource (`employee_directory`) as offline so any agent doing thorough discovery encounters it.

2. **`EXTRACT_STALE` (warning, not error).** An extract whose `last_refreshed_at` is older than `stale_after_minutes`. Returned in the `warnings` array of any tool that touches that datasource. The data is still queryable, but the agent should consider calling `refresh_datasource` if recency matters. Seeded: `pipeline_health` (extract refreshed daily, seeded 36 hours old).

3. **`QUERY_TIMEOUT` (fatal error).** A query exceeds the 5-second budget. Recovery: add filters, reduce result size, or use `run_sql` with a more selective query. Mocked behavior: we don't actually time out queries; rather, queries with no filters on the largest datasource (`web_analytics`, ~90k rows) and no aggregation hit a synthetic timeout. This nudges the agent toward better query design.

4. **`RESULT_TOO_LARGE` (fatal error).** Result row count exceeds `limit` (default 10000) for `query_datasource`, or 1,000,000 for `run_sql`. Recovery: paginate, aggregate, or export. The agent learns that "give me everything" doesn't scale.

**Why these four and not more?** They cover the classes of failure that change agent behavior. Add more (e.g. `PERMISSION_DENIED`, `RATE_LIMITED`) and we drift from the "no auth, no real-time" non-goals. The four chosen are read-only, deterministic from seed state plus query shape, and each requires the agent to do something distinct in response.

**Why surface staleness as a warning rather than an error?** Errors block; warnings inform. A 36-hour-old pipeline extract is still useful for many questions (e.g. structural questions about deal stages), but useless for "what's the pipeline today?" The agent should make the call. Modeling this as a non-fatal `warnings` array - a small but distinct contract from the `error` envelope - teaches that distinction.

---

## 9. Seed Data Strategy

The seed catalog is what an agent will *actually* encounter, so it needs to be plausible, varied, and exercise every tool path.

### 9.1 Datasource Catalog (10 sources)

Sized within the brief's "5–15 sources" and "few hundred to ~100k rows" envelopes. Names, descriptions, and field schemas are realistic enough that `get_datasource` reads like a real catalog.

| # | Datasource | Type | Rows | Refresh | Status | Purpose |
|---|---|---|---|---|---|---|
| 1 | `Sales Orders` | live | 50,412 | n/a | online | Order-level fact table since 2023. Fields: `order_id, order_date, customer_id, store_id, region, channel, product_id, product_category, units, gross_revenue, discount, net_revenue`. |
| 2 | `Inventory Levels` | extract | 12,180 | daily | online | SKU × store × date. Fields: `sku, store_id, snapshot_date, on_hand_units, on_order_units, reorder_threshold, lead_time_days`. |
| 3 | `Web Analytics` | live | 89,541 | n/a | online | Session-level. Fields: `session_id, user_id, country, device_type, browser, landing_page, event_time, duration_ms, page_views, bounced, converted, conversion_revenue`. |
| 4 | `Customer Directory` | live | 8,127 | n/a | online | Customer master. Fields: `customer_id, signup_date, segment, lifetime_value, region, status, last_purchase_at`. |
| 5 | `Marketing Campaigns` | extract | 612 | weekly | online | Campaign × date spend. Fields: `campaign_id, campaign_name, channel, start_date, end_date, spend, impressions, clicks, conversions, attributed_revenue`. |
| 6 | `Support Tickets` | live | 22,491 | n/a | online | Ticket-level. Fields: `ticket_id, opened_at, closed_at, channel, severity, category, status, csat`. |
| 7 | `Product Catalog` | live | 1,243 | n/a | online | SKU master. Fields: `sku, product_name, category, subcategory, list_price, cost, margin, launch_date, status`. |
| 8 | `Store Directory` | extract | 384 | monthly | online | Store master. Fields: `store_id, store_name, region, country, opened_date, square_feet, manager_name`. |
| 9 | `Pipeline Health` | extract | 4,521 | daily | online | Opportunity records. Fields: `opportunity_id, stage, owner, account_name, amount, expected_close_date, created_at, last_activity_at`. **Seeded as 36 hours stale (`stale_after_minutes: 1440`)** to surface `EXTRACT_STALE`. |
| 10 | `Employee Directory` | extract | 3,217 | weekly | **offline** | HR master. Fields: `employee_id, department, title, location, hire_date, status`. **Seeded offline** to surface `DATASOURCE_OFFLINE`. |

### 9.2 Realism Guarantees

The seed generator must satisfy these properties (verified by tests in §11):

- **Plausible distributions.** Revenue follows a heavy-tailed long-tail (some big-deal outliers); session durations are log-normal; CSAT is bimodal around 4 and 5. No flat or uniform synthetic data.
- **Plausible correlations.** Marketing spend correlates with web sessions; web sessions with conversions; conversions with sales orders. A query like "which channel drove the most conversions" returns *different* answers across timeframes, not constant.
- **Realistic dates.** Order dates span 2023-01-01 through 2026-05-04 (the seed "now"). Session timestamps in the last 90 days are denser than older ones (recency-weighted).
- **Variety in dimensions.** 5 regions (`NA`, `EMEA`, `LATAM`, `APAC`, `MEA`), 4 channels (`retail`, `web`, `mobile`, `partner`), 8 product categories, 380+ stores. Pies and bars look like real BI dashboards.
- **Plausible stale extracts.** `Pipeline Health.last_refreshed_at` is exactly 36h before the seed "now"; staleness threshold is 24h, so `is_stale: true` deterministically.
- **Mixed casing.** Store names mix title case and uppercase ("Downtown Manhattan", "FLAGSHIP NYC"); workbooks, owners, and projects do similar. Fuzzy search must handle this.
- **Some null values.** ~3% of `customer_id` on web sessions are null (anonymous traffic), ~5% of `closed_at` on support tickets are null (open tickets). Filters on `is_null` should hit non-empty results.

### 9.3 Workbooks (40, across 7 projects)

Distributed roughly:

| Project | Workbook count | Examples |
|---|---|---|
| Executive | 4 | Executive Monday Review; CEO Weekly Snapshot; Board Pack 2026-Q1; Cross-Functional KPIs |
| Sales Operations | 9 | Sales Performance Q1 2026; Same-Store Sales Tracker; Pipeline Health Dashboard; Sales Rep Leaderboard; Regional Mix; Channel Performance; Discount Impact; Quota Attainment; New Deal Velocity |
| Marketing | 6 | Marketing ROI by Channel; Web Conversion Funnel; Campaign Calendar; Top Landing Pages; Paid vs Organic; Campaign Cohorts |
| Customer Success | 5 | Customer Segment Health; Lifetime Value by Cohort; Churn Risk Heatmap; CSAT Trends; NPS by Segment |
| Operations | 6 | Inventory Replenishment; Stockout Risk; Lead Time Tracker; Supplier Performance; Warehouse Throughput; Returns Analysis |
| Product | 5 | Product Catalog Performance; Margin by Subcategory; New Launch Tracker; SKU Velocity; Discontinued Items |
| Support | 5 | Support Ops Daily; Ticket Backlog; Severity Mix; Channel Mix; CSAT Trend |

Total: 40 workbooks (within the brief's 30–80). Each workbook has 4–12 views (the brief's range), drawn primarily from its primary datasource but mixing in 1–2 cross-datasource views where realistic (e.g. Executive Monday Review pulls from Sales, Pipeline, Web, and Support).

### 9.4 Views

Total: ~280 views (40 workbooks × 7 avg views). Chart-type distribution roughly:

| Chart type | Share |
|---|---|
| `bar` | ~40% |
| `line` | ~30% |
| `pie` | ~10% |
| `treemap` | ~10% |
| `histogram` | ~10% |

Most views have 1–3 default filters (a recent date window, a specific region or category, a status filter). Some have none. View names are descriptive ("Revenue by Region", "Top 10 SKUs by Margin", "Pipeline by Stage"), not generic ("Chart 1").

### 9.5 Seed-time artifacts

- **Thumbnails:** every workbook and view has a precomputed 320×200 PNG under `fs/thumbnails/`. These are real renders of the seed data using the same renderer as `render_view`/`render_workbook`, just at thumbnail size.
- **One pre-existing export and one pre-existing render** exist at seed time as `expired_at: <past>` rows. This lets tests verify expiry handling without waiting 24 hours.

### 9.6 Seed Generation Approach

The seed is generated by a single deterministic script (seeded random) that:

1. Creates the `projects` rows.
2. Creates the `datasources` rows and emits their physical `data_<short_id>` tables.
3. Generates rows per datasource by sampling from the realism distributions (§9.2).
4. Enriches with cross-datasource correlations (e.g., insert orders for customers who exist; sessions before orders for converted users).
5. Creates the `datasource_fields` rows in display order.
6. Creates the `workbooks` and `views` per project.
7. Creates `view_default_filters`.
8. Renders thumbnails into `fs/thumbnails/`.
9. Writes one expired export and render row.
10. Optionally backs up the resulting SQLite file as `lookout-seed.db` for harness consumption.

**Why deterministic?** Test runs against the seed must be reproducible. A fixed seed reproduces the same data every time so `query_datasource` test assertions can be exact.

---

## 10. Technical Implementation Notes

This section calls out the gnarly parts - anywhere a careful implementer will save themselves a day by reading first.

### 10.1 The structured query → SQL translator

`query_datasource` constructs SQL programmatically from validated inputs. The translation is roughly:

```
SELECT  <select clause>
FROM    <data_table>
WHERE   <filters AND ed>
GROUP BY <group_by>
ORDER BY <order_by>
LIMIT   <limit>  OFFSET <offset>
```

**Implementation steps (in order):**

1. **Resolve the datasource and its `data_table`** from `datasources` by `datasource_id`. Reject if not found / offline.
2. **Validate every field reference** (in `select`, `filters`, `group_by`, `order_by`) against `datasource_fields`. Translate field names to physical column names (they're the same here, but go through the lookup so we can support computed fields later).
3. **Validate aggregations** against the field's `data_type`. `sum`/`avg` require numeric. `count`/`count_distinct`/`min`/`max` work on any type.
4. **Validate `group_by` consistency:** every non-aggregated `select` field must be in `group_by`. Otherwise `MISSING_GROUP_BY`.
5. **Translate filters** to parameterized SQL. **Always use placeholders** (`?`) - never string-interpolate values. This both prevents injection in the validator-bypass case *and* lets SQLite use indexes properly.
6. **Apply pagination.** Decode the cursor; on first page, offset=0. After the query, encode the next cursor with the new offset and a hash of the query inputs.
7. **Run with a 5-second timeout** via `sqlite3.set_progress_handler` or wall-clock check. On timeout, raise `QUERY_TIMEOUT`.
8. **Apply the row cap.** If returned rows exceed `limit`, truncate and set `truncated = 1` in `queries`. If actual count exceeds the hard cap (1,000,000), return `RESULT_TOO_LARGE` instead.
9. **Pagination is via `OFFSET`/`LIMIT` against the underlying data table.** The query is re-executed for each page of `query_datasource` and `run_sql`. We *do not* materialize results into a temp table because (a) seed data is static, so re-execution returns the same rows, and (b) materialization adds TTL bookkeeping for negligible benefit at the row scales we support. The cursor's hash check (§10.6) guards against parameter changes mid-pagination.
10. **Log to `queries`.** Insert a row with the input shape, row_count, truncated flag, and duration. This is an audit/debug aid; no tool returns it directly other than the `query_id` echoed in the response.

### 10.2 The SQL sandbox for `run_sql`

`run_sql` opens a *separate* SQLite connection per call:

- Connection mode: `mode=ro&immutable=0` (read-only).
- Database: only the datasource's `data_<short_id>` table is exposed. Implemented by **attaching a fresh in-memory database with only that table** copied/aliased as `data`. Other system tables, the metadata catalog, etc. are not visible.
- The SQL string is parsed using SQLite's built-in `EXPLAIN` first; if `EXPLAIN` errors, return `INVALID_SQL` with the parser message. If `EXPLAIN` produces opcodes other than read-only opcodes (`OpenWrite`, `Insert`, `Update`, `Delete`, `Cast`-to-DDL, etc.), reject before execution.
- Statement count check: SQLite executes only the first statement on `execute()`, but to be safe we reject any input containing `;` outside of strings (a simple lexer is enough).
- Wall-clock timeout: 5 seconds (same as structured queries).
- Result row cap: 1,000,000 (same).
- **Pagination wrapping.** The agent's SQL is wrapped for pagination: `SELECT * FROM ( <agent_sql> ) AS sub LIMIT ? OFFSET ?`. This works whether or not the agent included their own `LIMIT` - the inner `LIMIT` is preserved inside the subquery, and the outer `LIMIT/OFFSET` paginate the resulting set. If wrapping fails (e.g. the agent's SQL is invalid as a subquery), the original `INVALID_SQL` error from the parser is preserved.

The point: `run_sql` is powerful but cannot escape the sandbox. We document this clearly in the tool description so the agent doesn't try to query metadata tables there.

### 10.3 Filter operator translation

Operators map to SQL roughly as:

| Operator | SQL |
|---|---|
| `eq` | `field = ?` |
| `neq` | `field != ?` |
| `gt`, `gte`, `lt`, `lte` | `field <op> ?` |
| `between` | `field BETWEEN ? AND ?` |
| `in` | `field IN (?, ?, …)` |
| `not_in` | `field NOT IN (?, ?, …)` |
| `is_null`, `is_not_null` | `field IS NULL` / `field IS NOT NULL` |
| `contains` | `LOWER(field) LIKE LOWER(?)` with `%value%` |
| `starts_with` | `LOWER(field) LIKE LOWER(?)` with `value%` |
| `ends_with` | `LOWER(field) LIKE LOWER(?)` with `%value` |

Date/datetime values are stored as ISO-8601 strings; SQLite's lexicographic comparison on ISO-8601 is correct for `<`/`>`/`BETWEEN`. Date filter values are parsed and re-emitted in canonical form before binding to ensure `'2026-01-01'` and `'2026-1-1'` both work.

### 10.4 Chart rendering

Renders are produced by a small renderer module the harness wires up. The spec doesn't pin the implementation, but the contract is:

```
render(chart_type, data_rows, columns, config, width, height, theme, format) -> bytes
```

For the implementer's convenience, **Vega-Lite + a headless renderer (e.g. `vega-cli` or the `altair_saver` Python package) is the recommended choice.** Vega-Lite supports all five chart types declaratively and produces both PNG and SVG. A typical render takes 200–800 ms for the volumes we see at view-data scale, well inside the human-perceived "instant" budget.

**Inputs the renderer receives:**

- The view's `chart_type` and `config` (axes, group-by, color, aggregation).
- The query result (already aggregated and sorted by `query_datasource` semantics - the renderer never touches raw rows beyond plotting).
- The view's `name` and the field display names for axis labels.
- Theme and dimensions.

**Output:** raw bytes. The tool wrapper writes to `fs/renders/{render_id}.{ext}` and returns the path.

`render_workbook` is a wrapper that:

1. Loads the workbook and its views.
2. Renders each view (in parallel, capped at 4 concurrent renders).
3. Composes them into a 2-column or 3-column grid (3-column when view_count ≥ 9 *and* width ≥ 1600).
4. Adds the workbook title at the top and view names above each chart.
5. Writes the composite to `fs/renders/{render_id}.png` (or SVG via `<svg>` composition).

If any individual view fails, its slot in the grid contains a placeholder ("⚠ Render failed: <code>") and the failure is reported in the response's `warnings`.

### 10.5 Fuzzy search implementation

`search_lookout` uses **SQLite FTS5** for tokenized full-text matching across a derived index:

```sql
CREATE VIRTUAL TABLE search_index USING fts5(
  kind,             -- 'datasource' | 'workbook' | 'view'
  entity_id,        -- the prefixed id
  name,             -- searchable
  description,      -- searchable
  context,          -- e.g. 'in workbook X' or 'in project Y'
  tokenize = 'porter unicode61'
);
```

The index is rebuilt at seed time and on the rare update events. Scoring uses FTS5's BM25 with a small boost for `name` matches (×3) over `description` and `context` (×1).

For names like "Pipeline Health by Stage" matching the query "pipeline", the porter tokenizer normalizes well. Quoting is supported (`"pipeline health"`) for exact phrase matching.

If FTS5 isn't available in the SQLite build, fall back to a `LIKE %query%` scan - slower but functionally similar at our scale.

### 10.6 Cursor encoding & validation

```python
def encode_cursor(offset: int, query_signature: str) -> str:
    payload = {"o": offset, "h": hashlib.sha256(query_signature.encode()).hexdigest()[:8]}
    return base64.urlsafe_b64encode(json.dumps(payload, separators=(",", ":")).encode()).decode().rstrip("=")
```

`query_signature` is a stable string formed from sorted query parameters (e.g. `"ds=ds_…&select=region,sum(gross_revenue)&filters=…&page_size=25"`). On decode, recompute the signature; if it doesn't match the cursor's `h`, return `INVALID_CURSOR`. This prevents the agent from accidentally reusing a cursor across changed query parameters and getting confusing partial pages.

### 10.7 CSV writing details

- Library: Python's stdlib `csv` module with `dialect='excel'` (RFC-4180-ish, `\r\n` line endings, `"`-quoted, embedded quotes doubled).
- Headers: per `column_labels`. Display names may contain spaces and special characters; the CSV quoting handles this correctly.
- Numeric values: written as their string repr (`f"{x}"`); no scientific notation forcing.
- Date values: ISO-8601 string (`YYYY-MM-DD`); datetime as `YYYY-MM-DDTHH:MM:SSZ`.
- Null values: empty cell.
- Encoding: UTF-8, no BOM. (BOM breaks downstream parsers more often than it fixes Excel.)

### 10.8 JSON writing details

- A single JSON array of objects, written via streaming write (`json.dumps` per row is fine at our scale; up to 1M rows with 10–20 fields each = ~200–500 MB and we'd already fail `EXPORT_TOO_LARGE` before that).
- Numeric values are JSON numbers; nulls are `null`; booleans are `true`/`false`; strings/dates are JSON strings.
- Output is *not* pretty-printed (whitespace-compact) to reduce file size.

### 10.9 Datetime handling for staleness

All staleness comparisons use the harness-provided "now":

```
is_stale = (connection_type == 'extract'
            and stale_after_minutes is not None
            and last_refreshed_at is not None
            and (now - last_refreshed_at).total_seconds() / 60 > stale_after_minutes)
```

The harness exposes "now" as a single function. For tests, "now" can be pinned. The mock never reads the system clock directly - that would make tests flaky.

### 10.10 What to do if a render or export fails partway

- **Render:** if the renderer raises, no file is written; the call returns `RENDER_FAILED`. No partial file pollutes `fs/renders/`.
- **Export:** the writer writes to a temp file (`{id}.{ext}.tmp`) and atomically renames on success. On failure, the temp file is removed. No partial file under `fs/exports/`.

This is critical for agents that read the path immediately after the call returns - they should never see a half-written file.

### 10.11 Concurrency and idempotency

Each tool call is independent. SQLite is opened in WAL mode at the harness layer, allowing concurrent readers. Mutations from `refresh_datasource` are wrapped in a single transaction; concurrent refreshes of the same datasource are serialized at the database level. Idempotent calls (e.g. `list_datasources`) are safe to retry.

### 10.12 Error message hygiene

Every error message includes:
- The offending value (e.g. the bad ID, the field name that wasn't found).
- A hint where helpful ("Did you mean `gross_revenue`?" - based on a Levenshtein-2 match against the field list).

The hint is opportunistic: if a `FIELD_NOT_FOUND` has a single Levenshtein-2 candidate in the datasource's field list, include `details.suggestion`. Otherwise leave it off. This nudges the agent toward the right call without pretending we know exactly what they meant.

---

## 11. Testing Strategy

Tests are organized in three layers, in order of cost-to-write and value-per-test.

### 11.1 Schema-level tests (cheapest, run on every CI build)

- Every seed datasource has at least one row.
- Every workbook has 4–12 views (per the brief).
- Every view has a valid `chart_type` and references a real datasource.
- Every `view_default_filter` references a real `field_id` belonging to the view's datasource.
- The seeded-offline datasource (`Employee Directory`) exists with `status: "offline"`.
- The seeded-stale extract (`Pipeline Health`) has `last_refreshed_at` exactly 36 hours before "now" and `stale_after_minutes = 1440`.
- ID prefix correctness: every row in `datasources` has an id starting `ds_`, etc.
- Foreign-key integrity (SQLite `PRAGMA foreign_key_check` returns empty).

### 11.2 Tool contract tests (one per tool, plus one per major code path)

For each tool:

- Happy-path call returns a response that conforms to the documented schema (validated against a JSON Schema generated from the spec).
- Invalid IDs (wrong prefix, wrong length, non-existent) produce the expected error code.
- Pagination: requesting page 2 with the cursor from page 1 returns the next slice; reusing a cursor with a changed parameter returns `INVALID_CURSOR`.
- For tools that touch a datasource: the offline datasource produces `DATASOURCE_OFFLINE`; the stale datasource produces a top-level `warnings` entry.
- For `query_datasource`: a query that selects a non-aggregated column without `group_by` produces `MISSING_GROUP_BY`; a `sum()` on a string column produces `INVALID_AGGREGATION`; a query with no filters on `web_analytics` returns `RESULT_TOO_LARGE`.
- For `run_sql`: a query that contains `INSERT INTO data …` is rejected with `INVALID_SQL`; multi-statement input is rejected; a query referencing a table named `users` is rejected.
- For `apply_view_filters`: `merge_strategy` modes produce different `applied_filters` outputs; default override semantics work for all three modes.
- For `render_view` / `render_workbook`: file is written before the response returns; reading it via the `fs` reference returns valid PNG/SVG bytes (PNG: starts with `\x89PNG`; SVG: starts with `<svg`).
- For `export_view_data` / `export_query_results`: file at `file_path` exists and is valid CSV (parses cleanly, header row matches `column_labels`); JSON parses cleanly to an array of objects.

### 11.3 Workflow integration tests (replicates each W1–W8)

For each of the eight workflows, write an end-to-end test that drives the tool sequence and asserts on the final state:

- **W1:** `list_datasources()` includes 10 entries; for each, `get_datasource()` returns a non-empty `fields` array.
- **W2:** `search_lookout("executive")` returns a workbook; `get_workbook(that_id)` returns 4–12 views.
- **W3:** `get_view(vw_revenue_by_region)` → `apply_view_filters(filter_overrides=[region IN [NA]])` returns rows with only NA values.
- **W4:** `query_datasource(...)` aggregating `Sales Orders` by region returns five regions with non-zero revenue.
- **W5:** Two `query_datasource` calls (`Q1 2026`, `Q4 2025`) return different sums with non-trivial difference.
- **W6:** `get_view(...)` returns `summary_stats` with `min`, `max`, `mean` matching independently-computed values from a `query_datasource` call over the same data.
- **W7:** `render_view(...)` writes a PNG ≥ 5KB. `render_workbook(...)` writes a PNG ≥ 30KB.
- **W8:** `export_view_data(view_id, format="csv")` writes a valid CSV; the row count matches `apply_view_filters(view_id)` row count.

### 11.4 Realism / correctness tests

- **Distribution:** Pearson correlation between `Marketing Campaigns.spend` and `Web Analytics.sessions` (joined by date) is positive and > 0.4.
- **Variation:** A `query_datasource` over Sales Orders by region for two different quarters returns *different* totals (no two regions identical, no two quarters identical).
- **Determinism:** Re-running the seed script with the same random seed produces an identical SQLite file (byte-for-byte after `VACUUM`).
- **Schema diversity:** Across the 10 datasources, at least 6 distinct data types appear in `datasource_fields` (`integer`, `float`, `string`, `boolean`, `date`, `datetime`).
- **Search:** `search_lookout("pipline")` (intentional typo) returns the Pipeline Health workbook with a top-3 score.

### 11.5 What I deliberately don't test

- **Renderer pixel exactness.** Vega-Lite output isn't byte-stable across versions; we test that *some* PNG is written, of *plausible* size, and is a valid PNG header. Pixel-diff testing would be high-maintenance.
- **Full performance benchmarks.** We assert the 5-second budget kicks in on a known-slow query; we don't measure exact ms numbers across CI environments.
- **Concurrent calls.** v1 is per-request; the harness handles connection lifecycle. A multi-call concurrency suite is a future-extension item.

---

## 12. Explicit Trade-offs

What I'm consciously *not* building, and the reasoning.

| Choice | What I'm not doing | Why |
|---|---|---|
| Unified entity model | A separate `dashboards` table distinct from workbooks. | The brief flattens the hierarchy; a workbook acts as a dashboard at render time. The third tier doesn't change agent behavior and doubles surface area. |
| Filter logic | OR / NOT in the structured filter language. | A boolean expression tree adds parser, validator, error-message complexity. AND covers the common case; `run_sql` is the escape hatch for the rest. |
| Period-over-period comparison | A dedicated `compare_periods` tool. | The brief says explicitly: "the analyst querying the data... and analyzing the returned data themselves." Composition keeps the surface tight. |
| Calculated fields | User-defined formula columns. | Adds a query-rewriter and a formula language. Out of scope; described as a v2 extension. |
| Joins across datasources | A `query_join` or multi-source SQL. | `run_sql` is scoped to one datasource for sandbox safety. Multi-source joins are a v2 extension via a "blended datasource" entity (Tableau pattern). |
| Real-time data | Live-streaming updates, change detection. | Explicit non-goal. Everything is static seed data. |
| Authentication | Multi-user, RBAC, sharing model. | Explicit non-goal. |
| Versioning | Workbook history, undo, branch/restore. | Read-only mock; no need. |
| Map / geo charts | A `map` chart type with lat/long projection. | The brief lists five chart types ("to start with"); maps add a coordinate system, projection logic, and tile rendering. v2. |
| Subscription / scheduling | Email a workbook on a schedule. | Out of scope; no side effects beyond the mock. |
| Natural language query | "Ask Data"-style NL → query parsing. | The agent itself is the natural-language layer. We don't rebuild that here. |
| Field-level descriptions in search | `search_lookout` matching field names. | Field-level discovery is what `get_datasource` is for. Indexing all 600+ fields would diffuse the relevance signal. |
| Render annotations | Tooltip data, drill-down regions, axis interactivity. | Output is a static image; interactivity is a v2 thing. |
| Streaming responses | Partial result streaming for large queries. | MCP doesn't define streaming for tool calls in v1; we use pagination. |
| Cost / token-budget tools | A `count_rows`-style fast-path tool. | `query_datasource` with `select: [{ "field": "*", "aggregation": "count" }]` does this in one call. No separate tool needed. |

A short summary of why these trade-offs are right: each one is either explicitly out of the brief, has a near-equivalent composition path that keeps the surface smaller, or adds disproportionate complexity for a low-frequency workflow.

---

## 13. Future Extensions

The design is intentionally extensible without breaking changes. This section sketches what v2/v3 would look like; *none of this is in v1*.

| Extension | Schema change | Tool change | Notes |
|---|---|---|---|
| **More chart types** (`scatter`, `area`, `box_plot`, `heatmap`) | Add to the `views.chart_type` allowed set; add per-type config schema rules. | None - `render_view` already treats `chart_type` as an enum and dispatches to the renderer. | Renderer must learn the new types. |
| **Calculated fields** | New `calculated_fields(id, datasource_id, name, expression, data_type)` table. | `query_datasource` learns to expand calc fields into SQL via a small expression compiler. | Requires a safe expression language (see SQLite's `expr.c` allow-list). |
| **Multi-source joins / blended datasources** | New `blended_datasources` table with `join_spec` JSON. | `query_datasource` accepts a blended id; `run_sql` exposes both tables. | Mirrors Tableau's data-blending feature. |
| **Workbook layout fidelity** | New `workbook_layout` table with absolute positioning. | `render_workbook` reads the layout; current grid becomes the default. | For pixel-faithful dashboard recreations. |
| **Audit log** | New `tool_call_audit(id, tool, input_json, output_summary, timestamp)` table. | None - middleware writes to it. | Useful for debugging agent behavior. |
| **Soft permissions / project scoping** | New `agent_role` field per call (header). | All `list_*`/`get_*` filter by role. | Mock-only; not real auth. |
| **Streaming results** | None at storage. | `query_datasource` and exports gain an `Accept-Encoding: stream` mode. | Pending MCP spec evolution. |
| **Field aliases** (display name overrides per agent) | New `field_overrides` table. | None. | Lets an agent rename `gross_revenue` to "Revenue" globally. |

Each extension changes the schema additively (new tables, new optional columns) or the tool surface additively (new optional inputs, new tools). No existing tool input or output shape needs to change.

---

## Appendix A - Quick Reference: Tool Inventory

| # | Tool | Category | Reads | Writes |
|---|---|---|---|---|
| 1 | `list_datasources` | Discovery | `datasources` | - |
| 2 | `get_datasource` | Discovery | `datasources`, `datasource_fields` | - |
| 3 | `list_projects` | Discovery | `projects`, `workbooks` | - |
| 4 | `list_workbooks` | Discovery | `workbooks`, `projects`, `datasources` | - |
| 5 | `get_workbook` | Discovery | `workbooks`, `views`, `datasources`, `projects`, fs/thumbnails | - |
| 6 | `list_views` | Discovery | `views`, `workbooks`, `datasources` | - |
| 7 | `get_view` | Discovery | `views`, `view_default_filters`, datasource data table, fs/thumbnails | - |
| 8 | `search_lookout` | Search | `search_index` (FTS5) | - |
| 9 | `query_datasource` | Querying | datasource data table | `queries` |
| 10 | `run_sql` | Querying | datasource data table | `queries` |
| 11 | `apply_view_filters` | View ops | `views`, `view_default_filters`, datasource data table | - |
| 12 | `render_view` | Rendering | `views`, `view_default_filters`, datasource data table | `renders`, fs/renders/{id}.{ext} |
| 13 | `render_workbook` | Rendering | `workbooks`, `views`, `view_default_filters`, datasource data table | `renders`, fs/renders/{id}.{ext} |
| 14 | `export_view_data` | Export | `views`, `view_default_filters`, datasource data table | `exports`, fs/exports/{id}.{ext} |
| 15 | `export_query_results` | Export | datasource data table | `queries`, `exports`, fs/exports/{id}.{ext} |
| 16 | `refresh_datasource` | Operations | `datasources` | `datasources.last_refreshed_at` |

---

## Appendix B - JSON Type Definitions

For implementer reference, the structured types referenced in this document:

```ts
type Filter = {
  field: string;                    // technical name or fld_<id>
  operator:
    | "eq" | "neq" | "gt" | "gte" | "lt" | "lte"
    | "between" | "in" | "not_in"
    | "contains" | "starts_with" | "ends_with"
    | "is_null" | "is_not_null";
  value?: ScalarOrArray;            // omitted for is_null/is_not_null
};

type Select = {
  field: string;                    // technical name; "*" only allowed with count
  aggregation?: "sum" | "avg" | "count" | "count_distinct" | "min" | "max" | null;
  alias?: string;
};

type OrderBy = {
  field: string;                    // alias or technical name
  direction: "asc" | "desc";
};

type StructuredQuery = {
  select: Select[];
  filters?: Filter[];
  group_by?: string[];
  order_by?: OrderBy[];
  limit?: number;
};

type Warning = {
  code: string;                     // e.g. "EXTRACT_STALE"
  message: string;
  details?: object;
};

type ErrorEnvelope = {
  error: {
    code: string;
    message: string;
    details?: object;
  };
};
```

---

*End of specification.*
