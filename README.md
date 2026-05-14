# Rain Control Tower — Setup Guide
## SQL → Google Sheets → Dashboard → GitHub Pages

---

## PART 1: Google Sheet Column Specification

Your SQL query must write **one aggregated row per city** to a Google Sheet tab named `CITY_METRICS`.

The dashboard reads this sheet directly. Column names must match exactly (case-insensitive, underscores).

### Required Columns (in order)

| # | Column Name | Type | Example | SQL Notes |
|---|---|---|---|---|
| 1 | `city` | Text | Mumbai | City name — consistent spelling |
| 2 | `bau_pipeline` | Number | 3.5 | Static — set once, don't overwrite in SQL |
| 3 | `surface_tat` | Number | 5 | Static — set once |
| 4 | `air_tat` | Number | 2 | Static — set once |
| 5 | `pipeline_days` | Number | 6.8 | `COUNT(in_transit_orders) / AVG(daily_outflow_last7d)` |
| 6 | `long_tail_pct` | Number | 9.2 | `COUNT(delivered WHERE actual_date > promised_date + 1) / COUNT(delivered_today) * 100` |
| 7 | `sla_breach_pct` | Number | 13.1 | `COUNT(delivered WHERE actual_date > promised_date) / COUNT(delivered_today) * 100` |
| 8 | `inflow` | Number | 420 | Orders dispatched from hub today |
| 9 | `outflow` | Number | 310 | Orders delivered today |
| 10 | `aging_1d` | Number | 180 | In-transit orders that are 1 day past promised ETA |
| 11 | `aging_2d` | Number | 90 | In-transit orders that are 2 days past promised ETA |
| 12 | `aging_3d` | Number | 55 | In-transit orders that are 3 days past promised ETA |
| 13 | `aging_gt3d` | Number | 35 | In-transit orders that are >3 days past promised ETA |

---

### Reference SQL Query (adapt to your schema)

```sql
-- Run hourly. Writes one row per city into Google Sheet via your pipeline.
-- Columns bau_pipeline, surface_tat, air_tat are static config —
-- join them from a config table or hardcode per city.

SELECT
  o.delivery_city                                           AS city,
  cfg.bau_pipeline,
  cfg.surface_tat,
  cfg.air_tat,

  -- Pipeline Days: in-transit orders / 7-day avg daily outflow
  ROUND(
    COUNT(CASE WHEN o.status = 'in_transit' THEN 1 END)
    / NULLIF(AVG(daily_out.daily_count), 0),
  2)                                                        AS pipeline_days,

  -- Long Tail %: delivered today where actual > promised + 1 day
  ROUND(
    COUNT(CASE WHEN o.status = 'delivered'
               AND DATE(o.delivered_at) = CURRENT_DATE
               AND DATE(o.delivered_at) > DATE(o.promised_eta) + INTERVAL '1 day'
          THEN 1 END)
    / NULLIF(COUNT(CASE WHEN o.status = 'delivered'
                        AND DATE(o.delivered_at) = CURRENT_DATE
                   THEN 1 END), 0) * 100,
  2)                                                        AS long_tail_pct,

  -- SLA Breach %: delivered today where actual > promised
  ROUND(
    COUNT(CASE WHEN o.status = 'delivered'
               AND DATE(o.delivered_at) = CURRENT_DATE
               AND DATE(o.delivered_at) > DATE(o.promised_eta)
          THEN 1 END)
    / NULLIF(COUNT(CASE WHEN o.status = 'delivered'
                        AND DATE(o.delivered_at) = CURRENT_DATE
                   THEN 1 END), 0) * 100,
  2)                                                        AS sla_breach_pct,

  -- Inflow: dispatched today
  COUNT(CASE WHEN DATE(o.dispatched_at) = CURRENT_DATE THEN 1 END)
                                                            AS inflow,

  -- Outflow: delivered today
  COUNT(CASE WHEN o.status = 'delivered'
              AND DATE(o.delivered_at) = CURRENT_DATE
         THEN 1 END)                                        AS outflow,

  -- Aging buckets: in-transit orders N days past promised ETA
  COUNT(CASE WHEN o.status = 'in_transit'
              AND CURRENT_DATE - DATE(o.promised_eta) = 1
         THEN 1 END)                                        AS aging_1d,

  COUNT(CASE WHEN o.status = 'in_transit'
              AND CURRENT_DATE - DATE(o.promised_eta) = 2
         THEN 1 END)                                        AS aging_2d,

  COUNT(CASE WHEN o.status = 'in_transit'
              AND CURRENT_DATE - DATE(o.promised_eta) = 3
         THEN 1 END)                                        AS aging_3d,

  COUNT(CASE WHEN o.status = 'in_transit'
              AND CURRENT_DATE - DATE(o.promised_eta) > 3
         THEN 1 END)                                        AS aging_gt3d

FROM orders o
JOIN city_config cfg ON cfg.city = o.delivery_city

-- 7-day rolling outflow subquery
LEFT JOIN (
  SELECT delivery_city,
         AVG(cnt) AS daily_count
  FROM (
    SELECT delivery_city,
           DATE(delivered_at) AS d,
           COUNT(*) AS cnt
    FROM orders
    WHERE status = 'delivered'
      AND delivered_at >= CURRENT_DATE - INTERVAL '7 days'
    GROUP BY 1, 2
  ) x
  GROUP BY delivery_city
) daily_out ON daily_out.delivery_city = o.delivery_city

GROUP BY o.delivery_city, cfg.bau_pipeline, cfg.surface_tat, cfg.air_tat
ORDER BY o.delivery_city;
```

---

### Static Config Table (city_config)

Create this in your DB and join it into the query above:

```sql
CREATE TABLE city_config (
  city          VARCHAR(50) PRIMARY KEY,
  bau_pipeline  DECIMAL(4,1),  -- pre-monsoon avg pipeline days
  surface_tat   INT,            -- surface courier SLA in days
  air_tat       INT             -- BD Air TAT in days
);

INSERT INTO city_config VALUES
  ('Mumbai',    3.5, 5, 2),
  ('Delhi',     3.0, 4, 2),
  ('Bengaluru', 3.2, 5, 2),
  ('Chennai',   3.8, 5, 3),
  ('Hyderabad', 3.3, 5, 2),
  ('Kolkata',   4.0, 6, 3),
  ('Pune',      3.0, 4, 2),
  ('Ahmedabad', 3.5, 5, 2);
-- Add all your cities. BAU = compute from last 60 days pre-monsoon data.
```

---

## PART 2: Google Sheet Setup

### Step 1 — Create the Sheet

1. Go to [sheets.google.com](https://sheets.google.com) → New Spreadsheet
2. Rename it: **1mg Rain Control Tower**
3. Rename Sheet1 to: **CITY_METRICS**

### Step 2 — Add Headers (Row 1)

In row 1, add these exact headers:
```
city | bau_pipeline | surface_tat | air_tat | pipeline_days | long_tail_pct | sla_breach_pct | inflow | outflow | aging_1d | aging_2d | aging_3d | aging_gt3d
```

### Step 3 — Paste Sample Data to Test

Paste the 8 sample cities (from the dashboard's sample data) in rows 2–9 to verify the connection works before your SQL pipeline is live.

### Step 4 — Publish the Sheet as CSV

This is how the dashboard reads it — no API keys needed.

1. **File → Share → Publish to web**
2. In the first dropdown: select **CITY_METRICS** (the tab name)
3. In the second dropdown: select **Comma-separated values (.csv)**
4. Click **Publish** → Confirm
5. Copy the URL it gives you. It will look like:
   ```
   https://docs.google.com/spreadsheets/d/SHEET_ID/pub?gid=0&single=true&output=csv
   ```
6. **This is your Sheet CSV URL.** Paste it into the dashboard's URL input field.

> ⚠️ Important: The Sheet must remain published to web for the dashboard to read it.
> The data is not sensitive (aggregated city-level metrics only, no order IDs).
> If you need to restrict access, use the GitHub Pages URL restriction below instead.

### Step 5 — Connect Your SQL Pipeline

Configure your hourly SQL job to overwrite rows 2 onwards in the CITY_METRICS tab.
Most SQL-to-Sheets pipelines (dbt + Sheets connector, Airbyte, custom script) support this.
Keep row 1 (headers) intact — only overwrite data rows.

---

## PART 3: GitHub Pages Deployment (Free, Permanent URL)

### Step 1 — Create GitHub Account

1. Go to [github.com](https://github.com) → Sign Up
2. Choose the free plan. No credit card needed.
3. Verify your email.

### Step 2 — Create a New Repository

1. Click the **+** icon (top right) → **New repository**
2. Repository name: `rain-control-tower`
3. Set to **Public** (required for free GitHub Pages)
4. Click **Create repository**

### Step 3 — Upload the Dashboard File

1. On your new repo page, click **Add file → Upload files**
2. Upload the `index.html` file (the dashboard file provided to you)
3. In the commit message box type: `Initial dashboard`
4. Click **Commit changes**

### Step 4 — Enable GitHub Pages

1. In your repo, go to **Settings** (top tab)
2. In the left sidebar: **Pages**
3. Under "Source": select **Deploy from a branch**
4. Branch: **main** | Folder: **/ (root)**
5. Click **Save**
6. Wait 2–3 minutes. Refresh the page.
7. GitHub will show: **Your site is live at `https://YOUR-USERNAME.github.io/rain-control-tower/`**

### Step 5 — Connect Your Sheet to the Dashboard

1. Open your GitHub Pages URL in a browser
2. The dashboard loads with sample data by default
3. Paste your Sheet CSV URL (from Part 2, Step 4) into the input field at the top
4. Click **Load Sheet**
5. Your live city data appears immediately
6. The URL + Sheet URL is now saved in your browser's local storage — it remembers on next visit

### Step 6 — Share with Your Team

Send your team this URL:
```
https://YOUR-USERNAME.github.io/rain-control-tower/
```

Each team member:
1. Opens the URL in their browser (Chrome/Firefox/Edge — any modern browser)
2. Pastes the Sheet CSV URL once
3. Clicks Load Sheet
4. Bookmarks it — done

The dashboard auto-refreshes when they click "↻ Refresh Now", or they can set a browser tab auto-refresh extension.

---

## PART 4: Updating the Dashboard Later

If you need to change anything in the dashboard (new metrics, visual tweaks):

1. Edit the `index.html` file locally
2. Go to your GitHub repo → click `index.html` → click the pencil (edit) icon
3. Paste the updated code → **Commit changes**
4. GitHub Pages auto-deploys in ~2 minutes

Or drag and drop the updated file via **Add file → Upload files** and commit.

---

## Summary: What Your Team's Workflow Looks Like

```
Every hour:
  SQL query runs → writes 8-13 rows to Google Sheet CITY_METRICS tab

Ops team (any time):
  Opens https://YOUR-USERNAME.github.io/rain-control-tower/
  Clicks "Refresh Now" to pull latest Sheet data
  Checks City Overview for RED cities
  If city is RED → go to Clickpost → activate BD Air rule for that city
  Log action in Decision Log sheet

Every 4 hours:
  Re-check dashboard
  If RED city's pipeline has been below trigger for 2 checks → deactivate Air rule
  Log deactivation in Decision Log sheet
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Dashboard shows "Could not load Sheet" | Make sure Sheet is published (Part 2 Step 4). Re-publish if needed. |
| Data looks wrong / columns mismatched | Check column order in Sheet matches exactly the 13 columns listed in Part 1 |
| GitHub Pages URL not working | Wait 5 minutes after setup. Check Settings → Pages for the correct URL. |
| City names don't match | SQL must output city names exactly as they appear in city_config table |
| Sheet URL not saved after browser restart | Re-paste the URL and click Load Sheet. It saves to browser local storage. |
