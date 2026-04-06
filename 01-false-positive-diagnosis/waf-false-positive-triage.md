# WAF False Positive Diagnosis — Triage Runbook

**Category:** Rule Management / Alert Quality  
**Platform:** Imperva Cloud WAAP (concepts apply broadly)  
**Severity Impact:** Medium — false positives block legitimate users and erode trust in WAF policies

---

## 🎯 Purpose

This runbook provides a structured decision workflow for WAF engineers to **confirm or rule out false positives** when a security rule triggers on traffic that may be legitimate.

The core challenge: a WAF rule firing doesn't automatically mean an attack. It means the traffic matched a pattern. Your job is to determine whether that match is meaningful.

---

## 🔄 Triage Workflow

```
Alert received
      │
      ▼
1. Identify the triggering rule and violation type
      │
      ▼
2. Examine the raw request (URI, method, headers, body, User-Agent)
      │
      ▼
3. Check source context (IP reputation, ASN, geolocation, rate)
      │
      ├── Known bad (threat intel match, blacklisted ASN) ──► CONFIRM ATTACK, escalate
      │
      └── Unknown / legitimate-looking ──► Continue investigation
                    │
                    ▼
4. Analyze traffic volume and temporal pattern
      │
      ├── Burst from single IP / narrow ASN ──► LIKELY ATTACK
      │
      └── Distributed, low-rate, known ISP ──► POSSIBLE FALSE POSITIVE
                    │
                    ▼
5. Cross-reference with application context
      │
      ├── Known integration? (mobile app, partner API, monitoring tool) ──► LIKELY FALSE POSITIVE
      │
      └── No known legitimate source ──► ESCALATE for deeper analysis
                    │
                    ▼
6. Decision: Tune / Allowlist / Escalate / Confirm block
```

---

## 📋 Step-by-Step Guide

### Step 1 — Identify the triggering rule

Document before doing anything else:

| Field | What to capture |
|---|---|
| Rule ID / Signature | Which specific rule fired |
| Violation type | SQLi, XSS, RFI, Custom rule, Rate limit, etc. |
| Action taken | Block, Alert only, Challenge |
| Event count | How many times in the observation window |
| Affected URL / path | Which endpoint is generating the hits |

> **Why this matters:** A single rule generating 10,000 alerts on a `/api/search` endpoint is a completely different problem than 3 alerts on `/admin/login`. Volume and endpoint context change everything.

---

### Step 2 — Examine the raw request

Look at the actual HTTP request that triggered the rule:

**Key fields to inspect:**
- `URI path` + `query string` — does the flagged parameter make sense for this endpoint?
- `Request body` — if POST/PUT, what payload triggered the match?
- `User-Agent` — is this a browser, mobile SDK, known bot, or custom tool?
- `Referer` — does the request come from a known legitimate flow?
- `Content-Type` — expected for this endpoint?
- `X-Forwarded-For` / source IP — is there NAT or proxy involved?

**Red flags that lean toward REAL ATTACK:**
- Payload contains clear attack patterns (`UNION SELECT`, `<script>`, `../../../`, `cmd=`)
- Multiple distinct attack types in same session
- No valid `Referer` for a form submission endpoint
- User-Agent matches known attack tools (`sqlmap`, `nikto`, `masscan`)
- IP has no prior legitimate history on the site

**Red flags that lean toward FALSE POSITIVE:**
- Payload looks like normal application data that happens to match a pattern
  - Example: A search term like `price > 100 AND category = 'shoes'` triggering SQLi rules
- User-Agent is a known mobile SDK (`okhttp`, `Alamofire`, `Axios`)
- High volume from a single known ISP serving many legitimate users (mobile carrier NAT)
- Traffic follows normal business hours pattern

---

### Step 3 — Check source context

**IP / ASN analysis:**
- Is the source IP in a threat intelligence feed?
- Is the ASN a residential ISP, mobile carrier, cloud provider, or known bad actor range?
- Is geolocation consistent with your expected user base?

**For Imperva WAAP specifically:**
- Check the "IP Reputation" field in the event details
- Look at the "Client Classification" — is it classified as Human, Bot, or Unknown?
- Check if the IP appears in the allowlist or has prior clean history

> **Key insight:** A mobile carrier IP (e.g., Claro, Tigo, AT&T mobile) serving thousands of users behind NAT will generate high-volume alerts that look alarming but are almost always false positives if the User-Agent matches expected mobile app patterns.

---

### Step 4 — Analyze traffic pattern

**Temporal analysis questions:**
- When did the alerts start? Was there a deployment, app update, or campaign launch around that time?
- Is the pattern consistent (steady rate = likely automation) or spiky (burst = possible scan/attack)?
- Does it follow business hours or is it 24/7?

**Volume analysis questions:**
- What percentage of total traffic does this represent?
- Is it isolated to one IP/ASN or distributed across many sources?
- Is the same payload appearing across many different source IPs? (Coordinated attack signal)

---

### Step 5 — Cross-reference with application context

This is where you need input beyond the WAF logs:

**Questions to ask:**
- Is there a known integration that sends automated traffic to this endpoint? (Payment processor, analytics SDK, internal monitoring)
- Did the application team recently change how they send data to this endpoint?
- Is there a mobile app that uses this API? What SDK does it use?
- Are there third-party services (CDN, load balancer) that might alter headers or payloads?

> **For ABP specifically:** Traffic from your own mobile app (using `okhttp`, `Alamofire`, `Axios`) will frequently trigger bot detection rules because SDKs don't behave exactly like browser traffic. This is expected behavior, not an attack. See the [ABP Traffic Classification runbook](../03-bot-protection/abp-traffic-classification.md) for detail.

---

### Step 6 — Decision framework

| Finding | Recommended Action |
|---|---|
| Clear attack pattern + bad IP reputation | **Confirm block.** Document and escalate if high severity. |
| Ambiguous pattern, legitimate-looking source | **Set to Alert-only temporarily** and monitor for 24–48h before deciding |
| Known legitimate integration triggering rule | **Create targeted allowlist** with minimum required scope (specific IP + path + parameter) |
| Rule too broad, catching normal app behavior | **Flag for rule tuning.** Don't just allowlist — fix the rule. |
| Unable to determine without more data | **Escalate to SIEM correlation** before making changes. Never allowlist without evidence. |

---

## ✅ Allowlist Decision Checklist

Before creating any allowlist entry, confirm all of the following:

- [ ] The source has been identified (specific IP, IP range, or known service)
- [ ] The legitimate use case has been documented
- [ ] The allowlist is scoped to the minimum necessary (specific path, not site-wide)
- [ ] The change has been approved by the security lead or client
- [ ] A review date has been set (allowlists should not be permanent by default)
- [ ] SIEM correlation has been checked if available (don't allowlist what you can't explain)

> **Warning:** Site-wide allowlists are almost always wrong. Always scope to the narrowest possible combination of source + path + parameter.

---

## 📝 False Positive Documentation Template

When you confirm a false positive and make a change, document it:

```
Date: 
Rule ID affected:
Endpoint / path:
Source (IP/ASN/Service):
Trigger pattern:
Root cause:
Evidence reviewed:
Action taken:
Scope of change:
Approved by:
Review date:
```

---

## 🔗 Related Runbooks

- [DDoS Burst vs. Attack Triage](../02-ddos-alert-triage/ddos-burst-vs-attack.md)
- [ABP Traffic Classification](../03-bot-protection/abp-traffic-classification.md)
- [Allowlist Decision Framework](../04-waf-rule-tuning/allowlist-decision-framework.md)
