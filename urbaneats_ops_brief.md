# UrbanEats · 11 AM Ops Review Brief
**FDE Assignment · Delivered to VP of Operations & Regional Managers**

---

## Verbal Brief — 5 Sentences

**Sentence 1 — Data Readiness:**
The 150-order export is clean and ready for analysis, with 1 field requiring attention: `delivery_time_mins` is absent for 6 orders and has been filled using zone-level medians before the risk model ran, ensuring nothing in the summary is skewed by missing values.

**Sentence 2 — Priority Hotspot:**
The single combination that operations must address this week is **Wrap & Roll in the Central zone**, where 75% of its orders are flagged at high cancellation risk — the highest rate across all 25 restaurant-zone pairs in the dataset.

**Sentence 3 — Automated Morning Briefing:**
Starting tomorrow, every regional manager will receive a structured ops briefing at **07:30 each morning** via Slack and email, covering total order volume, cancellation rate, average delivery time, and the zone with the highest complaint count — with a red-alert escalation triggered automatically whenever the cancellation rate crosses the **20%** threshold.

**Sentence 4 — Known Gap:**
The system scores cancellation risk based on order characteristics available at placement time, but it cannot account for external factors such as weather events, local festivals, or road closures in the **North** and **Central** zones that may temporarily inflate delays and complaints beyond what the data alone would predict.

**Sentence 5 — Success Metric:**
The intervention is working if, 4 weeks from today, the high-cancellation-risk order rate for the **Wrap & Roll Central** cluster falls below **40%** — a 35-percentage-point reduction from its current 75% — measured on a rolling 7-day window using the same risk scoring pipeline.

---
