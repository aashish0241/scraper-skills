---
name: scraper-antibot
description: Handle bot protection while building or fixing a scraper. Research the site, detect the vendor (Cloudflare, Turnstile, PerimeterX, DataDome, reCAPTCHA, hCaptcha), pick the cheapest working client, and escalate from 429 (proxy rotation) to 403/CAPTCHA (TLS impersonation, mobile API, cookies, residential proxy, Camoufox, solver service). Use when a request is blocked, rate limited, or shows a challenge page.
---

# Scraper Anti-Bot Handling

Use during research (before writing a spider) or when an existing spider starts getting 429, 403, or a CAPTCHA. Only scrape what is permitted; ask the user before using paid proxies or solver services.

Full detail: `references/how-to-start-scraping.md` (workflow) and `references/antibot-overview.md` (per-system handling and approaches used in the internal scraper).

## 1. Research first (no spider code yet)

1. Find the cheapest data source, in order: JSON/mobile API, embedded JSON (`__NEXT_DATA__`, JSON-LD), sitemap, plain HTML, Camoufox as last resort.
2. Check if JS rendering is needed: is the data in `curl` output / view-source? Does an XHR return it? Only use a browser if the data exists solely in the rendered DOM.
3. Identify the vendor from status, headers, cookies and body:
   - Cloudflare: `cf-ray`, `__cf_bm`/`cf_clearance`, "Just a moment"
   - PerimeterX: `_px*`, `X-Px-*`, press-and-hold
   - DataDome: `datadome` cookie, `captcha-delivery.com`
   - Akamai: `_abck`, `bm_sz`; Imperva: `incap_ses_*`; Kasada: `x-kpsdk-*`
   - Only 429/`Retry-After`: plain rate limiting
4. Run the client matrix on the same request and keep the cheapest client that returns real data on 5+ requests in a row: `requests`, `httpx`, `curl_cffi` (`chrome`, `chrome131`, `safari18_0_ios`, `firefox`), `hrequests`, mobile app API, cookie bootstrap, Camoufox + cookie reuse, proxy.
5. Record vendor, matrix results and the winning recipe in `RESEARCH.md` (cookie/token names only, never values).

## 2. Escalation ladder (move down only when the level fails)

| Level | Symptom | Action |
|-------|---------|--------|
| 0 | any | browser-like headers, fresh session, `DOWNLOAD_DELAY`, backoff, honor `Retry-After` |
| 1 | **429** | lower concurrency, retry with backoff, **rotate proxies** (datacenter is fine) |
| 2 | **403 / CAPTCHA** | `curl_cffi` impersonation, then mobile app API or app User-Agent, then cookie bootstrap, then **residential proxy**, then cool-down with a fresh session and IP |
| 3 | challenge persists | Camoufox clears it, copy cookies + `navigator.userAgent` to a plain client, re-solve on 403/503 |
| 4 | CAPTCHA persists | solver service (CapSolver, 2Captcha, Anti-Captcha) after asking the user |

Rules: cap retries (2 to 3 session refreshes) then log and skip; keep IP, UA and TLS fingerprint the same between the browser step and the plain-client steps; detect challenge pages (`px-captcha`, "been denied", "just a moment", `captcha-delivery`) because some blocks return 200.

## 3. Challenge techniques

- Cloudflare/Turnstile: Camoufox, wait for `cf_clearance`, click the checkbox in the `challenges.cloudflare.com` frame, or `camoufox_captcha`; token from a solver service.
- PerimeterX: iOS app `X-Px-*` headers with `safari18_0_ios`; auto press-and-hold on `#px-captcha` with a manual fallback.
- DataDome: app User-Agent, cookie bootstrap + retry, residential proxy + humanized browser.
- reCAPTCHA v2/v3, hCaptcha, image CAPTCHA: solver service.
- Camoufox settings: `humanize=True`, `geoip=True`, `headless="virtual"` if headless is flagged.

## Rules

- Prior decisions win (for example a site that already uses its mobile API).
- If no client works, stop and ask the user; list what was tried and the options.
- Never commit real cookies, tokens, or client IDs.
