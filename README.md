# AI Lead Finder

An AI "open loop" that builds your offer list automatically, every day:

**Find leads → analyze their problems → score them → write custom outreach → save/send → track replies → learn → repeat.**

Pick a niche (dentists, realtors, restaurants, gyms, coaches, Etsy sellers, small
YouTubers), and the system finds businesses through official APIs, measures real
problems on their websites (no online ordering, no booking, slow mobile site,
outdated design), scores how much they need help, and drafts a personal message
based on the actual problem it found. You approve messages before anything is sent.

## Status

Planning stage. The code is built step by step with Codex:

1. `AGENTS.md` — project rules Codex reads before every task
2. `docs/BUILD_PROMPTS.md` — 12 prompts to paste into Codex, in order

## Accounts and keys you need

| What | Used for |
| --- | --- |
| Google Cloud: Places API (New) + PageSpeed Insights API | Finding local businesses, website speed/mobile checks |
| OpenAI or Anthropic API key | Problem analysis + writing outreach |
| Google Sheets service account (or just CSV) | Saving leads |
| HubSpot free CRM private app token (optional) | CRM sync |
| A separate sending email domain + mailbox | Sending outreach |
| YouTube Data API key (optional) | Small YouTuber niche |

## Safe sending

- Official APIs and public websites only; no scraping of LinkedIn, Instagram, Facebook or TikTok.
- `SEND_MODE=draft` by default; nothing is sent without approval.
- Daily send cap, permanent suppression list, real name, address and opt-out in every email.
- Email rules differ by country (CAN-SPAM in the US, PECR in the UK, consent rules in
  Greece and other EU countries). Check them for every market you target.
- Use a separate, warmed-up sending domain with SPF, DKIM and DMARC.
