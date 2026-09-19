# TenChat Content Agent — portfolio case

Sanitized public showcase of a private AI-integration project: a personal automation agent that drafts expert posts for TenChat (a Russian professional social network) and requires explicit human approval before anything goes out, built by Roman Melnikov (AI Integration Engineer).

This is a personal content-workflow tool, not a commercial SaaS product.

## What this shows

- Full case description, my role, and what the project proves: [`PORTFOLIO.md`](PORTFOLIO.md)
- Architecture (no production identifiers): [`docs/architecture.md`](docs/architecture.md)
- Safety/security decisions: [`docs/security.md`](docs/security.md)
- Tests and eval coverage: [`docs/evals.md`](docs/evals.md)
- A 2-3 minute walkthrough script: [`docs/walkthrough.md`](docs/walkthrough.md)
- Screenshots: [`screenshots/`](screenshots/) — to be added

## Stack

Python 3.11, GitHub Actions, Google Sheets API (service account), YandexGPT, Telegram Bot API (polling), a Retrieval-Augmented-Generation (RAG) service on Yandex Cloud Functions reused from a separate project as an internal fact-grounding API.

## Result

- Migrated a working no-code (n8n) MVP to a self-contained, testable Python service without losing any working behavior.
- Fully automated content pipeline with a mandatory human approval gate before anything is published.
- Diagnosed and fixed three real production issues caused by platform limitations, not application bugs (see `docs/security.md` and `PORTFOLIO.md`).
- Added a RAG-grounding step so the model drafts from real project facts instead of confidently inventing plausible-sounding details.
- A small but real regression-test suite (smoke tests on faked clients, no network or secrets) re-run after every change.

## Limitations

- Not a commercial product — a personal workflow tool for one TenChat account.
- Publishing into TenChat itself stays a manual, human action by design (see `docs/architecture.md` for why).
- This repository is a sanitized showcase, not the working engineering repo: production identifiers, live endpoints, spreadsheet/service-account details, and internal development history are intentionally excluded (see `docs/security.md`).

## Full project on request

The full private repository (complete source, GitHub Actions workflow, and the paired RAG service) and a live walkthrough are available on request.
