# UrbanEats — Delivery Operations Intelligence
### FDE Masterclass · Assignment 2
**Domain:** Food Delivery & Operations Analytics

---

## Project Overview

UrbanEats is a food delivery platform operating across 5 city zones. Cancellation rates and delivery complaints rose 22% over the last quarter. This project identifies exactly where the breakdowns are happening — by restaurant, zone, and rider — and automates a daily morning briefing so ops managers no longer pull reports manually.

**Tools used:** Google Colab · Python · Great Expectations · LangChain · Groq (free LLM) · n8n Cloud · Slack · Gmail

---

## Repository Structure

```
urbaneats_assignment2/
│
├── UrbanEats_profiling_gx.ipynb              ← Part A + B
├── UrbanEats2_classifier_langchain.ipynb     ← Part C1 + C2
├── UrbanEatsn8n_workflow.json                ← Part D
├── urbaneats_ops_brief.md                    ← Part E
│
├── urbaneats_delivery_orders.csv             ← Original dataset (150 rows)
├── urbaneats_delivery_orders_scored.csv      ← Scored dataset with cancel columns
│
├── urbaneats_operations_audit.html           ← Generated: data profile report
├── urbaneats_gx_data_docs.html               ← Generated: GX quality gate results
├── urbaneats_alerts.json                     ← Generated: LangChain ops alerts
│
└── README.md                                 ← This file
```

---

## Dataset

**File:** `urbaneats_delivery_orders.csv`
**Size:** 150 rows × 11 columns

| Column | Description |
|---|---|
| `order_id` | Unique order identifier |
| `order_date` | Date of order placement |
| `restaurant_name` | One of 5 restaurants |
| `delivery_zone` | One of 5 zones: North, South, East, West, Central |
| `order_value` | Order value in INR |
| `delivery_time_mins` | Minutes from order to delivery (6 missing) |
| `rider_rating` | Customer rating for rider, 1.0–5.0 (6 missing) |
| `order_status` | Delivered / Cancelled / Delayed / Refunded |
| `payment_method` | Cash / Card / UPI / Wallet |
| `discount_applied` | INR discount applied to order |
| `customer_complaints` | Number of support tickets for this order |

**Order status distribution:**

| Status | Count | % |
|---|---|---|
| Delivered | 44 | 29.3% |
| Refunded | 37 | 24.7% |
| Cancelled | 36 | 24.0% |
| Delayed | 33 | 22.0% |

> Cancelled + Refunded = **73 orders (48.7%)** — nearly half of all orders are non-successful deliveries.

---

## Part A — Data Profiling & Operational Audit

**Notebook:** `UrbanEats_profiling_gx.ipynb` (Cells 1–8)

**How to run:**
1. Open in Google Colab
2. Upload `urbaneats_delivery_orders.csv` to the Colab file panel
3. Add and run this install cell first:
   ```
   !pip install fg-data-profiling -q
   ```
4. Run all cells top to bottom

**Key findings:**

- `delivery_time_mins` — **6 values missing**, spread across all 4 order statuses. Imputed using zone-level medians before modelling.
- `rider_rating` — **6 values missing**, partially concentrated in Cancelled orders (MNAR — structurally expected when no rider was assigned).
- `order_value` — all values positive, minimum ₹123, no zero or negative values.
- Cancelled + Refunded orders represent **48.7%** of all orders.

**Outputs generated:**
- `urbaneats_operations_audit.html` — full interactive data profile (download from Colab file panel)

---

## Part B — Great Expectations Quality Gate

**Notebook:** `UrbanEats_profiling_gx.ipynb` (Cells 9–14)

**6 expectations enforced:**

| # | Column | Rule | Operational Reason |
|---|---|---|---|
| 1 | `order_id` | Not null | Null ID makes the record untraceable |
| 2 | `order_id` | Unique | Duplicates inflate all volume metrics |
| 3 | `delivery_time_mins` | Between 10–120 mins (mostly 95%) | Outside range = data entry error or test order |
| 4 | `rider_rating` | Between 1.0–5.0 (mostly 95%) | Out-of-range values corrupt rider performance tables |
| 5 | `order_status` | One of: Delivered, Cancelled, Delayed, Refunded | Unknown values break status-based KPI dashboards |
| 6 | `order_value` | Between ₹50–5,000 | Below ₹50 = test transaction; above ₹5,000 = B2B catering |

**Result:** All 6 expectations **PASSED**. The `mostly=0.95` tolerance on expectations 3 and 4 accommodates the known 4% structural nulls from Cancelled/Refunded orders.

**Outputs generated:**
- `urbaneats_gx_data_docs.html` — styled GX results report (download from Colab file panel)

---

## Part C1 — Cancellation Risk Classifier

**Notebook:** `UrbanEats2_classifier_langchain.ipynb` (Cells 1–8)

**How to run:**
1. Open in Google Colab (new tab)
2. Upload `urbaneats_delivery_orders.csv` to the file panel
3. Add and run this install cell first:
   ```
   !pip install langchain langchain-groq scikit-learn -q
   ```
4. Run all cells top to bottom

**Model:** Random Forest (`n_estimators=100`, `random_state=42`)
**Training data:** 80 rows (Delivered + Cancelled only, per assignment spec)
**Split:** 80/20 train/test
**Missing values:** `delivery_time_mins` filled with zone-level median before split

**Feature importance:**

| Feature | Importance |
|---|---|
| `delivery_time_mins` | 33.4% |
| `discount_applied` | 20.9% |
| `order_value` | 19.5% |
| `delivery_zone` | 15.4% |
| `restaurant_name` | 10.8% |

**Evaluation metrics (test set):**

| Metric | Score |
|---|---|
| Precision | 0.600 |
| Recall | 0.375 |
| F1-Score | 0.462 |

**Scoring:** All 150 orders scored. A `cancel_probability ≥ 0.55` is classified as `high` risk.
- High risk: **49 orders**
- Low risk: **101 orders**

**Top hotspot:** `Wrap & Roll | Central` — 75% high cancellation risk rate (highest across all 25 restaurant-zone pairs)

---

## Part C2 — Ops Alert Messages with LangChain + Groq

**Notebook:** `UrbanEats2_classifier_langchain.ipynb` (Cells 9–14)

**LLM:** `llama-3.1-8b-instant` via Groq (free tier — no credit card required)
**Framework:** LangChain LCEL chain (`PromptTemplate | ChatGroq | StrOutputParser`)

**Before running Cell 10:**
1. Click the 🔑 Secrets icon in the Colab left sidebar
2. Add a secret named `GROQ_API_KEY`
3. Paste your key from [console.groq.com](https://console.groq.com)
4. Toggle "Notebook access" to ON

**13 restaurant-zone hotspots identified (>30% high risk rate):**

| Restaurant | Zone | Risk Rate |
|---|---|---|
| Wrap & Roll | Central | 75.0% |
| Burger Hub | Central | 70.0% |
| Wrap & Roll | West | 60.0% |
| Spice Garden | West | 57.1% |
| Pizza Palace | Central | 50.0% |
| Spice Garden | Central | 50.0% |
| Wrap & Roll | East | 50.0% |
| Sushi Bay | Central | 44.4% |
| Wrap & Roll | North | 44.4% |
| Sushi Bay | East | 42.9% |
| Spice Garden | South | 37.5% |
| Spice Garden | North | 33.3% |
| Sushi Bay | North | 33.3% |

**Alert quality evaluation (3 criteria):**
- ✅ Specificity — names the exact restaurant-zone combination
- ✅ Actionability — contains a concrete operational action
- ✅ No hedging — no "it seems", "perhaps", "might" language

**Outputs generated:**
- `urbaneats_delivery_orders_scored.csv` — 150 rows, 13 columns including `cancel_probability` and `cancel_risk`
- `urbaneats_alerts.json` — all 13 hotspot alerts in structured JSON

---

## Part D — n8n Morning Ops Briefing Automation

**File:** `UrbanEatsn8n_workflow.json`
**Platform:** n8n Cloud (free trial at [app.n8n.cloud](https://app.n8n.cloud))

### Workflow — 8 nodes

```
Schedule Trigger (07:30 daily)
        ↓
HTTP Request — Fetch Scored CSV
        ↓
Code — Parse CSV & Compute KPIs
        ↓
If — Cancellation Rate > 20%?
   ↓ TRUE                    ↓ FALSE
Slack — RED ALERT       Slack — GREEN Summary
        ↓                         ↓
Gmail — RED ALERT Email   Gmail — GREEN Summary Email
```

### Node details

| Node | What it does |
|---|---|
| Schedule Trigger | Fires at 07:30 every day (cron: `30 7 * * *`) |
| HTTP Request | Fetches `urbaneats_delivery_orders_scored.csv` from Google Drive public URL |
| Code (JavaScript) | Parses CSV, computes: total orders, cancellation rate, avg delivery time, worst zone by complaints, worst restaurant by cancellations, top cancel-risk pair |
| If node | Branches on `cancellationRate > 0.20` |
| Slack — RED ALERT | Posts red alert to `#ops-alerts` channel with all KPIs + AI alert text |
| Slack — GREEN Summary | Posts green all-clear to `#ops-alerts` channel |
| Gmail — RED ALERT Email | Sends same content as Slack RED to ops manager email |
| Gmail — GREEN Summary Email | Sends same content as Slack GREEN to ops manager email |

### How to import and configure

1. Log in to [app.n8n.cloud](https://app.n8n.cloud)
2. Click **Workflows → + → Import from file** → select `UrbanEatsn8n_workflow.json`
3. **HTTP Request node:** update URL to your Google Drive direct download link:
   `https://drive.google.com/uc?export=download&id=YOUR_FILE_ID`
4. **Both Slack nodes:** connect via OAuth → select your workspace → set channel to `#ops-alerts`
5. **Both Gmail nodes:** connect via Google OAuth → set "Send To" to your email address
6. Click **Test workflow** to run immediately without waiting for 07:30
7. Toggle **Active** switch ON to enable daily automation

---

## Part E — 11 AM Ops Review Brief

**File:** `urbaneats_ops_brief.md`

A 5-sentence verbal brief delivered to the VP of Operations and regional managers covering:
1. Data readiness and the specific missing field
2. The single highest-risk restaurant-zone hotspot with exact risk rate
3. What the automated morning briefing delivers and when
4. One known gap in the system's scope
5. The single metric that confirms the intervention is working after 4 weeks

---

## How to Reproduce Everything from Scratch

### Step 1 — Get your free Groq API key
Go to [console.groq.com](https://console.groq.com) → sign up → API Keys → Create Key. Save the key starting with `gsk_...`

### Step 2 — Run Notebook 1 in Colab
```
1. Open UrbanEats_profiling_gx.ipynb in Google Colab
2. Upload urbaneats_delivery_orders.csv to the file panel
3. Add first cell: !pip install fg-data-profiling great-expectations -q
4. Runtime → Run all
5. Download: urbaneats_operations_audit.html + urbaneats_gx_data_docs.html
```

### Step 3 — Run Notebook 2 in Colab
```
1. Open UrbanEats2_classifier_langchain.ipynb in Google Colab (new tab)
2. Upload urbaneats_delivery_orders.csv to the file panel
3. Add first cell: !pip install langchain langchain-groq scikit-learn -q
4. Add GROQ_API_KEY to Colab Secrets (🔑 icon) with notebook access ON
5. Runtime → Run all
6. Download: urbaneats_delivery_orders_scored.csv + urbaneats_alerts.json
```

### Step 4 — Set up n8n Cloud workflow
```
1. Upload urbaneats_delivery_orders_scored.csv to Google Drive (set to public)
2. Import UrbanEatsn8n_workflow.json into n8n Cloud
3. Update HTTP Request node URL with your Google Drive file ID
4. Connect Slack OAuth on both Slack nodes → channel: #ops-alerts
5. Connect Gmail OAuth on both Gmail nodes → update Send To email
6. Click Test workflow → verify Slack message and email arrive
7. Toggle Active → ON
```

---

## Key Findings Summary

| Metric | Value |
|---|---|
| Total orders analysed | 150 |
| Cancellation rate | 24.0% |
| Cancelled + Refunded combined | 48.7% |
| Orders with high cancel risk | 49 / 150 |
| Hotspot pairs identified (>30% risk) | 13 |
| Worst hotspot | Wrap & Roll — Central (75%) |
| Most complained zone | North (48 complaints) |
| Top cancellation predictor | `delivery_time_mins` (33.4% importance) |
| Intervention success threshold | Wrap & Roll Central risk rate < 40% after 4 weeks |

---

*FDE Masterclass · Assignment 2 | Dataset: synthetic, modelled on real food delivery OMS exports*
