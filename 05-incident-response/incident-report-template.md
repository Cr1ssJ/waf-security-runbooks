# Security Incident Report Template

**Category:** Incident Response / Client Communication  
**Platform:** Platform-agnostic  
**Purpose:** Structured template for client-facing security incident reports covering WAF, DDoS, and bot-related events

---

## 🎯 Purpose

Consistent incident reporting builds client trust, creates an audit trail, and ensures that key stakeholders understand both what happened and what was done about it. This template covers the five core sections of an effective security incident report.

---

## 📄 Report Structure

```
1. Incident Summary          ← Executive-level, one paragraph
2. Attack Signatures         ← Technical detail with event counts
3. Observations              ← Analysis and context
4. Mitigation Status         ← What was done and current state
5. Recommended Actions       ← What should happen next
```

---

## Template

---

### [CLIENT NAME] — Security Incident Report

**Date:** `[YYYY-MM-DD]`  
**Period covered:** `[Start datetime] – [End datetime]`  
**Protected assets affected:** `[Domain(s) / Application(s)]`  
**Prepared by:** `[Engineer name]`  
**Classification:** `[Confidential / Internal]`

---

### 1. Incident Summary

> *One paragraph, non-technical, suitable for a manager or executive. Answer: What happened? When? Was it mitigated? Is there any ongoing risk?*

On `[date]`, `[protected asset]` experienced a `[type of event — e.g., multi-vector attack / DDoS attempt / credential stuffing campaign]` originating primarily from `[general source description — e.g., distributed sources across multiple ASNs]`. The event was detected at `[time]` and mitigation was `[activated automatically / applied manually]` at `[time]`. At the time of this report, `[the attack has subsided / mitigation remains active / no further activity has been observed]`.

---

### 2. Attack Signatures

> *Technical section. Document each distinct attack vector with event counts, timeframes, and indicators.*

#### 2.1 `[Attack Type 1 — e.g., HTTP Flood / DDoS]`

| Field | Value |
|---|---|
| Detection time | `[datetime]` |
| Duration | `[X minutes / hours]` |
| Peak traffic | `[X req/s or Gbps]` |
| Total events | `[count]` |
| Top source IPs/ASNs | `[ASN1 (X%), ASN2 (X%)]` |
| Target endpoints | `[/path1, /path2]` |
| Mitigation action | `[Block / Challenge / Alert]` |

#### 2.2 `[Attack Type 2 — e.g., Credential Stuffing / SQLi Attempts]`

| Field | Value |
|---|---|
| Detection time | `[datetime]` |
| Duration | `[X minutes / hours]` |
| Total events | `[count]` |
| Top source IPs/ASNs | `[ASN1 (X%), ASN2 (X%)]` |
| Target endpoints | `[/login, /api/auth]` |
| Techniques observed | `[e.g., distributed low-rate, UA rotation]` |
| Mitigation action | `[Block / Challenge / Alert]` |

*Add sections for additional attack vectors as needed.*

---

### 3. Observations

> *Analysis section. What does the data mean? What patterns were observed? Any noteworthy context?*

**Traffic analysis:**
During the observation period, traffic to `[asset]` showed `[description of pattern — e.g., a sustained increase beginning at HH:MM, concentrated on the following endpoints: ...]`.

**Source characteristics:**
The majority of malicious traffic originated from `[cloud infrastructure / residential IPs / distributed botnet]`. `[Notable observation about source diversity, geolocation, or ASN concentration]`.

**Attack sophistication:**
`[Assessment of attacker capability — e.g., The traffic showed signs of evasion including UA rotation and low request rates designed to avoid threshold-based detection / The attack used straightforward volumetric flooding with no evasion attempts]`.

**Potential impact:**
`[Assessment of what could have happened if mitigation had not been in place — avoid confirming any breach or data exposure without confirmed evidence]`.

---

### 4. Mitigation Status

> *Current state. What protections are active? Is the threat resolved?*

| Protection | Status | Activated |
|---|---|---|
| WAF rules | `[Active / Modified / Unchanged]` | `[datetime or N/A]` |
| DDoS mitigation | `[Active / Deactivated / Not triggered]` | `[datetime or N/A]` |
| Rate limiting | `[Active / Adjusted]` | `[datetime or N/A]` |
| IP blocks | `[Applied / Not applied]` | `[datetime or N/A]` |
| ABP policy changes | `[Applied / Pending]` | `[datetime or N/A]` |

**Current threat status:** `[Resolved / Ongoing — monitoring / Resolved — recommend continued monitoring for X hours]`

---

### 5. Recommended Follow-Up Actions

> *Concrete, prioritized actions. Assign ownership where possible.*

| Priority | Action | Owner | Deadline |
|---|---|---|---|
| High | `[e.g., Review and tighten rate limiting thresholds on /api/login]` | `[Security team / Client team]` | `[date]` |
| Medium | `[e.g., Enable CAPTCHA challenge on password reset endpoint]` | `[Application team]` | `[date]` |
| Low | `[e.g., Update allowlist to remove expired entry for X]` | `[Security team]` | `[date]` |

---

*End of report.*

---

## 📝 Usage Notes

- Replace all `[bracketed]` fields with actual data
- Remove sections not applicable to the incident type
- Use metric data (event counts, timestamps, percentages) — avoid vague language like "many" or "significant"
- Never include raw customer data, PII, or credentials in the report
- Use UTC timestamps consistently, noting the client's local timezone in parentheses

---

## 🔗 Related Runbooks

- [DDoS Alert Triage](../02-ddos-alert-triage/ddos-burst-vs-attack.md)
- [False Positive Diagnosis](../01-false-positive-diagnosis/waf-false-positive-triage.md)
- [ABP Traffic Classification](../03-bot-protection/abp-traffic-classification.md)
