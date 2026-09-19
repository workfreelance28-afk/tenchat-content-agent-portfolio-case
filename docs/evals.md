# Tests and evals

## Smoke-test suite

```text
5 scenarios, all passing
```

The orchestration logic (generation, approve/redo/skip routing, time- and date-gating) is covered by a smoke-test suite that runs against faked Google Sheets, YandexGPT, and Telegram clients — no network calls, no secrets, no external services required. It re-runs after every change to the orchestration code.

Scenarios covered:

- **Generation + approve flow** — a topic due today generates exactly one post, is saved to the drafts log, and is marked used in the content plan; a second run on the same day does not generate again; an "approve" button click updates the draft's status and edits the Telegram message.
- **Skip flow** — a "skip" button click updates status and message text without touching generation.
- **Redo flow** — a "redo" click regenerates the post (verified to actually be a different draft, not a cache hit), sends a new message with fresh buttons, and disables the buttons on the old message.
- **Time-gating** — before the configured generation hour, no message is sent, regardless of what's in the content plan.
- **Unknown-action resilience** — an unrecognized callback action is logged and skipped rather than raising and aborting the whole run, so one malformed or future button type can't take down the scheduled job.

## Why this matters for the role

The test count is small on purpose — the goal wasn't "test everything," it was pinning down the specific branches that had already caused real incidents once (double-generation, a stuck approval state, a run that silently stops early). Every one of the five scenarios corresponds to a behavior the project actually depends on in production, not a hypothetical edge case.

The RAG-grounding step added later (see `PORTFOLIO.md`) is deliberately fail-open at the integration boundary rather than covered by this suite: a RAG outage is meant to degrade generation quality, never to break the pipeline, so the property being protected is "the agent still runs," which the existing time/date/idempotency tests already exercise regardless of whether RAG succeeds.
