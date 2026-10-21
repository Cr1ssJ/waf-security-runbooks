# ABP: Phased Rollout and Traffic Classification

| Field | Value |
|---|---|
| Platform | Imperva Cloud WAAP — Advanced Bot Protection (ABP) |
| Version | 0.1 (draft) |
| Last validated | October 2026 |
| Audience | WAF engineers and SOC analysts |
| Language | English · [Español](abp-despliegue-por-fases.es.md) |

---

## 1. Purpose

ABP tells people apart from automation. The goal is **not to block bots**; it is to block **harmful** bots without affecting real users, legitimate search engines or your own integrations.

This runbook explains how to roll ABP out in phases, how to classify traffic flagged as suspicious and which mistakes to avoid. The core idea: **a bot is not the same as an attack**; what matters is whether that automated traffic is legitimate for *that* application.

## 2. Principles

1. Diagnose before acting.
2. Build a traffic baseline before creating rules.
3. Always move in two stages: first **Passive/Alert mode**, where **no action is applied** and you only observe; then an **active action**, which can be **CAPTCHA** or **Block**. Alert and CAPTCHA are not the same thing: Alert does not act on traffic, CAPTCHA does. Never skip the observation stage.
4. Validate false positives in a non-production environment before going to production.
5. Identify legitimate traffic (first- and third-party) before enabling blocking conditions.
6. Prefer **behavioral** signals over signatures that are easy to rotate.

## 3. Policy Scope

| Aspect | What to do |
|---|---|
| Granularity | Prefer **path- or directory-level** policies for a well-defined scope. For internal sites with a known, restricted audience, **per-site** scope is usually simpler and safer. |
| Uniqueness | Directories must be **unique** across all sites in the same *website group*. A repeated directory causes conflicts. |
| Default policy | Do not assume how traffic cascades to the default policy: **verify it empirically in Passive mode** before relying on it. |
| Grouped sites | Two sites with different traffic may share a group for operational reasons, but this limits how independently you can tune them. Document the decision and revisit it as the sites mature. |

## 4. Roadmap by Pillars

A clean way to plan ABP is to split it into pillars. Each one goes through the phases in section 5.

| Pillar | What it looks for | Suggested logic | Cautions |
|---|---|---|---|
| **1. Traffic hygiene** | Requests that no real browser produces: missing User-Agent, badly normalized paths, missing token, cloud-provider origin. | Evaluate each signal separately in Passive/Alert, then decide how to combine them. | A missing token is normal for non-browser integrations. Cloud providers may host **your own** infrastructure: confirm an inventory of legitimate ASNs before blocking. |
| **2. Manipulation detection** | Traffic that tampers with its own token or interrogation (for example, invalid checksums or challenge manipulation). | `any` (OR): **one** signal is enough. | High-confidence signals, but still validate in Passive/Alert first. |
| **3. Environment emulation detection** | Automated browsers or clients that lie about their identity (impostors, webdriver, evasion tooling). | Combine with other signals; lower-precision ones stay in Passive/Alert until combined with other indicators, and only then move to an active action. | Some fingerprint-tampering signals have known false positives on uncommon devices. Example signals to review (confirm names in your console): `browser_impostor`, `fingerprint_tampering_*`. |
| **4. Environment separation** | Keeping test traffic from polluting production. | Depends on architecture. | **Optional**, and may be put on hold by the application owner. |

> **Rule of thumb:** use `any` (OR) to detect **manipulation** and `all` (AND) for **correlated fraud**. This cuts false positives where a single signal is not enough.

## 5. Rollout Phases

The criterion to advance is **evidence**, not the calendar. Phase length depends on the site's traffic volume.

> **Important:** Alert and CAPTCHA are different things. In **Passive/Alert mode no action is applied** to traffic; **CAPTCHA and Block are active actions**.

| Phase | What happens | Exit criterion |
|---|---|---|
| **0. Baseline** | Inventory normal traffic across the five dimensions (section 6): User-Agent, ISP/ASN, paths, hours and session behavior. Identify first- and third-party integrations. | A documented inventory of legitimate and unknown traffic exists. |
| **1. Passive / Alert (no action)** | Enable conditions without applying any action to traffic: observe and measure only. False positives are analyzed. | You understand what volume and type of traffic each condition flags, and false positives are explained and handled (minimal exception, rule adjustment or application fix). |
| **2. Active action (CAPTCHA or Block)** | A real action is applied to traffic. Choose by the condition's confidence: **CAPTCHA** for ambiguous traffic and **Block** only for **high-confidence** conditions already validated in testing. | The action does not affect legitimate traffic during the agreed observation period. |
| **3. Maintenance** | Periodic reviews: are exceptions still valid? Did the application change? Are new techniques appearing? | Documented review. |

**Control practices:**

- Keep a tracker with separate **Pre-Production** and **Production** sheets, plus a summary and a legend.
- Every change is validated in pre-production before production.
- When advising a customer, phrase recommendations as *"evaluate additional conditions based on observed behavior"*, and always state that it will be validated before reaching production.

## 6. The Five Analysis Dimensions

Always analyze several dimensions **together**; none is conclusive on its own.

| Dimension | What to look at | How to read it |
|---|---|---|
| **User-Agent** | HTTP client families, attack tools, empty or random strings. | First filter, but trivially spoofed. Never decide on UA alone. |
| **ISP / ASN** | Residential, mobile, cloud, hosting, corporate, anonymizers. | Residential and mobile networks share IPs (NAT): high volume from one IP can be many people. Cloud and hosting imply automation: is it yours or someone else's? |
| **Path** | Authentication, payments, admin, search, catalog, system files. | The more sensitive the path, the lower the tolerance for bot signals. |
| **Time** | Business hours, uniform rate, night-time spikes, slow growth. | A perfectly constant rate 24/7 is usually a script. A marked day/night cycle suggests people. |
| **Session behavior** | Cookies, tokens, requests per session, exact request repetition. | Sessions with no history or zeroed metrics plus ML signals are strong indicators. |

## 7. Safe Logic Patterns (Handling Missing Values)

The ABP rule engine is designed to **fail open**: when a value is missing, the condition evaluates to `nil` and the rule **does not fire**. This prevents accidental blocks, but it also means a badly written rule may **not block what you think it does**.

```
; WRONG: if there is no User-Agent, the condition is nil and does NOT fire
headers.user_agent == "curl"

; RIGHT: covers both absence and match
(any (not headers.user_agent?) (headers.user_agent starts_with "curl"))
```

```
; Order matters: check existence first, then read
(all headers.x_custom? (matches headers.x_custom re"^abc"))
```

| Topic | Remember |
|---|---|
| String literals | They require a **whole-string** match. For paths with variable parts, use regular expressions: `re"..."`. |
| Paths | The path arrives **decoded**. To detect encoding anomalies, use the non-normalized-path signal. |
| Negations | To negate a combination apply De Morgan: "neither A nor B" equals "not A **and** not B", not "not A **or** not B". |

```
; Example: path with a variable numeric identifier
(matches request.path re"^/api/v[0-9]+/orders/[0-9]+$")
```

## 8. Exceptions for Legitimate Bots Across Multiple Sites

To permit a legitimate bot (for example, an uptime monitor), combine **three conditions** and guard each against missing values:

```
(all
  headers.user_agent?
  headers.host?
  (in ip 203.0.113.0/24)
  (matches headers.user_agent re"^MonitorBot/")
  (headers.host == "app.example.com"))
```

- **IP ∧ User-Agent ∧ Host**: an attacker would need to spoof all three at once.
- For search engines, also verify **reverse DNS with forward confirmation**; User-Agent alone is not enough.
- Before creating the exception, check which mechanism keeps events. See [Exceptions Without Losing Visibility](../05-excepciones-y-tuning/exceptions-and-visibility.en.md).

## 9. Cookies and Subdomains

- `Domain=example.com` and `Domain=.example.com` are functionally identical (RFC 6265).
- A cookie **without** a `Domain` attribute is host-only and does **not** reach other subdomains. If the ABP token must travel across subdomains, review this attribute.

## 10. Common Mistakes

| Mistake | Consequence | Alternative |
|---|---|---|
| Geoblocking when the user base is international | Blocks travelers and legitimate customers. | ASN- and behavior-based controls. |
| Enabling data-center blocking without an inventory | You may block your own infrastructure hosted in the cloud. | Confirm an inventory of legitimate ASNs first. |
| Using a residential-proxy signal as a permanent, isolated hard block | Can teach the attacker to rotate identifiers. | Soft mitigation or combination with other signals (per guidance received from Imperva). |
| Basing rules solely on User-Agent or fingerprint | Evaded by rotating the value. | Behavioral rules (session metrics + ML signals). |
| Assuming the cascade to the default policy | Uncovered paths or unexpected coverage. | Verify in Passive. |
| Repeating directories within the same website group | Policy conflicts. | Unique directories. |
| Searching an IP with CIDR notation in console filters | UI filters accept an exact IP or free text, not CIDR. | Use CIDR only inside rules, not in search filters. |

## 11. Investigation Template

```
Policy / scope:
Site type (public / internal exposed / authentication / API):
Observation period:
Total / suspicious / mitigated volume (count and %):

--- User-Agent (top 3) ---
Classification:
--- ISP / ASN (top 3) ---
Classification (residential / mobile / cloud / hosting / anonymizer):
--- Target paths (top 3) ---
Sensitivity:
--- Time pattern ---
Uniform / spiky / business hours:
--- Session behavior ---

Verdict per condition:  [ ] Legitimate  [ ] Malicious  [ ] Mixed  [ ] No data
Overall verdict:
  [ ] Mostly false positives → tune
  [ ] Mixed → per-condition action
  [ ] Mostly malicious → policy is working
  [ ] Escalate → insufficient data
Recommended actions (phrased as evaluation, validated in pre-production):
```

## 12. Threat Context (to Make Sense of What You See)

| Technique | What it attempts | Typical signals | Reference |
|---|---|---|---|
| Credential stuffing | Trying stolen username/password pairs. | Many POSTs to authentication, no token, from hosting or residential proxies. | OWASP Automated Threats: *Credential Stuffing* |
| Scraping | Copying prices, inventory or content. | Repetitive GETs on catalog or search, uniform rate, one resource hit very hard. | OWASP Automated Threats: *Scraping* |
| Reconnaissance and scanning | Hunting for admin panels, config files and framework diagnostic endpoints. | "Dictionary" paths, from hosting, with generic or missing User-Agent. | OWASP Automated Threats: *Vulnerability Scanning* |
| Business-flow abuse | Automating bookings, purchases or account creation. | Sessions with no history, rotated fingerprints, off-hours spikes. | OWASP API Security Top 10: *Unrestricted Access to Sensitive Business Flows* |

**Field note:** the same hosting providers recur across unrelated incidents. Document the **ASN**, not just the IP: the pattern is what lets you anticipate.

## 13. Closing an Analysis

When reporting, use plain language and a fixed structure: **What was detected? → Source → Observed behavior → Context and risk → Status and next steps.** Present conclusions as **working hypotheses**, not confirmed facts, unless there is direct evidence.
