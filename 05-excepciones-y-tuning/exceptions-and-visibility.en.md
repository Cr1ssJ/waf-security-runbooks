# Exceptions Without Losing Visibility: Allowlist and ACL

| Field | Value |
|---|---|
| Platform | Imperva Cloud WAAP (WAF and ABP) |
| Version | 0.1 (draft) |
| Last validated | October 2026 |
| Audience | WAF engineers and SOC analysts |
| Language | English · [Español](excepciones-y-visibilidad.es.md) |

---

## 1. Purpose

An exception is a **deliberate reduction in protection**. Sometimes it is necessary (a legitimate integration trips a rule), but the mechanism you choose decides whether you can still see what happens afterwards or whether you go blind.

> **Principle:** when in doubt, do not create an exception. Alerting, tuning the rule or fixing the application are usually safer options.

## 2. The Most Overlooked Fact

> **Traffic that matches an allowlist does not generate security events: not in the console and not in the SIEM.**

This was confirmed in real operation. Validate it in your own account before relying on it. Consequences:

- You cannot audit that traffic after the fact.
- If the allowed source is compromised, its malicious traffic is **invisible**.
- SIEM correlation rules never fire on it.

## 3. Mechanism Comparison

> Based on what we observed in our environment. Confirm each mechanism's behavior in your account.

| Mechanism | Generates events? | Visible in SIEM? | When to use |
|---|---|---|---|
| **Allowlist** | No | No | Last resort, only when no other option exists. |
| **ACL-based exception** | Yes | Yes | **Default choice** for any exception. |
| **Allow / Ignore action in a rule** | Reduced visibility | Reduced visibility | Only if an ACL does not solve the case. |

## 4. Decision Tree

1. **Is it really a false positive?** Review the full request, the source and the application context.
2. **Can it be fixed by tuning the rule or fixing the application?** If the rule is too broad, fix it; do not work around it.
3. **Can I use an ACL?** If yes, **stop here**.
4. **Is an allowlist the only option?** Then apply the minimum scope and document the blind spot you are accepting.

## 5. Scope: Most to Least Restrictive

```
1. IP + Path + Parameter + Rule       ← ideal
2. IP + Path + Rule
3. IP + Rule
4. IP range + Path + Rule
5. Path + Rule                        ← only if the source legitimately varies
6. Rule only                          ← almost never correct
7. Site-wide                          ← almost always a mistake
```

> **Mind the position:** the real scope of an exception depends on **where** it is defined. An exception in a policy evaluated first (for example, the default policy) can apply to an entire website group, even if you only needed it for one site.

## 6. Pre-Exception Checklist

```
Before choosing a mechanism:
[ ] Can this be solved with an ACL?  If yes: use it. Done.

Evidence:
[ ] Full request(s) reviewed
[ ] Source IP / ASN identified and verified
[ ] User-Agent confirmed legitimate (for search engines: reverse DNS with forward confirmation)
[ ] Path and parameter documented
[ ] Volume and pattern assessed
[ ] Business justification confirmed with the application team

Risk:
[ ] You understand which area will lose visibility
[ ] Impact assessed if that source is compromised
[ ] Controls that still apply are listed
[ ] Minimum scope defined

Approval and closure:
[ ] Change approved by the security owner or the customer
[ ] The customer knows excepted traffic will not appear in reports
[ ] Logged in change control
[ ] Review date set
```

## 7. Lifecycle

```
Creation ──► 30-day review ──► 90-day review ──► Annual audit ──► Keep / Modify / Remove
```

At each review ask: is the source still active? Did the use case change? Is the scope still the minimum? Can it now be converted to ACL to regain visibility? **The goal of every review is to remove the allowlist or convert it.**

## 8. Real Cases (Anonymized)

### Case A — A search-engine crawler that explained much of the traffic

**Situation.** Anomalous 404 errors were being investigated on lookup paths of a public portal. A significant share of the traffic was a legitimate search-engine crawler (verified through reverse DNS), allowed by an exception in the default policy.

**Takeaways.**

- Because that policy is evaluated **first** in the website group, the exception was not limited to the site where it was needed.
- Checking reverse DNS prevented misclassifying the traffic. The remaining IPs were mostly residential and mobile networks, consistent with real users.
- A hosting IP running automated scanning was also found, **unrelated** to the original problem. It was recommended to handle it in a separate case.

**Lesson:** separate your findings. An application problem and a scanner can coexist on the same paths.

### Case B — Hacking-tool detection rules in Alert mode

**Situation.** A hacking-tool detection rule was deployed across several sites in **Alert** mode, precisely to observe before blocking.

**Outcome.** False positives appeared from legitimate HTTP clients: Windows utilities, Java libraries, Postman, `python-requests` and Ruby clients, used by integrations and development teams.

**Lesson:** without the observation phase (Passive/Alert) these integrations would have been blocked. User-Agent classification does not tell you **intent**; it only describes the tool.

### Case C — A redirect rule with no Alert mode

**Situation.** Redirect rules apply their action immediately; there is no "Alert mode" equivalent. They also normally **take priority: they are evaluated before security rules**. So once active, a redirect can act on traffic before security rules evaluate it (confirm the evaluation order in your account).

**Solution.** Create a parallel **shadow rule** with the same condition and an Alert action, and keep the **real redirect rule switched off (disabled)** while you validate with it. Only when the evidence confirms the condition affects just the expected traffic do you turn the redirect on.

## 9. Change Log Template

```
Date:
Mechanism (Allowlist / ACL / Allow in rule):
Rule or condition affected:
Site / path:
Source (IP / ASN / service):
Pattern that triggered detection:
Root cause:
Evidence reviewed:
Scope of change:
Accepted blind spot:
Approved by:
Review date:
```

## 10. Explaining It to a Non-Technical Audience

> "An exception lets certain traffic through without being blocked. If it also switches off the logs, we can no longer see what that traffic does. That is why we prefer exceptions that let it through **and** keep recording it. Before taking any change to production, we validate it in a test environment."
