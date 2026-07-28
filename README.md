# WhatsApp Lead Qualification Bot — Architecture Notes

Architecture documentation for a production WhatsApp bot that captures, qualifies and schedules leads for a credit brokerage, replacing a manual phone screening process.

**The source and the client are private.** This repo documents the engineering: how the system is structured, how it's monitored, and what the failures taught. No business rules, no client identity, no conversational data.

---

## What it does

Prospects arrive from paid social campaigns into WhatsApp. The bot handles the conversation 24/7, qualifies eligibility, and books an appointment with a human advisor. Every qualified lead lands in a CRM sheet, generates a card in the sales pipeline, and fires an internal notification — with no human involvement until the appointment itself.

The point wasn't automation for its own sake. Screening calls were consuming the hours that could go into closing.

---

## Architecture

It is deliberately **not** a monolith. Nine independent n8n workflows, each with its own trigger:

| Workflow | Trigger | Function |
|----------|---------|----------|
| Main bot | Inbound message webhook | Conversation, qualification, scheduling |
| Uptime canary | External cron, every 3 min | Public healthcheck carrying no secrets |
| Error handler | On-failure of main bot | Alerts on failure of the primary flow |
| Follow-up | Cron, business hours | Re-engages leads who went quiet |
| Appointment sync | Daily cron | Reconciles manual reschedules into the CRM |
| Appointment reminder | Daily cron | Sends reminders — depends on sync running first |
| Prompt staging | Webhook | Parallel environment for testing conversational changes |
| Comment-to-DM | Platform webhook | Converts public comments into private conversations |
| Privacy policy | GET webhook | Serves the required policy page |

Splitting them this way means a failure in the follow-up cron can't take down the conversation that's happening right now. The error handler is the one workflow that must never be disabled — it's the only thing that reports when the primary flow breaks.

**Data flow for a lead:** paid campaign → messaging API → n8n webhook → LLM with conversation history → CRM sheet → pipeline card → internal notification.

---

## Monitoring

The uptime canary is a public endpoint that carries **no secrets and touches no data** — its only job is to answer. An external service hits it every three minutes and alerts on silence, which catches the whole chain: server down, proxy down, orchestrator down, DNS down.

It was running before launch, not added after the first outage. A bot that silently stops answering is worse than a bot that was never deployed, because the leads still arrive and nobody knows they're being dropped.

---

## Prompt versioning, and knowing when to stop patching

The conversational prompt is explicitly versioned in the repo, one file per version.

The system went through **35+ documented hot-fix iterations**. Then something became measurable: four consecutive hot-fixes produced four new regressions. Each patch fixed its target case and broke a different one.

That was the signal to stop patching and rewrite. Not a hunch — a pattern in the log.

The rewrite wasn't driven by intuition either. It came out of a quantitative audit of the real funnel: every category of conversation abandonment was documented with its frequency and severity *before* deciding whether to rewrite that section, add a rule, or drop it entirely. The result is a traceable decision log — finding → prompt change → outcome.

Prompt deploys are versioned scripts with mandatory sequential numbering. Versions are never skipped, so the deployed state is always reconstructible from the repo.

---

## Deployment and quality gates

**Deploy by API, never through the UI.** The orchestrator's UI auto-saves, which means an open browser tab can silently overwrite a deployed workflow. Every deploy is a script that performs a write followed by an immediate read-back for validation.

That read-back exists because of a specific failure: a workflow with missing credentials stayed *active* and kept serving traffic with broken nodes — no error, no alert, just silently dropping leads. Deploying without verifying the deployed state is deploying blind.

**Pre-commit hooks** run secret detection with custom rules for the specific API key formats in this stack, on top of the default ruleset, plus merge-conflict and large-file checks.

**Data audits.** Dedicated scripts classify historical conversations to measure abandonment rates by category. Product decisions come from that measurement rather than from reading transcripts and forming an impression.

---

## Stack

Meta Ads → WhatsApp Cloud API · n8n on Docker behind Traefik · Anthropic Claude for conversation · Google Sheets as CRM · Trello for pipeline · Telegram for internal alerts · Better Stack for external uptime monitoring

**Snapshot:** 78 commits over roughly a month of active development · 130+ versioned operational scripts, none of which run against production without the write-then-verify cycle · 14 living documents covering setup, monitoring architecture, prompt design decisions, test scenarios and data audits.

---

*Jorge Rangel — [LinkedIn](https://www.linkedin.com/in/jorge-rangel87/)*
