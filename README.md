# Amazon Affiliate Deals Userbot

A Telethon **UserBot** that watches your source channels/groups, finds Amazon links (including short links and third-party shorteners), converts them to your **affiliate links**, and reposts each deal to a destination channel with a product title and image.

Settings and sources are stored in **Neon PostgreSQL** and managed live from Telegram — no redeploy needed.

## Features

- Detects Amazon links in source posts: full URLs, `amzn.to` / `a.co` style short links, scheme-less links, and any other shortener that lands on Amazon (redirects are followed, up to 8 hops).
- Rebuilds each link as `https://www.amazon.<tld>/dp/<ASIN>?tag=<your-tag>` and strips old tracking params.
- Multiple deals in one post are split into separate posts, each with its own caption text.
- Product title and image scraped from the Amazon page; falls back to the source image or plain text if scraping fails.
- Optional **posting bot** (`BOT_TOKEN`): posts via a bot with a "🛒 Check Price" button instead of the userbot account.
- Admin control from Telegram (DM / Saved Messages / bot chat): add or remove sources, set tag, destination, header and footer.
- `/status` live-checks every source and flags deleted, left, or migrated chats.
- Safety: SSRF guard on all fetched hosts, redirect/ASIN cache, per-post resolve cap, flood-wait handling, auto-reconnect.

## Project structure

```
main.py           # Userbot, admin commands, deal posting, startup
utils.py          # Link parsing, affiliate conversion, scraping, post formatting
database.py       # asyncpg layer for Neon + in-memory cache
requirements.txt
```

## Requirements

- Python 3.10+
- A Telegram account (the userbot) with `API_ID` / `API_HASH` from <https://my.telegram.org>
- A Neon PostgreSQL database

```bash
pip install -r requirements.txt
```

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `API_ID` | ✅ | Telegram API ID |
| `API_HASH` | ✅ | Telegram API hash |
| `STRING_SESSION` | ✅ | Telethon string session (see Setup) |
| `DATABASE_URL` | ✅ | Neon connection string |
| `ADMIN_IDS` | ❌ | Extra admin user IDs, comma/space separated (the account owner is always admin) |
| `AFFILIATE_TAG` | ❌ | Initial Amazon tag (can be set later with `/set_tag`) |
| `DESTINATION_CHANNEL` | ❌ | Initial destination ID or @username |
| `SOURCE_CHANNEL` | ❌ | Initial source ID(s)/username(s), comma/space separated |
| `BOT_TOKEN` | ❌ | If set, a bot posts deals (with button) and also accepts admin commands in DM |
| `SEND_DELAY` | ❌ | Seconds between posts (default `2`) |
| `MULTI_DEAL_USE_SOURCE_MEDIA` | ❌ | `1` = reuse the source media on every deal of a multi-deal post (default `0`) |
| `EXTRA_SHORT_HOSTS` | ❌ | Extra short-link domains, comma separated |

Env values for destination/tag/sources only **seed** the database; after that, the database is the source of truth.

## Setup

1. **Generate the session string** (run locally):
   ```bash
   python main.py --login
   ```
   Enter phone, code, and 2FA if enabled. Copy the printed `STRING_SESSION` and keep it secret.

2. **Set the environment variables** (locally or on your host).

3. **Run:**
   ```bash
   python main.py
   ```
   Tables are created automatically on first start.

4. **Configure from Telegram** (send to the userbot's Saved Messages, or to the bot if `BOT_TOKEN` is set):
   ```
   /set_tag yourtag-21
   /set_dest @your_channel
   -1001234567890        # send a source channel/group ID to add it
   ```

## Admin commands

| Command | Description |
|---|---|
| `/start` | Welcome + quick setup guide |
| `/help` | Command list |
| `/status` | Active sources (live health check) and destination |
| `/set_tag <tag>` | Amazon affiliate tag |
| `/set_dest <id \| @username>` | Destination channel |
| `/set_header <text>` / `off` | Text above the product title |
| `/set_footer <text>` / `off` | Text below the product title |
| `/del_source <channel_id>` | Remove a source |
| *(send ID / @username / t.me link)* | Add a source |

> The userbot account must be a member of every source and of the destination. If a posting bot is used, it must be an admin in the destination.

## Deploying on Railway

1. Push the 4 files to a GitHub repo and create a Railway project from it.
2. Add the environment variables above in the service settings.
3. Start command: `python main.py`
4. Use Neon's connection string as `DATABASE_URL` (the pooled endpoint works — `channel_binding` is stripped and the statement cache is disabled automatically).

## Post format

```
[header]

🛍️ <product title>

[footer]

👉 Check Price        (link line; with BOT_TOKEN a "🛒 Check Price" button is used instead)
```

## Notes

- Keep `STRING_SESSION` and `DATABASE_URL` private; never commit them.
- Using a userbot is against Telegram's ToS if abused — keep volume and `SEND_DELAY` reasonable.
- Amazon may serve captchas to scrapers; the bot then falls back to the source text/image.
