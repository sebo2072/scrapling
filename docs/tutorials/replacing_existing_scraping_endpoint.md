# Replacing an Existing App Scraping Endpoint with Scrapling

If your app already has a scraping endpoint, the safest migration path is usually an **adapter-style replacement**: keep your endpoint contract the same, and swap the scraping implementation behind it.

## 1) Pick the fetcher that matches your current scraper

- Use `Fetcher` if your current scraper is mostly HTTP requests (fastest and simplest).
- Use `DynamicFetcher` if pages require JavaScript rendering.
- Use `StealthyFetcher` if you face anti-bot protections and need stronger evasion.

See the fetcher comparison in [Fetchers basics](../fetching/choosing.md).

## 2) Keep your endpoint response schema unchanged

The easiest low-risk migration is:

1. Keep existing request payloads/route paths as-is.
2. Keep existing output JSON keys as-is.
3. Replace only the old scraper function body with Scrapling selectors.

This lets clients switch to the new backend behavior without any API integration changes.

## 3) Minimal synchronous adapter pattern

```python
from dataclasses import dataclass
from scrapling.fetchers import Fetcher

@dataclass
class ScrapeResult:
    title: str | None
    links: list[str]


def scrape_with_scrapling(url: str) -> ScrapeResult:
    page = Fetcher.get(
        url,
        timeout=30,
        retries=3,
        stealthy_headers=True,
        impersonate="chrome",
    )

    title = page.css("title::text").get()
    links = page.css("a::attr(href)").getall()

    return ScrapeResult(title=title, links=links)
```

## 4) Async endpoint integration pattern

If your app already uses async handlers, prefer `AsyncFetcher`:

```python
from scrapling.fetchers import AsyncFetcher

async def scrape_with_scrapling_async(url: str) -> dict:
    page = await AsyncFetcher.get(
        url,
        timeout=30,
        retries=3,
        stealthy_headers=True,
        impersonate="chrome",
    )

    return {
        "status": page.status,
        "title": page.css("title::text").get(),
        "links": page.css("a::attr(href)").getall(),
    }
```

## 5) Add session reuse for higher-throughput endpoints

If your endpoint is called frequently, use `FetcherSession` (or dynamic/stealth sessions) to reuse connections and cookies across requests.

This usually improves performance and reduces repeated setup overhead compared to one-off calls.

## 6) Migrate parser logic incrementally

If your old code uses BeautifulSoup or custom parsing, convert in layers:

1. Replace network call first (`requests`/old browser -> Scrapling fetcher).
2. Replace one extraction block at a time (`find_all`/XPath -> `css`/`xpath` in Scrapling).
3. Keep old assertions/tests per field while swapping internals.

Scrapling supports CSS, XPath, and BeautifulSoup-like APIs on the response object, which makes incremental conversion easier.

## 7) Enable adaptive mode for unstable page structures

If selectors frequently break after site redesigns, configure adaptive parsing:

```python
from scrapling.fetchers import Fetcher

Fetcher.configure(adaptive=True)
```

Then use `auto_save=True` when extracting key elements during successful runs and `adaptive=True` when replaying selectors later.

## 8) Production hardening checklist

- Set strict `timeout`, `retries`, and retry delays.
- Capture and log response status, URL, and selected proxy metadata for debugging.
- Use proxy rotation for high-volume or block-prone targets.
- Separate fetch errors (network/bot blocks) from parse errors (selector misses) in logs/metrics.
- Add circuit-breaker/rate-limit controls at the endpoint layer.

## 9) Rollout strategy to reduce risk

1. **Shadow mode:** Run old scraper and Scrapling in parallel for a subset of traffic.
2. **Compare mode:** Diff key extracted fields and track mismatch rate.
3. **Canary mode:** Route a small percentage fully to Scrapling.
4. **Full cutover:** Remove old scraper once mismatch/error rates stabilize.

## 10) Practical migration examples

- Existing `requests + lxml` endpoint -> `Fetcher` + Scrapling selectors.
- Existing Selenium endpoint -> `DynamicSession` for browser-backed extraction.
- Existing Playwright + anti-bot workaround endpoint -> `StealthySession`.

This approach keeps your API contract stable while letting Scrapling improve robustness, parser ergonomics, and anti-bot handling behind the scenes.
