# ig-carousel

Local-first Instagram carousel automation for AI/Claude education content.

**Pipeline:** drop sources in `inbox/` → Claude drafts a 7-slide carousel → renders in your brand template → emails you a digest → you tap Approve on your phone → app posts to Instagram at the next research-optimal slot.

The only manual step is approval. Everything else is automatic.

## What it does

- **Watches** `~/Desktop/Instagram/inbox/` for PDFs, screenshots, `.md`/`.txt` notes, `.url` files, or text files containing URLs.
- **Generates** carousels via the `claude` CLI (uses your Max plan; no API key needed).
- **Renders** slides via Playwright at 1080×1350 portrait in a custom Anthropic-inspired warm-minimal template (cream / navy / terracotta).
- **Emails** a digest with inline previews + ✅ / ✏️ / ❌ buttons (Gmail SMTP).
- **Polls** Gmail (IMAP) every 5 min for your reply.
- **Posts** approved carousels via the Meta Graph API at peak ET windows (Tue/Wed/Thu/Sat 14:00 ET by default).
- **Stories**: posts a daily 9:16 reuse of the latest carousel hook at 10:00 ET.
- **Refreshes** the long-lived access token monthly.

## Cadence (research-backed)

- 4 carousels/week (Tue/Wed/Thu/Sat) — sweet spot for IG growth (Genviral 2M-post study, 2026).
- Daily Stories — algorithm signal of activity.
- 3–5 hashtags per post (IG enforces a 5-tag cap as of Dec 2025).
- 4:5 portrait, not 1:1 square (20% more screen real estate).

## Prerequisites

- macOS / Linux
- **Python 3.11+ recommended** (`brew install python@3.11`). 3.9 still works but boto3 drops support on 2026-04-29.
- Claude Code CLI (already installed if you have a Max plan)
- Instagram **Business or Creator** account, linked to a Facebook Page
- A Meta for Developers app with `instagram_content_publish` permission
- A GitHub account (a public repo is used for image hosting)
- A Gmail account with an [App Password](https://myaccount.google.com/apppasswords)

## Install

```bash
cd /Users/ani/Desktop/Instagram

# Create a virtualenv (recommended)
python3.11 -m venv .venv      # falls back to python3 if 3.11 not installed
source .venv/bin/activate

# Install the app
pip install -e .

# One-time: download the Chromium binary used for slide rendering
playwright install chromium

# Run the interactive setup wizard
ig setup
```

The wizard walks through each secret and validates every connection. If a check fails, fix and re-run `ig setup`.

## Daily workflow

1. **Drop sources** into `inbox/` whenever you want to feed the queue.
   - PDFs, `.md`, `.txt`, `.png`/`.jpg` screenshots, `.url` files (or text files with URLs in them).
2. **Receive a digest email** at 8:00 ET on post days.
3. **Tap a button on your phone:**
   - **✅ Approve** — sends an empty reply with `[IG-APPROVE-<token>]` subject.
   - **✏️ Request edits** — opens a reply where you write notes; app regenerates the post.
   - **❌ Reject** — discards the post.
4. **App posts** at the scheduled slot. You get a log line in `logs/ig.log`.

## CLI reference

```bash
ig setup                      # interactive setup wizard
ig doctor                     # validate all connections
ig add <file_or_url>          # add a source manually
ig status                     # queue + recent posts + unused sources
ig dryrun                     # generate + render + email digest, no publish
ig preview <post_id>          # open rendered slides in macOS Preview
ig regen <post_id>            # regenerate (uses edit_notes if any)
ig publish <post_id>          # manually publish (bypasses scheduler)
ig poll-approvals             # one-shot IMAP poll
ig send-digest                # send digest of current drafts
ig refresh-token              # refresh Meta long-lived token
ig run                        # start the scheduler (foreground)
```

## Verification (smoke-test path)

1. `ig doctor` — all five checks should be OK.
2. Drop one Anthropic blog URL into `inbox/url.txt`:
   ```
   https://www.anthropic.com/news/some-post
   ```
   Within ~10 seconds you should see a `sources` row appear (`ig status` to verify).
3. `ig dryrun` — generates a draft, renders 7 slides, uploads to R2, emails you.
4. Open the email on your phone — confirm the previews render and buttons work.
5. Reply with `[IG-APPROVE-<token>]` (the buttons do this for you).
6. `ig poll-approvals` — should flip status from `pending_approval` to `approved`.
7. `ig publish <post_id>` — first real post goes live.
8. `ig run` — start the scheduler. It will fire on the next scheduled slot.

## Brand template

Edit `ig/templates/brand.css` to tune the visual. v1 ships with:

- Background `#F5F0E8` (warm cream)
- Body text `#1A2332` (deep navy)
- Accent `#C8553D` (terracotta)
- Headlines: Source Serif Pro
- Body: Inter
- Code: JetBrains Mono

All three Google Fonts load at render time via Playwright (no font installation needed).

## Layout

```
Instagram/
├── inbox/              # drop sources here
├── archive/            # processed sources land here
├── previews/           # rendered slide JPEGs (post-00001/, ...)
├── logs/               # ig.log (rotating)
├── ig.db               # SQLite
├── .env                # secrets (created by `ig setup`)
└── ig/                 # source code
    ├── cli.py          # entrypoint
    ├── config.py       # env + paths
    ├── db.py           # SQLite schema + queries
    ├── schemas.py      # pydantic models for Claude output
    ├── claude_runner.py  # `claude -p` wrapper
    ├── ingest.py       # watchdog + PDF/URL/image parsers
    ├── generate.py     # carousel generation prompt + flow
    ├── render.py       # Playwright HTML→JPEG
    ├── templates/      # Jinja2 + brand.css
    ├── host.py         # Cloudflare R2 upload
    ├── digest.py       # email composition + Gmail SMTP
    ├── approve.py      # IMAP poll loop
    ├── post.py         # Meta Graph API carousel publish
    ├── stories.py      # daily 9:16 story
    ├── token.py        # long-lived token refresh
    ├── scheduler.py    # APScheduler cron jobs
    └── setup_wizard.py # interactive .env builder
```

## Realistic growth expectations

With consistent posting at peak ET times, expect:
- **Week 1–2:** 30–80 followers (algorithm cold-start)
- **Week 3–4:** 80–200 followers/week
- **Month 2+:** potential 200–500/week if a post hits Explore
- **1k followers:** realistic in 6–10 weeks; faster only with paid ads, collabs, or a viral hit

The app is built for compounding, not a sprint.

## Out of scope (v2 candidates)

- Reels generation (video script + TTS + b-roll)
- Comment/DM auto-reply
- Cross-posting to LinkedIn/Twitter/Threads
- Meta Ads API integration
- Engagement analytics dashboard
- A/B testing of hook variants

## Troubleshooting

- **Claude CLI not found:** `which claude` should print a path. If not, ensure your shell loads `~/.local/bin` or wherever `claude` lives. Edit `CLAUDE_BIN=` in `.env` to a full path if needed.
- **`claude -p` returns malformed JSON:** the generator retries once with stricter instructions. Persistent failures indicate a prompt issue — check `logs/ig.log`.
- **Meta `400` on publish:** usually an image URL not publicly fetchable (check R2 public-URL setting) or token expired (`ig refresh-token`).
- **No digest email:** check `ig doctor`; verify Gmail App Password (not your account password); check spam folder.
- **Approval reply not registering:** subject line must contain `[IG-APPROVE-<token>]` exactly; tap the button (don't compose a fresh email). Run `ig poll-approvals` to force a check.
