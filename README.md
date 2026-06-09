# B2C Pipeline Analytics Dashboard

A zero-dependency, single-file analytics dashboard for the B2C HubSpot pipeline, joined with Google Ads spend.

Three tabs:
- **Pipeline** — the HubSpot funnel (leads → Erstgespräch → Abschlussgespräch → won/lost).
- **Ads** — Google Ads spend, impressions, clicks, CPC, CTR.
- **Performance** — the two joined per campaign: **CPL, CAC, and a cost-per-funnel-stage ladder**.

The **Ads** and **Performance** tabs unlock once Google Ads data is loaded.

---

## Quick start

```bash
cd ~/b2c-pipeline-dashboard
python3 -m http.server 8000
# open http://localhost:8000
```

On startup the dashboard loads `data/deals.csv` (pipeline) and `data/ads.csv` (Google Ads) if present. Sample files are included so all three tabs work out of the box.

---

## Weekly update routine

You have two ways to load a fresh export — pick whichever is easier.

### Option A — Load in the UI (no file juggling)

1. Export your **HubSpot deals** report and/or your **Google Ads** report as **CSV or Excel (`.xlsx`)**.
2. In the dashboard, click **＋ Load file** in the top-right (or just **drag the file anywhere onto the page**).
3. The dashboard **auto-detects** whether it's a deals or an ads file, validates it, and updates instantly.

Excel files are parsed in-browser (first sheet, date cells normalised to `YYYY-MM-DD`) and run through the same validation as CSV.

Each file is remembered separately in your browser, so they stay loaded across refreshes. The header shows `deals: … · ads: …`. Click **Reset to files** to discard uploads and return to the bundled samples.

### Option B — Replace the files on disk

1. Export from HubSpot / Google Ads.
2. Save / overwrite `data/deals.csv` and/or `data/ads.csv`.
3. Refresh `http://localhost:8000`.

No build step, no install — just save and refresh.

### Validation checks

Whenever you load a CSV, the dashboard runs these checks before applying it:

- **Format detection** — simplified vs. raw HubSpot export.
- **Create-date column present** — *fatal* if missing; the file is not loaded.
- **Valid rows** — counts rows with a parseable create date, flags any skipped.
- **Date range** of the data.
- **Campaigns found**, and any **unmapped UTM values** that fell back to "Other".
- **Status split** (won / absagen / in process).
- **Integrity warnings** — rows with both a won and an absage date, and absagen with no reason.

A green panel means all good; yellow means loaded with warnings worth a look; red means the file was rejected and your previous data is untouched.

---

## Folder structure

```
b2c-pipeline-dashboard/
├── index.html        ← entire app (HTML + CSS + JS, no dependencies)
├── data/
│   ├── deals.csv     ← bundled sample HubSpot export
│   └── ads.csv       ← bundled sample Google Ads export
└── README.md
```

---

## Column reference

The dashboard accepts both the **simplified** and **raw HubSpot export** formats.

| Field | Simplified column | Raw HubSpot column |
|---|---|---|
| Create date | `CreateDate` | `Date entered "Neuer Lead (B2C)"…` |
| Erstgespräch date | `Erstgespräch` | `Date entered "Erstgespräch durchgeführt (B2C)"…` |
| Abschlussgespräch date | `Abschlussgespräch` | `Date entered "Abschlussgespräch durchgeführt (B2C)"…` |
| Signed date | `Angebot unterschrieben` | `Date entered "Angebot unterschrieben (B2C)"…` |
| Handover date | `Handover` | `Date entered "Handover Installation (B2C)"…` |
| Absage date | `Absage` | `Date entered "Absage (B2C)"…` |
| UTM campaign | `utm_campaign` | `utm_campaign` |
| Absage reason | `Absage - Grund` | `Absage - Grund` |

The raw export column names contain a date suffix (e.g. `– täglich`). The app matches on the prefix so any suffix is fine.

---

## Google Ads data

### How to export
In Google Ads: **Campaigns** → set the date range → **Segment → Time → Week** → add the columns below → **Download** as CSV or Excel.

Required columns (EN or DE both work): **Campaign / Kampagne**, **Week / Woche**, **Impressions / Impressionen**, **Cost / Kosten**, **Clicks / Klicks**. **Conversions** is read but shown only for reference (see below). The app automatically strips the report's title/date preamble and the `Total / Gesamt` rows, and drops paused (all-zero) campaigns.

### How the join works
Campaigns are matched by **K-code** extracted from the campaign name (`K1a`, `K2`, `K8`, …) on both the HubSpot `utm_campaign` and the Google Ads campaign name. New campaigns map automatically; anything without a K-code falls to **Other**.

| Metric | Source |
|---|---|
| Spend, Impressions, Clicks, CPC, CTR | Google Ads (billing/serving truth) |
| Leads, Erstgespräch, Closed Won, attribution | HubSpot (CRM truth) |
| **CPL** = Spend ÷ Leads, **CAC** = Spend ÷ Closed Won | the join |

> Google Ads' own **Conversions** column is **not used** for CPL/CAC — its tracking is unreliable. It's shown on the Ads tab for reference only. All cost-per metrics are driven off HubSpot counts.

Spend and outcomes are both attributed to the **lead's creation cohort** (the period the lead came in), so CPL/CAC reflect true unit economics. Recent cohorts look incomplete until deals mature.

---

## Dashboard panels

### Global controls (sticky bar)

| Control | Effect |
|---|---|
| **Pipeline / Ads / Performance** | Switches tab (Ads & Performance need Google Ads data loaded) |
| **Monthly / Weekly** | Switches the cohort granularity for all panels |
| **Cohort selector** | Filters all panels to a single month or week |
| **All / Paid / Organic** | Filters by channel (Paid = a Search campaign with a K-code; Organic = everything else) |
| **Campaign multiselect** | Show/hide individual campaigns across all tabs |
| **Total / By campaign** | Switches the funnel chart between single-color and stacked view (Pipeline tab) |

### Section 1 — Metrics strip
Six summary cards for the selected cohort: Leads, Erstgespräch, Abschlussgespräch, Closed Won, Absagen, In Process. Each shows absolute count and its rate relative to leads.

### Section 2 — Pipeline funnel
Bar chart with the five funnel steps: Neuer Lead → Erstgespräch → Abschlussgespräch → Closed Won → Closed Lost. In **By campaign** mode each bar is stacked by campaign with a tooltip showing the campaign name, count, and % of that step's total.

### Section 3 — Pending & Absagen breakdown
Two side-by-side cards. Each card breaks the cohort into three pipeline stages (*Vor Erstgespräch*, *Nach Erstgespräch*, *Nach Abschlussgespräch*). The Absagen card additionally shows the top 3 rejection reasons per stage as inline mini-bars.

### Section 4 — Week-over-week (weekly mode only)
Two-column layout comparing the previous week to the selected week. Metrics: Leads, Erstgespräch, Erst rate, Closed Won, Close rate, Absagen, Absage rate, Falsche Kontaktinfos. Delta badges are direction-aware (green = good, red = bad).

### Section 5 — Leads over time
Line chart showing weekly lead volume for every campaign in the dataset. Uses all weeks in the CSV regardless of the cohort selector. Series toggle buttons above the chart are **independent** from the global campaign filter — hide/show any line without changing the rest of the dashboard.

### Ads tab
Cards (Spend, Impressions, Clicks, CPC, CTR, Conversions*), a **weekly spend-over-time** chart (Total / By-campaign), and a sortable **by-campaign table**. *Conversions is Google-tracked and shown for reference only.*

### Performance tab
The join. Cards: Spend, Leads, **CPL**, Closed Won, **CAC**, Lead→Won %. A **cost-per-stage ladder** (Cost / Lead → Cost / Erstgespräch → Cost / Abschlussgespräch → Cost / Acquisition) showing what each surviving lead costs deeper in the funnel. A **by-campaign table** with spend, leads, CPL, won, CAC — campaigns with spend but no leads show "—" (a tracking gap worth investigating); free channels (Organic/Referral) show €0 CPL.

---

## Campaign colour reference

| Label | UTM segment | Colour |
|---|---|---|
| K1a | K1a Dynamischer Stromtarif (Long-Tail) | `#e8f547` |
| K1b | K1b Dynamischer Stromtarif (Head-Term) | `#60a5fa` |
| K1c | K1c Dynamischer Stromtarif (Research)  | `#a78bfa` |
| K2  | K2 HEMS                                | `#2dd4bf` |
| K3  | K3 Smart Meter Tarif                   | `#fb923c` |
| K5  | K5 PV-Eigenverbrauch                   | `#f472b6` |
| K6  | K6 Competitor: Heartbeat               | `#34d399` |
| K7  | K7 Competitor: Tibber                  | `#fbbf24` |
| Organic  | No UTM                            | `#94a3b8` |
| Referral | UTM contains "Freunde" or "bullfinch" | `#6b7280` |
| Other    | Any other UTM                     | `#475569` |
