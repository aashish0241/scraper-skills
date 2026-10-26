# How to Start Scraping a New Website

A step-by-step workflow for a new target: research the site first, pick the cheapest method that works, and escalate only when blocked. It follows the approach used in `aashish_webscraping_team` (Scrapy + `curl_cffi` + Camoufox) and the playbook in `.claude/skills/scraper-research/SKILL.md`.

> Only scrape what you are allowed to. Check the site's Terms of Service and applicable law, and prefer an official API where one exists.

## Contents

1. [Golden rule](#1-golden-rule)
2. [Phase 1: Research the website](#2-phase-1-research-the-website)
3. [Phase 2: Detect the security vendor](#3-phase-2-detect-the-security-vendor)
4. [Phase 3: Pick the request client](#4-phase-3-pick-the-request-client)
5. [Phase 4: Escalation ladder (429, 403, CAPTCHA)](#5-phase-4-escalation-ladder)
6. [CAPTCHA and challenge techniques](#6-captcha-and-challenge-techniques)
7. [Record the research](#7-record-the-research)
8. [Cheat sheet](#8-cheat-sheet)

---

## 1. Golden rule

**Use the cheapest method that returns real data on 5+ requests in a row.** A browser is the most expensive option (slow, heavy, easier to flag). Go down this list and stop at the first that works:

1. Public or internal JSON API (XHR/fetch in the network tab, mobile app API, GraphQL)
2. Embedded JSON in the HTML (`__NEXT_DATA__`, JSON-LD `application/ld+json`, `window.__STATE__`)
3. Sitemaps and `robots.txt` (for URL discovery)
4. Plain HTML parsed with CSS/XPath
5. Browser automation (Camoufox), last resort

---

## 2. Phase 1: Research the website

Do not write spider code yet. Answer these questions first.

### 2.1 Understand what you need
- Which scraper type: product URLs, sitemap, product detail, reviews, profile?
- Which fields are required? Map each one to a source later.
- Is there an existing spider for this site or a similar platform (Shopify, Magento, Next.js)? Reuse its pattern.

### 2.2 Open the site like a user
1. Open the page in Chrome with DevTools, **Network** tab, filter **Fetch/XHR**. Reload and click through (pagination, filters, "load more").
2. Look for JSON responses that already contain the data (product list, reviews, prices).
3. Note third-party widgets: reviews are often Yotpo, Bazaarvoice or PowerReviews and have their own public API.

### 2.3 Does the site need JavaScript rendering?

| Test | How | Meaning |
|------|-----|---------|
| View source | `Ctrl+U` or `curl -s <url>`, search for a product name or price | Present: server-rendered, no browser needed |
| Disable JS | DevTools, Command menu, "Disable JavaScript", reload | Data still visible: no rendering needed |
| Embedded JSON | Search the source for `__NEXT_DATA__`, `application/ld+json`, `window.__`, `dataLayer` | Data is in the HTML, parse the JSON |
| XHR after load | Data absent in source but a Network request returns it | Call that API directly, skip the browser |
| Data only in the rendered DOM | Absent from source and from every request | Browser rendering is needed (Camoufox) |

Quick check from the terminal:

```bash
curl -sI "https://example.com/product/1" | head -20          # status + headers
curl -s  "https://example.com/product/1" | grep -c "price"    # is the data in raw HTML?
curl -s  "https://example.com/product/1" | grep -o '__NEXT_DATA__' | head -1
```

### 2.4 Check discovery and volume
- `https://site.com/robots.txt` and `sitemap.xml` for the URL list.
- How pagination works, page size, total count, any cap.
- Test at least **5 URLs**: normal, on sale, out of stock, multi-variant, no reviews, and an invalid 404.

---

## 3. Phase 2: Detect the security vendor

Send a normal request and inspect status, headers, cookies and the body.

| Vendor | Fingerprint |
|--------|-------------|
| **Cloudflare / Turnstile** | `cf-ray` header, `__cf_bm` / `cf_clearance` cookies, "Just a moment..." page, 403/503 challenge |
| **PerimeterX / HUMAN** | `_px*` cookies, `X-Px-*` headers, "Press & Hold" page, `px-captcha` |
| **DataDome** | `datadome` cookie, `captcha-delivery.com` iframe |
| **Akamai** | `_abck`, `bm_sz`, `ak_bmsc` cookies, "Access Denied" with a reference id |
| **Imperva / Incapsula** | `incap_ses_*`, `visid_incap_*` cookies |
| **Kasada** | `x-kpsdk-*` headers |
| **reCAPTCHA / hCaptcha** | Captcha widget on the page |
| **Rate limiting only** | 429 or `Retry-After`, no challenge |

Which codes mean what:

| Response | Usually means |
|----------|---------------|
| **200 with real data** | Fine |
| **200 with a challenge page** | Soft block: check the body for "captcha", "denied", "Just a moment" |
| **429** | Rate limit, too many requests from this IP |
| **403** | Fingerprint, IP reputation or challenge block |
| **503 + Cloudflare page** | Cloudflare JS challenge |
| **401 / empty data** | Missing cookie or token, try a cookie bootstrap |

---

## 4. Phase 3: Pick the request client

Run the **same request** (same URL, headers, cookies) with each client and record status, body size and whether real data came back.

| # | Client | Notes |
|---|--------|-------|
| 1 | `requests` | Baseline with a browser User-Agent and full headers |
| 2 | `httpx` (HTTP/1.1 and `http2=True`) | Different TLS/HTTP2 fingerprint |
| 3 | `curl_cffi` with `chrome`, `chrome131`, `safari`, `safari18_0_ios`, `firefox` | Fixes TLS/JA3 blocks |
| 4 | `hrequests`, `tls_client`, `rnet` | Other fingerprints |
| 5 | Scrapy native downloader | What the spider uses unless another client is wired in |
| 6 | **Mobile app API** (iOS/Android headers or app User-Agent) | Web blocked but the app API is open |
| 7 | Camoufox, then reuse its cookies and UA in a plain client | JS challenge or required cookie |
| 7a | **Cookie bootstrap**: one plain request to the domain, keep `Set-Cookie`, send on later requests | Site sets session/anti-bot cookies on first visit |
| 8 | Proxy, lower concurrency, delays | Geo-lock or 429 after a few requests |

Winners we have seen: `curl_cffi` fixed JB Hi-Fi (429 to non-browser TLS); the Kogan app User-Agent passes DataDome; Adore Beauty's iOS app headers pass PerimeterX. Always try these before a browser.

Also vary headers (copy a real browser request), cookies, `Referer`/`Origin` and HTTP version.

If **no client works**, stop. List what was tried and the options (proxy, captcha solver, tokens, another data source), and ask before spending on paid services.

---

## 5. Phase 4: Escalation ladder

Move down one level only when the current one fails. Always keep the ladder in this order, because each step costs more.

### Level 0: Clean and polite baseline
- Browser-like headers, a realistic User-Agent, a fresh session.
- Lower concurrency and add `DOWNLOAD_DELAY` (RH needed 4 s between requests, otherwise pagination returned 403).
- Honor `Retry-After`. Use exponential backoff with jitter.

### Level 1: 429 Too Many Requests, rotate proxies

429 means the IP is rate limited. Fix: slow down, then **rotate the proxy** so requests spread across IPs.

1. Lower concurrency and add a delay. Often enough on its own.
2. Retry on 429 with backoff (`RETRY_HTTP_CODES = [429, 500, 502, 503, 504]`).
3. Rotate proxies: a new IP per request or per small batch of requests.
4. Datacenter proxies are enough for simple rate limits. Keep a pool and drop IPs that keep failing.

```python
# Scrapy: per-request proxy rotation (sketch)
import random

PROXIES = ["http://user:pass@proxy1:8000", "http://user:pass@proxy2:8000"]

class RotatingProxyMiddleware:
    def process_request(self, request, spider):
        request.meta["proxy"] = random.choice(PROXIES)

    def process_response(self, request, response, spider):
        if response.status == 429:
            retry = request.copy()
            retry.dont_filter = True
            retry.meta["proxy"] = random.choice(PROXIES)  # new IP on retry
            return retry
        return response
```

```python
# settings.py
DOWNLOADER_MIDDLEWARES = {"internalprogram.middlewares.RotatingProxyMiddleware": 543}
RETRY_HTTP_CODES = [429, 500, 502, 503, 504]
RETRY_TIMES = 3
```

> Status: `aashish_webscraping_team` does not configure proxies today. Add this middleware when a site returns 429 even after throttling. Ask before buying a paid proxy plan.

### Level 2: 403 Forbidden or a CAPTCHA, fix the fingerprint, then use a stronger proxy

403 or a CAPTCHA means the site judged the client or IP as a bot, so more of the same IP will not help. Work through these in order:

1. **Fix the TLS fingerprint**: switch to `curl_cffi` impersonation (`chrome`, `safari18_0_ios`, `firefox`).
2. **Try the mobile app API or app User-Agent** (Adore Beauty, Kogan).
3. **Bootstrap cookies**: hit the home page first and keep `datadome`, `_abck`, `bm_sz`, `_px*` cookies (TripAdvisor, RH).
4. **Switch to residential proxies** if the IP reputation is the problem. Datacenter ranges are often blocked outright by Cloudflare, DataDome and PerimeterX; residential IPs look like real users.
5. **Wait and cool down** on a block: stop that session for a period (for example 1 to 5 minutes), then retry with a fresh session and new IP. Do not hammer a blocked IP.
6. **Use a real browser** (Camoufox) to pass the challenge and get clearance cookies (see Level 3).

Rule of thumb for time: if a block repeats after a cool-down and a new session, move to the next level instead of looping. Cap retries (we use 2 to 3 session refreshes) and then log and skip.

### Level 3: Browser to pass the challenge, then reuse the cookies

1. Launch **Camoufox** (`headless="virtual"` or a visible window if headless is flagged).
2. Open the site, solve or wait out the challenge (see section 6).
3. Read `page.context.cookies()` (for example `cf_clearance`) and `navigator.userAgent`.
4. Send those cookies and that exact UA to a fast client (`curl_cffi` with `impersonate="firefox"`, `httpx`, or Scrapy).
5. On 403/503 again, re-run the browser step to refresh clearance. Cap the refreshes.
6. Keep the **same IP, UA and TLS fingerprint** between the browser step and the plain-client steps. Vendors bind cookies to them.

Examples in the repo: `himalayas` (`cf_clearance`), `upwork` (cookies + auth token, then GraphQL), `casio` (cookies + UA, then GraphQL), `trustpilot`.

### Level 4: Solver services (paid, ask first)

If an image CAPTCHA, reCAPTCHA v2/v3, hCaptcha or Turnstile still appears and the browser cannot clear it, use a solver provider: **CapSolver**, **2Captcha** or **Anti-Captcha**. Not used in any spider yet.

```
Level 0  headers, delay, backoff
Level 1  429  ─► slow down ─► rotate proxies
Level 2  403 / CAPTCHA ─► TLS fingerprint ─► mobile API ─► cookies ─► residential proxy ─► cool-down
Level 3  Camoufox clears challenge ─► reuse cookies in a plain client
Level 4  CAPTCHA solver service
```

---

## 6. CAPTCHA and challenge techniques

| Challenge | Technique | Used in repo |
|-----------|-----------|--------------|
| **Cloudflare "Just a moment"** | Camoufox, wait for `cf_clearance`, click the Turnstile checkbox in the `challenges.cloudflare.com` frame, then reuse cookies | `himalayas`, `autodoc` |
| **Cloudflare, library solver** | `camoufox_captcha.solve_captcha(captcha_type="cloudflare")` with `disable_coop=True` | `comfortiva` reviews |
| **Turnstile token** | Solver service (CapSolver / 2Captcha) returns a token to submit | not yet |
| **PerimeterX press-and-hold** | Auto press-and-hold on `#px-captcha` in the PX frame; fall back to a visible window for a human; refresh the session on 403 | `skyscanner`, `yeti` |
| **PerimeterX, app traffic** | iOS app `X-Px-*` headers with `safari18_0_ios` impersonation | `adorebeauty` |
| **DataDome** | App User-Agent, or cookie bootstrap and retry, or residential proxy + humanized browser | `kogan`, `tripadvisor`, `rh` |
| **reCAPTCHA v2 / v3** | Solver service; v3 needs a real browser environment for a good score | not yet |
| **hCaptcha** | Solver service (human workers) | not yet |
| **Text / image CAPTCHA** | Solver service (OCR or human) | not yet |
| **Detect and skip** | Detect the CAPTCHA page and skip the URL instead of looping | `elkay` |

General anti-detection settings for Camoufox:

- `humanize=True` for human-like cursor movement.
- `geoip=True` so locale and timezone match the IP.
- `headless="virtual"` (virtual display) when plain headless is flagged. Skyscanner needed this fix.
- Random delays, scrolling, realistic window size, and a consistent fingerprint per session.
- Do not send requests faster than a person could.

Block detection helper (keep it in the spider):

```python
BLOCK_MARKERS = ("px-captcha", "been denied", "just a moment", "captcha-delivery", "access denied")

def is_blocked(response) -> bool:
    if response.status in (403, 429, 503):
        return True
    body = response.text[:5000].lower()
    return any(m in body for m in BLOCK_MARKERS)
```

---

## 7. Record the research

Write the findings in `RESEARCH.md` next to the spider (`internalprogram/src/spiders/<site>/`) before planning or coding:

- Mode: new or fix
- Data source chosen and why the others were rejected
- Request recipe: client, profile, headers, cookies (names only, never real values), lifetime and refresh rule
- Anti-bot vendor and the client matrix results (status, size, real data yes/no)
- Required fields and the field map (field, source, path, example)
- Spider inputs (`keywords`, `product_urls` or `urls`, each with one working default)
- Edge cases and risks

Then plan, implement and QA. Never commit real cookie values, tokens or client IDs.

---

## 8. Cheat sheet

| Symptom | First action | Next action |
|---------|--------------|-------------|
| Data is in raw HTML or embedded JSON | Plain request, parse JSON | none |
| Data comes from an XHR | Call that API directly | Add cookie bootstrap if 401/empty |
| **429** | Slow down, backoff | **Rotate proxies** |
| 403 on plain `requests` | `curl_cffi` impersonation | Mobile API / app UA |
| 403 with `datadome`, `_px`, `cf_` cookies | Cookie bootstrap | Residential proxy, then Camoufox |
| Cloudflare "Just a moment" | Camoufox + Turnstile click | Reuse `cf_clearance` in a plain client |
| PerimeterX press-and-hold | Mobile app headers | Camoufox auto press-and-hold, manual fallback |
| reCAPTCHA / hCaptcha / image CAPTCHA | Camoufox, check if avoidable | Solver service (ask first) |
| Block repeats after a new session and IP | Cool down, cap retries | Log, skip, ask the user |
