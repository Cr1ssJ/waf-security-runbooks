# Allowlist Decision Framework

**Category:** WAF Rule Management  
**Platform:** Imperva Cloud WAAP (applies broadly)  
**Severity Impact:** Critical — allowlist mistakes permanently weaken your security posture

---

## 🎯 Purpose

An allowlist is a deliberate exception to your security policy. Every allowlist entry is a controlled reduction in protection. This runbook governs when to create one, how to scope it, and when to review it.

**The core principle:** When in doubt, don't allowlist. Alert-only mode, rule tuning, or application-side fixes are almost always safer options.

---

## ⚠️ Critical: Allowlists Kill Visibility — Logs Are Not Generated

This is one of the most important operational facts about WAF allowlists that is often overlooked:

> **Traffic that matches an allowlist entry does NOT generate security events — not in the WAF, and not in any connected SIEM.**

This means:

- You cannot audit allowlisted traffic after the fact
- You cannot detect abuse of an allowlisted IP or path
- If the allowlisted source is ever compromised, its malicious traffic becomes invisible
- SIEM correlation rules will never fire on allowlisted requests — even if the request is malicious

### Allowlist vs. ACL — Always Prefer ACLs

| Mechanism | Generates events/logs | SIEM visibility | Reversible detection | Use when |
|---|---|---|---|---|
| **Allowlist** | ❌ No | ❌ Blind | ❌ No | Almost never — only when ACL is not an option |
| **ACL (Access Control List)** | ✅ Yes | ✅ Full | ✅ Yes | Default choice for any exception |

**ACLs allow the traffic through but still generate log entries.** This means:

- The SIEM still receives the events
- Anomaly detection rules can still fire
- You maintain an audit trail
- If the source behavior changes, you will know

### The Practical Rule

**Default to ACL. Use allowlist only when the platform provides no other mechanism.**

If you must use an allowlist, apply the absolute minimum scope — specific IP + specific path + specific rule — precisely because you are creating a permanent blind spot in your visibility for everything that scope covers.

A broad allowlist (e.g., site-wide for a large IP range) doesn't just reduce protection — it eliminates your ability to detect anything from that source, ever, as long as the allowlist exists.

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

**Never create a site-wide allowlist for a specific IP without understanding every endpoint that IP can now reach — and accepting that you will be completely blind to everything that IP does across the entire site.**

---

## 📋 Pre-Allowlist Checklist

```
Before choosing an allowlist, ask first:
[ ] Can this be handled with an ACL instead? (ACLs generate logs — allowlists do not)
[ ] If ACL is available: USE ACL, not allowlist. Stop here.
[ ] If allowlist is the only available mechanism: continue below.

Evidence gathering:
[ ] Raw request(s) reviewed
[ ] Source IP / ASN identified and verified
[ ] User-Agent confirmed legitimate
[ ] Endpoint and parameter documented
[ ] Traffic volume and pattern assessed
[ ] Business justification confirmed with application team

Risk assessment:
[ ] Understood that this source will be INVISIBLE in logs and SIEM after allowlisting
[ ] Impact if this source IS compromised and begins sending malicious traffic
[ ] Existing controls that still apply (rate limiting, other rules)
[ ] Minimum scope determined — remember: every additional scope = additional blind spot

Approval:
[ ] Change reviewed by security lead or client
[ ] Client/stakeholder explicitly informed that allowlisted traffic will not appear in reports
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
- Can this now be replaced with an ACL to restore visibility?

> **The goal of every allowlist review is to eliminate the allowlist or convert it to an ACL.** Allowlists should be temporary by default. Every day an allowlist exists is another day you have no visibility into that traffic.

---

## 🔗 Related Runbooks

- [False Positive Triage](../01-false-positive-diagnosis/waf-false-positive-triage.md)
- [ABP Traffic Classification](../03-bot-protection/abp-traffic-classification.md)
- [WAF Rule Tuning Methodology](./rule-tuning-methodology.md)
