# WAF Security Runbooks

> Operational runbooks for Web Application Firewall engineers — covering false positive diagnosis, DDoS alert triage, rule tuning, and incident response workflows. Platform-agnostic methodology with references to Imperva WAAP.

---

## 📋 What's in this repository?

This repository contains practical, opinionated runbooks derived from real-world WAF operations. These are not vendor documentation rewrites — they document the *decision logic* and *investigation workflows* that WAF engineers use daily to distinguish noise from real threats, tune policies without creating blind spots, and respond to incidents efficiently.

**Who is this for?**
- WAF / Application Security Engineers
- Security Operations Center (SOC) Analysts dealing with web traffic alerts
- Cloud Security professionals managing WAAP platforms
- Anyone preparing for roles involving Imperva, Cloudflare, or similar platforms

---

## 📁 Repository Structure

```
waf-security-runbooks/
│
├── README.md
│
├── 01-false-positive-diagnosis/
│   └──  waf-false-positive-triage.md
│   
│   
│
├── 02-ddos-alert-triage/
│   └── ddos-burst-vs-attack.md
│   
│   
│
├── 03-bot-protection/
│   ├── abp-traffic-classification.md
│   └── mobile-sdk-fingerprints.md
│   
│
├── 04-waf-rule-tuning/
│   └── allowlist-decision-framework.md
│   
│   
│
├── 05-incident-response/
│   └── incident-report-template.md

```

---

## 🗂️ Runbook Index

| # | Runbook | Description |
|---|---|---|
| 01 | [False Positive Diagnosis](./01-false-positive-diagnosis/) | Structured triage workflow to confirm or rule out false positives in WAF alerts |
| 02 | [DDoS Alert Triage](./02-ddos-alert-triage/) | How to distinguish Imperva burst detections from real volumetric attacks |
| 03 | [Bot Protection & ABP](./03-bot-protection/) | ABP traffic classification — mobile SDKs, good bots, and malicious patterns |
| 04 | [WAF Rule Tuning](./04-waf-rule-tuning/) | Methodology for tuning WAF rules without creating security blind spots |
| 05 | [Incident Response](./05-incident-response/) | Multi-vector attack response flow and client-facing report templates |

---

## 🔧 Platform Coverage

| Platform | Coverage Level |
|---|---|
| **Imperva Cloud WAAP** | Primary — most runbooks reference Imperva concepts and terminology |
| **Cloudflare WAF** | Secondary — equivalent concepts noted where applicable |
| **Generic / Platform-agnostic** | All core methodologies apply across platforms |

---

## ⚠️ Important Notes

- No client data, proprietary configurations, or sensitive information is included in any runbook.
- All examples use synthetic/anonymized traffic patterns.
- Runbooks reflect operational experience and may not represent the vendor's official recommended approach.

---

## 🤝 Contributing

Pull requests are welcome. If you have operational experience with WAF platforms and want to contribute a runbook or correction, please open an issue first to discuss the scope.

---

*Author: Cristian Jiménez — WAF Engineer | Cloud Security*  
*[LinkedIn](https://www.linkedin.com/in/cristian-jimenez1603/) · [GitHub](https://github.com/Cr1ssJ)*
