# n8n_security_automation_showcase
n8n Automations 

Four automation workflows built for a real Security Operations function,
generalised for public sharing: no company names, internal domains,
credentials, ticket IDs, or proprietary tooling references.

Each workflow folder contains:
- A **README** explaining the problem, the architecture, the interesting
  engineering decisions, and the measurable impact.
- A **sanitised `workflow.json`** ‚Äî an importable n8n workflow skeleton
  showing the actual node structure and logic pattern, with all
  credentials, URLs, and identifiers replaced by clearly-marked
  placeholders (`REPLACE_WITH_...`).

## Why this exists

Most "SOAR/automation" bullet points on a resume are unverifiable.
This repo exists to make the engineering behind that phrase inspectable:
real node graphs, real design tradeoffs (and the mistakes that led to
them), not just a diagram.

## Workflows

| Workflow | Problem it solves | Highlights |
|---|---|---|
| [Attack Surface Monitor](workflows/shodan-attack-surface-monitor/) | Continuously discover internet-exposed services across a growing domain portfolio and only alert on things that actually matter | Reduced a 75+ ticket/run noisy output to 1 high-signal finding via a severity filter; cut external API usage by 94% by combining per-domain queries into one |
| [Phishing Triage Pipeline](workflows/phishing-triage-pipeline/) | Turn user-reported phishing emails into automatic IOC extraction, URL scanning, and threat-intel logging | Webhook-driven, HMAC-signature-verified trigger; automatic pivot from ticketing-system webhook payload to raw `.eml` analysis |
| [Vulnerability Advisory ‚Üí Ticket Pipeline](workflows/vuln-advisory-to-ticket-pipeline/) | Aggregate vulnerability scanner advisories (HIGH/CRITICAL only) into a single actionable ticket per run instead of one ticket per finding | Parallel polling branches, LLM-assisted structuring of unstructured advisory text, per-run aggregation to avoid ticket-flooding |
| [Offboarding Device Audit](workflows/offboarding-device-audit/) | Automatically pull removable-media (USB) activity history for a departing employee's device as part of offboarding | Ticketing-system-triggered, endpoint-management-API-driven evidence collection with zero manual analyst lookup |

## A note on the JSON files

These are **not** literal exports from a production system. Each
`workflow.json` was rebuilt from scratch to demonstrate the same node
types, connection structure, and logic used in the original ‚Äî with
every organisation-specific value (domains, project keys, credential
IDs, webhook paths, Slack channel IDs) replaced by a placeholder.
Import them into n8n to see the actual shape of the automation; you'll
need to supply your own credentials and configuration values before
they'll run.

## Stack

n8n (self-hosted), plus integrations with: an internet-exposure
scanning API (Shodan-style), a cloud/vuln-posture platform (Wiz-style),
an EDR/endpoint-management platform, a ticketing system's REST API and
webhooks, Slack, and a threat-intel platform (MISP-style) for IOC
logging.

<img width="462" height="642" alt="image" src="https://github.com/user-attachments/assets/43de96e3-f324-4092-9d3c-a67ff12136af" />
