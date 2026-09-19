# Walkthrough (2-3 minutes)

Goal: show the engineering pipeline and the platform-limitation debugging, not just "a bot that posts to TenChat" — and do it without revealing production identifiers, secrets, or live endpoints.

## Framing

> This is TenChat Content Agent — a private automation tool I use as a portfolio case for migrating a no-code prototype to a testable, production-diagnosed Python service. It drafts expert posts and requires my explicit approval in Telegram before anything is considered final; publishing into TenChat itself stays a manual step by design, since TenChat has no public API and the alternatives I evaluated (paid workarounds, browser automation) weren't ones I was willing to risk an account on.
>
> The interesting engineering isn't the LLM call — it's what it took to keep this running reliably: a no-code workflow that kept posting after I'd "disabled" it, a GitHub Actions scheduler that silently ran far less often than configured, and a Telegram approval window that expires faster than a low-frequency job can reliably respond to. All three are documented with root cause and fix in `docs/security.md`.
>
> Most recently, I added a grounding step: before generating a post, the agent asks a separately built RAG service — a Yandex Cloud Function from another project of mine — what I actually did related to the topic, and only that becomes the source of concrete detail in the prompt. That's why the posts talk about real project specifics instead of confident-sounding invention.

## What to show

1. **Pipeline overview** — walk through the flow described in `docs/architecture.md`: external scheduler → GitHub Actions → content plan lookup → RAG lookup → YandexGPT draft → Telegram approval → manual publish.
2. **My role and the migration story** — frame it as: built it twice on purpose (n8n first, then Python), not because the first attempt failed, but because I needed more control once real usage started surfacing platform limits.
3. **A generated post with RAG grounding** — show a Telegram message with the three buttons (Publish / Redo / Skip), and point out that the specific project details in the text came from the RAG lookup, not the model's own guesswork.
4. **The Redo path** — click "Redo" (or show a prior example) and point out that it's a genuinely different draft, with fresh buttons, and that the old message's buttons get disabled so there's never ambiguity about which draft is live.
5. **Production incidents, briefly** — mention one or two of the three real issues in `docs/security.md` (the duplicate-posting n8n workflow is the most memorable) to show this wasn't a "build once, done" project.
6. **Tests** — mention the five-scenario smoke-test suite (`docs/evals.md`) and that it runs against faked clients, so it can be re-run after every change without touching real Google Sheets, YandexGPT, or Telegram.

*(Screenshots for this walkthrough — the Telegram approval message, a GitHub Actions run log, the smoke-test output — are being added; see `screenshots/`.)*

## What not to show

- Any production identifiers: the real spreadsheet ID, Telegram chat ID, Yandex Cloud folder ID, or the RAG service's endpoint URL.
- Real drafted post content tied to identifiable, non-public project details beyond what's already in this portfolio.

## Closing line

> This project isn't meant to prove that "an LLM can write social posts" — it's meant to show what it actually takes to keep an AI-assisted automation reliable once it leaves the demo stage: platform limitations you have to diagnose yourself, a human approval gate you don't compromise on, and grounding the model in real facts instead of trusting it to know what you actually did.
