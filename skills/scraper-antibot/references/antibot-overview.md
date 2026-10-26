# Anti-Bot & CAPTCHA Handling Guide

How we handle the anti-bot and CAPTCHA systems we run into while automating web access.

> **Use responsibly.** Only automate sites you are permitted to access. Respect each site's Terms of Service, `robots.txt`, and applicable law. Prefer an official API or data partnership where one exists.

> New to a target? Start with [how-to-start-scraping.md](how-to-start-scraping.md): research the site, detect the security vendor, pick a client, and escalate from 429 to 403/CAPTCHA.

## Table of Contents

- [Supported Anti-Bot Systems](#supported-anti-bot-systems)
- [Standard Escalation Rules](#standard-escalation-rules)
- [Per-System Handling](#per-system-handling)
- [Solver Services](#solver-services)
- [Browser / WebDriver Options](#browser--webdriver-options)
- [Approaches Tried in the Internal Scraper](#approaches-tried-in-the-internal-scraper)
- [Recommended Stack](#recommended-stack)
- [Quick Decision Table](#quick-decision-table)

---

## Supported Anti-Bot Systems

| # | System | Type |
|---|--------|------|
| 1 | Cloudflare & Turnstile | IP reputation, fingerprinting, JS and behavioral challenges |
| 2 | Google reCAPTCHA (v2, v3) | Image challenge (v2), silent behavior score (v3) |
| 3 | hCaptcha | Image classification |
| 4 | Text / Image CAPTCHA | Distorted text or image |
| 5 | DataDome | CDN-level behavioral biometrics and ML anomaly detection |
| 6 | PerimeterX (HUMAN Security) | JS sensor data and behavioral analysis |

---

## Standard Escalation Rules

Apply these in order. Move to the next step only when the current one still gets blocked.

### 1. Clear cookies and retry
- Remove cookies, cache, and session/local storage.
- Retry the request with a clean session.
- A CAPTCHA is often triggered by bad or flagged cookies.

### 2. Rotate proxies
- If still blocked, rotate the IP address.
- Change IPs frequently to avoid rate limits and IP/fingerprint tracking.
- Datacenter proxies are enough for simple protections.

### 3. Switch to residential proxies
- If normal proxies fail, use residential proxies.
- Residential IPs look like real users and work better against stronger protections such as Cloudflare.

### 4. Use a different WebDriver
- Some sites detect common Selenium setups.
- Switch driver or automation framework to reduce detection.
- Dynamic content usually needs a real browser driver anyway.

### 5. Use a stealth-oriented browser tool
- **Camoufox**: strong against Cloudflare and other anti-bot systems.
- **NoDriver**: reduces WebDriver detection.
- **Playwright**: better stealth and modern automation support.

### 6. Use CDP (Chrome DevTools Protocol)
- If WebDriver detection persists, control the browser over CDP.
- CDP is lower level and tends to look more like a normal browser session.

### 7. Additional techniques
- Randomize browser fingerprints.
- Mimic human behavior: mouse movement, delays, scrolling.
- Rotate user agents and headers.
- Match the TLS and browser fingerprint to the claimed user agent.
- Avoid sending too many requests too quickly.

### 8. CAPTCHA-solving services
If an image CAPTCHA or reCAPTCHA v2/v3 still appears, use a solver provider. They handle:
- Image CAPTCHA
- hCaptcha
- reCAPTCHA v2 / v3
- Turnstile

### 9. Final goal
Combine all of the following to maximize the success rate:

- Clean sessions
- Proxy rotation
- Residential IPs
- Stealth browsers
- Human-like behavior
- CAPTCHA-solving services

---

## Per-System Handling

### 1. Cloudflare & Turnstile

**Cloudflare** uses IP reputation, browser fingerprinting, and JS challenges.
- **Approach:** Use **Camoufox**, which spoofs the browser fingerprint to pass challenges. Pair with good (residential) IPs.

**Turnstile** is Cloudflare's invisible behavioral challenge.
- **Approach:** Use **CapSolver** or **2Captcha** to generate a valid token, then submit it with the request.

### 2. Google reCAPTCHA (v2 / v3)

**v2** shows image challenges and tracks behavior.
- **Approach:** **CapSolver** or **2Captcha** (human or ML solvers).

**v3** runs silently and returns a behavior-based score.
- **Approach:** **CapSolver**, using real browser environments to achieve a good score.

### 3. hCaptcha

Image classification tasks; a privacy-focused alternative to reCAPTCHA.
- **Approach:** **CapSolver** or **2Captcha** via human solver networks.

### 4. Text / Image CAPTCHA

A distorted text or image the user must read and type.
- **Approach:** **CapSolver** or **2Captcha** using OCR and human solvers.

### 5. DataDome

Behavioral biometrics and ML anomaly detection at the CDN level.
- **Approach:** Humanized browser automation plus residential proxy rotation.

### 6. PerimeterX (HUMAN Security)

JS sensor data collection and behavioral analysis.
- **Approach:** Realistic browser fingerprints plus behavioral mimicry.

---

## Solver Services

| Service | Model | Notes |
|---------|-------|-------|
| **CapSolver** | Automated, ML-based | Fast; supports most CAPTCHA types |
| **2Captcha** | Human worker network | Highly accurate |
| **Anti-Captcha** | Human + automated hybrid | Balanced speed and accuracy |

---

## Browser / WebDriver Options

- **Playwright**
- **Camoufox**
- **Ulixee**
- **NoDriver**
- **Selenium**
- **CloakBrowser**

---

## Approaches Tried in the Internal Scraper

Source: `jyaba/aashish_webscraping_team/internalprogram/src/spiders/` (Scrapy project, 150+ spiders). This section was built by reading the spider code and the repo's own research playbook (`.claude/skills/scraper-research/SKILL.md`). It reflects what the code does today, not a log of every failed attempt, because failed experiments are mostly not recorded in the repo.

### The repo's own playbook (cheapest first)

1. Public or internal JSON API (XHR, mobile app API, GraphQL).
2. Embedded JSON in the HTML (`__NEXT_DATA__`, JSON-LD, `window.__STATE__`).
3. Sitemaps and `robots.txt` for URL discovery.
4. Plain HTML via CSS/XPath.
5. Camoufox browser automation, as a last resort.

For each target we run a **request-client matrix** and keep the cheapest client that returns real data on 5+ requests in a row:

`requests` → `httpx` (HTTP/1.1 and HTTP/2) → `curl_cffi` (profiles `chrome`, `chrome131`, `safari`, `safari18_0_ios`, `firefox`) → `tls_client` / `rnet` / `aiohttp` → Scrapy native → mobile app API → Camoufox, then reuse its cookies in a plain client → cookie bootstrap (one plain request, keep `Set-Cookie`) → proxy and lower concurrency.

Rules in the playbook: if no client works, stop and ask; always ask before using paid services (proxies, captcha solvers); prior decisions win (for example Adore Beauty uses the mobile API, not a browser).

### Approaches in use, by technique

| Approach | What it does | Spiders using it |
|----------|--------------|------------------|
| **Mobile app API / app User-Agent** | Web is blocked, the app endpoint is open. Send the app's headers or UA. | `adorebeauty` (iOS app headers incl. `X-Px-*` for PerimeterX + `safari18_0_ios`), `kogan` (Kogan app UA gets past DataDome), `yandex` (Yandex Maps Android UA), `acmetools` (mobile UA), `apartments` (mobile UA) |
| **`curl_cffi` / `scrapy-impersonate`** (TLS/JA3 fingerprint match) | Plain HTTP with a browser's TLS fingerprint, no browser needed. | `jbhifi` (429 to non-browser TLS, fixed with `chrome`), `harveynorman` urls (`safari18_0_ios`), `adorebeauty` (`safari18_0_ios`), `yeti` (`chrome131`), `acmetools` (`chrome146`), `zoopla`, `aerosoles`, `whoscored`, `momoshoptw` (`chrome`), `duckduckgo` (`chrome110`) |
| **Camoufox, then reuse cookies in a fast client** | Browser solves the challenge once; the clearance cookies and UA go to `curl_cffi`/`httpx`/Scrapy. | `himalayas` (`cf_clearance`, re-solved on 403/503), `upwork` (cookies + auth token, then GraphQL), `casio` (cookies + UA, then GraphQL), `trustpilot` (cookies, then Next.js data calls), `adorebeauty` urls (key fetched in browser as fallback), `yeti` |
| **Camoufox Turnstile click** | Find the `challenges.cloudflare.com` frame and click the checkbox. | `autodoc`, `himalayas` |
| **`camoufox_captcha` solver library** | Library solves the Cloudflare challenge inside Camoufox. | `comfortiva` reviews |
| **Camoufox PerimeterX press-and-hold** | Auto press-and-hold on `#px-captcha`; falls back to a visible browser for a human to solve; session refresh on 403. | `skyscanner`, `yeti` (detects `px-captcha`/"been denied" pages) |
| **Camoufox as renderer only** | Site needs JS rendering; no explicit challenge handling. | `guatecompras`, `zoominfo`, `google` travel, `paige`, `catrustpilot`, `f6s`, `naukri`, `schwab`, `jampaper`, `evercommerce`, `harveynorman` detail + sitemaps |
| **Camoufox `humanize` + `geoip`** | Human-like cursor movement and IP-matched locale. | `instagram`, `upwork`, `himalayas` |
| **Cookie bootstrap / capture** | First request sets cookies (`datadome`, `_abck`, `bm_sz`, `_px*`), later requests reuse them. | `tripadvisor` (DataDome cookie captured on the 403, then retry), `rh` (cookie jar kept, DataDome), `adorebeauty` urls (home page first for PerimeterX cookies) |
| **Throttling and retries** | Slow down and retry on block codes. | `rh` (`DOWNLOAD_DELAY = 4`, else pagination returns 403), `kogan` (retry on 403/429/5xx) |
| **Block detection, then skip or refresh** | Detect challenge pages and either skip or rebuild the session. | `elkay` (flags CAPTCHA page and skips), `himalayas`, `skyscanner` |
| **Other data-source tricks** | Avoid the protected page entirely. | `harveynorman` (reads the build id from the unprotected 404 page), `fantasticfurniture` (Yotpo widget JSON API instead of the site) |
| **Playwright** | Plain Playwright for simple dynamic pages. | `fema`, `flashscore`, `forexfactory` |
| **`hrequests`** | Alternative TLS-spoofing client. | `apartments`, `adorebeauty` urls |

### Per-protection summary

| Protection | What worked in our spiders | Spiders |
|------------|----------------------------|---------|
| **Cloudflare / Turnstile** | Camoufox solves it (Turnstile click or `camoufox_captcha`), then cookies go to an `impersonate=firefox` client. | `himalayas`, `autodoc`, `comfortiva`, `guatecompras` |
| **PerimeterX** | iOS app headers + `safari18_0_ios` impersonation (Adore Beauty); auto press-and-hold in Camoufox with a manual fallback and session refresh (Skyscanner); `curl_cffi` + Camoufox (Yeti). | `adorebeauty`, `skyscanner`, `yeti` |
| **DataDome** | App User-Agent (Kogan); cookie capture and retry (TripAdvisor); cookie jar + 4 s delay (RH). | `kogan`, `tripadvisor`, `rh` |
| **TLS fingerprint blocks (429/403 on plain clients)** | `curl_cffi` impersonation, no browser. | `jbhifi`, `zoopla`, `aerosoles`, `whoscored`, `acmetools`, `duckduckgo` |
| **reCAPTCHA / hCaptcha / image CAPTCHA** | No solver service is wired in. Only a CAPTCHA page is detected and skipped (`elkay`). | none |

### Not found in the code

- **Solver services (CapSolver, 2Captcha, Anti-Captcha):** no API integration in any spider. Only the `camoufox_captcha` library and manual solving are used.
- **Proxies (residential or rotating):** no proxy configuration in `settings.py` or any spider (the `proxy` hits in `yandex`/`remington`/`hertz` are URL or field names, not proxy use). The playbook says to ask before adding paid proxies.
- **NoDriver, Selenium, Ulixee, CloakBrowser:** not used in this project. Camoufox is the main browser, with Playwright for simple pages.
- **Project settings:** `ROBOTSTXT_OBEY = False` and no downloader middleware, throttling or retry settings at project level; those are set per spider.

### Lessons so far

- Look for a mobile API or app User-Agent first; it was the cheapest fix for PerimeterX (Adore Beauty), DataDome (Kogan) and 403s generally.
- Match the TLS fingerprint with `curl_cffi` before reaching for a browser; it fixed most 429/403 blocks.
- When a browser is needed, use it only to get clearance cookies and a UA, then reuse them in a plain client. A challenge can come back, so re-solve on 403/503 and cap the retries.
- Keep the same IP, UA and TLS fingerprint across the browser step and the plain-client steps, since vendors bind cookies to them.
- Headless Camoufox sometimes hits challenges that `headless="virtual"` or a visible window does not (see the Skyscanner headless fixes).

---

## Recommended Stack

```
Clean session
   └─► Proxy rotation (datacenter → residential)
         └─► Stealth browser (Camoufox / NoDriver / Playwright, CDP if needed)
               └─► Human-like behavior + fingerprint randomization
                     └─► CAPTCHA solver (CapSolver / 2Captcha / Anti-Captcha)
```

## Quick Decision Table

| Protection | First choice | Fallback |
|------------|--------------|----------|
| Cloudflare | Camoufox + residential proxy | NoDriver / CDP |
| Turnstile | CapSolver / 2Captcha token | Stealth browser |
| reCAPTCHA v2 | CapSolver / 2Captcha | Anti-Captcha |
| reCAPTCHA v3 | CapSolver (real browser env) | Stealth browser + human-like behavior |
| hCaptcha | CapSolver / 2Captcha | Anti-Captcha |
| Text / Image | CapSolver / 2Captcha | Anti-Captcha |
| DataDome | Humanized automation + residential proxies | CDP-based browser |
| PerimeterX | Realistic fingerprint + behavior mimicry | Stealth browser + residential proxies |
