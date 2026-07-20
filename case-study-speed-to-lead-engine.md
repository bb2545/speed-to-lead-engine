# Case Study: Speed-to-Lead Engine

**A Make.com automation that turns "we'll get back to you" into "we
already did" — built for B2B SaaS teams who can't afford to lose a lead
to slow follow-up.**

## The Real Problem

Most CRM implementations treat a new deal as something a sales rep will
"see eventually." But in B2B SaaS, eventually is a competitive risk.
Research from MIT — analyzing over 2 million sales leads — found that
contacting a lead within 5 minutes makes a company 21x more likely to
qualify it, compared to waiting just 30 minutes. Yet the average business
takes close to two days to respond. The cost isn't visible on a dashboard;
it shows up as a demo booked with a competitor before your own rep ever
picks up the phone.

## What Was Built

A Make.com scenario that watches HubSpot for new deals and, without any
manual step, fans out three things in parallel: a Slack alert to the sales
team, a logged row in Google Sheets for pipeline tracking, and — critically
— an automatic acknowledgment email to the prospect themselves, pulled
directly from their HubSpot contact record via the Deals-to-Contacts
association API. Each of the three branches carries its own error handling,
scaled to what that channel's failure actually costs the business: a missed
Slack ping is skipped, while a missed customer email or pipeline log
retries three times before surfacing as an issue that needs a human.

## The Debugging Story Worth Telling

During testing, the customer-facing email began silently failing delivery
with no obvious cause — even after ruling out the email address itself and
standard whitespace trimming. Root-cause investigation eventually traced
it to an invisible Thai combining character contaminating the contact
record, most likely from an input-language mismatch during manual data
entry — a realistic risk for any CRM populated by multilingual teams.
Rather than patching the symptom with a downstream filter, the fix was
applied at the data source, and the diagnosis process itself (isolating
the failure to data vs. code vs. formula syntax) is documented step by
step for future reference.

## Verified, Not Assumed

Before calling this done, the system was tested against three real
scenarios — not just a single happy-path run:

- **Live auto-trigger test:** a real deal was created in HubSpot with no
  manual "Run" in Make. All three channels fired correctly within the
  scenario's polling window.
- **Forced failure test:** the email delivery step was deliberately broken
  to confirm the retry logic actually waits and retries on schedule (3
  attempts, 2 minutes apart) rather than just being configured and never
  exercised.
- **Edge-case test:** a deal was created with no linked contact on purpose,
  to confirm that one broken record queues in isolation rather than
  halting the pipeline for every other lead.

## Business Impact

| | |
|---|---|
| Response window (demo environment) | Automatic, within 15 minutes — zero manual triggering |
| Response window (production-ready) | Near-instant with a webhook trigger or paid platform tier |
| Delivery reliability | 100%, with automatic retry verified under real failure conditions |
| Fault isolation | Confirmed — a bad record never takes down the pipeline |

---

*Built with Make.com, HubSpot, Slack, Google Sheets, and Gmail. Full
technical write-up and architecture diagram available in the project
README.*
