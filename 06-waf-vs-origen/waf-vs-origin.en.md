# Is It the WAF or the Origin? Diagnosing Errors, mTLS and Certificates

| Field | Value |
|---|---|
| Platform | Imperva Cloud WAAP |
| Version | 0.1 (draft) |
| Last validated | October 2026 |
| Audience | WAF engineers, infrastructure and support teams |
| Language | English · [Español](waf-vs-origen.es.md) |

---

## 1. Purpose

When an application fails (blank page, 5xx errors, TLS errors), the WAF is the first suspect. Sometimes rightly so; often not. This runbook helps you **prove with evidence** where the chain breaks instead of presuming it.

> **Principle:** diagnose before acting and validate empirically. Present conclusions as working hypotheses until direct evidence exists.

## 2. Step 1 — Scope the Problem

| Question | Why it matters |
|---|---|
| Does it affect everyone or a subset? | A subset points to something that group has in common. |
| External users affected, internal or VPN users not? | Internal users often reach the origin through a different path that skips the WAF. |
| Only authenticated users? | May indicate an authentication issue at the origin, not filtering. |
| Constant or intermittent? | Intermittent suggests multiple nodes or load balancing with inconsistent configurations. |
| When did it start and what changed? | Look for overlap with application updates, certificate rotations or configuration changes. |

## 3. Step 2 — Follow the Request Hop by Hop

```
Client  ──►  Imperva (WAF/CDN)  ──►  Load balancer / Origin  ──►  Application
```

| Hop | Evidence to collect |
|---|---|
| **Client** | HAR file, response code and body, content type, size. |
| **Imperva** | Request event: was it **blocked** or **passed to the origin**? Confirm in your console the field equivalent to *"passed to server"*. |
| **Origin / load balancer** | Server logs: did the request arrive? With what status code and response time? |

**The deciding question:** *does the request reach the origin?* If Imperva passed it and the origin logs show the error, the problem is on the origin side.

> **Typical origin signature:** a response of **identical size** and **unexpected content type** for a static resource (for example, a small HTML error where a JavaScript file should be) points to a load balancer or origin, not the WAF.

## 4. Case — Intermittent Errors Only for Authenticated Users (NTLM)

**Symptom.** External users saw a blank page after authenticating; internal users did not. The issue was intermittent and began after an application update.

**Initial hypotheses (ruled out with evidence).** Load balancing across nodes with inconsistent deployments, or WAF blocking. The application vendor stated requests never reached their server.

**Confirmed root cause.** Imperva's *Origin Connection Reuse* setting **multiplexes** TCP connections to the origin. **NTLM** authentication needs to complete its entire handshake over **a single TCP connection**; once the connection was shared, authentication broke. IIS logs showed hundreds of HTTP 500 errors (`sc-win32-status 1346`, `ERROR_BAD_IMPERSONATION_LEVEL`) **exclusively** for authenticated users.

**Fix.** Disable *Origin Connection Reuse* for that site. It was live-tested with Imperva support across multiple users and devices without reproducing the issue.

**Recommendations.**

- Confirm with Imperva support whether this setting will remain configurable in your version; Imperva has communicated planned changes to it.
- Long term, evaluate moving load-balancer authentication from **NTLM to Kerberos**.

**Lesson:** a cause that looked like an origin problem turned out to be an interaction between the WAF and the authentication protocol. Origin log evidence closed the case.

## 5. mTLS: The Three Mandatory Questions

Before touching anything in an mTLS case, answer:

1. Is the problem **Client → Imperva**, **Imperva → Origin**, or both?
2. Does the origin validate the certificate in the **TLS handshake** or through an **HTTP header**?
3. Was there a recent **certificate upload or rotation**? Dates often coincide with the incident start.

### Facts worth having straight

| Fact | Consequence |
|---|---|
| The option to **send client certificate details to the origin** only **forwards headers**; it does **not** perform real mTLS. | It does not satisfy an origin that requires mTLS. |
| For **Imperva → Origin** mTLS, the certificate is uploaded in Imperva's origin mTLS section. | Without that upload, the origin receives no valid client certificate. |
| Some load balancers validate the certificate **at the handshake**, not via headers. | Forwarding headers does not help. |
| Two TLS renegotiations followed by an abrupt close. | The server **requires** a client certificate that the client **is not presenting**. |
| On Windows, `CRYPT_E_NO_REVOCATION_CHECK`. | Usually a false positive: the system cannot reach the internal revocation list. |

### Helpful commands

```
# Test with a client certificate (on Windows, skipping revocation checks)
curl -v --ssl-no-revoke --cert client.pem --key client.key https://app.example.com/

# Inspect the certificate chain the server presents
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts
```

## 6. Certificate Migration

**Symptom.** After moving a site from an account-level certificate to a per-site one, SSL fails.

**Observed cause.** If the *Imperva generated certificate* option is **disabled** (in the site's general settings, SSL Support section), the site falls back to a custom certificate and SSL breaks.

**What to do.**

- Check that option in the site settings. **Looking at the certificate type in the certificate list is not enough.**
- To validate chain compatibility, confirm with an external TLS analysis tool that **two certification paths** exist where applicable.

## 7. Beware of Performance Metrics

The performance dashboard's "client errors" reflect **HTTP behavior**, not TLS cipher compatibility. If the handshake fails before HTTP is processed, the WAF **cannot detect** that client-side failure. Do not use those metrics to rule out a TLS problem.

## 8. Imperva-Side Validation Checklist

Useful to formally document that Imperva is not the cause (for example, when a vendor or team questions it):

```
[ ] The affected request appears in Imperva events
[ ] It shows as passed to the origin (not blocked)
[ ] No security or ABP rules triggered on that request
[ ] Redirect, rewrite and cache rules applying to the path were reviewed
[ ] The certificate the site serves and its SSL configuration were confirmed
[ ] The origin connection setting (connection reuse) was reviewed
[ ] Behavior was compared with and without going through Imperva (if possible)
[ ] Cross-checked against origin logs (status, size, time)
[ ] Verified twice in different time windows
```

## 9. How to Report It

Present the result as **your own technical analysis**, with working hypotheses and attached evidence. Suggested structure: **What was detected? → Source → Observed behavior → Context and risk → Status and next steps.** If the fix depends on a configuration change, state that it will be validated outside production before being applied.
