# Advanced Bot Protection — Traffic Classification Runbook

**Category:** Bot Management / ABP Analysis  
**Platform:** Imperva Advanced Bot Protection (ABP)  
**Severity Impact:** High — misclassifying legitimate bot traffic blocks real users; missing malicious bots enables scraping, credential stuffing, and fraud

---

## 🎯 Purpose

Advanced Bot Protection generates two primary traffic categories: **Suspicious** and **Blocked**. The challenge is that not all Suspicious traffic is malicious, and not all traffic that looks clean is legitimate.

This runbook documents the methodology for classifying ABP traffic across a set of policies grouped by scope and purpose — distinguishing:

1. **Legitimate automated traffic** (mobile SDKs, known crawlers, monitoring tools)
2. **Ambiguous traffic** requiring deeper investigation
3. **Genuinely malicious bots** (scrapers, credential stuffers, vulnerability scanners)

---

## 🗂️ ABP Policy Types — Context Matters First

Before classifying any traffic, understand what the policy is protecting:

| Policy scope | Expected traffic mix | Baseline assumption |
|---|---|---|
| **Public web / marketing site** | Browsers + crawlers + some bots | Mixed — expect legitimate scrapers and search bots |
| **API policy (app-specific)** | Programmatic clients (mobile SDKs, internal services) | High volume of "bot-like" traffic that is legitimate |
| **Login / authentication endpoint** | Browsers + some automation | Any non-browser traffic is higher risk |
| **E-commerce / checkout** | Browsers + payment integrations | Very low tolerance for bot traffic |
| **Admin / internal endpoints** | Humans only (ideally) | Any bot signal should be investigated |

> **Critical insight:** An "App API" policy will show high Suspicious traffic almost by definition, because mobile SDKs don't carry full browser fingerprints. This is **expected** — it does not mean the policy is failing or that you're under attack.

---

## 📊 The Two-Bucket Problem

Imperva ABP classifies traffic into:

- **Suspicious:** Traffic that exhibits bot-like characteristics but hasn't been confirmed as malicious
- **Mitigated:** Traffic that was blocked/challenged based on confirmed bot signals

The key question for each bucket is not just "is this a bot?" but **"is this bot traffic legitimate for this application?"**

---

## 🔬 Investigation Dimensions

For any ABP traffic analysis, work through these five dimensions systematically:

---

### Dimension 1 — User-Agent Analysis

User-Agent strings are the fastest first filter. Classify them into:

**Tier 1 — Known legitimate (likely false positive if blocked):**
| User-Agent pattern | Source | Notes |
|---|---|---|
| `okhttp/x.x.x` | Android HTTP client library | Used by most Android apps. Very common in API policies. |
| `Alamofire/x.x.x` | iOS HTTP client library | Standard iOS networking. Expect in mobile-heavy APIs. |
| `axios/x.x.x` | JavaScript HTTP client | Frontend or Node.js apps. Common in web APIs. |
| `python-requests/x.x.x` | Python requests library | Could be legitimate (internal tools) or automated scanning |
| `Go-http-client/x.x` | Go HTTP client | Internal services, monitoring, or microservices |
| `passd` | Apple Wallet / Passbook | iOS Wallet passes. Legitimate if app supports Wallet. |
| `Googlebot`, `bingbot` | Search engine crawlers | Legitimate if on public-facing pages |
| `curl/x.x.x` | cURL | Ambiguous — developer testing or automated tool |

**Tier 2 — Ambiguous (require additional context):**
- Generic strings: `Mozilla/5.0` without browser-specific tokens
- Outdated browser versions (e.g., IE6, Chrome 40) — could be spoofed
- Empty or null User-Agent — almost always automated

**Tier 3 — High-risk signals:**
| Pattern | Risk | Reason |
|---|---|---|
| Tool names: `sqlmap`, `nikto`, `masscan`, `nmap` | Critical | Known attack tools |
| `python-requests` at high rate to sensitive endpoints | High | Automated scanning or credential stuffing |
| Random/garbage strings | High | UA randomization to evade detection |
| Empty UA | High | Most legitimate clients set a UA |
| Mismatched UA + behavior | High | Browser UA but behaves like a script |

---

### Dimension 2 — ISP / ASN Analysis

The source network tells you a lot about the traffic intent:

**ASN categories and their implications:**

| ASN type | Examples | Implication |
|---|---|---|
| **Residential ISP** | Claro, Tigo, Movistar, Comcast | Legitimate users. NAT means many users share one IP. High volume from one IP ≠ one attacker. |
| **Mobile carrier** | AT&T, T-Mobile, América Móvil | Same as residential. Very common source of false positives. |
| **Cloud provider** | AWS, GCP, Azure, DigitalOcean | No legitimate user browses from cloud IP. Any traffic here is automated. May be legitimate (internal tool) or malicious (bot hosted on cloud). |
| **Hosting / VPS** | Vultr, Linode, Hetzner, OVH | Same as cloud — automated traffic. Requires UA + behavior context. |
| **Corporate / enterprise** | Large company ranges | Could be legitimate (employees) or proxy/VPN. |
| **Known bad actors** | Bulletproof hosters, known abuse ASNs | Block by default. |
| **TOR exit nodes** | Various | Anonymization tool. Block or challenge depending on policy. |
| **VPN providers** | NordVPN, ExpressVPN, etc. | Anonymization. Depends on your threat model. |

> **For mobile app APIs specifically:** The majority of legitimate traffic will arrive from residential ISP and mobile carrier ASNs. High volume from a single mobile carrier IP is almost always many real users behind NAT, not a single attacker.

---

### Dimension 3 — Path / Endpoint Analysis

The endpoint being targeted changes the risk calculus significantly:

**High-sensitivity endpoints (any bot signal should be investigated):**
- `/login`, `/signin`, `/auth` — credential stuffing target
- `/password-reset`, `/forgot-password` — account takeover vector
- `/checkout`, `/payment`, `/order` — fraud target
- `/admin/*`, `/api/admin/*` — unauthorized access attempt
- `/register`, `/signup` — fake account creation

**Medium-sensitivity endpoints (context-dependent):**
- `/api/search` — could be legitimate high-volume queries or scraping
- `/api/products`, `/api/catalog` — price scraping target
- `/api/user/*` — depends on what data is exposed

**Lower-sensitivity endpoints (bot traffic may be legitimate):**
- `/api/health`, `/api/status` — monitoring tools
- `/sitemap.xml`, `/robots.txt` — crawler behavior
- Static assets (`/static/*`, `/images/*`) — CDN behavior

---

### Dimension 4 — Temporal Pattern Analysis

When does the traffic occur, and what does the timing tell you?

**Patterns and their interpretations:**

| Pattern | Likely interpretation |
|---|---|
| Traffic follows business hours (9am–6pm, weekdays) | Human-driven or business-process automation |
| Traffic is perfectly uniform (e.g., exactly 100 req/min, 24/7) | Script with rate limiting — automation |
| Traffic spikes at odd hours (2am–4am local time) | Off-hours automation — suspicious |
| Traffic correlates with app release / update date | Mobile SDK update — legitimate |
| Sudden spike with no business context | Possible attack or scraping campaign |
| Traffic gradually increases over days | Slow bot / low-and-slow attack — harder to detect |

**For mobile app traffic:** Expect traffic spikes correlating with:
- App store updates (sudden UA version bump)
- Marketing push notifications (burst of activity from user base)
- Business hours of primary user geography

---

### Dimension 5 — HTTP Method + Request Behavior

**Method distribution:**
- `GET` heavy on API endpoints: could be scraping or legitimate reads
- `POST` heavy on login endpoints: credential stuffing signal
- `HEAD` requests at high rate: content discovery / reconnaissance
- Unusual methods (`OPTIONS`, `TRACE`, `PUT`, `DELETE`) on unexpected endpoints: testing/scanning

**Behavioral signals:**
- **No cookies** on endpoints that normally require session: automated client
- **No `Referer`** on form submissions: direct automated POST
- **Sequential IDs** in parameters: enumeration attack (`/user/1`, `/user/2`, `/user/3`)
- **Same exact payload** repeated: replay attack or primitive scraper
- **High entropy in parameters**: randomization to avoid pattern detection (sophisticated bot)

---

## 🚦 Classification Decision Matrix

After working through all five dimensions, use this matrix:

| Bot-like UA | Cloud/VPS ASN | Sensitive endpoint | Suspicious behavior | Classification |
|---|---|---|---|---|
| ✅ (okhttp/Alamofire) | ❌ (mobile/residential) | ❌ (API reads) | ❌ | **Legitimate — mobile SDK** |
| ❌ (browser UA) | ✅ (AWS/GCP) | ✅ (login) | ✅ (no cookies, uniform rate) | **Malicious — credential stuffing** |
| ✅ (python-requests) | ✅ (DigitalOcean) | ✅ (catalog/price) | ✅ (sequential params) | **Malicious — price scraper** |
| ✅ (Googlebot) | ❌ (Google ASN) | ❌ (public pages) | ❌ | **Legitimate — search crawler** |
| ❌ (empty UA) | ✅ (Hetzner) | ✅ (login) | ✅ (high rate) | **Malicious — automated attack** |
| ✅ (python-requests) | ❌ (corporate) | ❌ (API) | ❌ (low rate, business hours) | **Likely legitimate — internal tool** |
| ❌ (randomized UA) | ✅ (VPS) | ✅ (checkout) | ✅ (fraud indicators) | **Malicious — fraud bot** |

---

## 🏷️ ABP Flag Reference

Key flags to understand when reviewing ABP events:

| Flag | Meaning | Investigation priority |
|---|---|---|
| `no_token` | Request missing the ABP JavaScript token (no browser execution) | High — non-browser client. Context needed. |
| `invalid_token` | Token present but failed validation | High — tampering or token reuse attempt |
| `expired_token` | Token present but too old | Medium — could be cached/proxied request or slow client |
| `bad_ua` | User-Agent flagged as known bot/tool | High — check against Tier 3 list above |
| `headless` | Headless browser detected | High — automated browser (Puppeteer, Selenium, Playwright) |
| `rate_limit` | Source exceeded request rate threshold | Medium — context needed (NAT? or single attacker?) |

> **`no_token` in app API policies:** If an API policy protects a mobile app endpoint, the mobile SDK will **never** execute JavaScript — so `no_token` will be the most common flag for all mobile traffic. This is expected. Do not treat `no_token` on an App API policy the same way as `no_token` on a web login page.

---

## 📝 ABP Traffic Investigation Template

Use this template when investigating an ABP policy with elevated Suspicious or Mitigated traffic:

```
Policy name:
Policy type (Web / App API / Login / etc.):
Observation period:
Total traffic volume:
Suspicious traffic volume + percentage:
Mitigated traffic volume + percentage:

--- Top User-Agents ---
1.
2.
3.
Classification:

--- Top ISPs / ASNs ---
1.
2.
3.
Classification (residential / mobile / cloud / bad):

--- Top Target Paths ---
1.
2.
3.
Sensitivity level:

--- Temporal Pattern ---
Peak hours:
Distribution (uniform / bursty / business hours):

--- Verdict by condition type ---
Bad User Agents:     [ ] Legitimate  [ ] Malicious  [ ] Mixed  [ ] Unknown
Rate Limiting:       [ ] Legitimate  [ ] Malicious  [ ] Mixed  [ ] Unknown
Force Identify:      [ ] Legitimate  [ ] Malicious  [ ] Mixed  [ ] Unknown
no_token:            [ ] Legitimate  [ ] Malicious  [ ] Mixed  [ ] Unknown

--- Overall verdict ---
[ ] Predominantly false positives — recommend tuning
[ ] Mixed — requires per-condition action
[ ] Predominantly malicious — policy performing correctly
[ ] Escalate — insufficient data for conclusion

Recommended actions:
```

---

## 🔗 Related Runbooks

- [False Positive Diagnosis](../01-false-positive-diagnosis/waf-false-positive-triage.md)
- [Mobile SDK Fingerprints](./mobile-sdk-fingerprints.md)
- [Allowlist Decision Framework](../04-waf-rule-tuning/allowlist-decision-framework.md)
