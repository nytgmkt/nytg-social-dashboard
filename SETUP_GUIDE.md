# NYTG Social Media Dashboard — Setup Guide

> This guide covers everything needed to recreate the project from scratch on a new machine, new Supabase project, or new hosting environment.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Required Accounts & Services](#2-required-accounts--services)
3. [Step-by-Step Setup](#3-step-by-step-setup)
   - 3.1 [Clone & Verify the Project](#31-clone--verify-the-project)
   - 3.2 [Create the Supabase Project](#32-create-the-supabase-project)
   - 3.3 [Create the Database Tables](#33-create-the-database-tables)
   - 3.4 [Set Up Row Level Security (RLS)](#34-set-up-row-level-security-rls)
   - 3.5 [Create User Accounts](#35-create-user-accounts)
   - 3.6 [Configure index.html](#36-configure-indexhtml)
   - 3.7 [Run Locally](#37-run-locally)
   - 3.8 [Deploy to Production](#38-deploy-to-production)
4. [Configuration Reference](#4-configuration-reference)
5. [Common Issues & Solutions](#5-common-issues--solutions)
6. [.env.example](#6-envexample)
7. [Recreating from Scratch Checklist](#7-recreating-from-scratch-checklist)

---

## 1. Prerequisites

| Tool | Required | Purpose |
|---|---|---|
| Modern browser | ✅ Required | Chrome 90+, Firefox 88+, Safari 14+, Edge 90+ |
| Git | ✅ Required | Clone and push code |
| Text editor | ✅ Required | VS Code recommended |
| Python 3 or Node.js | Optional | Local HTTP server (avoids `file://` issues) |
| Supabase account | ✅ Required | Auth + database backend |
| GitHub account | Optional | Version control + GitHub Pages hosting |

**No Node.js, no npm, no build step is required** for the app itself. Python/Node are only needed if you want a local dev server instead of opening the file directly.

---

## 2. Required Accounts & Services

### 2.1 Supabase (free tier is sufficient)

- Sign up at [https://supabase.com](https://supabase.com)
- Free tier includes: 500 MB database, 1 GB file storage, 50,000 MAU auth
- You need: **Project URL** and **anon (public) API key**

### 2.2 GitHub (optional, for hosting)

- Used for version control and GitHub Pages deployment
- Free tier is sufficient
- Repository: `nytgmkt/nytg-social-dashboard`

### 2.3 Google Fonts (no account needed)

- Fonts are loaded via CDN automatically
- No API key required
- Works offline only if the browser has cached the fonts previously

---

## 3. Step-by-Step Setup

### 3.1 Clone & Verify the Project

```bash
git clone https://github.com/nytgmkt/nytg-social-dashboard.git
cd nytg-social-dashboard
ls
# Should see: index.html, PROJECT_DOCUMENTATION.md, BUSINESS_LOGIC.md, SETUP_GUIDE.md
```

Verify the file is intact — it should be a single `index.html` around 2,500 lines:

```bash
wc -l index.html
# Expected: ~2500 lines
```

---

### 3.2 Create the Supabase Project

1. Log in to [https://supabase.com/dashboard](https://supabase.com/dashboard)
2. Click **New project**
3. Fill in:
   - **Name:** `nytg-social-dashboard` (or any name)
   - **Database Password:** Save this securely — you'll need it if you ever use direct DB access
   - **Region:** Choose closest to your users (e.g., Southeast Asia → Singapore `ap-southeast-1`)
4. Wait ~2 minutes for the project to provision
5. Go to **Project Settings → API**
6. Copy and save:
   - **Project URL** — looks like `https://xxxxxxxxxxxx.supabase.co`
   - **anon / public key** — the `eyJhbGci...` JWT string under "Project API Keys"

> **Note:** The `anon` key is safe to expose in browser-side code. Security is enforced by Row Level Security policies, not key secrecy.

---

### 3.3 Create the Database Tables

Go to **Supabase Dashboard → SQL Editor** and run the following SQL blocks one at a time.

#### Table 1: `nytg_posts` (Social Media Posts)

```sql
CREATE TABLE nytg_posts (
  id                 UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  brand              TEXT,
  year               INTEGER,
  month              TEXT,
  reach              INTEGER DEFAULT 0,
  total_engagement   INTEGER DEFAULT 0,
  percent_reach      NUMERIC DEFAULT 0,
  percent_engagement NUMERIC DEFAULT 0,
  post_type          TEXT,
  category           TEXT,
  permalink          TEXT,
  views              INTEGER DEFAULT 0,
  reactions          INTEGER DEFAULT 0,
  comments           INTEGER DEFAULT 0,
  shares             INTEGER DEFAULT 0,
  created_at         TIMESTAMPTZ DEFAULT now()
);
```

#### Table 2: `nytg_ads` (Ad Spending)

```sql
CREATE TABLE nytg_ads (
  id               UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  year             TEXT,
  month            TEXT,
  ad_objective     TEXT,
  ad_name          TEXT,
  ad_set_name      TEXT,
  amount_spent     NUMERIC DEFAULT 0,
  reach            INTEGER DEFAULT 0,
  impressions      INTEGER DEFAULT 0,
  results          NUMERIC DEFAULT 0,
  cost_per_result  NUMERIC DEFAULT 0,
  cpm              NUMERIC DEFAULT 0,
  created_at       TIMESTAMPTZ DEFAULT now()
);
```

#### Table 3: `nytg_crm` (CRM / Lead Tracking)

```sql
CREATE TABLE nytg_crm (
  id               UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  date             TEXT,
  customer_name    TEXT,
  channel          TEXT,
  stage            TEXT,
  product_category TEXT,
  contact_person   TEXT,
  est_quantity     TEXT,
  est_value        TEXT,
  notes            TEXT,
  bounce           TEXT,
  created_at       TIMESTAMPTZ DEFAULT now()
);
```

#### Table 4: `user_roles` (Access Control)

```sql
CREATE TABLE user_roles (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id    UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  role       TEXT NOT NULL CHECK (role IN ('admin', 'viewer')),
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id)
);
```

Verify all 4 tables exist:

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
-- Expected: nytg_ads, nytg_crm, nytg_posts, user_roles
```

---

### 3.4 Set Up Row Level Security (RLS)

RLS must be enabled so the `anon` key cannot bypass access controls.

Run in **SQL Editor**:

```sql
-- Enable RLS on all tables
ALTER TABLE nytg_posts  ENABLE ROW LEVEL SECURITY;
ALTER TABLE nytg_ads    ENABLE ROW LEVEL SECURITY;
ALTER TABLE nytg_crm    ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_roles  ENABLE ROW LEVEL SECURITY;

-- nytg_posts: authenticated users can read/write
CREATE POLICY "auth_read_posts"  ON nytg_posts FOR SELECT USING (auth.role() = 'authenticated');
CREATE POLICY "auth_insert_posts" ON nytg_posts FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "auth_delete_posts" ON nytg_posts FOR DELETE USING (auth.role() = 'authenticated');

-- nytg_ads: authenticated users can read/write
CREATE POLICY "auth_read_ads"    ON nytg_ads FOR SELECT USING (auth.role() = 'authenticated');
CREATE POLICY "auth_insert_ads"  ON nytg_ads FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "auth_delete_ads"  ON nytg_ads FOR DELETE USING (auth.role() = 'authenticated');

-- nytg_crm: authenticated users can read/write
CREATE POLICY "auth_read_crm"    ON nytg_crm FOR SELECT USING (auth.role() = 'authenticated');
CREATE POLICY "auth_insert_crm"  ON nytg_crm FOR INSERT WITH CHECK (auth.role() = 'authenticated');
CREATE POLICY "auth_delete_crm"  ON nytg_crm FOR DELETE USING (auth.role() = 'authenticated');

-- user_roles: users can only read their own role
CREATE POLICY "read_own_role"    ON user_roles FOR SELECT USING (auth.uid() = user_id);
```

> **Why this matters:** Without RLS, anyone who discovers your `anon` key can read/delete your entire database via the Supabase API. With RLS, only authenticated (logged-in) users can access data.

---

### 3.5 Create User Accounts

#### Step A — Create the Auth User

Go to **Supabase Dashboard → Authentication → Users → Add user**:

- **Email:** `admin@yourdomain.com`
- **Password:** Choose a strong password
- **Auto Confirm User:** ✅ Check this (skips email confirmation)

Repeat for each user who needs access.

#### Step B — Assign a Role

After creating the user, you need their UUID. Run in **SQL Editor**:

```sql
SELECT id, email FROM auth.users ORDER BY created_at DESC;
```

Then insert their role:

```sql
-- Make a user an admin
INSERT INTO user_roles (user_id, role)
VALUES ('paste-uuid-here', 'admin');

-- Make a user a viewer
INSERT INTO user_roles (user_id, role)
VALUES ('paste-uuid-here', 'viewer');
```

> **Role logic:** If a user has no row in `user_roles`, the app treats them as `viewer` by default. If Supabase is unreachable, the app assumes `admin` (offline mode).

---

### 3.6 Configure index.html

Open `index.html` in your text editor. Search for `createClient` (around line 724):

```js
const _supabase = window.supabase?.createClient?.(
  'https://YOUR_PROJECT_REF.supabase.co',
  'YOUR_ANON_KEY'
) ?? null;
```

Replace the two placeholder values with your actual Supabase credentials from Step 3.2:

```js
const _supabase = window.supabase?.createClient?.(
  'https://abcdefghijklmnop.supabase.co',          // ← your Project URL
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.ey...'   // ← your anon key
) ?? null;
```

**Save the file.** That is the only configuration change required.

---

### 3.7 Run Locally

#### Option A — Python (recommended, built into macOS/Linux)

```bash
cd nytg-social-dashboard
python3 -m http.server 8080
```

Open your browser at: **http://localhost:8080**

#### Option B — Node.js

```bash
npx serve .
# Opens at http://localhost:3000
```

#### Option C — VS Code Live Server

1. Install the **Live Server** extension in VS Code
2. Right-click `index.html` → **Open with Live Server**
3. Browser opens automatically at `http://127.0.0.1:5500`

#### Option D — Open Directly (no server)

```bash
open index.html        # macOS
start index.html       # Windows
xdg-open index.html    # Linux
```

> **Warning:** Opening via `file://` URL can block `localStorage` in some browser security settings. Use a local server if you experience auth or data persistence issues.

#### Verify the Setup

1. The login overlay should appear
2. Sign in with the admin credentials you created in Step 3.5
3. You should see the dashboard with "ยังไม่มีข้อมูล" (No data yet)
4. Click **☁️ Data Hub** → **⬇️ Pull from Supabase** (should succeed with empty data)
5. Try uploading a sample via **☁️ Data Hub → ▶ ดู Sample Data**

---

### 3.8 Deploy to Production

#### Option A — GitHub Pages (free, recommended)

```bash
# Push to main branch
git add index.html
git commit -m "Configure for production"
git push origin main
```

In GitHub repository settings:
1. Go to **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: `main` / `/ (root)`
4. Click **Save**

Your app will be live at: `https://nytgmkt.github.io/nytg-social-dashboard/`

> Allow 2–5 minutes for the first deployment.

#### Option B — Netlify (drag and drop)

1. Go to [https://netlify.com](https://netlify.com)
2. Drag the `index.html` file onto the Netlify dashboard
3. Done — you get a random URL like `https://amazing-darwin-123456.netlify.app`
4. Optionally configure a custom domain in Netlify settings

#### Option C — Supabase Storage (serve from same project)

1. Supabase Dashboard → **Storage → New bucket** → name: `public-site`
2. Set bucket to **Public**
3. Upload `index.html`
4. Access at: `https://YOUR_PROJECT_REF.supabase.co/storage/v1/object/public/public-site/index.html`

#### Option D — Any Static Host

The file is a single `.html` file with no dependencies to install. Any static host works:
- **Cloudflare Pages** — connect GitHub repo, auto-deploys on push
- **Vercel** — same as Cloudflare Pages
- **AWS S3 + CloudFront** — upload file, enable static website hosting
- **Company intranet** — copy file to any web server directory

---

## 4. Configuration Reference

### Where Configuration Lives

This project has **no `.env` file** and **no config files**. All configuration is a two-line change inside `index.html`:

```
index.html
  Line ~724: Supabase URL
  Line ~725: Supabase anon key
```

### All Configurable Values

| Setting | Location in index.html | Current Value | How to Change |
|---|---|---|---|
| Supabase URL | `createClient(` first arg | `https://bsnblfk...supabase.co` | Replace string |
| Supabase anon key | `createClient(` second arg | `eyJhbGci...` JWT | Replace string |
| Auth timeout (ms) | `setTimeout(..., 15000)` in `signIn()` | `15000` (15 sec) | Change number |
| Supabase batch size | `i+=200` in `sbSync()`/`sbBgSync()` | `200` rows | Change number |
| Month range | `MONTH_ORDER` array | Jan 2025 – Apr 2026 | Extend array |
| Excluded categories | `EXCLUDE_CATS` array | `['Greeting','Other']` | Add/remove strings |
| Category colors | `CAT_COLORS` object | Navy/blue palette | Change hex values |
| Brand colors | `BRAND_COLORS` array | 10 blue-family shades | Change hex values |
| Static category data | `STATIC_CAT` object | 2025 + 2026 targets | Update percentages |
| App title | `<title>` tag | NYTG · Social Media Dashboard | Edit HTML |
| Sidebar logo text | `<div class="sb-logo">NYTG` | `NYTG.` | Edit HTML |

### Extending the Month Range

The `MONTH_ORDER` constant controls the valid month sequence. Data outside this range will not sort correctly. To extend into 2027:

```js
// Find MONTH_ORDER in index.html (~line 748) and append:
const MONTH_ORDER = [
  'Jan 2025','Feb 2025', ..., 'Dec 2025',
  'Jan 2026','Feb 2026', ..., 'Dec 2026',  // add these
  'Jan 2027','Feb 2027', ..., 'Apr 2027'   // extend as needed
];
```

### Adding a New Dashboard Page

1. Add page ID string to the `pages` array (keep order)
2. Add a new `.ntab` div in `<nav class="sb-nav">` at the matching index position
3. Add a `<div class="panel" id="page-{id}">` in the main content area
4. Create a `render{PageName}()` JavaScript function
5. Call that function from `showPage()` when the new page ID is active

> **Critical:** The `.ntab` DOM order must exactly match the `pages` array index order. Breaking this makes the wrong sidebar item highlight.

---

## 5. Common Issues & Solutions

### Issue: Login Overlay Appears Even After Signing In (Refresh Bug)

**Symptom:** You sign in, refresh the page, and the login form reappears.

**Cause:** The Supabase session is not being restored from localStorage on page load.

**Solution:**
1. Open browser DevTools → Application → Local Storage
2. Check if `sb-*-auth-token` keys exist for your Supabase URL
3. If they do, the bug is in the code — check that `initAuth()` calls `getSession()`, not `onAuthStateChange(INITIAL_SESSION)`
4. If they don't exist, Supabase is not saving the session. Check that the URL is correct and the anon key is valid

```js
// Correct pattern in initAuth():
const { data: { session } } = await _supabase.auth.getSession();
if (session) { await _onSessionReady(session); }
```

---

### Issue: Sign In Button Does Nothing / Stays Disabled

**Symptom:** Click "เข้าสู่ระบบ", button stays disabled, nothing happens after 15 seconds.

**Cause A:** Wrong Supabase URL or anon key in `index.html`.

**Fix A:** Open DevTools → Console. Look for network errors on `supabase.co`. Verify the URL and key match your Supabase project.

**Cause B:** User email doesn't exist in Supabase Auth.

**Fix B:** Supabase Dashboard → Authentication → Users. Confirm the email exists and is confirmed.

**Cause C:** User exists but password is wrong.

**Fix C:** Supabase Dashboard → Authentication → Users → click the user → "Send password reset".

**Cause D:** Browser is blocking the Supabase CDN.

**Fix D:** Check DevTools → Console for `Failed to load resource` errors. Ensure you're not behind a firewall that blocks `cdn.jsdelivr.net` or `supabase.co`.

---

### Issue: Sync Returns 400 Error

**Symptom:** "Sync error: ..." toast appears. Console shows HTTP 400 from Supabase.

**Cause A:** Table schema mismatch — a column in the insert payload doesn't exist in the table.

**Fix A:** Compare the fields in `_postsToSupa()`, `_adsToSupa()`, `_crmToSupa()` against the actual table columns in Supabase Dashboard → Table Editor. Remove any extra fields.

**Cause B:** RLS is blocking the delete/insert.

**Fix B:** Check that the RLS policies from Step 3.4 are applied. In Supabase Dashboard → Authentication → Policies, you should see 3 policies per data table.

**Cause C:** The user is not authenticated (JWT expired).

**Fix C:** Sign out and sign back in to refresh the session token.

---

### Issue: Pull Returns Empty Data / "No Data" After Pull

**Symptom:** Pull completes successfully but dashboard shows empty state.

**Cause A:** The Supabase tables are empty — no data has been synced yet.

**Fix A:** Upload a CSV file first (or use Sample Data), then Sync to push to Supabase, then Pull.

**Cause B:** `activeBrand` is set to `nic` but all data was synced under `nytg` brand.

**Fix B:** Switch to the NYTG brand button in the sidebar before pulling.

---

### Issue: Menu Items Not Visible in Sidebar

**Symptom:** Sidebar appears but is empty (no nav items).

**Cause:** A CSS rule like `nav { display: none }` is hiding the `<nav class="sb-nav">` element inside the sidebar.

**Fix:** Search `index.html` for `nav{display:none` or `nav { display: none`. If found, remove it. This rule was added in an older version to suppress the top navbar and accidentally hides the sidebar menu.

---

### Issue: Charts Show "Canvas already in use" Error

**Symptom:** Console error: `Canvas is already in use. Chart with ID X must be destroyed before the canvas can be reused.`

**Cause:** A chart was not destroyed before re-rendering.

**Fix:** Ensure every chart creation is preceded by `destroyChart(canvasId)`. Example:

```js
destroyChart('myChartId');
charts.myChartId = new Chart(document.getElementById('myChartId'), { ... });
```

---

### Issue: Data Duplicates After Sync

**Symptom:** After syncing, the same posts appear twice when you pull.

**Cause:** The delete operation in `sbSync()` failed silently, but the insert still ran.

**Fix:** The current code already checks delete errors. If duplicates persist:
1. Manually clear the table: Supabase Dashboard → Table Editor → `nytg_posts` → delete all rows
2. Click Sync again from the dashboard

---

### Issue: Month Filter Chips Out of Order

**Symptom:** Month chips appear in wrong order (e.g., "Mar 2026" before "Jan 2026").

**Cause:** A month string in your data doesn't match the format in `MONTH_ORDER` exactly.

**Fix:** Check that months in your CSV are in `"MMM YYYY"` format (e.g., `Jan 2025`, not `January 2025` or `01-2025`). The app normalises this during `parseDateStr()`, but if the month field is passed in directly without parsing, it may not match.

---

### Issue: localStorage Data Lost After Closing Browser

**Symptom:** Every time the browser is closed and reopened, data disappears.

**Cause A:** Browser is set to clear storage on close (Private/Incognito mode or browser setting).

**Fix A:** Use a normal browsing window, not Incognito. Check browser privacy settings.

**Cause B:** Data exceeds localStorage quota (~5–10 MB).

**Fix B:** Reduce dataset size or use Supabase as the primary store (sync up, then pull fresh each session).

---

### Issue: Excel Upload Returns "No valid data found"

**Symptom:** Uploading an `.xlsx` file shows a toast error about no valid data.

**Cause:** The first sheet does not contain the expected column headers, or the file uses a different sheet structure.

**Fix:**
1. Open the file in Excel and check: is the data on Sheet 1?
2. Check that required column names match exactly (case-sensitive). See required columns in `BUSINESS_LOGIC.md` Section 3.
3. If headers are on row 2 (with a title row on row 1), delete row 1 so headers are on the first row.

---

### Issue: Supabase Operations Fail in Production but Work Locally

**Symptom:** Auth works locally, but deployed version (GitHub Pages etc.) cannot sign in.

**Cause:** Supabase blocks requests from unauthorized origins.

**Fix:** Supabase Dashboard → Authentication → URL Configuration:
- Add your production URL to **Allowed Origins**: e.g., `https://nytgmkt.github.io`
- Add to **Site URL**: `https://nytgmkt.github.io/nytg-social-dashboard/`
- Add to **Redirect URLs**: same URL

---

## 6. .env.example

> This project does not use a `.env` file — all config is in `index.html`. This template is provided as a reference for anyone migrating to a build-based setup (Vite, webpack, etc.).

```env
# ─────────────────────────────────────────────
# NYTG Social Media Dashboard — Environment Variables
# Copy this file to .env and fill in real values.
# Never commit .env to version control.
# ─────────────────────────────────────────────

# ── Supabase ──────────────────────────────────
# Project URL from: Supabase Dashboard → Project Settings → API → Project URL
SUPABASE_URL=https://your-project-ref.supabase.co

# Anon/Public key from: Supabase Dashboard → Project Settings → API → anon public
# This key is safe to expose in browser-side code.
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.your-anon-key-here

# Service role key — NEVER use this in browser code.
# Only needed if you build a server-side admin tool.
# From: Supabase Dashboard → Project Settings → API → service_role (secret)
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.your-service-role-key-here

# ── App Settings ──────────────────────────────
# How long (ms) to wait before showing a sign-in timeout error
AUTH_TIMEOUT_MS=15000

# Number of rows to insert per Supabase batch request
SUPABASE_BATCH_SIZE=200

# ── Deployment ────────────────────────────────
# The public URL where the app is hosted (used for Supabase auth redirects)
SITE_URL=https://nytgmkt.github.io/nytg-social-dashboard/
```

---

## 7. Recreating from Scratch Checklist

Use this checklist if you are setting up the project on a new machine or new Supabase project.

### Supabase Setup
- [ ] Created new Supabase project
- [ ] Copied Project URL
- [ ] Copied anon public key
- [ ] Created `nytg_posts` table with correct columns
- [ ] Created `nytg_ads` table with correct columns
- [ ] Created `nytg_crm` table with correct columns
- [ ] Created `user_roles` table with correct columns
- [ ] Enabled RLS on all 4 tables
- [ ] Created RLS policies (SELECT, INSERT, DELETE for data tables; SELECT own row for user_roles)
- [ ] Added production URL to Supabase Allowed Origins
- [ ] Added production URL to Supabase Site URL and Redirect URLs

### User Accounts
- [ ] Created at least one Auth user in Supabase Dashboard
- [ ] User email confirmed (Auto Confirm checked)
- [ ] Inserted a row in `user_roles` for each user with correct role (`admin` or `viewer`)

### Code Configuration
- [ ] Opened `index.html` and replaced Supabase URL at line ~724
- [ ] Replaced Supabase anon key at line ~725
- [ ] Verified `MONTH_ORDER` covers your current date range
- [ ] Verified `STATIC_CAT` percentages match your actual category targets

### Local Testing
- [ ] Started local HTTP server (`python3 -m http.server 8080`)
- [ ] Opened http://localhost:8080 in browser
- [ ] Login overlay appears
- [ ] Can sign in with admin credentials
- [ ] Dashboard loads without console errors
- [ ] "Sample Data" button works and renders charts
- [ ] Sync button works (success toast)
- [ ] Pull button works (reloads data)
- [ ] Data Editor tab visible (admin role)

### Production Deployment
- [ ] Pushed latest `index.html` to main branch
- [ ] GitHub Pages / Netlify / host configured and deployed
- [ ] Production URL loads the login page
- [ ] Sign in works from production URL
- [ ] Supabase allowed origins include the production URL

---

*For architecture details, see `PROJECT_DOCUMENTATION.md`.  
For business rules and formulas, see `BUSINESS_LOGIC.md`.*
