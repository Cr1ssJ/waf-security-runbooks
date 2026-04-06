# Allowlist Decision Framework

**Category:** WAF Rule Management  
**Platform:** Imperva Cloud WAAP (applies broadly)  
**Severity Impact:** Critical — allowlist mistakes permanently weaken your security posture

---

## 🎯 Purpose

An allowlist is a deliberate exception to your security policy. Every allowlist entry is a controlled reduction in protection. This runbook governs when to create one, how to scope it, and when to review it.

**The core principle:** When in doubt, don't allowlist. Alert-only mode, rule tuning, or application-side fixes are almost always safer options.

---

## ❌ When NOT to Create an Allowlist

- You haven't confirmed the traffic is legitimate (evidence required)
- You don't know the exact source of the traffic
- The request is flagged by multiple independent rules (overlapping detection = stronger signal)
- SIEM correlation hasn't been performed when access is available
- The proposed allowlist scope is broader than the minimum necessary
- The change hasn't been reviewed/approved

---

## ✅ When an Allowlist IS Appropriate

- The source is fully identified (specific IP, IP range, or service)
- The legitimate use case is documented and understood
- The scope can be narrowed to specific path + parameter + source
- The change is approved by the appropriate authority
- A review date is set

---

## 🔍 Allowlist Scoping — Narrowest Possible

Always apply the most restrictive combination of conditions possible:

**Scope hierarchy (most restrictive → least restrictive):**

```
1. IP + Path + Parameter + Rule ID       ← ideal
2. IP + Path + Rule ID
3. IP + Rule ID
4. IP range + Path + Rule ID
5. Path + Rule ID                        ← only if source varies legitimately
6. Rule ID only                          ← almost never correct
7. Site-wide / global                    ← almost always wrong
```

**Never create a site-wide allowlist for a specific IP without understanding every endpoint that IP can now reach.**

---

## 📋 Pre-Allowlist Checklist

```
Evidence gathering:
[ ] Raw request(s) reviewed
[ ] Source IP / ASN identified and verified
[ ] User-Agent confirmed legitimate
[ ] Endpoint and parameter documented
[ ] Traffic volume and pattern assessed
[ ] Business justification confirmed with application team

Risk assessment:
[ ] Impact if this source IS malicious (what could it access?)
[ ] Existing controls that still apply (rate limiting, other rules)
[ ] Minimum scope determined

Approval:
[ ] Change reviewed by security lead or client
[ ] Documented in change log

Implementation:
[ ] Narrowest possible scope applied
[ ] Rule set to Alert-only first (before full allowlist if possible)
[ ] Review date set (default: 90 days)
```

---

## 🗓️ Allowlist Lifecycle

```
Creation ──► 30-day review ──► 90-day review ──► Annual audit ──► Keep / Remove / Modify
```

**At each review, ask:**
- Is the source still active and using this endpoint?
- Has the use case changed?
- Is the scope still the minimum necessary?
- Has the application changed in a way that makes this allowlist unnecessary?

---

## 🔗 Related Runbooks

- [False Positive Triage](../01-false-positive-diagnosis/waf-false-positive-triage.md)
- [ABP Traffic Classification](../03-bot-protection/abp-traffic-classification.md)
- [WAF Rule Tuning Methodology](./rule-tuning-methodology.md)
