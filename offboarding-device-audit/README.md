# Offboarding Device Audit

Triggered by an offboarding ticket, this workflow automatically pulls a
departing employee's removable-media (USB) activity history from the
endpoint management platform and attaches it to the ticket — no manual
analyst lookup required.

## Problem

Offboarding checklists often include "check whether this person copied
data to removable media before leaving," but doing that lookup by hand
for every departure (across dozens of employees per month) is exactly
the kind of repetitive, easy-to-skip step automation should own.

## Architecture

```
Webhook (offboarding ticket created)
   │
   ▼
Normalise Fields
   │   - extract employee identifier, device ID, ticket ID from the
   │     ticketing system's payload shape
   ▼
If: is this ticket type actually an offboarding request?
   │
   ▼ (yes)
Endpoint Management API: fetch device(s) for this employee
   │
   ▼
Endpoint Management API: fetch removable-media event history for device
   │   - filtered to a configurable lookback window (e.g. last 90 days)
   ▼
Format Findings (Code node)
   │   - summarise device count, event count, and any large transfers
   ▼
Comment on Ticket with formatted findings
   │
   ▼
If: any removable-media activity found in the lookback window?
   │
   ├── yes → flag ticket for manual review (label/priority bump)
   └── no  → comment "no removable media activity found," no further action
```

## Design decisions worth knowing about

**Ticket-triggered, not scheduled.**
This runs off the offboarding ticket's creation event rather than a
nightly batch job — the audit needs to be available by the time HR/IT
actually process the departure, not the next morning.

**Findings become a ticket comment, not just a raw data dump.**
The endpoint management API can return a lot of raw event data. Rather
than pasting that directly into the ticket, a formatting step
summarises it into what an analyst or HR reviewer actually needs to
act on: device count, total events, and specifically flagging any
large transfers — with the option to pull the full raw log separately
if something looks worth digging into.

**A clear "found nothing" path, not just a "found something" path.**
An audit workflow that only posts a comment when it finds something
creates ambiguity: did it run and find nothing, or did it silently
fail? Explicitly posting "no removable media activity found in the
last 90 days" makes a clean result distinguishable from a workflow
that never ran.

## Files

- [`workflow.json`](workflow.json) — sanitised, importable n8n workflow
  skeleton demonstrating this node structure.
