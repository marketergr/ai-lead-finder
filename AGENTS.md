# AI Lead Finder — project rules

## What this project is
A daily "open loop" that finds businesses in a niche, detects real problems
(outdated website, no booking, no online ordering, weak listings, poor titles),
scores how much they need help, writes a personalized outreach message based on
the problem found, saves everything to SQLite/Sheets/CRM, sends only approved
messages, tracks replies, and learns which problems and angles get responses.

Loop: find -> analyze -> score -> write -> save/send -> track -> learn -> repeat.

## Tech rules
- Python 3.11+, package name `leadfinder`, CLI with Typer, dashboard with Streamlit.
- SQLite through sqlmodel. Every step reads from and writes to the DB, so any
  step can be re-run alone: `python -m leadfinder <step> --niche <slug>`.
- All config in `niches/<slug>.yaml` and `.env` (never hardcode keys).
- Every external call goes through one client module with retries, timeouts,
  rate limiting, and a local cache (`.cache/`) so re-runs cost nothing.
- All LLM calls go through `leadfinder/llm.py`. Provider chosen by env var
  (LLM_PROVIDER=openai|anthropic). Always request JSON, validate with pydantic,
  retry once on invalid JSON, then mark the lead `needs_review`.
- Prompts live in `prompts/*.md` as templates, not inside Python strings.
- Log every run to the `runs` table: counts per step, errors, estimated cost.
- Write pytest tests for every module. Use recorded fixtures, never live APIs
  in tests.

## Data model (SQLite tables)
- `leads`: id, niche, name, website, phone, email, address, google_place_id,
  rating, review_count, source, found_at, status (new, analyzed, scored,
  drafted, approved, sent, replied, won, lost, do_not_contact, needs_review)
- `signals`: lead_id, key, value, checked_at
- `problems`: lead_id, problem_code, severity 1-5, evidence, confidence 0-1
- `scores`: lead_id, score, tier, reason, weights_version
- `messages`: lead_id, variant, angle, subject, body, followup_body, status,
  sent_at, reply_at, reply_type (positive, neutral, negative, unsubscribe)
- `suppression`: email or domain that must never be contacted again
- `runs`: date, niche, counts per step, errors, API cost estimate

## Hard safety rules (never break these)
- Only use official APIs or public business websites. Do NOT scrape LinkedIn,
  Instagram, Facebook or TikTok pages, and never bypass logins or CAPTCHAs.
- Respect robots.txt and fetch at most 1 page per second per domain.
- AI findings must include evidence (a quote, a URL, or a measured value).
  No evidence -> the problem is dropped. Never invent facts about a business.
- Sending is OFF by default (SEND_MODE=draft). Messages need status `approved`
  before the mailer touches them. Daily send cap from config (default 20).
- Every email includes the sender's real name, business address, and a one-line
  opt-out. Any unsubscribe or negative reply -> add to `suppression` forever.
- Check `suppression` before drafting and again before sending.

## Working style
- Make the smallest change that completes the task. Do not refactor other modules.
- After each task: list files changed, the command to run it, and how to test it.
- Keep README.md updated with setup and usage.
- The build plan lives in `docs/BUILD_PROMPTS.md`. Follow it step by step.
