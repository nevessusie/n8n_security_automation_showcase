# Attack Surface Monitor

Continuously scans an organisation's domain portfolio for internet-exposed
services (via an internet-exposure scanning API), filters out noise, and
opens tickets only for findings that are actually actionable — with
automatic ticket lifecycle management (dedup, reopen-detection, auto-close).

## Problem

Running exposure scans against a growing list of domains (organic growth
plus acquisitions) and manually triaging results doesn't scale, and a
naive "alert on everything" approach produces too much noise to be
useful — analysts start ignoring the channel.

## Architecture

```
Schedule Trigger
   │
   ▼
Build Domain List  ──────────► single combined query
   │                             (one API call covering every domain,
   │                              instead of one call per domain)
   ▼
Exposure Scan API Call
   │
   ▼
Flatten + Filter (Code node)
   │   - dedupe raw results
   │   - apply severity filter:
   │       include if (port ∈ {admin/db ports}) OR (CVE flagged)
   │       else drop
   ▼
Host Enrichment (per finding)
   │
   ▼
Ticketing System: search open tickets for this finding ──► already open? skip
   │
   ▼ (not found)
Ticketing System: search closed/resolved tickets ──► was previously closed
   │                                                  and reopened? flag it
   ▼ (genuinely new)
Create Ticket (structured description: host, port, service, CVEs, first seen)
   │
   ▼
Slack Notification (digest, not per-finding — avoids channel spam)

[Separate scheduled branch]
Reconciliation Job: for every open ticket created by this workflow,
re-check whether the finding still exists → if resolved, auto-close ticket
```

## Design decisions worth knowing about

**Single combined query instead of a per-domain loop.**
The first version looped over each domain and issued one API call per
domain — at ~17 domains, that's 17 calls (and API credits) per run. Most
exposure-scanning APIs support comma-separated hostname filters in a
single query. Restructuring to one combined query cut credit consumption
by roughly 94% with no loss of coverage. Everything downstream (the
per-finding pipeline) is unaware of whether findings came from 1 query or
17 — the restructure only touched the query-building stage.

**A severity filter, not a keyword filter.**
Early unfiltered output produced 75+ candidate tickets per run — far too
noisy to triage. The filter that actually worked was structural, not
keyword-based: only surface a finding if it's on a known admin/database
port (SSH, RDP, SMB, common DB ports, etc.) **or** the scanning API itself
has flagged a CVE against the service. That single rule took output from
75+ candidates down to a small number of genuinely actionable findings
per run.

**Full ticket lifecycle, not just creation.**
A monitor that only creates tickets (and never closes them) trains
analysts to ignore it once the backlog fills with stale, already-fixed
findings. This workflow's final stage re-checks every open
workflow-created ticket on a schedule and auto-closes it once the
underlying exposure is gone — while distinguishing a "closed and now
recurring" finding (reopen) from a genuinely new one, since those
deserve different handling.

## Files

- [`workflow.json`](workflow.json) — sanitised, importable n8n workflow
  skeleton demonstrating this node structure.
