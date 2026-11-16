# GGC Data Studio Automation — Plan of Action (DRAFT for review)

> **Status:** Draft v1 — for review before any code is built.
> Goal: stop manually pasting department Excel/CSV data into a Google Sheet, and instead
> push all non-GA4 data sources into Google Sheets (or BigQuery) automatically so
> Looker Studio (Data Studio) stays current with zero manual edits.

---

## 1. What you told me

- Department heads email you Excel/CSV exports from third-party tools.
- You manually paste updated values into a Google Sheet.
- That sheet feeds Looker Studio (Data Studio).
- GA4 already connects to Looker Studio via the native connector (no work needed).
- Scope for this first pass: **localIQ, Sprout Social, Meta Business Suite, Flowcode**.
- You will revise this plan before we build.

## 2. What I found in the raw data (your sample exports)

### A. Meta Business Suite — `Dec-31-2025_Nov-11-2026_1822393032338707.csv`
- **Post-level / Instagram only.** 245 rows, 1 row per post.
- Columns: `Post ID, Account ID, Account username, Account name, Description,
  Duration (sec), Publish time, Permalink, Post type, Data comment, Date,
  Views, Likes, Shares, Comments, Saves, Reach, Follows`.
- `Post type`: `IG reel` (130), `IG carousel` (60), `IG image` (55).
- `Date` column is always the literal value `Lifetime` — the real date is in `Publish time`
  (format `MM/DD/YYYY HH:MM`). The filename date range says `Dec-31-2025`–`Nov-11-2026`.
- **Multiple accounts** in one file: `georgiagwinnett` (224 rows) plus 10 smaller/dept
  accounts (athletics teams, CIEE Rome, Watch The Yard, etc.).
- `Description` contains **embedded newlines and commas** → must be parsed with a real CSV
  parser, never naive split (my first column-cut attempt broke here).

### B. Sprout Social — `Profile Performance ... Jan 1, 2026 - Nov 12, 2026.csv`
- **Daily / profile-level / cross-network.** 1,580 rows = 316 days × 5 networks.
- One row per `(Date, Network, Profile)`.
- Networks: `Facebook, Instagram, LinkedIn, TikTok, YouTube`.
- Profiles: `Georgia Gwinnett College` (FB/LI/YT), `georgiagwinnett` (IG), `'@georgiagwinnett2005` (TikTok — note the **leading apostrophe**, likely a CSV-injection guard).
- ~60 columns: Audience, growth, impressions, video views, engagements, engagement rate,
  reactions, comments, shares, link clicks, saves, plus many network-specific sent/received
  breakdowns. Most are sparse per row (only the relevant network's columns are populated).
- Dates in `DD-MM-YYYY` format (`01-01-2026`). Numbers carry thousands separators
  (`26,390`) and percents (`1.72%`) → must be normalized before math in Sheets/Studio.

### C. Both files
- Both carry a `:Zone.Identifier` metadata file (Windows "downloaded from internet" tag).
  Harmless, but should be excluded from the repo/processing (add to `.gitignore`).

### D. Flowcode — `flowcodes-2026-01-01-2026-10-31/` (new)
- Exports as a **set of ~8 small CSVs per report**, each a different view. The date in each
  filename is the report's *end date*, not the whole range (all files say `2026-01-01` here
  but the folder covers Jan 1–Oct 31, 2026).
- `main_chart`: **daily time series**, one row per date: `Rank, Date, Scans, Unique Scanners`.
  Date format is ISO-8601 with offset (`2026-01-06T00:00:00-05:00`) — the only source with a
  clean, sortable date. This is the primary table for Looker Studio.
- `top_performing_codes`: `Rank, Code Name, Scans` — per QR-code totals.
- `top_performing_tags`: `Rank, Tag Name, Scans` — per-tag totals. **Note:** some tag names
  contain embedded commas and quotes (e.g. `"Add-""digital-sign"""`) → needs real CSV parsing.
- `top_performing_cities` (`Rank, City Name, Postal Code, Scans`), `top_performing_states`
  (`Rank, State, Scans`), `most_popular_day` (`Rank, Day, Scans`), `most_popular_hour`
  (`Rank, Hour, Scans`), `most_popular_operating_system` (`Rank, Operating System, Scans`).
- `top_metrics_summary` is a **summary pivot table** (`Today / Last 7 / Last 30 / Last 90 /
  All Time`) — not row-oriented; likely exclude or store separately.
- Granularity: **daily** in `main_chart`; the rest are **aggregate/ranked** totals.

### E. LocalIQ — `LocalIQ-YTD-Marketing Activity Report.pdf` (new — **PDF, not CSV**)
- **Important caveat:** LocalIQ only exports to **PDF** (3 pages), not CSV.
- Page 1–2: charts ("Leads By Source" over months; "Calls / Forms & Emails / Chats").
- Page 3: the useful tabular data — **"How people are discovering you"**:
  `Total / Organic Search / Direct Site Traffic / LocaliQ Media / Dash by LocaliQ / Search
  Engine Marketing / Facebook Lead Ads / ... / Other / Social / Directory Listing / Other
  Paid Search Campaigns`, with columns `Site Visits, Calls, Forms & Emails, Chats`.
- This is **aggregate YTD data** (Jan 1–Nov 16, 2026), not daily.
- Because it's a PDF, ingestion requires **PDF text extraction** (not a plain CSV read).
  Apps Script can read PDFs but text extraction from arbitrary PDF layouts is fragile; a
  better path is to have the PDF **parsed into a clean table** (manually once to confirm
  structure, or via a tool) before it hits the Sheet. See section 6.2 for options.

## 3. The core design decision — DECIDED: Option A (Sheets), BigQuery later

> **Decision (from review):** Use **Google Sheets as the single source of truth** now, with
> **Apps Script** automating the sheet updates. Pivot to **BigQuery** later for future-proofing.

| Option | How data gets to Looker Studio | Status |
|---|---|---|
| **A. Sheets-only** | Apps Script pushes each source into its own tab; Looker Studio reads the sheet. | **CHOSEN — build now** |
| **B. BigQuery + Studio** | Load every source into BigQuery tables; Looker Studio connects to BigQuery. | **Future pivot** (v2/v3) |

Why now: you need to present tomorrow and BigQuery setup is too heavy for today. Sheets gives
you maximum automation quickly and stays compatible with your current workflow.

**Future-proofing design principle:** I'll build the Apps Script so the *normalization logic* is
decoupled from the *destination* (a function returns clean rows; a thin writer sends them to a
Sheet tab). When you're ready for BigQuery, only the writer changes — the normalize step is reused
as-is. This makes the eventual pivot cheap.

## 4. Proposed architecture (Option A — Sheets as single source of truth)

```
   DEPARTMENT HEADS UPLOAD CSVs
        |
        v
[Google Drive drop folder]  (one per source, e.g. "Incoming/Meta", "Incoming/Sprout")
        |
        v
[APPS SCRIPT INGEST]  <-- runs on a time-based trigger (automated)
   |  1. watch Drive folder for new CSVs
   |  2. read + parse CSV (real CSV parser, handles embedded newlines/commas)
   |  3. normalize: unify dates -> YYYY-MM-DD, strip , % $ from numbers
   |  4. map onto unified schema
   |  5. append clean rows into that source's Sheet tab  <-- WRITER (swappable later)
        |
        v
[Master Google Sheet]  <-- one tab per source (single source of truth)
   - tab: meta_posts
   - tab: sprout_daily
   - tab: flowcode_daily
   - tab: flowcode_top_codes / _tags / _cities / _states (aggregates)
   - tab: localiq_summary
        |
        v
[Looker Studio]  <-- native connectors
   - GA4  (native connector, already works, no work needed)
   - Google Sheets (connects to each tab above)
```

### 4.1 Standardized, unified schema (one per source, all share a common "date" key)
This common `date` key is what lets Looker Studio report across all tools on one timeline.

- **meta_posts** (post level): `date, account_username, account_name, post_type,
  post_url, views, likes, shares, comments, saves, reach, follows, duration_sec`
- **sprout_daily** (daily profile level): `date, network, profile, audience,
  net_audience_growth, impressions, video_views, engagements, engagement_rate,
  reactions, comments, shares, post_link_clicks, saves` (+ optional network-specific columns)
- **flowcode_daily** (daily): `date, scans, unique_scanners` (from `main_chart`).
- **flowcode_top_codes / _tags / _cities / _states** (aggregate): `rank, name, scans`
  (+ `postal_code` for cities). Parsed from their respective CSVs.
- **localiq_summary** (aggregate YTD): `source, site_visits, calls, forms_emails, chats`
  — derived from the PDF's "How people are discovering you" table. Granularity is
  **year-to-date, not daily** (LocalIQ only provides this).
- All numeric columns are cleaned (stripped of commas/`%`/`$`), all dates normalized to ISO
  `YYYY-MM-DD` in Eastern Time, all timezones consistent.

### 4.2 Automation path — DECIDED: Apps Script (Google native, free)
- You already use it (`google-sheets-cleaner.js`), it's free, needs no infra, and schedules
  itself with time-based triggers. This is the highest-automation, lowest-friction option for
  tomorrow's deadline.
- App Script will: watch a Drive folder → parse → normalize → append to the right Sheet tab.
- Later, to pivot to BigQuery: only the "writer" function changes; the normalize step is reused.
- (Python remains an option only if we hit Sheets limits; not needed for v1.)

## 5. Suggested phased roadmap

### Phase 1 — Foundation & all four sources (schemas now defined)
1. Add `.gitignore` for `*.csv` and `*:Zone.Identifier` so processed data isn't committed.
2. Build the **Apps Script normalizer + ingester** (decoupled normalize vs. write) that:
   - parses **Meta, Sprout, Flowcode** CSVs correctly (real CSV parser, handles embedded
     newlines/commas, e.g. in `Description` and Flowcode `top_performing_tags`),
   - reads **LocalIQ** from a converted CSV (section 6.2, Option 1),
   - unifies date formats → `YYYY-MM-DD` in **Eastern Time**,
   - strips `,`, `%`, `$` from numbers, handles blanks,
   - maps each source onto the unified schemas (section 4.1),
   - **filters to Jan 1, 2026 onward** (drops anything before the start date),
   - appends clean rows to the matching Sheet tab.
3. Stand up the **Master Google Sheet** with a tab per source:
   `meta_posts`, `sprout_daily`, `flowcode_daily`, `flowcode_top_*`, `localiq_summary`.
4. **Backfill** current data from Jan 1, 2026 through today into the tabs (2025 ignored).
5. Wire a **scheduled Apps Script trigger** so future uploads get picked up automatically.
6. *(Optional, recommended)* Add the **email→Drive** automation (section 6.1) so monthly
   emails auto-land in the Incoming folder — removing the manual drop.

### Phase 2 — LocalIQ PDF handling + Incoming folders
7. Convert the LocalIQ PDF to CSV (manual copy, section 6.2 Option 1) each month, or pursue
   Option 3 (ask LocalIQ for a raw export) to eliminate even that.
8. Stand up **per-source Incoming folders** (and/or email routing) so each source lands in the
   right place automatically. Filter to Jan 1, 2026 onward for these too.
9. Keep **daily granularity** for all sources (rollups to weekly/monthly happen in Looker Studio,
   not at ingest time).

### Phase 3 — Reporting
10. Build Looker Studio data sources + a unified dashboard (date-range filter drives all charts).
11. Add a **reconciliation/validation** tab or report that flags missing dates or zero rows
    per source, so you know when someone hasn't uploaded.

### Phase 4 — Future BigQuery pivot (not now)
12. When ready: reuse the same normalize functions, swap the "writer" to load BigQuery tables,
    and repoint Looker Studio data sources from Sheets to BigQuery. Normalization is unchanged.

## 6. Decisions so far (from your review)

1. **Target:** **Google Sheets (Option A)** — single source of truth. ✅
2. **Ingest mechanism:** **Apps Script** with scheduled triggers. ✅
3. **BigQuery:** deferred to future Phase 4; design keeps the pivot cheap. ✅
4. **Cadence:** Department heads upload **once per month**. ✅
5. **Start date:** **January 1, 2026 going forward.** 2025 data stays on the old dashboards;
   this new data source feeds new charts/visuals only. (No need to re-do existing visuals.) ✅
6. **Granularity (Sprout):** Data is inherently **daily** (one row per network per day).
   Storing daily is trivial and future-proofs weekly/monthly rollups — **we'll keep daily**,
   which lets you slice weekly or monthly in Looker Studio without re-ingesting. ✅
7. **Accounts:** **Keep dept/athletics sub-accounts separate** (as-is), don't roll up. ✅
8. **Timezone:** **Eastern Time (EST/EDT, America/New_York)** for all dates. ✅
9. **Flowcode:** samples received → schemas defined (see 4.1). ✅
10. **LocalIQ:** samples received → **PDF-only export** (see 6.2 for handling). ⚠️
11. **Granularity (Flowcode):** keep **daily** in `flowcode_daily` (from `main_chart`); the
    top-codes/tags/cities/states are stored as aggregate totals as provided. ✅
12. **Granularity (LocalIQ):** **year-to-date aggregate** only (no daily data available). ⚠️

### 6.1 Research: Can we automate "email the CSV → lands in Drive"? — YES
**Short answer: yes, fully automate-able via Apps Script, no extra tools needed.**
Apps Script can read a Gmail message's attachments and save them straight to a Drive folder:

- **GmailApp.search(query)** → find emails (e.g. by sender or subject, label-based).
- **message.getAttachments()** → get the attached files.
- **attachment.copyBlob() → folder.createFile(blob)** → save the attachment into the
  "Incoming" Drive folder, which your ingest script then picks up.

So the flow can become: department heads email the CSV to a dedicated Gmail address (or label)
→ a scheduled Apps Script runs every ~15–60 min → detects the email → pulls the attachment →
saves it to the Drive drop folder → your ingest script parses it into the Sheet. **Zero manual
dropping.**

Caveats / design notes:
- **Routing:** Each department head emailing the *same* address works if you route by **sender
  or subject line** (e.g. subject must contain "Sprout" / "Meta" / "localIQ" / "Flowcode") so
  the script knows which source + which Sheet tab it belongs to. Alternatively, give each source
  its own email alias/subject tag.
- **Dedupe:** track processed email IDs (in a Script property or a "processed" label) so a
  re-scan doesn't double-import. Simple and reliable.
- **Trigger:** a time-based trigger (every 15–60 min) is free and has no infra cost.
- **Fallback:** keep the manual Drive-drop as a backup; the script handles both (it just
  watches the folder and/or scans email).
- This works for **CSV attachments on any email** — so it applies to Sprout, Meta, localIQ,
  Flowcode alike once their schemas are added.

This is a strong candidate for Phase 1/2 — it removes the "I have to drop the file" step, which
was the last manual action in your pipeline.

### 6.2 Research: LocalIQ only exports PDF — how do we handle it?
**Situation:** LocalIQ has no CSV export; the "Marketing Activity Report" comes as a 3-page PDF
with one useful table ("How people are discovering you": source × site visits/calls/forms/chats).

**Options (pick one):**

1. **One-time manual conversion → CSV (recommended for v1).**
   - The table on page 3 is simple and clean. Once per report period, copy it into a small CSV
     (`source, site_visits, calls, forms_emails, chats`) and drop it in the LocalIQ Incoming
     folder. Your normal CSV ingester handles the rest, exactly like the other sources.
   - **Pros:** zero fragile PDF parsing; fits the existing pipeline; reliable. **Cons:** a small
     manual step per month (matches your monthly cadence, so it's one quick copy/paste).

2. **Apps Script PDF text extraction.**
   - Apps Script can read a PDF's text via `DocumentApp`/pdf blob, but extracting a *table* from
     an arbitrary PDF layout is brittle (columns/tables rarely map cleanly to text lines).
   - **Pros:** fully automated. **Cons:** high risk of parsing errors that corrupt your data
     silently; needs careful validation. Not recommended as the primary path for v1.

3. **Ask LocalIQ for a CSV/data export.**
   - Many platforms can export raw data (or expose it via API/integration) even if the dashboard
     only shows a PDF button. Worth one support request to your LocalIQ rep.
   - **Pros:** if available, it's the cleanest (matches all other sources). **Cons:** depends on
     their offering; may not be available or may cost more.

**Recommendation:** start with **Option 1** (manual copy → CSV) so LocalIQ is handled by the same
pipeline with no fragile parsing. Optionally pursue **Option 3** in parallel. This keeps the
monthly effort to a single copy/paste while everything else stays automated.

## 7. Answers to your questions (resolved)

1. Cadence: monthly uploads. → email→Drive automation (6.1) removes even the manual drop.
2. localIQ / Flowcode: samples provided → schemas defined (see 4.1 / 6.2).
3. Start: **Jan 1, 2026** forward; 2025 untouched on old dashboards.
4. Granularity: **keep daily** (source is daily; enables weekly/monthly rollups later).
5. Accounts: keep sub-accounts separate.
6. Timezone: **Eastern Time (America/New_York)**.

## 8. What I need from you next (to build Phase 1)

- Confirm the resolved decisions above (incl. the LocalIQ PDF handling — section 6.2).
- Confirm the **Master Google Sheet** we'll build into (or let me scaffold it).
- Decide: do you want the **email→Drive automation** built in Phase 1, or defer to Phase 2
  (manual Drive-drop for the first cycle is fine either way)?
- Decide whether to pursue **LocalIQ raw export** (section 6.2 Option 3) or rely on the
  monthly manual PDF→CSV copy (Option 1).

---

*This plan is intentionally high-level for your review. Once you revise it, I'll turn it
into concrete Apps Script code, the Master Sheet schema, and setup steps.*
