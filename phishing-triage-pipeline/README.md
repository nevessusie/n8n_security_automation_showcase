# Phishing Triage Pipeline

Turns a user-reported phishing email into automated IOC extraction, URL
reputation scanning, and threat-intel logging — triggered directly by
the ticketing system's webhook rather than polling a mailbox.

## Problem

Users report suspicious emails via a "report phishing" button, which
files a ticket with the original email attached. Manually opening every
report, extracting URLs/IPs, checking them, and logging confirmed
threats doesn't scale, and delays in triage matter — a real phish sitting
in an inbox for hours is a live risk.

## Architecture

```
Webhook (ticket-created event from ticketing system)
   │
   ▼
Verify Webhook Signature (HMAC, Code node)
   │   - reject if signature doesn't match
   │   - protects against spoofed/replayed webhook calls
   ▼
Normalise Fields (Code node)
   │   - extract ticket ID, reporter, attachment reference from the
   │     ticketing system's payload shape
   ▼
If: is this actually a phishing-report ticket?
   │   - filters out unrelated webhook events firing on the same endpoint
   ▼ (yes)
Comment on Ticket — "report received, triage in progress"
   │
   ▼
List Ticket Attachments → Download Attachment (.eml)
   │
   ▼
Extract IOCs (Code node)
   │   - parses the raw .eml for URLs, IPs, sender domain
   │   - drops anything matching an internal allow-list
   │     (own domains, known vendors) before scanning
   ▼
urlscan.io Submission + Result Retrieval
   │
   ▼
Classify Verdict (Code node)
   │   - score-based malicious / suspicious / clean classification
   ▼
   ├── malicious/suspicious → Slack alert to SecOps channel
   │                          + push IOCs to threat-intel platform (MISP-style)
   └── clean → close ticket with "no threat found" comment
```

## Design decisions worth knowing about

**Signature verification on the webhook, not just an open endpoint.**
Any workflow that exposes a public webhook URL needs to verify the
request actually came from the system it claims to — otherwise anyone
who finds the URL can trigger arbitrary "ticket" processing. The
verification step computes an HMAC over the raw request body using a
shared secret and rejects anything that doesn't match, before any
downstream processing happens.

**Raw `.eml` attachment, not a forwarded/re-typed summary.**
The workflow pulls the actual original email as a `message/rfc822`
attachment rather than relying on a human-typed summary of "here's the
suspicious link" — this preserves full original headers, which matters
for sender-domain verification and for distinguishing a spoofed sender
from a genuinely compromised one.

**Allow-list filtering before scanning, not after.**
Every internal/known-vendor domain found in the email gets filtered out
*before* hitting the URL-scanning API, not after — this avoids wasting
scan quota and avoids a false "suspicious" verdict on a legitimate
internal link that happens to appear alongside a genuinely malicious
one in the same email.

**Confirmed threats get logged to a threat-intel platform, not just
closed.**
A ticket-only workflow loses the IOC the moment the ticket is closed.
Pushing confirmed-malicious IOCs (URLs, IPs, sender domains) into a
threat-intel platform means the next time the same indicator shows up —
in this workflow or any other detection surface — it's already known.

## Files

- [`workflow.json`](workflow.json) — sanitised, importable n8n workflow
  skeleton demonstrating this node structure.
