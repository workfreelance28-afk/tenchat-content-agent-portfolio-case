# TenChat Content Agent: portfolio case

## What it is

TenChat Content Agent is a private automation tool that drafts expert posts for TenChat, delivers each draft to Telegram for approval, and requires an explicit human decision — publish, redo, or skip — before anything is treated as final. Actual publishing into TenChat is still done by hand, by design.

It is not positioned as a commercial product. Its value as a portfolio case is that it's a real, evolving system: it started as a no-code workflow, was migrated to code once the no-code tool's limits became a problem, and has since had several real production issues diagnosed and fixed from GitHub Actions logs rather than hypothesized from a spec.

## My role

I designed and built the whole pipeline twice: first as an n8n workflow (schedule trigger → content plan lookup → LLM generation → Telegram approval → status bookkeeping in Google Sheets), then as a 1:1 behavioral port to a self-contained Python service once I needed more control than the no-code tool could give me.

Beyond the initial build, I was responsible for keeping it running: diagnosing why a disabled n8n workflow kept posting anyway, working around an unreliable GitHub Actions scheduler, fixing a Telegram API timing issue, and — most recently — integrating a separate Retrieval-Augmented-Generation (RAG) service I'd already built for another project so the model stops inventing facts and starts using real ones.

## Problem

A general-purpose LLM asked to write "an expert post about topic X" has two failure modes that matter for a public professional-network audience:

- it reaches for generic, AI-sounding openers ("In today's fast-changing world, X is becoming increasingly important...");
- when asked to be concrete about "what I actually did," it invents plausible-sounding but fabricated details, because nothing in its prompt tells it what actually happened.

Separately, the automation layer itself needed to survive several platform quirks that aren't documented anywhere and only show up in production: a no-code tool that doesn't fully stop when you disable it, a "cron" scheduler that silently runs far less often than configured, and an approval API with a short, undocumented expiry window.

## Solution

```text
External scheduler (workaround for GitHub Actions' unreliable cron)
  -> GitHub Actions (workflow_dispatch)
  -> Python agent
       -> content plan lookup (Google Sheets, status = "planned")
       -> RAG fact lookup (Yandex Cloud Function, reused from another project)
       -> YandexGPT draft, grounded in the RAG facts when available
       -> Telegram message with inline buttons: Publish / Redo / Skip
       -> status bookkeeping (Google Sheets: drafts + content plan)
  -> human decision in Telegram
  -> (approved) manual copy-paste publish into TenChat
```

The system prompt bans a specific list of generic openers and explicitly forbids inventing statistics or facts that weren't given to it. Since the model previously had nothing but a topic title and category to work from, it would fall back to generic phrasing or plausible-sounding invention. The fix wasn't a better prompt alone — it was giving the model something real to ground itself in: a question built from the topic is sent to a RAG service (a Yandex Cloud Function backed by a vector index of my own project write-ups), and its answer is injected into the prompt as the only allowed source of specific detail, with the injected text explicitly marked as data rather than instructions.

Full architecture detail (without production identifiers): [`docs/architecture.md`](docs/architecture.md). Safety/security decisions: [`docs/security.md`](docs/security.md). Test coverage: [`docs/evals.md`](docs/evals.md).

## Result

- Ported a working n8n MVP (schedule trigger, Google Sheets read/write, LLM generation, three-button Telegram approval) to Python with identical behavior, verified against the same test scenarios.
- Diagnosed and fixed three real production issues that were platform limitations, not application logic bugs (see `docs/security.md`):
  - a "disabled" n8n workflow kept generating duplicate posts because disabling a workflow in the UI doesn't stop its independent schedule trigger — fixed by unpublishing it outright;
  - GitHub Actions' `schedule:` trigger ran far less often than configured (hours instead of minutes) — worked around with an external scheduler dispatching runs through the GitHub REST API;
  - Telegram's callback-approval window expires faster than the automation's run cadence, causing a benign-but-unhandled `400` — fixed by making the acknowledgment call fail-safe and reordering it after the state change it doesn't gate.
- Added a date-fencing check after a topic scheduled for a future day was generated a day early — the generation step now explicitly checks the topic's own date, not just "did anything already run today."
- Integrated an existing RAG service as an internal fact-grounding API for a second, independent project — reusing infrastructure instead of duplicating it.
- A small smoke-test suite (five scenarios, faked clients, no network or secrets) is re-run after every change to the orchestration logic.

## Stack

Python 3.11, GitHub Actions (`workflow_dispatch`), an external scheduler used to work around GitHub's unreliable native cron, Google Sheets API via a service account, YandexGPT, Telegram Bot API (polling, not webhook), and a Retrieval-Augmented-Generation service on Yandex Cloud Functions reused from a separate project.

## What this project proves

- Migrating a working no-code prototype to code without regressing behavior, verified with tests rather than assumed.
- Diagnosing production issues from logs and platform behavior, not just from application code.
- Designing around a third-party platform's undocumented limitations (unreliable scheduling, short-lived approval tokens, a no-code tool that doesn't fully stop) instead of assuming the platform behaves as advertised.
- Reusing one AI service (RAG) as an internal API for a second, unrelated agent instead of duplicating the retrieval logic.
- Grounding LLM output in real, retrieved facts and treating retrieved text as untrusted data in the prompt.
- Keeping a human approval gate as a hard requirement, not a nice-to-have, throughout an architecture rewrite.

## Limitations

- Personal workflow tool for one TenChat account, not a multi-tenant product.
- TenChat has no public developer API; automating the final publish step was evaluated and explicitly rejected (paid third-party workarounds don't guarantee API access either, and browser automation risks Terms-of-Service violations and account suspension) — publishing stays a manual, human action.
- This case is a sanitized showcase built from the private repository — production identifiers, live endpoints, spreadsheet IDs, and internal development history are intentionally left out. The full private repository and a live walkthrough are available on request.
