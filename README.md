# n8n-crm-airtable-ai

AI-powered CRM intake built in [n8n](https://n8n.io). Inbound sales leads arrive by email, get parsed and scored by Claude, and are written to an Airtable CRM — with an append-only interaction history and a Slack alert for hot leads.

This is Project 8 of my Phase 4 (n8n Mastery) bootcamp work.

## What it does

When an email lands on a label-gated address (`you+crm@gmail.com`), the workflow:

1. **Catches it** with a Gmail Trigger filtered to a single label, so only tagged leads enter the pipeline.
2. **Parses and scores it** with an AI Agent (Claude Sonnet 4.6) that reads the raw email body and extracts `name`, `company`, `phone`, a `leadScore` of Hot / Warm / Cold, and a one-line `notes` summary. A Structured Output Parser with a strict JSON Schema (including an `enum` on `leadScore`) guarantees the output matches the Airtable fields exactly.
3. **Upserts the contact** into an Airtable **Contacts** table, matching on **Email** so a returning lead updates their existing record instead of creating a duplicate.
4. **Logs the interaction** as a new row in an append-only **Interactions** table — a full timestamped history of every touchpoint.
5. **Alerts on hot leads** — an IF node checks `leadScore == "Hot"`, and only then posts a formatted message to a Slack channel.

## Flow

```
Gmail Trigger ──► AI Agent (Claude) ──┬──► Airtable: Contacts (upsert on Email)
                  + Structured Parser ├──► Airtable: Interactions (append)
                                      └──► IF (leadScore == Hot) ──true──► Slack: Send message
```

## Design notes

- **Email is the upsert match key.** The AI never outputs an email address (it could hallucinate one); the real sender address is pulled from the Gmail message metadata (`from.value[0].address`) and used as the match key. The most recent email's details win on update — standard CRM dedupe behaviour.
- **Two tables, two behaviours.** Contacts is one living record per person (upsert). Interactions is append-only (create) — so the history grows while the contact record stays current.
- **Stage is fixed to `New` on intake.** Pipeline progression (New → Contacted → Qualified …) is a human decision, not something the AI overwrites.
- **The enum guardrail.** The output parser constrains `leadScore` to exactly `Hot`, `Warm`, or `Cold`, so the AI can't write a stray value into the Airtable single-select field.

## Setup

This export contains **credential references only** — no secrets. To run it yourself:

1. **Import** `crm-airtable-ai.json` into your n8n instance.
2. **Create the credentials** (each node references one by name; reconnect them to your own):
   - Airtable Personal Access Token (scopes: `data.records:read`, `data.records:write`, `schema.bases:read`, with access to your base)
   - Gmail OAuth2
   - Anthropic API
   - Slack OAuth2
3. **Swap in your own IDs** wherever you see a `REPLACE_WITH_YOUR_...` placeholder:
   - Airtable base ID and the two table IDs (Contacts, Interactions)
   - Gmail label ID for your lead label
   - Slack channel ID for your alerts channel
4. **Build the Airtable base** with two tables:
   - **Contacts:** Name, Email, Phone, Company, Stage (single select: New/Contacted/Qualified/Proposal/Won/Lost), Lead Score (single select: Hot/Warm/Cold), Source, Last Contacted (date), Notes
   - **Interactions:** Summary, Contact Email, Channel (single select: Email/Form/Phone/Manual), Lead Score (single select: Hot/Warm/Cold), Received At (date + time)
5. **Set up the Gmail label** + a filter that applies it to mail sent to your `+crm` plus-address.
6. **Invite the Slack app** to your alerts channel if the message node returns `not_in_channel`.

## Stack

n8n · Airtable · Gmail · Anthropic Claude Sonnet 4.6 · Slack

## Notes

`Last Contacted` on the Contacts table is currently left unset by the workflow; `Received At` on Interactions is stamped with `$now.toISO()` at processing time. Stage auto-progression and linked-record relationships between the two tables are possible future enhancements.
