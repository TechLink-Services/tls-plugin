---
name: incident-runbook
description: >
  Use this skill when the user is dealing with an operational problem, something that went
  wrong, or a situation requiring immediate action. Trigger phrases include: "there's an
  incident", "something went wrong", "we have a problem", "technician didn't show up",
  "job fell through", "payment failed", "invoice is wrong", "client is complaining",
  "delivery is late", "deliverable wasn't sent", "work order was mis-assigned",
  "urgent reschedule", "technician no-show", "billing error", "client escalation",
  "disputed charge", "what do I do about", or any description of an unexpected issue
  that needs a structured response. Covers field and scheduling incidents, billing and
  payment errors, and client deliverable problems.
metadata:
  version: "0.1.0"
---

When this skill triggers, help the user respond to an operational incident quickly and systematically. Identify the incident type, load the appropriate runbook steps, and guide the user through resolution.

## Incident classification

First, determine which category the issue falls into based on the user's description:

1. **Field / scheduling incident** — technician no-show, mis-assigned work order, urgent reschedule needed, technician unreachable, job site access problem.
2. **Billing / payment error** — failed payment, incorrect invoice, disputed charge, client overbilled, payment not applied correctly.
3. **Client deliverable issue** — deliverable not sent on time, wrong deliverable sent, client hasn't received expected output, deliverable quality dispute.

If the incident spans multiple categories, address the most urgent first and flag the others.

## Core behavior

1. Ask one clarifying question if the category isn't clear from the description.
2. State the identified incident type so the user can confirm or correct it.
3. Walk through the resolution steps from `references/runbook-steps.md` for that incident type.
4. Use the dev-portal connector to pull relevant context where helpful (e.g., the work order in question, the invoice, the client record).
5. After each step, check if the user needs help with that step before moving to the next one.
6. At the end, summarize what was done and suggest any follow-up actions (e.g., logging the incident, notifying a manager, updating the work order status).

## Tone

Stay calm and action-oriented. Users reaching for this skill are often stressed. Lead with what to do, not what went wrong.
