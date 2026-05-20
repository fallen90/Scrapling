---
name: stealth-scrapling
description: Stealth-first web scraping preset. Wraps the Scrapling skill with anti-detection defaults (real Chrome, WebRTC blocking, canvas noise). Use when scraping sites with aggressive bot detection — Instagram-tier protection, Cloudflare, DataDome, etc. Delegates all scraping to the scrapling-official skill.
version: "0.1.0"
metadata:
  homepage: "https://scrapling.readthedocs.io/en/latest/index.html"
  openclaw:
    emoji: "🥷"
    homepage: "https://scrapling.readthedocs.io/en/latest/index.html"
    requires:
      bins:
        - python3
      anyBins:
        - pip
        - pip3
      skills:
        - scrapling-official
---

# Stealth Scrapling

A stealth-first preset layer over the [Scrapling skill](../Scrapling-Skill/SKILL.md). This skill does not replace Scrapling — it configures it with anti-detection defaults proven against Fingerprint.com's Smart Signals.

**This skill requires the `scrapling-official` skill to be installed.** All scraping is delegated to it.

## When to use this skill

- Target site has aggressive bot detection (Cloudflare, DataDome, PerimeterX, Fingerprint)
- Target site blocks headless browsers or automation tools
- You need to appear as a real human browser
- Default Scrapling commands get blocked or return empty/challenge pages

## Stealth defaults

Every command and code snippet MUST use these defaults unless the user explicitly overrides them:

### CLI commands

Always use `stealthy-fetch` (never `get` or `fetch`) with these flags:

```bash
scrapling extract stealthy-fetch "<URL>" output.md \
  --real-chrome \
  --block-webrtc \
  --hide-canvas \
  --network-idle \
  --ai-targeted \
  --wait 5000
```

### Python code

Always use `StealthyFetcher` or `StealthySession` with these parameters:

```python
from scrapling.fetchers import StealthyFetcher, StealthySession

# One-off fetch
page = StealthyFetcher.fetch(
    url,
    headless=True,
    real_chrome=True,
    block_webrtc=True,
    hide_canvas=True,
    network_idle=True,
    wait=5000,
)

# Session (multi-page flows, maintains cookies)
with StealthySession(
    headless=True,
    real_chrome=True,
    block_webrtc=True,
    hide_canvas=True,
) as session:
    page = session.fetch(url, network_idle=True, wait=5000)
```

### Async code

```python
from scrapling.fetchers import StealthyFetcher, AsyncStealthySession

# One-off async
page = await StealthyFetcher.async_fetch(
    url,
    headless=True,
    real_chrome=True,
    block_webrtc=True,
    hide_canvas=True,
    network_idle=True,
    wait=5000,
)

# Async session
async with AsyncStealthySession(
    headless=True,
    real_chrome=True,
    block_webrtc=True,
    hide_canvas=True,
    max_pages=3,
) as session:
    page = await session.fetch(url, network_idle=True, wait=5000)
```

## Benchmark context

These defaults were validated against Fingerprint.com's Browser Smart Signals (2026-05-20):

| Signal | DynamicFetcher | StealthyFetcher | **Stealth + real_chrome** |
|---|---|---|---|
| Suspect Score | 29 | 22 | **14** |
| Bot Detection | WebDriver (medium) | not_detected | **not_detected** |
| Developer Tools | true | false | **false** |
| Tampering | false (high) | true (low) | **false (high)** |
| Anti-Detect Browser | false | true | **false** |
| VM Score | 0.106 | 0.056 | **0.055** |

`real_chrome=True` is the key — it uses the host's actual Chrome binary instead of patched Chromium, so there are no stealth patches for fingerprinting services to detect.

## Escalation ladder

If a site still blocks you with these defaults:

1. **Add `--solve-cloudflare`** — handles Cloudflare challenge pages
2. **Add proxy rotation** — `--proxy "http://user:pass@host:port"`
3. **Add `--wait-selector ".target-element"`** — wait for specific content instead of relying on network idle
4. **Increase wait** — `--wait 10000` for heavy JS sites
5. **Try `--no-headless`** — some sites detect headless even with real Chrome

## Spiders

For crawls, configure the spider to use a stealth session:

```python
from scrapling.spiders import Spider, Request, Response
from scrapling.fetchers import AsyncStealthySession

class StealthSpider(Spider):
    name = "stealth_crawl"
    start_urls = ["https://target.com"]
    concurrent_requests = 3
    download_delay = 2

    def configure_sessions(self, manager):
        manager.add("stealth", AsyncStealthySession(
            headless=True,
            real_chrome=True,
            block_webrtc=True,
            hide_canvas=True,
        ), lazy=True)

    async def parse(self, response: Response):
        yield {"title": response.css("h1::text").get()}
```

## What this skill does NOT do

- It does not replace or modify the Scrapling skill
- It does not add new scraping capabilities
- For Scrapling's full API, parsing, selectors, XHR capture, etc. — refer to the `scrapling-official` skill

## Guardrails

All guardrails from the Scrapling skill apply. Additionally:
- `real_chrome=True` launches your actual Chrome — close other Chrome windows first to avoid profile conflicts
- Higher stealth = slower. Don't use this for bulk scraping where speed matters more than evasion
- Respect rate limits. Stealth doesn't mean invisible — rapid requests from one IP still get flagged
