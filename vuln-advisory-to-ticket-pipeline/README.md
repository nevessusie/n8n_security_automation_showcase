# Vulnerability Advisory → Ticket Pipeline

Polls a cloud security posture platform for high-severity vulnerability
advisories and threat-intel items, structures the unstructured advisory
text into consistent fields, and aggregates everything into one ticket
per run instead of flooding the queue with one ticket per finding.

## Problem

A vulnerability-scanning platform can produce many advisories per day.
Filing one ticket per finding either floods the ticket queue (if
everything gets a ticket) or requires a human to manually triage the
platform's dashboard every day (if nothing is automated) — neither
scales.

## Architecture

```
Schedule Trigger (every 15 min)
   │
   ├── Branch A: Vulnerability Advisories
   │      │
   │      ▼
   │   Fetch Advisories (HIGH + CRITICAL severity only)
   │      │
   │      ▼
   │   Structure via LLM Agent
   │      │   - advisory text isn't uniformly structured across sources;
   │      │     an LLM step normalises it into consistent JSON fields
   │      │     (affected asset, CVE, severity, recommended fix)
   │      ▼
   │   Deduplicate (against items already surfaced in a prior run)
   │      │
   │      ▼
   │   Aggregate → Create ONE ticket for this run's findings
   │
   └── Branch B: Threat Intel Items
          │
          ▼
       Fetch Threat Intel Center Items (today only)
          │
          ▼
       Structure via LLM Agent
          │
          ▼
       Deduplicate
          │
          ▼
       Aggregate → Create ONE ticket for this run's findings
```

## Design decisions worth knowing about

**One ticket per run, not one ticket per finding.**
The platform this pulls from can return many findings in a single poll.
Filing a ticket per finding creates a queue nobody can keep up with.
Aggregating every finding from a single run into one ticket (with each
finding as a structured section in the description) keeps the signal
without the flood — an analyst opens one ticket and sees everything that
needs attention from that run.

**LLM-assisted structuring, with a deliberate fallback in mind.**
Advisory and threat-intel text isn't uniformly shaped — different
sources describe severity, affected assets, and references differently.
Rather than writing a brittle parser for every possible text shape, an
LLM step converts free-text advisories into a consistent JSON structure.
The tradeoff: this adds a point of failure (API errors, rate limits)
that a pure-code parser wouldn't have, so the production version needs
retry logic around this step — an LLM call that fails shouldn't silently
drop a HIGH-severity finding.

**Separating "vulnerability advisory" from "threat intel item."**
These two feeds look similar (both are "things a security platform
flagged today") but have different shapes and different urgency
profiles — an advisory has a clear severity rating tied to a specific
asset, while a threat-intel item might be a general warning about an
actor or technique with no direct severity field. Keeping them as
separate branches (rather than forcing one schema onto both) avoids
building fragile assumptions like "every item has a severity field"
into the aggregation logic.

**Aggregation via a plain data structure, not a black-box "aggregate"
node.**
An early version relied on a generic aggregation node that wrapped
output in a nested structure the downstream ticket-creation step wasn't
expecting, producing tickets with empty descriptions. Building the
aggregation as an explicit step that outputs a flat, predictable object
made the failure mode visible and fixable, instead of quietly losing
data inside a node whose internal behaviour wasn't obvious from the UI.

## Files

- [`workflow.json`](workflow.json) — sanitised, importable n8n workflow
  skeleton demonstrating this node structure.
