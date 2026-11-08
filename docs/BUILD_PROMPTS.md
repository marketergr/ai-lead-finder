# AI Lead Finder — Codex build prompts

Paste these into Codex in order, one per task. Each prompt says what to build
and how you'll know it works. Commit after each step that runs end to end.
If something breaks, paste the error back with:
"Fix this error without changing other modules."

Loop: Niche config -> Find leads -> Scan website -> AI problems -> Score lead ->
Write outreach -> You approve -> Send -> Track replies -> Learn -> (repeat daily)

---

## Prompt 1: Project skeleton

```text
Read AGENTS.md. Create the project skeleton for `leadfinder`:
- pyproject.toml with dependencies: typer, httpx, pydantic, sqlmodel, pyyaml,
  python-dotenv, beautifulsoup4, trafilatura, tenacity, streamlit, pytest.
- Package layout: leadfinder/{__main__.py, cli.py, config.py, db.py, llm.py,
  http.py} and empty subpackages sources/, enrich/, analyze/, score/,
  outreach/, export/, send/, track/, learn/. Folders: niches/, prompts/, tests/.
- db.py: sqlmodel tables leads, signals, problems, scores, messages,
  suppression, runs, exactly as described in AGENTS.md and with the lead status
  values new, analyzed, scored, drafted, approved, sent, replied, won, lost,
  do_not_contact, needs_review. Add `init_db()`.
- http.py: one shared httpx client with timeouts, tenacity retries, a per-domain
  rate limiter (1 req/sec) and a disk cache in .cache/ keyed by URL.
- .env.example with every key we will need (GOOGLE_API_KEY, LLM_PROVIDER,
  OPENAI_API_KEY, ANTHROPIC_API_KEY, LLM_MODEL, SMTP_*, IMAP_*, HUBSPOT_TOKEN,
  GOOGLE_SHEETS_ID, SEND_MODE=draft, DAILY_SEND_CAP=20, SENDER_NAME,
  SENDER_ADDRESS).
- CLI command `python -m leadfinder init` that creates the DB.
Done when: `python -m leadfinder init` creates leadfinder.db and `pytest` passes.
```

## Prompt 2: Niche config files

```text
Create a niche config system.
- config.py: pydantic model `NicheConfig` loaded from niches/<slug>.yaml with
  fields: slug, label, search_queries (list), locations (list of
  "city, country"), max_new_leads_per_run, my_offer (one sentence: what I sell),
  problem_checks (list of problem codes to look for), scoring_weights
  (dict problem_code -> points), positive_signals (dict signal -> points, e.g.
  good reviews means they have money and customers), language (en/el),
  tone (friendly/professional), cta (the soft call-to-action).
- Create 3 example files: niches/restaurants-athens.yaml (offer: online
  ordering + digital menu), niches/dentists-us.yaml (offer: online booking
  website), niches/gyms-uk.yaml (offer: website + lead capture).
- Problem code catalog in leadfinder/problems.py with a description for each:
  no_website, outdated_website, not_mobile_friendly, slow_website, no_ssl,
  no_booking_system, no_online_ordering, no_clear_offer, no_contact_form,
  weak_reviews, few_reviews, no_social_links, weak_listing_photos,
  poor_video_titles.
- CLI: `python -m leadfinder niches` lists configs and validates them.
Done when: the 3 YAML files validate and invalid ones show a clear error.
```

## Prompt 3: Lead finder (Google Places)

```text
Build sources/google_places.py using the Google Places API (New) Text Search
endpoint (https://places.googleapis.com/v1/places:searchText) with a field
mask for: id, displayName, formattedAddress, websiteUri,
nationalPhoneNumber, rating, userRatingCount, businessStatus, types.
- For each search_query x location in the niche config, page through results
  until max_new_leads_per_run NEW leads are found.
- Skip businesses that are not OPERATIONAL.
- Deduplicate by google_place_id and by website domain; skip anything in the
  suppression table.
- Save to `leads` with status `new` and source `google_places`.
- Define a `LeadSource` base class (method `find(niche) -> list[LeadIn]`) so we
  can add YouTube/Etsy sources later with the same interface.
- CLI: `python -m leadfinder find --niche restaurants-athens [--limit 20]`
  prints a table of new leads found.
- Tests with a recorded JSON fixture, no live API.
Done when: running find twice does not create duplicates.
```

## Prompt 4: Website and signal scanner

```text
Build the enrich step. For each lead with status `new`:
1. enrich/website.py: fetch the homepage (respect robots.txt) and up to 3
   internal pages whose links contain menu, order, book, booking, appointment,
   contact, pricing. Extract and store as `signals`:
   has_ssl, final_url, page_title, meta_description, last_copyright_year,
   has_viewport_meta, has_contact_form, emails_found (only public business
   emails shown on the site), has_booking_widget (detect Calendly, Acuity,
   SimplyBook, Booksy, Mindbody, Fresha, OpenTable, Resy, Zocdoc etc.),
   has_online_ordering (detect own ordering, DoorDash, Uber Eats, Wolt, efood,
   Deliveroo, Toast, Square Online links; record which), social_links,
   cms_generator (from meta generator), has_analytics_or_pixel, word_count,
   and a cleaned text excerpt (max 3,000 chars) saved for the AI step.
2. enrich/pagespeed.py: call PageSpeed Insights API (mobile strategy), store
   performance_score, seo_score, accessibility_score, lcp_seconds.
3. No website at all -> signal has_website=false.
These are FACTS measured by code, not AI. Store them with checked_at.
CLI: `python -m leadfinder enrich --niche <slug>`.
Done when: 10 real leads get signals and a failed fetch never crashes the run
(it records fetch_error and moves on).
```

## Prompt 5: AI problem analyzer

```text
Build analyze/problems.py and prompts/analyze_problems.md.
Input per lead: niche config, business name, rating, review_count, all
signals, and the text excerpt. The LLM must return JSON matching this schema:
{
  "problems": [
    {"problem_code": one of the niche's problem_checks,
     "severity": 1-5,
     "evidence": "exact quote, URL, or measured value from the input",
     "confidence": 0.0-1.0,
     "why_it_costs_them": "one sentence in plain words"}
  ],
  "strengths": ["short positive facts, e.g. 4.7 stars from 380 reviews"],
  "best_angle": "the single problem most likely to make them reply"
}
Rules in the prompt: use only the input data; if evidence is not in the input,
do not report the problem; prefer problems that cost them money or customers.
After the LLM call, validate in code: drop any problem whose evidence string
does not appear in the input or match a signal value, and drop confidence < 0.6.
Also add deterministic problems straight from signals without the LLM
(no_ssl, not_mobile_friendly when performance_score < 50 or no viewport,
no_online_ordering, no_booking_system, outdated_website when copyright year is
3+ years old).
Save to `problems`, set status `analyzed`. Log token usage and cost per lead.
CLI: `python -m leadfinder analyze --niche <slug>`.
Done when: tests show a hallucinated problem with fake evidence gets dropped.
```

## Prompt 6: Lead scoring

```text
Build score/scorer.py. Scoring is rules first, AI second:
- need_score (0-60): sum of scoring_weights for each problem x severity/5,
  capped at 60.
- ability_score (0-40): from positive_signals, e.g. rating >= 4.3 and
  review_count >= 50 means an established business that can pay; has a
  website/phone/email we can reach; is not a big chain (skip names that appear
  3+ times in our DB or match a chain list in config).
- score = need + ability. Tier A >= 70, B 50-69, C < 50.
- No reachable contact (no email and no contact form) -> tier C with reason
  "no contact route".
- Generate a one-line reason in code from the top factors, e.g.
  "No online ordering + weak mobile site + 4.6 stars from 410 reviews. High-value lead."
- Store weights_version so we can compare later.
CLI: `python -m leadfinder score --niche <slug>` prints the top 20 by score.
Done when: unit tests cover each tier and the chain filter.
```

## Prompt 7: Custom outreach writer

```text
Build outreach/writer.py and prompts/write_outreach.md. Only for tier A and B
leads not in suppression.
Input: niche config (my_offer, tone, language, cta), business name, strengths,
best_angle problem with its evidence and why_it_costs_them.
The LLM returns JSON: {subject, body, followup_body, angle, problem_code}.
Message rules in the prompt:
- 60-110 words. Plain text. No links in the first email.
- Line 1: a genuine, specific compliment from strengths (e.g. their reviews).
- Line 2: the ONE problem, stated with the evidence, in plain words.
- Line 3: what it probably costs them (lost orders, missed bookings).
- Line 4: offer a free, specific idea ("I put together a quick idea..."),
  not a pitch. Soft CTA from config, a yes/no question.
- No hype words (guaranteed, skyrocket, 10x), no fake familiarity, never claim
  you are a customer, never invent numbers.
- Subject: under 6 words, lowercase feel, mentions the business or problem.
- followup_body: 40 words, sent 4 days later if no reply.
- Write in the niche's language (Greek when language=el).
Generate 2 variants per lead with different angles (variant A uses best_angle,
variant B the second-best problem) so we can A/B test.
Append the footer from config in code (sender name, address, opt-out line),
not via the LLM. Save to `messages` with status `draft`.
CLI: `python -m leadfinder write --niche <slug>`.
Done when: a test checks word count, no links, footer present, and that the
problem mentioned matches a stored problem for that lead.
```

## Prompt 8: Save to Sheets, CSV and HubSpot

```text
Build the export step.
- export/csv_export.py: `leads_<niche>_<date>.csv` with name, website, phone,
  email, score, tier, reason, top problems, subject, body, status.
- export/sheets.py: append/update rows in a Google Sheet (service account,
  gspread). One tab per niche. Upsert by google_place_id so re-runs update
  instead of duplicating. Columns as in the CSV plus an "approve" checkbox
  column that the approval sync can read back.
- export/hubspot.py (optional, only if HUBSPOT_TOKEN is set): upsert company
  (by domain) and contact (by email), add score/tier/reason as custom
  properties, attach the draft message as a note.
- CLI: `python -m leadfinder export --niche <slug> --to csv|sheets|hubspot`.
- `python -m leadfinder sync-approvals --niche <slug>` reads the Sheet's approve
  column and sets matching messages to `approved`.
Done when: running export twice leaves one row per lead.
```

## Prompt 9: Approval dashboard

```text
Build a Streamlit app at leadfinder/dashboard.py (`streamlit run ...`).
Pages:
1. Review queue: drafted leads sorted by score. For each: business name,
   website link, score, tier, reason, the problems with evidence, both message
   variants in editable text areas. Buttons: Approve A, Approve B, Regenerate,
   Skip, Do not contact (adds to suppression).
2. Pipeline: counts by status per niche, and today's run log from `runs`.
3. Results: sent, replies, positive replies, reply rate by problem_code, by
   angle, by variant, by niche (simple tables + bar charts).
4. Settings view: shows SEND_MODE, daily cap, and today's sent count (read-only).
Edits made in the text area are saved to the message before approval.
Done when: I can approve 5 messages in under a minute and see them as approved
in the DB.
```

## Prompt 10: Safe sender

```text
Build send/mailer.py.
- Modes from env: SEND_MODE=draft (default, sends nothing), manual (sends only
  messages with status approved), auto (also auto-approves tier A messages
  whose lead has score >= AUTO_SEND_MIN_SCORE; I will only enable this later).
- Before each send: re-check suppression (email and domain), skip if this lead
  was contacted in the last 90 days, enforce DAILY_SEND_CAP, wait a random 60-180
  seconds between sends, only send between 09:00 and 17:00 in the lead's
  timezone, Monday-Friday.
- Send via SMTP (or Gmail API if configured) from the sending domain. Plain
  text. Add List-Unsubscribe header (mailto). Save Message-ID.
- Schedule follow-up: if no reply after 4 business days, send followup_body
  once, in the same thread. Never more than 1 follow-up.
- `--dry-run` flag prints what would be sent.
CLI: `python -m leadfinder send --niche <slug> [--dry-run]`.
Done when: tests prove draft mode never calls SMTP, the cap holds, and a
suppressed email is never sent.
```

## Prompt 11: Reply tracking and learning

```text
Build track/replies.py and learn/optimizer.py.
Replies:
- Poll the sending mailbox over IMAP, match replies to messages by
  In-Reply-To/References headers or sender email.
- Classify each reply with the LLM into positive, neutral, negative,
  unsubscribe, out_of_office, bounce (JSON output). Store reply_type and
  reply_at, set lead status replied.
- unsubscribe, negative or bounce -> add to suppression immediately and cancel
  any follow-up. Positive -> push to HubSpot as a deal (if enabled) and flag in
  the dashboard.
Learning:
- Weekly, compute reply rate and positive rate per niche x problem_code,
  angle, variant, subject style, and score tier (need at least 30 sends per group
  before trusting it).
- Write learn/insights.json with the winners and losers.
- The writer prompt loads insights.json: include the top 3 best-performing
  real messages as examples and tell it which angles to avoid.
- Suggest new scoring_weights (problems with higher positive reply rate get
  more points) as a proposed YAML diff in the dashboard. I approve it manually;
  never auto-change weights.
CLI: `python -m leadfinder track` and `python -m leadfinder learn`.
Done when: replies from a fixture mailbox are matched and classified correctly.
```

## Prompt 12: The daily open loop

```text
Build `python -m leadfinder run --niche <slug>` (or --all) that runs:
find -> enrich -> analyze -> score -> write -> export -> sync-approvals ->
send -> track, in that order, continuing past per-lead errors.
- Write a run summary to `runs` and print it: new leads, analyzed, A/B/C
  counts, drafts written, sent, replies, positive replies, errors, est. cost.
- Optional: email or Telegram me that summary each morning.
- Add a GitHub Actions workflow .github/workflows/daily.yml that runs at 07:00
  Europe/Athens on weekdays using repo secrets, and persists leadfinder.db as
  an artifact or commits it to a private data branch. Also document a cron
  alternative for a VPS in README.
- Add a lock so two runs never overlap.
Done when: one command runs the whole loop on a fresh DB and the summary shows
non-zero counts for find, analyze, score and write.
```

---

## Optional source add-ons (one prompt each, after the core works)

- **Small YouTubers:** "Add sources/youtube.py using the YouTube Data API v3:
  search channels by niche keywords with 1k-50k subscribers, pull the last 10
  video titles, view counts and thumbnails. Add problems poor_video_titles and
  inconsistent_uploads with evidence. Contact only via the business email the
  channel publishes in its About text."
- **Etsy sellers:** "Add sources/etsy.py using the official Etsy Open API v3
  (needs my API key): find shops by keyword, pull listing titles, photo counts
  and review counts. Add problems weak_listing_photos (fewer than 5 photos) and
  weak_titles."
- **Local Greek market:** "Add a Greek niche config with language=el and adapt
  prompts so outreach is natural Greek, polite plural (εσείς)."

## Fix-it prompt

```text
Here are 5 leads where the analysis was wrong: <paste lead ids + what is wrong>.
Find why (signal detection, prompt, or validation), fix the root cause, add
these as test fixtures, and show me the before/after output.
```

## Testing checklist (before sending anything)

- [ ] No duplicate leads after 3 runs
- [ ] At least 9 of 10 detected problems are true when you open the website yourself
- [ ] Tier A leads look like businesses you would actually want to pitch
- [ ] Every draft mentions a real, specific problem and reads like a human wrote it
- [ ] Draft mode sends zero emails; the daily cap and suppression list work in `--dry-run`
- [ ] Cost per 100 leads is logged (aim for under 2 EUR with a small model)
- [ ] SPF, DKIM and DMARC pass on the sending domain (check with mail-tester.com)
