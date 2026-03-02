---
title: "Building an Instagram to Telegram Bot"
date: 2026-03-01T17:37:00+01:00
lastmod: 2026-03-01T17:37:00+01:00
description: "A practical writeup on building a Telegram bot that automatically downloads and re-uploads Instagram media to a group chat. Covering the stack, the problems, and what actually broke."
summary: "A practical writeup on building a Telegram bot that automatically downloads and re-uploads Instagram media to a group chat. Covering the stack, the problems, and what actually broke."
categories: ["Projects", "Tooling"]
series: [] # If you write several articles on the same subject
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

## Introduction

My Telegram group shares Instagram links constantly. Every time someone posts one,
everyone has to leave Telegram, open Instagram, watch the reel, come back. Annoying enough that I decided to fix it.

Bots that do that already exist. I didn't use them. A bot sitting in your group chat reads every message sent in it. I'm not interested in handling that to code I didn't write and can't read. So I built my own.

---

## How It Works

The flow is straightforward:

1. A message arrive in the group containing an Instagram URL
2. The bot extracts the URL using a regex pattern
3. It attempts to download the media via `yt-dlp`
4. If the post is a photo, it falls back to `instaloader`
5. The file gets uploaded directly to the chat

Everything else in the code handles the edge cases around those five steps.

---

## The Stack

| Library | Role |
| --- | --- |
| `python-telegram-bot` | Telegram API communication |
| `yt-dlp` | Video and reel downloads |
| `instaloader` | Photo post fallback |
| `python-dotenv` | Secret management |

---

## Project Structure

```plaintext
tg-instagram-bot/
├── main.py
├── requirements.txt
├── .env.example
├── .env
└── .gitignore
```

---

## Implementation

### Detecting Instagram URLs

```python
INSTAGRAM_URL_PATTERN = re.compile(
    r"https?://(?:www\.)?instagram\.com/"
    r"(?:p|reel|reels|tv|stories)/[A-Za-z0-9_\-]+/?(?:\?[^\s]*)?"
)

def find_instagram_urls(text: str) -> list[str]:
    return INSTAGRAM_URL_PATTERN.findall(text)
```

The pattern covers posts (`/p/`), reels and stories. Any matching URLs in the message get extracted and processed in order.

### Downloading Media

`yt-dlp` handles videos and reels. For photo posts it explicitly raises:

```
ExtractorError: There is no video in this post
```

That error gets caught and routed to `instaloader` instead:

```python
def download_media(url: str, output_dir: str) -> list[Path]:
    try:
        with yt_dlp.YoutubeDL(build_ydl_options(output_dir)) as ydl:
            info = ydl.extract_info(url, download=True)
            entries = info.get("entries") or [info]
            paths = [resolve_file_path(ydl, entry) for entry in entries if entry]
            return [p for p in paths if p is not None]
    except Exception as exc:
        if not is_photo_post_error(exc):
            raise

        logger.info("Photo post detected, falling back to instaloader")
        return download_images(url, output_dir)
```

### Secret Management

Credentials live in a `.env` file, loaded at runtime by `python-dotenv`:

```plaintext
TELEGRAM_BOT_TOKEN=your_token_here
INSTAGRAM_COOKIES_FILE=/path/to/cookies.txt
ALLOWED_CHAT_IDS=-1001234567890
```

### Running as a Service

The bot runs under systemd on my home server:

```ini
[Unit]
Description=Instagram Telegram Bot
After=network.target

[Service]
WorkingDirectory=/home/user/services/tg-instagram-bot
ExecStart=/home/user/services/tg-instagram-bot/.venv/bin/python main.py
EnvironmentFile=/home/user/services/tg-instagram-bot/.env
Restart=always
User=user

[Install]
WantedBy=multi-user.target
```

Note the `ExecStart` points to the venv's Python binary, not the system one. Using the system Python means your dependencies aren't available.

---

## What Broke

### Python version conflicts

Running Python 3.14 locally. `python-telegram-bot` calls `asyncio.get_event_loop()` which changed behaviour in 3.14 and raises:

```
RuntimeError: There is no current event loop in thread 'MainThread'.
```

Fix:

```python
asyncio.set_event_loop(asyncio.new_event_loop())
```

Called once in `main()` before building the application.

### yt-dlp can't download photos

`yt-dlp` is a video downloader. It raises an explicit error on photo posts rather than silently failing. The dix was catching that specific error string and routing to `instaloader`, which handles both photos and carousels properly.

### Instagram blocks every scraping approach

Before landing on `instaloader`, I tried two other approaches for photos:

**HTML scraping**: `requests` fetches the page, regex looks for image URLs. Instagram serves a JavaScript shell. No data in the HTML.

**oEmbed API**: `instagram.com/oembed/?url=...` should return JSON with a thumbnail URL. It redirects to the same JavaScript shell.

Every approach Instagram had already anticipated. `instaloader` is a library specifically built to maintain this fight, it handles authentication, rate limiting and API changes. The right call was using it rather than trying to out-engineer Meta's platform team.

---

## Notes

- Private posts will fail regardless of cookies unless the authenticated account follows that profile. The bot returns a clean error message rather than crashing.
- `yt-dlp` breaks occasionally when Instagram changes its internals. `pip install -U yt-dlp` fixes it most of the time.
- Telegram bots are limited to 50MB per file via the Bot API. Anything larger gets a warning message instead.

---

## The Code

Full project on GitHub: [GreyCipher-sec/tg-instagram-bot](https://github.com/GreyCipher-sec/tg-instagram-bot)
