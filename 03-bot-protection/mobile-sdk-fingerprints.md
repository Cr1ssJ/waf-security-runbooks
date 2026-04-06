# Mobile SDK Fingerprints — ABP Reference

**Category:** Bot Management / False Positive Prevention  
**Platform:** Imperva Advanced Bot Protection  
**Purpose:** Quick reference for identifying legitimate mobile SDK traffic in ABP policies

---

## 🎯 Why This Matters

Mobile applications communicate with APIs using HTTP client libraries (SDKs) that don't behave like browsers. They don't execute JavaScript, don't carry browser cookies by default, and have distinct User-Agent patterns. This makes them appear "bot-like" to ABP systems.

Knowing these fingerprints prevents you from blocking or challenging your own users.

---

## 📱 Android SDK Fingerprints

### OkHttp (Most common Android HTTP client)
```
User-Agent: okhttp/3.x.x
User-Agent: okhttp/4.x.x
```
- **Used by:** Most Android apps, Retrofit (which wraps OkHttp), many popular apps
- **Characteristics:** No JavaScript execution, consistent UA format, typically no browser cookies
- **ABP behavior:** Will trigger `no_token` — this is expected
- **Risk level:** Low (when from residential/mobile ASN on expected API endpoints)

### Volley (Google's Android HTTP library)
```
User-Agent: Dalvik/x.x.x (Linux; U; Android x.x; [Device] Build/[Build])
```
- **Used by:** Older Android apps, Google's own apps
- **Characteristics:** UA includes device model and Android version
- **Risk level:** Low

---

## 🍎 iOS SDK Fingerprints

### Alamofire (Most common iOS HTTP client)
```
User-Agent: MyApp/1.0 Alamofire/5.x.x
User-Agent: Alamofire/5.x.x
```
- **Used by:** Most Swift-based iOS applications
- **Characteristics:** Often includes app name + Alamofire version
- **ABP behavior:** Will trigger `no_token`
- **Risk level:** Low (when from mobile ASN on expected API endpoints)

### URLSession (Native iOS networking)
```
User-Agent: MyApp/1.0 CFNetwork/1.x Darwin/x.x.x
```
- **Used by:** iOS apps using native networking without a third-party library
- **Characteristics:** CFNetwork is Apple's native networking framework
- **Risk level:** Low

### Apple Wallet / Passbook (`passd`)
```
User-Agent: passd/[version] CFNetwork/[version] Darwin/[version]
```
- **Used by:** iOS Wallet passes — triggered when users add/update passes
- **Characteristics:** Very specific pattern, low volume, predictable behavior
- **ABP behavior:** Will trigger `no_token` — legitimate if app has Wallet integration
- **Risk level:** Very low

---

## 🌐 Cross-Platform / Web SDK Fingerprints

### Axios (JavaScript HTTP client)
```
User-Agent: axios/0.x.x
User-Agent: axios/1.x.x
```
- **Used by:** React Native apps, Node.js backends, frontend JavaScript
- **Characteristics:** Very clean UA format
- **Risk level:** Low to medium — could also be used by scrapers

### React Native default
```
User-Agent: [AppName]/[version] (react-native/[version]; [platform])
```
- **Used by:** React Native mobile applications
- **Risk level:** Low

### Flutter (cross-platform apps)
```
User-Agent: Dart/x.x (dart:io)
```
- **Used by:** Flutter-based iOS and Android apps
- **Characteristics:** Dart runtime identifier
- **Risk level:** Low

---

## 🖥️ Backend / Service SDK Fingerprints

These may appear in API policies when internal services call APIs:

| SDK | User-Agent pattern | Common use |
|---|---|---|
| Python requests | `python-requests/x.x.x` | Internal tools, data pipelines |
| Go HTTP client | `Go-http-client/1.1` or `Go-http-client/2.0` | Microservices, internal APIs |
| Java HttpClient | `Java/x.x.x` or `Apache-HttpClient/x.x.x` | Backend services, Spring apps |
| Node.js fetch/got | `node-fetch/x.x.x` or `got/x.x.x` | BFF services, API gateways |
| curl | `curl/x.x.x` | Developer testing, shell scripts, monitoring |

---

## ⚠️ Spoofing Awareness

Attackers can spoof any User-Agent string. A `okhttp` User-Agent does NOT guarantee the request is from a legitimate Android app.

**Signals that a "mobile SDK" UA may be spoofed:**

| Signal | Description |
|---|---|
| Cloud/VPS ASN | Legitimate mobile apps come from mobile carriers, not AWS |
| Perfectly uniform request rate | Real mobile users are irregular; bots are precise |
| Targeting sensitive endpoints | Scrapers/stuffers spoof mobile UAs to avoid detection |
| No version variation | All requests from identical UA version = likely single tool |
| High sequential parameter patterns | Enumeration — real app users don't do this |

**Always combine UA analysis with ASN, endpoint, and behavioral analysis — never rely on UA alone.**

---

## 🔗 Related Runbooks

- [ABP Traffic Classification](./abp-traffic-classification.md)
- [False Positive Triage](../01-false-positive-diagnosis/waf-false-positive-triage.md)
