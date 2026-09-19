# Security and safety

## What this project does

- **Secrets stay out of code.** The YandexGPT API key, Telegram bot token, and Google service-account credentials are never committed; they are injected from GitHub Actions repository secrets at runtime. `.env` and local runtime artifacts are gitignored and are not part of this public case.
- **Human approval is a hard requirement, not a default.** Every generated post reaches Telegram with three explicit choices — Publish, Redo, Skip — and nothing is treated as final until a human picks one. The system does not auto-publish under any configuration.
- **No automated publishing into TenChat.** TenChat has no public developer API. A paid third-party workaround was evaluated and rejected (it doesn't guarantee open API access even on paid tiers), and browser automation of the TenChat web UI was evaluated and rejected as a Terms-of-Service and account-suspension risk. Publishing stays a deliberate manual action.
- **RAG input is treated as untrusted data in the prompt.** When the RAG service returns facts, they are injected into the YandexGPT prompt with an explicit instruction that the text is data to ground the answer in, not a command to follow — a standard prompt-injection defense, applied even though the current source is the author's own material.
- **RAG lookups fail open, not silently wrong.** If the RAG service is unreachable, times out, errors, or returns an empty answer, the agent logs it and generates the post without the extra context, rather than blocking generation or (worse) treating an error as "no relevant facts, therefore invent your own."
- **Idempotent generation.** A topic only generates once its own scheduled date has arrived, and only once per day even if the automation runs many times that day — both checks exist because of a real bug found in production (see below), not as speculative hardening.

## Real production issues this project surfaced and fixed

These are included because diagnosing them was a bigger part of the engineering work than the original build:

- **A "disabled" no-code workflow kept posting.** Disabling a workflow in the n8n UI stopped new manual runs but not its already-active schedule trigger, so duplicate posts with the old, buggy button behavior kept arriving daily after the Python version was supposed to have fully replaced it. Root-caused by directly inspecting the platform's pending-update queue; fixed by unpublishing the workflow outright rather than just disabling it.
- **GitHub Actions' `schedule:` trigger is unreliable at the intervals this project needed.** A 10-minute cron ran, in practice, anywhere from ~1 to ~3.5 hours apart; an "honest" hourly cron still wasn't precise enough once a fast reaction to Telegram button presses mattered. Worked around by having an external scheduler dispatch runs through the GitHub REST API's `workflow_dispatch` endpoint on two separate cadences.
- **Telegram's callback-acknowledgment window expires before a low-frequency automation can reliably respond.** This surfaced as an unhandled `400 Bad Request` that could abort processing of a real button click. Fixed by making the acknowledgment call fail safely (logged, non-fatal) and by reordering it after the state change it doesn't gate, so the important side effect (updating the post's status) always completes first.
- **A topic could generate a day early.** The "already generated today" check didn't also confirm the topic's own scheduled date had arrived, so a future-dated topic could be picked up and published ahead of schedule. Fixed with an explicit date-fence check.

## What this public case deliberately leaves out

This repository is a sanitized showcase built from a private engineering repository. The following are intentionally excluded and are not needed to evaluate the engineering approach:

- The real Google Sheets spreadsheet ID, Telegram chat ID, and Yandex Cloud folder ID.
- The RAG service's real endpoint URL (it is a separate private project's deployed Cloud Function).
- GitHub Actions repository secrets, the external scheduler's configuration, and any `.env` values.
- Internal stage-by-stage project documentation and decision logs — useful for the engineering team, not for evaluating the case.

Placeholders like `<spreadsheet-id>`, `<telegram-chat-id>`, and `<rag-service-url>` stand in wherever an infrastructure identifier would otherwise appear.

## Positioning

This is a personal content-automation tool for one TenChat account — not a commercial product, not a multi-tenant service, and not a claim that any step here is production-hardened beyond what a single-user workflow needs.
