# NYTG Social Media Dashboard — Business Logic Reference

> Extracted from `index.html` source code.  
> Last updated: June 2026

---

## Table of Contents

1. [Core Business Rules](#1-core-business-rules)
2. [Calculations & Formulas](#2-calculations--formulas)
3. [Data Parsing & Transformation Rules](#3-data-parsing--transformation-rules)
4. [Filter & Aggregation Logic](#4-filter--aggregation-logic)
5. [Data Flow Between Components](#5-data-flow-between-components)
6. [External Integrations](#6-external-integrations)
7. [Workarounds & Special Implementations](#7-workarounds--special-implementations)

---

## 1. Core Business Rules

### Brand Isolation

Each brand (`nytg` / `nic`) has **completely separate** data stored in localStorage:

- Keys are always prefixed: `nytg_posts`, `nic_posts`, `nytg_ads`, etc.
- Switching brands via `setBrand()` clears all in-memory arrays and reloads from localStorage
- Supabase sync/pull stores `brand` as a column in `nytg_posts`, but the UI filters by `activeBrand` only after pulling all rows
- **There is no cross-brand comparison view** — all charts and KPIs show the active brand only

### Role-Based Access Control (RBAC)

Two roles exist — `admin` and `viewer`:

| Feature | Admin | Viewer |
|---|---|---|
| Upload files | ✅ | ❌ |
| Data Editor | ✅ | ❌ |
| Sync / Pull (Supabase) | ✅ | ❌ |
| Data Hub tab | ✅ | ❌ |
| View all dashboards | ✅ | ✅ |

**Implementation:** A single CSS class `viewer-mode` on `<body>` hides all `.admin-only` elements.  
**Default fallback:** If no row in `user_roles` table → treated as `viewer`.  
**No Supabase mode:** If Supabase is unavailable at load time → treated as `admin`.  
**Redirect rule:** If a viewer lands on the Data Editor tab, they are automatically redirected to the Executive Summary tab.

### Category Exclusion Policy

Two categories are **always excluded** from all analytics and charts:

```js
const EXCLUDE_CATS = ['Greeting', 'Other'];
```

Posts tagged with `cat1` in this list are excluded unless they also have a `cat2` or `cat3` tag. The rationale: Greeting and Other posts are administrative/generic and skew performance averages.

### Ad Objective Classification

All ads belong to one of two objectives — the classification drives which section they appear in:

- **AWN (Awareness):** Ads where objective column does NOT contain "lead" (case-insensitive)
- **Lead:** Ads where objective column contains "lead" (case-insensitive)

Awareness ads track Reach, Impressions, CPM. Lead ads additionally track Results (conversions) and CPR (Cost Per Result).

### Static Category Data

The Category Content page displays **hardcoded** category distribution percentages for 2025 and 2026, not computed from uploaded CSV data:

```js
const STATIC_CAT = {
  '2025': [
    { cat: 'Sustainability', count: 52, pct: 36 },
    { cat: 'Innovation',     count: 52, pct: 36 },
    { cat: 'VI',             count: 41, pct: 28 }
  ],
  '2026': [
    { cat: 'VI',             count: 11, pct: 37 },
    { cat: 'Innovation',     count: 10, pct: 33 },
    { cat: 'Sustainability', count:  6, pct: 20 },
    { cat: 'Commodity',      count:  3, pct: 10 }
  ]
};
```

These are the pre-defined annual content mix targets/actuals used for the category overview cards. Dynamic data from uploads is used separately for trend and performance charts on the same page.

---

## 2. Calculations & Formulas

### Number Formatting — `fmt(n, d=0)`

```js
function fmt(n, d=0) {
  if (n == null || isNaN(n)) return '—';
  if (Math.abs(n) >= 1e6)    return (n / 1e6).toFixed(1) + 'M';
  if (Math.abs(n) >= 1e3)    return (n / 1e3).toFixed(1) + 'K';
  return n.toLocaleString('th-TH', { maximumFractionDigits: d });
}
```

| Value | Output |
|---|---|
| null or NaN | `—` |
| ≥ 1,000,000 | `1.2M` |
| ≥ 1,000 | `45.3K` |
| < 1,000 | Thai locale (e.g. `987`) |

### Reach Capping — `capReach(v)`

```js
function capReach(v) { return v > 100 ? 98 : v; }
```

Applied to every `%Reach` display. Facebook sometimes reports organic reach percentages above 100% (a platform data anomaly). Values above 100 are capped at 98 to prevent misleading KPI displays. The cap is applied **after** averaging across posts, not per-post.

### Base Aggregations

```js
function sum(arr, key) { return arr.reduce((s, d) => s + (d[key] || 0), 0); }
function avg(arr, key) {
  const vals = arr.map(d => d[key]).filter(v => v != null && !isNaN(v));
  return vals.length ? vals.reduce((a, b) => a + b, 0) / vals.length : 0;
}
```

`sum` tolerates missing keys (treats as 0). `avg` ignores nulls and NaN to avoid skewing the mean.

### KPI Definitions by Page

**Executive Summary** — uses entire `DATA` array (unfiltered by year/period):

| KPI | Formula |
|---|---|
| Total Reach | `sum(DATA, 'reach')` |
| Total Views | `sum(DATA, 'view')` |
| Interactions | `sum(DATA, 'interactions')` |
| Leads | `getLeadsTotal('all')` |
| Avg %Reach | `capReach(avg(DATA, 'pct_reach'))` |
| Avg %Engagement | `avg(DATA, 'pct_eng')` |
| Posts | `DATA.length` |

**Content Performance** — uses `filtered()` (respects year + month filters):

| KPI | Formula |
|---|---|
| Total Reach | `sum(filtered(), 'reach')` |
| Avg %Reach | `capReach(avg(filtered(), 'pct_reach'))` |
| Avg %Engagement | `avg(filtered(), 'pct_eng')` |
| Total Interactions | `sum(filtered(), 'interactions')` |
| Total Views | `sum(filtered(), 'view')` |
| Total Leads | `getLeadsTotal(activeYear)` — from ADS_DATA, not posts |

### Leads Total Calculation — `getLeadsTotal(year)`

```js
function getLeadsTotal(year) {
  return ADS_DATA
    .filter(r => r.objective === 'Lead' &&
                 (year === 'all' || String(r.start || '').startsWith(year)))
    .reduce((s, r) => s + (r.results || 0), 0);
}
```

**Key rule:** Leads are counted from **Ad data** (`ADS_DATA.results`), not from CRM records. Only ads with `objective === 'Lead'` contribute. The `year` filter matches the ad's start date string prefix.

### CPM & CPR Calculations (Ad Spending page)

```js
const totalImp = d.reduce((s, r) => s + r.impressions, 0);
const avgCPM   = totalImp > 0 ? totalSpend / totalImp * 1000 : 0;
const cprOverall = isLeadTab && totalResults > 0 ? totalSpend / totalResults : 0;
```

| Metric | Formula | Unit |
|---|---|---|
| CPM | (total_spend / total_impressions) × 1,000 | THB per 1K impressions |
| CPR | total_spend / total_results | THB per result |

CPR is only calculated when viewing the Lead objective tab (`isLeadTab` flag).

### Best Performing Post Selection

Posts are ranked by descending `pct_reach`:

```js
const sorted = [...d].sort((a, b) => b.pct_reach - a.pct_reach);
```

The post with the highest `%Reach` is displayed as the "Best Performing" highlight on the Content Performance page.

### Category Performance Ranking

For each category, average all `pct_reach` values across its posts, then sort descending:

```js
const catRanked = Object.entries(catPct)
  .map(([k, v]) => ({ cat: k, pct: v.reduce((a, b) => a + b, 0) / v.length }))
  .sort((a, b) => b.pct - a.pct);
```

This determines which categories appear highest on the Content Performance page.

### Analysis — Recommendation Logic (`renderAnalysis`)

Recommendations are generated by comparing aggregates against thresholds:

**"What Worked Well" conditions:**
- Categories with `avgPctReach >= overall avgPctReach` → listed as top performers
- Best post type by highest avg `pct_reach` → recommended
- Brand with highest `pct_reach` → highlighted
- If `avgEng > 1` → engagement rate is flagged as good
- Posts with `reach > 100,000` → counted as "high-reach posts"

**"What Needs Improvement" conditions:**
- Categories with `avgPctReach < overall avgPctReach` → listed as underperformers
- Worst post type → identified
- If `avgEng <= 1` → low engagement warning
- If `totalLeads === 0` → no leads from CRM warning
- If posts with brand tag `< 50% of total posts` → brand tagging warning

All next-step suggestions are always shown regardless of conditions.

---

## 3. Data Parsing & Transformation Rules

### CSV Delimiter Auto-Detection

```js
const sep = text.substring(0, headerEnd).split('\t').length >
            text.substring(0, headerEnd).split(',').length ? '\t' : ',';
```

Counts tab vs comma splits in the first line. The delimiter producing more columns wins. Default is comma on tie.

### CSV Row Parser (RFC 4180 compliant)

- Handles quoted fields with embedded commas/newlines
- `""` inside quoted fields → literal `"`
- Normalises `\r\n` and `\r` to `\n`
- Skips rows where every cell is empty

### Date Parsing — `parseDateStr(s)`

Two formats supported:

| Input Format | Example | Output |
|---|---|---|
| `MM/DD/YYYY` | `1/7/2025` | `{ date: '2025-01-07', month: 'Jan 2025', year: '2025' }` |
| `YYYY-MM-DD` | `2025-01-07` | `{ date: '2025-01-07', month: 'Jan 2025', year: '2025' }` |

Fallback: returns the first space-delimited token as `date`, empty string for `month`.

### Posts File Parsing (`parseCSV`, `parseContentExcel`)

**Required columns (validation fails if missing):** `Permalink`, `Publish time`, `Post type`

**Interaction field priority:**
```js
const rcs = getN(row, 'Reactions, comments and shares');
const te  = getN(row, 'Total Engagement');
const interactions = te || rcs;  // Total Engagement preferred; falls back to Reactions+Comments+Shares
```

**Date resolution priority:**
1. Parse `Publish time` column via `parseDateStr()`
2. If Year + Month columns exist and are valid → construct `YYYY-MM-01` date
3. Fallback to empty string if neither works

**Category fields:** `Category 1`, `Category 2`, `Category 3` → stored as `cat1`, `cat2`, `cat3` (string trim only, no normalization)

### Ads File Parsing — Objective Detection (`parseUnifiedAdsRow`)

```js
const objRaw  = String(r['Ad Objective'] || r['Objective'] || '');
const objective = objRaw.toLowerCase().includes('lead') ? 'Lead' : 'AWN';
```

Checks two possible column names. Case-insensitive substring match on "lead". Anything else → AWN.

**Results field priority (in order):**
```js
results: parseFloat(r['Results'])
      || parseInt(r['Total messaging contacts'])
      || parseInt(r['New messaging contacts'])
      || 0
```

**Date column priority:**
1. `Reporting starts`
2. `Date created`
3. `Start date`

### CRM Leads Parsing (`parseLeadsCSV`)

**Required columns:** `Date`, `Customer / Account Name`, `Channel`, `Stage`

**Product category cleanup:**
```js
product_cat: (row['Product Category'] || '').replace(/^None ?/,'').trim()
```

Strips leading "None" text that appears in some CRM exports.

**Row filter:** Only rows with `customer` OR `channel` value are kept. Blank rows are discarded.

### Deduplication Rules

| Data Type | Key | Function |
|---|---|---|
| Posts | `link` (Permalink URL) | `addMonthCSV` |
| CRM Leads | `customer + \| + date + \| + channel` | `addMonthLeadsCSV` |
| Ads | `ad_name + \| + start_date` | `addMonthAdsExcel` |

For posts: if a post has no link, it is always included (assumed unique). For leads: the composite key must be fully present to deduplicate.

### Supabase Field Mapping — Posts

**To Supabase (`_postsToSupa`):**

| JS Field | Supabase Column | Transform |
|---|---|---|
| `d.year` | `year` | `Number(d.year) \|\| 0` |
| `d.month` | `month` | `d.month.split(' ')[0]` → `"Jan 2025"` becomes `"Jan"` |
| `d.interactions` | `total_engagement` | direct |
| `d.pct_reach` | `percent_reach` | direct |
| `d.pct_eng` | `percent_engagement` | direct |
| `d.type` | `post_type` | direct |
| `d.cat1` | `category` | only cat1; cat2/cat3 lost |
| `d.link` | `permalink` | direct |
| `d.view` | `views` | direct |

**From Supabase (`_postsFromSupa`) — fields NOT restored:**

| Lost Field | Notes |
|---|---|
| `cat2`, `cat3` | Always `''` after pull |
| `title` | Always `''` after pull |
| `date` | Always `''` after pull |
| `impressions` | Always `0` after pull |
| `reels` | Always `0` after pull |
| `leads` | Always `0` after pull |
| `avg_sec` | Always `0` after pull |
| `link_clicks` | Always `0` after pull |

> **Impact:** Pulling from Supabase loses multi-category tagging, per-post lead counts, and video-specific metrics. The primary sync direction is local → cloud; pulling is for onboarding new devices.

### Supabase Field Mapping — Ads

**Not preserved through sync/pull:**

| Lost Field | Notes |
|---|---|
| `start` | Raw start date string — not in Supabase schema |

### Supabase Field Mapping — CRM

CRM stores 10 fields to Supabase. The local object has 15+ fields (opportunity, contact_role, lead_id, ad_creative, product_interest, sales_status are local-only).

---

## 4. Filter & Aggregation Logic

### Filter Chain

```
DATA (all posts)
  └─ yearFiltered(DATA)       ← activeYear filter
        └─ filtered()         ← activePeriod filter (month chip)
```

### Year Filter — `yearFiltered(data)`

Matches if **any** of three conditions pass:

```js
String(d.year) === String(activeYear)            // exact year field match
|| String(d.date || '').startsWith(activeYear)   // date string starts with year
|| (d.month && d.month.includes(activeYear))     // month string contains year ("Jan 2025" ⊇ "2025")
```

This triple-match handles inconsistent year data across upload formats.

### Period Filter — `filtered()`

```js
function filtered() {
  const base = yearFiltered(DATA);
  return activePeriod === 'all' ? base : base.filter(d => d.month === activePeriod);
}
```

Period filter is an **exact string match** on `d.month` (e.g. `"Jan 2025"`). Month chips are only shown for months present in the filtered dataset.

### Month Ordering

All month sorting uses a fixed lookup array:

```js
const MONTH_ORDER = [
  'Jan 2025','Feb 2025','Mar 2025','Apr 2025','May 2025','Jun 2025',
  'Jul 2025','Aug 2025','Sep 2025','Oct 2025','Nov 2025','Dec 2025',
  'Jan 2026','Feb 2026','Mar 2026','Apr 2026'
];
```

Months not in this list will sort to index -1 (appear first). The list currently covers Jan 2025 → Apr 2026 only; data beyond Apr 2026 will not sort correctly without updating this constant.

### Period Bar Generation

```js
const mUsed = MONTH_ORDER.filter(m => d.some(p => p.month === m));
```

Only months that have at least one post in the current year filter are shown as chips. The "All" chip is always shown first.

---

## 5. Data Flow Between Components

### Upload → Render Pipeline

```
User selects file
  └─ FileReader.readAsArrayBuffer / readAsText
        └─ parse function (parseCSV / parseContentExcel / parseUnifiedAdsRow / parseLeadsCSV)
              └─ validate required columns (validateCSVHeaders)
                    └─ DATA / ADS_DATA / LEADS_DATA assigned
                          └─ LS.save(type, data)           ← localStorage write
                                └─ sbBgSync(type, data)    ← async Supabase write
                          └─ updateBadge()
                          └─ showDashboard()
                          └─ renderAll()                   ← re-renders all 4 main pages
```

### Background Sync Trigger (LS.save → sbBgSync)

Every call to `LS.save()` automatically triggers `sbBgSync()` **unless** `_sbPulling === true`. This prevents sync from running while a pull is in progress (which would overwrite freshly-pulled data).

```
LS.save('posts', DATA)
  ├─ localStorage.setItem('nytg_posts', JSON.stringify(DATA))
  ├─ localStorage.setItem('nytg_posts_updated', ISO_timestamp)
  └─ if (!_sbPulling && type in ['posts','ads','leads'])
        └─ sbBgSync('posts', DATA)   ← async, silent errors
```

### Brand Switch Data Flow

```
setBrand('nic')
  └─ activeBrand = 'nic'
  └─ DATA=[], ADS_DATA=[], LEADS_DATA=[], AWN_DATA=[], LEAD_ADS_DATA=[]
  └─ initBrandData()
        ├─ LS.load('posts')  → reads 'nic_posts' from localStorage
        ├─ LS.load('ads')    → reads 'nic_ads'
        ├─ LS.load('leads')  → reads 'nic_leads'
        ├─ data found → showDashboard() + renderAll()
        └─ no data   → showEmptyState() → sbAutoPull()
```

### Year/Period Filter Flow

```
User clicks year tab → setYear('2025')
  └─ activeYear = '2025'
  └─ activePeriod = 'all'    ← always reset to 'all' on year change
  └─ renderAll()

User clicks month chip → setFilter('Jan 2025')
  └─ activePeriod = 'Jan 2025'
  └─ renderAll()

renderAll():
  └─ buildPeriodBar()         ← regenerates month chips for active year
  └─ renderExec()             ← uses DATA (unfiltered)
  └─ renderContent()          ← uses filtered()
  └─ renderCategory()         ← uses filtered()
  └─ renderBrand()            ← uses filtered()
```

### Lazy Page Rendering

CRM, Ads, Analysis, and Editor pages render **only when navigated to** or after a data change:

```
showPage('crm')
  └─ panels: hide all, show #page-crm
  └─ renderCrm()    ← triggered here, not in renderAll()

sbPull() completion:
  └─ renderAll()       ← exec/content/category/brand
  └─ renderLeadsAds()  ← leads from ads
  └─ renderCrm()       ← CRM panel
  └─ renderAds()       ← full ads panel
  └─ renderEditor()    ← data editor table
```

### Notes Data Flow

```
NOTES_DATA = { '2025-01': 'Campaign launched', '2025-03': 'Peak month' }

LS.save('notes', NOTES_DATA)
  └─ localStorage.setItem('nytg_notes', JSON.stringify(NOTES_DATA))
  └─ NOT synced to Supabase (notes are local-only)

Chart tooltip callback:
  └─ afterBody: ctx => {
       const note = NOTES_DATA[monthToKey(label)];
       return note ? ['📝 ' + note] : [];
     }
```

Notes appear as tooltip annotations on monthly trend charts. They are **not synced to Supabase** — notes are device/browser-local only.

---

## 6. External Integrations

### Supabase Auth

**Method:** Email + Password via `auth.signInWithPassword()`

**Session persistence:** Supabase JS stores the JWT in `localStorage` automatically. On refresh, `auth.getSession()` retrieves the cached session without a network call (if not expired).

**Why `getSession()` not `onAuthStateChange(INITIAL_SESSION)`:** The `INITIAL_SESSION` event fires before the DOM is ready on some browsers, causing a race condition where the login overlay flashes even with a valid session. Using `getSession()` directly at `DOMContentLoaded` guarantees timing.

**Sign-in timeout implementation:**
```js
let timedOut = false;
const tid = setTimeout(() => {
  timedOut = true;
  // show error, re-enable button
}, 15000);

// In callback:
clearTimeout(tid);
if (timedOut) return;  // Discard stale response
```

The 15-second timeout prevents the UI from staying in a "signing in..." disabled state indefinitely on slow connections.

### Supabase Database

**Delete pattern used for full table clear:**
```js
.delete().neq('id', '00000000-0000-0000-0000-000000000000')
```

This matches every row with a non-nil UUID — effectively "delete all." Supabase's PostgREST API does not allow `DELETE` without a filter, so this nil-UUID trick is required to clear a table.

**Batch insert size:** 200 rows per request. Supabase PostgREST has a default max payload size. Batching at 200 rows prevents 413 errors on large datasets.

**Concurrency guard (`_sbSyncing`):** Manual sync sets `_sbSyncing = true` immediately and clears it in a `finally` block. A second sync click while the first is running returns immediately, preventing the delete-then-insert from being interleaved with a second delete.

### SheetJS (xlsx) — Excel Parsing

**Usage pattern:**
```js
const wb = XLSX.read(new Uint8Array(e.target.result), { type: 'array' });
const ws = wb.Sheets[wb.SheetNames[0]];
const rows = XLSX.utils.sheet_to_json(ws, { defval: '' });
```

Reads the **first sheet** only. Uses `defval: ''` so missing cells return empty strings rather than `undefined`. Columns are accessed by their exact header string (case-sensitive).

**Used for:** Ads files (XLSX only), Posts files (XLSX + CSV supported)

**Not used for:** CRM/Leads files (CSV only)

### Chart.js — Chart Lifecycle

**Destroy-before-create pattern (required):**
```js
function destroyChart(id) {
  if (charts[id]) { charts[id].destroy(); delete charts[id]; }
  const el = document.getElementById(id);
  if (el) { const ex = Chart.getChart(el); if (ex) ex.destroy(); }
}
```

Two-step because the `charts` registry and Chart.js's internal canvas registry can get out of sync (e.g., if a chart was created outside this app's registry). Both are always cleared to prevent the "Canvas already in use" error on re-render.

**Global chart defaults (`CD` constant):**
```js
const CD = {
  plugins: { legend: { labels: { font: { family: 'DM Sans' }, ... } } },
  scales: {
    y: { grid: { color: 'rgba(...)' }, ticks: { font: { family: 'DM Mono' } } },
    x: { ... }
  }
};
```

All chart instances spread `...CD` as their base options, ensuring consistent typography and grid styling across all charts.

### Google Fonts

Loaded via a single `<link>` in `<head>`. Three families:
- **Syne** — headings, KPI values, brand logo
- **DM Sans** — all body text, buttons, labels
- **DM Mono** — all numeric/monospace values (KPIs, badges, table cells)

No fallback fonts are defined beyond the generic `sans-serif`. If Google Fonts is unavailable (offline), the app falls back to the browser's default sans-serif.

---

## 7. Workarounds & Special Implementations

### Workaround: `onAuthStateChange` Race Condition

**Problem:** `onAuthStateChange` fires an `INITIAL_SESSION` event very early in page load, sometimes before `localStorage` is fully available or before the app has finished initializing. This caused the login overlay to flash briefly — or permanently — even when a valid session existed.

**Solution:** Use `auth.getSession()` as the primary auth check at `DOMContentLoaded`. Register `onAuthStateChange` only to handle `SIGNED_OUT`:

```js
const { data: { session } } = await _supabase.auth.getSession();
if (session) { await _onSessionReady(session); }
else { _showLoginOverlay(); }

_supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_OUT') { _showLoginOverlay(); }
  // All other events (INITIAL_SESSION, USER_UPDATED, TOKEN_REFRESHED) ignored
});
```

### Workaround: Supabase Delete Without a WHERE Clause

**Problem:** Supabase PostgREST rejects `DELETE` requests with no filter (to prevent accidental full-table wipes via API).

**Solution:** Filter by `id != '00000000-0000-0000-0000-000000000000'`. The nil UUID is never a real primary key in Postgres `gen_random_uuid()` output, so this condition matches every real row.

```js
await _supabase.from('nytg_posts').delete()
  .neq('id', '00000000-0000-0000-0000-000000000000');
```

### Workaround: Month String Split for Supabase

**Problem:** Posts are stored locally with `month = "Jan 2025"` (full label for display), but Supabase's `nytg_posts.month` column stores only the 3-letter abbreviation `"Jan"`.

**Solution:** `_postsToSupa()` strips the year before upload:
```js
month: (d.month || '').split(' ')[0]  // "Jan 2025" → "Jan"
```

`_adsToSupa()` does the inverse — extracts year from the month string:
```js
year: d.month ? d.month.split(' ')[1] : ''  // "Jan 2025" → "2025"
```

### Workaround: localStorage Key Prefix for Multi-Brand

**Problem:** Both brands' data live in the same `localStorage` origin. Without namespacing, `nytg_posts` and `nic_posts` would collide.

**Solution:** The `LS` helper always prefixes with `activeBrand`:
```js
const LS = {
  key: (type) => `${activeBrand}_${type}`,
  ...
};
```

**Critical bug history:** An earlier version used `localStorage.removeItem('posts')` directly (no prefix), which silently failed because the actual key was `nytg_posts`. The fix was to iterate both brand prefixes explicitly during `sbPull()`:

```js
['nytg', 'nic'].forEach(brand => {
  ['posts','ads','leads','crm','ads_awn','ads_lead',
   'posts_updated','ads_updated','leads_updated',
   'crm_updated','ads_awn_updated','ads_lead_updated'
  ].forEach(type => localStorage.removeItem(`${brand}_${type}`));
});
```

### Workaround: Positional Navigation Instead of ID-Based

**Problem:** `showPage()` needs to sync two structures — the `pages` string array and the DOM `.ntab` elements — to highlight the correct sidebar item.

**Implementation:** `showPage(p)` finds the index of `p` in the `pages` array, then selects the nth `.ntab` element from the DOM:

```js
const pages = ['exec','content','category','brand','leads','crm','ads','editor','analysis','hub'];
// ...
const idx = pages.indexOf(p);
const tabs = document.querySelectorAll('.ntab');
tabs.forEach((t, i) => t.classList.toggle('active', i === idx));
```

**Constraint:** The order of `<div class="ntab">` elements in the HTML **must exactly match** the order in the `pages` array. Adding a new page requires updating both in tandem.

### Workaround: `nav{display:none}` Conflict

**Problem:** A CSS rule `nav { display: none !important }` was added to suppress the old `<nav>` top bar after the sidebar refactor. However, the sidebar itself contains a `<nav class="sb-nav">` element, which was hidden by the same rule, making all sidebar menu items invisible.

**Fix:** The `nav { display: none }` rule was removed entirely. The old top bar `<nav>` was replaced by the `<aside id="sidebar">` element, so no suppression rule is needed.

### Workaround: `publish_time` Column Removal

**Problem:** An earlier version of `_postsToSupa()` included a `publish_time` field in the Supabase insert payload. The `nytg_posts` table does not have this column, causing HTTP 400 errors on every sync.

**Fix:** `publish_time` was removed from `_postsToSupa()`. The `_postsFromSupa()` reverse mapper uses `date: ''` as a placeholder since the original publish date is not stored in Supabase.

### Workaround: Dual Chart Registry Cleanup

**Problem:** Chart.js maintains its own internal canvas registry. If the app's `charts` object loses a reference (e.g., page reload without destroying), re-creating a chart on the same `<canvas>` throws "Canvas is already in use."

**Solution:** `destroyChart(id)` performs two cleanup steps — deletes from the app registry AND queries Chart.js's own registry:

```js
function destroyChart(id) {
  if (charts[id]) { charts[id].destroy(); delete charts[id]; }
  const el = document.getElementById(id);
  if (el) {
    const ex = Chart.getChart(el);
    if (ex) ex.destroy();
  }
}
```

### Workaround: `_sbPulling` Guard for Background Sync

**Problem:** `LS.save()` triggers `sbBgSync()` automatically. If `sbPull()` is running and assigns new data via `LS.save()`, the background sync would immediately re-upload the freshly pulled data (a no-op but wasteful) and could race with the ongoing pull.

**Solution:** `sbPull()` sets `_sbPulling = true` before clearing localStorage and assigning data. `LS.save()` checks this flag before calling `sbBgSync()`. The flag is cleared in `sbPull()`'s `finally` block.

### Workaround: `renderLeads` Alias

**Problem:** Six places in the code called `renderLeads()`, but the function was never defined, causing `ReferenceError` crashes.

**Fix:** A stub was added that delegates to `renderCrm()`:

```js
function renderLeads() { renderCrm(); }
```

The name `renderLeads` refers to the same CRM panel. Both names are kept to avoid breaking call sites.

---

## Quick Reference — Thresholds & Constants

| Rule | Value |
|---|---|
| Reach cap threshold | `> 100` → display as `98` |
| Number format: K | `≥ 1,000` |
| Number format: M | `≥ 1,000,000` |
| Supabase batch size | 200 rows per insert |
| Auth sign-in timeout | 15,000 ms (15 seconds) |
| Sync delete filter | `.neq('id', '00000000-0000-0000-0000-000000000000')` |
| Post dedup key | `link` (Permalink URL) |
| Lead dedup key | `customer \| date \| channel` |
| Ad dedup key | `ad_name \| start_date` |
| Excluded categories | `['Greeting', 'Other']` |
| Objective: Lead | column contains `'lead'` (case-insensitive) |
| Objective: AWN | anything else |
| Low engagement threshold (analysis) | `avgEng <= 1` |
| High-reach post threshold (analysis) | `reach > 100,000` |
| Low brand-tagging threshold (analysis) | brand-tagged posts `< 50%` of total |
| Month range in MONTH_ORDER | Jan 2025 → Apr 2026 (16 months) |
| Default role (no Supabase row) | `'viewer'` |
| Default role (no Supabase at all) | `'admin'` |
