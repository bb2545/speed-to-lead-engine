# Speed-to-Lead Engine

An automated workflow that closes the gap between "a prospect requests a
demo" and "a human actually responds" — built on Make.com, HubSpot, Slack,
Google Sheets, and Gmail.

---

## 🔥 The Problem

In B2B SaaS, speed-to-lead is a competitive weapon, not a nice-to-have.
Research from MIT (published via Harvard Business Review) analyzed over 2
million sales leads and found that contacting a lead within 5 minutes makes
a company 100x more likely to make contact and 21x more likely to qualify
that lead, compared to waiting just 30 minutes. Yet the average business
takes roughly two full days to respond.

The gap isn't a lack of interest from sales reps — it's that new deals sit
silently in a CRM until someone happens to check it. No alert, no
acknowledgment to the prospect, no visibility into how many leads are
slipping through that window. By the time a rep follows up, the prospect
has often already booked a demo with a competitor.

## ✅ The Solution

This system watches HubSpot for new deals and, the moment one appears,
fans out three things in parallel — no manual triggering required:

1. **Alerts the sales team in Slack** with the deal name, value, and stage,
   so a rep can act immediately.
2. **Logs the deal to Google Sheets** for a running pipeline record and
   response-time tracking.
3. **Sends an automatic acknowledgment email to the prospect** — pulling
   their contact record via HubSpot's association API — so they know
   someone has received their request within minutes, not days, even
   before a rep has picked up the phone.

**Architecture:**

```
HubSpot (Watch Deals)
        │
        ▼
List Pipelines  →  List Associations (Deal → Contact)  →  Get a Contact
        │
        ▼
     Router
   ╱    │    ╲
Slack  Sheets  Gmail
 │       │       │
Skip   Retry   Retry
(on    (3x /   (3x /
fail)   2min)   2min)
```

Every branch has its own error handler tuned to the cost of failure for
that channel: Slack failures are skipped (low stakes — the deal is still
visible in HubSpot), while Sheets and Gmail retry three times at 2-minute
intervals, since a missed row or a missed customer acknowledgment carries
real business cost.

## 📊 Business Impact

| Metric | Result |
|---|---|
| Automatic response window (demo environment) | **Within 15 minutes**, with zero manual triggering — verified via live test with no manual "Run" |
| Automatic response window (production-ready) | Near-instant with a webhook trigger or paid HubSpot/Make tier — the 15-minute figure above reflects a free-tier polling limit, not a design constraint |
| Delivery reliability | 100% — Gmail delivery failures automatically retried 3x at 2-minute intervals; verified end-to-end in testing |
| Fault isolation | Confirmed — a malformed record (e.g. a deal with no linked contact) queues in isolation and does not block or crash processing of other leads |
| Industry benchmark this system targets | 5-minute response = 21x higher lead qualification rate (MIT / Harvard Business Review, Lead Response Management Study) vs. the ~47-hour average response time most businesses operate at today |

*All figures above marked "verified" were measured directly against this
build, not estimated. The system was tested by creating live HubSpot deals
and observing automatic end-to-end processing — including both a
successful path and two deliberate failure scenarios — without any manual
intervention in Make.com.*

## 🎬 Demo

*Video walkthrough — coming soon (recorded alongside Portfolio 1).*

---

## Tech Stack

- **Automation platform:** Make.com (Webhook/polling trigger, Router,
  multi-branch error handling)
- **CRM:** HubSpot (Deals, Contacts, Associations API)
- **Notification & logging:** Slack, Google Sheets
- **Customer-facing communication:** Gmail

## Known Limitations (v1 → v2 Roadmap)

- Polling interval is fixed at 15 minutes on the current free-tier plan;
  a webhook-based trigger or paid tier would close this to near-instant.
- Deals with more than one associated contact currently route to the
  first contact found; multi-contact handling is planned for v2.
- Deals with zero associated contacts are queued as an isolated incomplete
  execution rather than routed with a graceful fallback — a filtered
  fallback branch (e.g. notify the team without attempting a customer
  email) is planned for v2.
