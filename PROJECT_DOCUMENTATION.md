# NYTG Social Media Dashboard — Project Documentation

> **Last updated:** June 2026  
> **Branch:** `claude/wizardly-hypatia-SpsnU`  
> **Repository:** `nytgmkt/nytg-social-dashboard`

---

## 1. Project Overview and Purpose

The **NYTG Social Media Dashboard** is a single-page web application for Nanyang Textile Group (NYTG) marketing teams to track, analyze, and manage social media performance across two brands: **NYTG** and **NIC**.

### What it does

- **Content Performance** — Visualizes post reach, engagement, views, interactions, and category breakdown by month and year
- **Brand Performance** — Compares brands and post types side by side
- **CRM / Lead Tracking** — Tracks sales pipeline leads by channel, stage, product category, and contact person
- **Ad Spending** — Analyzes Facebook/Meta ad spend, reach, CPM, and results for two objectives: Awareness (AWN) and Lead Generation
- **Data Editor** — In-browser inline editor for posts, leads, ads, and notes data
- **Data Hub** — Upload new data files, append monthly data, and sync/pull from Supabase cloud storage
- **Analysis** — Auto-generated insights, performance rankings, and recommendations based on loaded data

### Who uses it

- **Admins** — Can upload files, edit data, sync to cloud, and see all sections
- **Viewers** — Read-only access to all dashboard views; upload/edit/sync controls are hidden

---

## 2. Tech Stack and Dependencies

### Frontend

| Technology | Purpose |
|---|---|
| HTML5 / CSS3 / Vanilla JavaScript | Single-file app, no build step or framework |
| [Chart.js 4.4.1](https://www.chartjs.org/) | All charts (bar, line, doughnut, horizontal bar) |
| [SheetJS (xlsx) 0.18.5](https://sheetjs.com/) | Parsing `.xlsx` / `.xls` Excel files |
| [@supabase/supabase-js@2](https://supabase.com/docs/reference/javascript) | Auth (email/password) + Postgres database client |
| Google Fonts | **Syne** (headings), **DM Sans** (body), **DM Mono** (numbers/badges) |

### Backend / Infrastructure

| Service | Purpose |
|---|---|
| [Supabase](https://supabase.com) | PostgreSQL database, Row Level Security, Auth |
| Browser `localStorage` | Client-side data cache (no server needed for offline use) |

### CDN Links (in `<head>`)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<link href="https://fonts.googleapis.com/css2?family=Syne:wght@400;500;600;700;800&family=DM+Sans:wght@300;400;500;600&family=DM+Mono:wght@400;500&display=swap" rel="stylesheet">
```

---

## 3. Folder Structure

```
nytg-social-dashboard/
├── index.html                  # Entire application (HTML + CSS + JS, ~2500 lines)
├── PROJECT_DOCUMENTATION.md    # This file
└── (no other files — all assets are CDN-loaded)
```

The entire application lives in a **single HTML file**. There is no build pipeline, no `node_modules`, and no separate CSS or JS files. Deployment is simply hosting `index.html` on any static file server (GitHub Pages, Netlify, Supabase Storage, etc.).

---

## 4. Main Application Flow (Step by Step)

```
Browser loads index.html
    │
    ▼
CSS & CDN scripts load (Chart.js, SheetJS, Supabase)
    │
    ▼
DOMContentLoaded → initAuth()
    │
    ├─ Supabase available?
    │   ├─ YES → getSession()
    │   │         ├─ Session found → _onSessionReady(session)
    │   │         │     ├─ _checkUserRole(userId) → 'admin' or 'viewer'
    │   │         │     ├─ applyRoleVisibility(role)
    │   │         │     ├─ Update nav user pill (email + role badge)
    │   │         │     └─ initBrandData()
    │   │         └─ No session → _showLoginOverlay()
    │   └─ NO  → _showMainApp() + applyRoleVisibility('admin') + initBrandData()
    │
    ▼
initBrandData()
    ├─ Load localStorage keys for activeBrand (posts, ads, leads, notes)
    ├─ If data found → showDashboard() + renderAll()
    └─ If no data → showEmptyState()
         └─ sbAutoPull() → fetch from Supabase if available

User interactions:
    ├─ Upload file → parse → save to localStorage → renderAll() → sbBgSync()
    ├─ Year tab → setYear() → renderAll()
    ├─ Month chip → setFilter() → renderAll()
    ├─ Brand button → setBrand() → initBrandData()
    ├─ Sidebar nav → showPage() → render that page
    ├─ Sync button → sbSync() → delete + re-insert all 3 Supabase tables
    └─ Pull button → sbPull() → clear localStorage → fetch all 3 tables → renderAll()
```

---

## 5. Key Components / Modules and Their Responsibilities

### Authentication (`initAuth`, `signIn`, `signOut`, `_onSessionReady`, `_checkUserRole`)

Manages Supabase email/password auth. On load, calls `getSession()` (not `onAuthStateChange`) to avoid race conditions on refresh. The `onAuthStateChange` listener handles only `SIGNED_OUT` events. Role is fetched from the `user_roles` table immediately after session is confirmed.

### Data Storage — `LS` Helper Object

Thin wrapper around `localStorage` that automatically prefixes keys with `activeBrand_`:

```
nytg_posts          nic_posts
nytg_ads            nic_ads
nytg_leads          nic_leads
nytg_notes          nic_notes
nytg_posts_updated  (ISO timestamp of last save)
... etc.
```

`LS.save()` automatically triggers `sbBgSync()` after every write (unless a pull is in progress).

### Supabase Sync (`sbSync`, `sbPull`, `sbBgSync`, `sbAutoPull`)

| Function | When Called | What it Does |
|---|---|---|
| `sbSync()` | Manual (Sync button) | DELETE all rows in 3 tables, then INSERT current local data in 200-row batches |
| `sbPull()` | Manual (Pull button) | Clear all localStorage, fetch all 3 tables, repopulate local arrays, re-render |
| `sbBgSync()` | Auto after `LS.save()` | Background upsert of a single data type (posts/ads/leads); guarded by `_sbSyncing` flag |
| `sbAutoPull()` | App init (if no local data) | Pull from Supabase silently; shows loading overlay |

Sync uses `.neq('id','00000000-0000-0000-0000-000000000000')` as the delete filter (affects all real rows; UUID nil is never a real PK).

### CSV/Excel Parsers (`parseCSV`, `parseLeadsCSV`, `parseContentExcel`, `parseUnifiedAdsRow`)

- Posts files: accepts CSV (auto-detects comma/tab), XLSX, or XLS
- CRM files: CSV only
- Ads files: XLSX/XLS
- `parseUnifiedAdsRow()` detects objective (`Lead` vs `AWN`) from the `Ad Objective` column
- `parseDateStr()` handles both `DD/MM/YYYY` and `YYYY-MM-DD` formats

### Rendering Pipeline (`renderAll` and per-page render functions)

`renderAll()` calls: `buildPeriodBar()` → `renderExec()` → `renderContent()` → `renderCategory()` → `renderBrand()`.

Other pages render lazily when `showPage()` activates them (CRM, Ads, Analysis, Editor render on first visit or data change).

Each render function:
1. Reads `filtered()` (DATA filtered by `activeYear` + `activePeriod`)
2. Computes aggregates (sums, averages, top-N)
3. Updates DOM elements directly (`innerHTML`, `textContent`)
4. Calls `destroyChart(id)` then creates new Chart.js instance

### Navigation (`showPage`, `openSidebar`, `closeSidebar`)

`showPage(p)` matches page ID against the ordered `pages` array and activates the matching `.ntab` by index. Page IDs and `.ntab` elements **must stay in sync** — order is positional, not by ID.

```js
const pages = ['exec','content','category','brand','leads','crm','ads','editor','analysis','hub'];
```

### Data Editor (`renderEditor`, `editorSaveChanges`, `editorExportCSV`)

Renders a fully editable HTML table for posts, leads, ads, or notes. Cells use `contenteditable`. On save, reads all `<tr>` cells back into the appropriate data array via `editorRowFromTr()` and persists to localStorage.

### RBAC (`applyRoleVisibility`)

Adds/removes `viewer-mode` class on `<body>`. CSS rule `.viewer-mode .admin-only { display: none !important }` hides all upload, edit, sync, and hub controls for viewers.

---

## 6. API Endpoints

There are **no custom backend API endpoints**. All data operations use the Supabase JavaScript client which communicates directly with the Supabase PostgREST API.

### Supabase Operations Used

| Operation | Supabase Call |
|---|---|
| Sign in | `auth.signInWithPassword({email, password})` |
| Sign out | `auth.signOut()` |
| Get session | `auth.getSession()` |
| Auth state changes | `auth.onAuthStateChange(callback)` |
| Get user role | `from('user_roles').select('role').eq('user_id', id).single()` |
| Delete all posts | `from('nytg_posts').delete().neq('id', '00000000-...')` |
| Insert posts batch | `from('nytg_posts').insert(rows)` |
| Fetch all posts | `from('nytg_posts').select('*')` |
| Delete/insert/fetch ads | same pattern on `nytg_ads` |
| Delete/insert/fetch CRM | same pattern on `nytg_crm` |

---

## 7. Database Schema

### `nytg_posts` — Social Media Posts

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key (auto-generated) |
| `brand` | text | `'nytg'` or `'nic'` |
| `year` | integer | e.g. `2025` |
| `month` | text | 3-letter month, e.g. `'Jan'` |
| `reach` | integer | Organic reach count |
| `total_engagement` | integer | Total interactions |
| `percent_reach` | numeric | % Reach (0–100) |
| `percent_engagement` | numeric | % Engagement rate |
| `post_type` | text | e.g. `'Video'`, `'Image'`, `'Reel'` |
| `category` | text | Primary category tag (cat1) |
| `permalink` | text | Facebook post URL |
| `views` | integer | Video view count |
| `reactions` | integer | Like/love/etc count |
| `comments` | integer | Comment count |
| `shares` | integer | Share count |

### `nytg_ads` — Ad Spending

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `year` | text | Year string |
| `month` | text | Month string |
| `ad_objective` | text | `'AWN'` (Awareness) or `'Lead'` |
| `ad_name` | text | Facebook ad name |
| `ad_set_name` | text | Ad set name |
| `amount_spent` | numeric | Spend in THB |
| `reach` | integer | People reached |
| `impressions` | integer | Total impressions |
| `results` | numeric | Conversions / ThruPlays / etc. |
| `cost_per_result` | numeric | CPR in THB |
| `cpm` | numeric | Cost per 1,000 impressions |

### `nytg_crm` — CRM / Lead Tracking

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `date` | text | Date string (DD/MM/YYYY) |
| `customer_name` | text | Customer / account name |
| `channel` | text | Lead source channel (e.g. `'FB-NYTG'`) |
| `stage` | text | Sales stage (e.g. `'Follow Up'`, `'Closed Won'`) |
| `product_category` | text | Product interest category |
| `contact_person` | text | Internal sales contact |
| `est_quantity` | text | Estimated quantity |
| `est_value` | text | Estimated deal value |
| `notes` | text | Free-text notes |
| `bounce` | text | Optional bounce/churn flag |

### `user_roles` — Access Control

| Column | Type | Notes |
|---|---|---|
| `user_id` | UUID | References `auth.users.id` |
| `role` | text | `'admin'` or `'viewer'` |

> **Note:** Row Level Security (RLS) must be configured in Supabase so users can only read their own role row and the anon key can read/write data tables.

---

## 8. Environment Variables / Configuration

This project has **no `.env` file** — it is a static HTML file. Configuration is hardcoded in `index.html` at the top of the `<script>` block:

```js
const _supabase = window.supabase?.createClient?.(
  'https://bsnblfkggxjqxuhdqbwf.supabase.co',      // SUPABASE_URL
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'        // SUPABASE_ANON_KEY
) ?? null;
```

### If you need to deploy to a different Supabase project

1. Open `index.html` and locate the `_supabase` initialization (search for `createClient`)
2. Replace the URL with your project URL: `https://<ref>.supabase.co`
3. Replace the anon key with your project's `anon` public key (safe to expose in browser)
4. Create the required tables (see Database Schema above) in your Supabase project
5. Create rows in `user_roles` for each authorized user

### `.env` Template (for future migration to a build-based setup)

```env
SUPABASE_URL=https://your-project-ref.supabase.co
SUPABASE_ANON_KEY=eyJhbGci...your-anon-key
```

---

## 9. How to Run the Project Locally

### Prerequisites

- Any modern web browser (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+)
- No Node.js, no npm, no build tools required

### Option A — Open Directly (simplest)

```bash
# Clone the repository
git clone https://github.com/nytgmkt/nytg-social-dashboard.git
cd nytg-social-dashboard

# Open in browser
open index.html          # macOS
start index.html         # Windows
xdg-open index.html      # Linux
```

> **Note:** Some browsers block `localStorage` on `file://` URLs. If you see issues, use Option B.

### Option B — Local HTTP Server (recommended)

```bash
# Using Python (built into macOS/Linux)
python3 -m http.server 8080
# Then open: http://localhost:8080

# Using Node.js (if installed)
npx serve .
# Then open: http://localhost:3000

# Using VS Code
# Install the "Live Server" extension, then right-click index.html → "Open with Live Server"
```

### Option C — Deploy to GitHub Pages

1. Push `index.html` to the `main` branch of a GitHub repository
2. Go to **Settings → Pages → Source → Deploy from branch → main**
3. Access at `https://<org>.github.io/<repo>/`

---

## 10. Data Upload Guide

### Supported File Formats

| Data Type | Formats | Source |
|---|---|---|
| Posts / Content | `.csv`, `.xlsx`, `.xls` | Facebook Insights export |
| CRM / Leads | `.csv` | Internal CRM export |
| Ad Spending | `.xlsx`, `.xls` | Facebook Ads Manager export |

### Required Column Names

**Posts CSV/XLSX** (case-sensitive):
```
Permalink, Publish time, Post type, Category 1, Category 2, Category 3,
%Reach, %Engagement Rate, Reach, View, Year, Month, Brand, Link clicks,
Reactions, Comments, Shares
```

**CRM CSV**:
```
Date, Customer / Account Name, Channel, Stage, Product Category,
Contact Person, Estimated Quantity, Est. value, Notes
```

**Ads XLSX**:
```
Ad name, Ad set name, Ad Objective, Amount spent (THB), Results,
Reach, Impressions, CPM (cost per 1,000 impressions) (THB), Reporting starts
```

### Upload Workflow

1. **Load New** — Replaces all existing data for that type; prompts confirmation
2. **Add Month** — Appends new rows; deduplicates automatically:
   - Posts: by `permalink` (link)
   - CRM: by `customer|date|channel`
   - Ads: by `ad_name|start_date`

---

## 11. Global State Reference

| Variable | Type | Description |
|---|---|---|
| `DATA` | Array | All loaded posts for active brand |
| `ADS_DATA` | Array | Unified ads (AWN + Lead objectives) |
| `LEADS_DATA` | Array | CRM lead records |
| `AWN_DATA` | Array | Legacy: awareness-only ads |
| `LEAD_ADS_DATA` | Array | Legacy: lead-only ads |
| `NOTES_DATA` | Object | Monthly notes keyed by `YYYY-MM` |
| `activeBrand` | string | `'nytg'` or `'nic'` |
| `activePeriod` | string | `'all'` or `'Jan 2025'` etc. |
| `activeYear` | string | `'all'`, `'2025'`, or `'2026'` |
| `currentUserRole` | string\|null | `'admin'`, `'viewer'`, or `null` |
| `currentUserEmail` | string\|null | Signed-in user email |
| `_sbSyncing` | boolean | Prevents duplicate sync calls |
| `_sbPulling` | boolean | Prevents background sync during pull |
| `charts` | Object | Map of Chart.js instances by canvas ID |
| `editorType` | string | Active editor tab: `'posts'`, `'leads'`, `'ads'`, `'notes'` |

---

## 12. Color Palette (CSS Custom Properties)

```css
:root {
  --bg:         #F0F4F8;   /* Page background */
  --surface:    #FFFFFF;   /* Card / panel background */
  --surface2:   #E8EEF5;   /* Table row alt / muted areas */
  --ink:        #0D1F3C;   /* Primary text / dark navy */
  --ink2:       #3A5070;   /* Secondary text */
  --muted:      #7A96B0;   /* Labels, captions */
  --border:     #C8D8E8;   /* Dividers */
  --teal:       #1A6B8A;   /* Primary accent */
  --teal2:      #0F4C75;   /* Darker accent */
  --teal-lt:    #DCE9F5;   /* Light accent background */
  --blue:       #5B9BD5;   /* Sky blue / secondary accent */
  --blue-lt:    #DCE9F5;   /* Light blue background */
  --amber:      #2E6DA4;   /* Tertiary accent (blue family) */
  --amber-lt:   #DCE9F5;   /* Light amber background */
  --red:        #C0351C;   /* Error / danger / Event category */
  --purple:     #3B5998;   /* Facebook blue / VI category */
  --shadow:     0 1px 4px rgba(26,58,107,.08), 0 4px 16px rgba(26,58,107,.05);
}
```

### Category Colors (`CAT_COLORS`)

| Category | Color | Hex |
|---|---|---|
| Innovation | Deep Navy | `#1A3A6B` |
| Sustainability | Teal Blue | `#1A6B8A` |
| VI | Medium Blue | `#2E6DA4` |
| Commodity | Sky Blue | `#5B9BD5` |
| ESG | Dark Navy | `#0F4C75` |
| Event | Red | `#C0351C` |

---

## 13. Known Limitations

- **Single-file architecture** — All code is in one 2500-line HTML file. Changes require careful in-file editing.
- **No automated tests** — No test suite; all validation is manual.
- **Browser localStorage limit** — ~5–10 MB per origin. Large datasets (10,000+ rows) may exceed this.
- **Supabase anon key is public** — This is expected for Supabase; security is enforced by Row Level Security (RLS) policies, not key secrecy.
- **Multi-brand sync** — `sbSync()` currently syncs only the active brand's data. Switching brands and syncing is a separate operation.
- **No offline-first conflict resolution** — If two users sync simultaneously, the last writer wins.
- **Legacy ad inputs** — `awnFileIn`, `leadAdsFileIn` and related handlers exist for backwards compatibility; new uploads should use the unified `adsUnifiedFileIn` input.
