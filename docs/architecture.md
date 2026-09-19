# Architecture

## Request flow

```text
External scheduler (cron-job.org)
  -> POST /repos/<owner>/<repo>/actions/workflows/agent.yml/dispatches (GitHub REST API)
  -> GitHub Actions run (workflow_dispatch)
  -> Python agent (single entry point, runs to completion, no long-lived process)
       -> content plan lookup (Google Sheets, next row with status="planned")
       -> date fence: skip if the topic's own scheduled date hasn't arrived yet
       -> RAG fact lookup (question built from the topic, sent to a separate
          Yandex Cloud Function; fail-open — a RAG failure never blocks generation)
       -> YandexGPT draft, grounded in RAG facts when available
       -> Telegram message with inline buttons: Publish / Redo / Skip
       -> status bookkeeping (Google Sheets: append to drafts, mark topic used)
  -> (separately, every run) poll Telegram for pending button clicks
       -> route by action: approve / skip / redo
       -> redo re-runs the RAG lookup + YandexGPT generation, posts a new
          message with fresh buttons, and disables the old message's buttons
  -> human decision in Telegram
  -> (approved) manual copy-paste publish into TenChat — not automated
```

## Component responsibilities

- **External scheduler** — dispatches GitHub Actions runs on two cadences (a lower-frequency one for post generation, a higher-frequency one so Telegram button clicks are processed quickly) because GitHub's own `schedule:` trigger does not run reliably enough for either purpose on this repository — see `security.md`.
- **Content plan client (Google Sheets)** — the source of truth for what to write about and when; a simple two-sheet model (a dated topic queue, and a drafts/status log) instead of a database, since the workload is a handful of rows per day.
- **RAG client** — a thin HTTP client for a separately deployed service; treated as an optional enrichment, not a dependency. Any failure, timeout, or empty response degrades to "generate without extra context" rather than failing the run.
- **YandexGPT client** — builds the prompt, injecting RAG-provided facts as explicitly labeled, untrusted data when present, and enforces a fixed set of generation parameters (temperature differs between first-draft and regenerate requests).
- **Telegram client** — sends messages with inline keyboards, polls for callback queries, edits/acknowledges them; polling was chosen over a webhook because there is no requirement for sub-second reaction time (the final publish step is manual either way).
- **Orchestration (main entry point)** — ties the above together, applies the human-approval-gate routing (approve/skip/redo), and contains the date-fencing and "already generated today" checks that keep a single scheduled post from firing twice or firing early.

## Trust boundaries

- The content plan (what to write, when) lives in Google Sheets, not in the model's own judgment — the agent always generates from an explicit row, never from an open-ended "pick something" prompt.
- RAG-provided text is explicitly marked in the prompt as data the model must ground itself in, not as an instruction — this is a deliberate prompt-injection defense, since the RAG answer ultimately traces back to files the agent's own author controls, but the same pattern would hold if that source were ever less trusted.
- The model's output is never auto-published — a human decision in Telegram is required before a post is even considered "approved," and TenChat publication itself is always a manual step outside this system.
- Secrets (LLM API key, Telegram bot token, Google service-account credentials) are never committed; they are injected as GitHub Actions repository secrets at runtime.

## Deployment boundaries

- The agent runs as a single short-lived process per GitHub Actions run — there is no always-on server, which keeps the design simple at the cost of the scheduling reliability problems described in `security.md`.
- The RAG service the agent calls is deployed independently (a separate Yandex Cloud Function, from a separate project) and is treated as an external dependency with its own lifecycle, not bundled into this repository.

Production identifiers — the real spreadsheet ID, Telegram chat ID, Yandex Cloud folder ID, and RAG service endpoint — are intentionally omitted from this document; see `security.md` for why.
