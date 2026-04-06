# DDoS Alert Triage — Burst Detection vs. Real Attack

**Category:** DDoS / Volumetric Threat Analysis  
**Platform:** Imperva Cloud WAAP  
**Severity Impact:** High — misclassifying a burst as an attack (or vice versa) leads to either over-reaction or under-response

---

## 🎯 Purpose

Not every DDoS alert is a DDoS attack. WAF platforms like Imperva use statistical burst detection that can fire on **legitimate traffic spikes** — flash sales, app launches, marketing campaigns, or even poorly configured health checks.

This runbook helps you quickly distinguish:
- **Burst detection:** Imperva's threshold-based detection firing on a legitimate spike
- **Real DDoS attack:** Volumetric or application-layer attack targeting your infrastructure

---

## 🔄 Triage Decision Tree

```
DDoS alert received
        │
        ▼
1. What type of alert? (Network/Volume vs. Application Layer)
        │
        ├── Network / Volumetric (L3/L4) ──► Go to Section A
        │
        └── Application Layer (L7 / HTTP flood) ──► Go to Section B
```

---

## Section A — Volumetric / Network Layer Alert

### Key questions:

| Question | Attack signal | Burst / FP signal |
|---|---|---|
| Traffic volume vs. baseline | 10x–100x+ above normal | 2x–5x, explainable by event |
| Traffic protocol distribution | Unusual mix (UDP flood, SYN flood) | Normal HTTP/S distribution |
| Source IP diversity | Thousands of unique IPs (botnet) | Concentrated in known ISPs |
| Geographic origin | Unexpected countries | Matches normal user base |
| Payload variety | Randomized to evade signature | Consistent, normal request structure |
| Duration | Sustained over minutes/hours | Short burst (<5 min), then normalizes |

### Imperva-specific checks:
- Navigate to DDoS protection dashboard → review the **attack timeline**
- Check if Imperva automatically activated **DDoS mitigation mode**
- Look at the **traffic composition** tab — what percentage is flagged as attack traffic?
- Review the **top source IPs** and their reputation scores

---

## Section B — Application Layer (L7) Alert

Application layer DDoS is harder to distinguish from legitimate traffic because it uses valid HTTP requests.

### Behavioral analysis:

**Indicators pointing to REAL attack:**
- Requests target a resource-intensive endpoint (search, login, checkout, API endpoints that hit the DB)
- Very high rate from a narrow set of IPs or ASNs
- Requests have no session context (no cookies, no referrer chain)
- User-Agent strings are generic, randomized, or missing
- Requests arrive at perfectly uniform intervals (automated tool signature)
- Same exact request repeated at high rate (low entropy in request parameters)

**Indicators pointing to LEGITIMATE spike:**
- Traffic spike correlates with a known event (product launch, email campaign, news coverage)
- Requests have valid session structure (cookies, referrers, varied parameters)
- User-Agent distribution matches normal browser/app fingerprints
- Traffic normalizes on its own within 10–30 minutes
- Source IPs are diverse residential/mobile IPs

---

## 🔧 Threshold Context

> **Key operational principle:** A low-volume DDoS alert on Imperva does NOT automatically mean a low-severity event, and it does NOT automatically mean a real attack.

Imperva uses **dynamic thresholds** based on historical traffic baselines for each protected site. The threshold is automatically calculated — which means:

- A site with normally low traffic (e.g., 100 req/s) can trigger a DDoS alert at 300 req/s — a volume that would be invisible on a high-traffic site
- **Raising the threshold** prevents these alerts but creates blind spots for low-and-slow attacks
- **Lowering the threshold** increases sensitivity but floods the SOC with false positives

### Threshold change impact matrix:

| Change | Effect on detection | Effect on alert volume | Risk |
|---|---|---|---|
| Raise threshold | Less sensitive to small bursts | Fewer alerts | May miss legitimate attacks below new threshold |
| Lower threshold | More sensitive | More alerts (many FP) | Alert fatigue, SOC desensitization |
| Keep current + context | Balanced | Moderate | Requires analyst judgment per alert |

**Always document threshold changes and monitor for 48–72h after any adjustment.**

---

## 📊 Investigation Checklist

```
[ ] Alert timestamp noted — what was happening in the application at that time?
[ ] Alert type identified (volumetric vs. application layer)
[ ] Traffic volume compared to 7-day and 30-day baseline
[ ] Source IP / ASN distribution reviewed
[ ] User-Agent distribution reviewed
[ ] Target endpoint(s) identified — are they resource-intensive?
[ ] Geographic origin checked — expected or anomalous?
[ ] Mitigation auto-activated? If yes, is it still active?
[ ] Application team notified / queried about any campaigns or events
[ ] Verdict reached: Attack / Burst / Inconclusive
```

---

## 📝 Alert Classification Output

After completing the checklist, document your verdict:

```
Alert ID / Timestamp:
Alert type:
Peak traffic volume:
Baseline traffic volume:
Source characteristics:
Target endpoint(s):
Contextual events (if any):
Mitigation action taken (auto / manual / none):
Verdict: [ ] Real attack  [ ] Legitimate burst  [ ] Inconclusive
Rationale:
Follow-up actions:
```

---

## ⚙️ Post-Incident: Threshold Review Decision

After handling a confirmed false positive burst, evaluate whether a threshold adjustment is warranted:

| Scenario | Recommendation |
|---|---|
| First occurrence, no clear pattern | Monitor — do not change threshold yet |
| Recurring alerts from same known source | Consider selective threshold adjustment for that site |
| Business event generates consistent spikes | Coordinate with application team to set pre-emptive exceptions |
| Legitimate traffic permanently higher | Review and raise threshold with documented justification |

---

## 🔗 Related Runbooks

- [False Positive Triage](../01-false-positive-diagnosis/waf-false-positive-triage.md)
- [ABP Traffic Classification](../03-bot-protection/abp-traffic-classification.md)
- [Incident Report Template](../05-incident-response/incident-report-template.md)
