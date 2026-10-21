# Guía rápida: del síntoma al runbook / Quick guide: from symptom to runbook

Usa esta tabla cuando no sabes por dónde empezar. / Use this table when you are not sure where to start.

| Síntoma / Symptom | Primera pregunta / First question | Runbook |
|---|---|---|
| Una regla se activa sobre tráfico que parece legítimo / A rule fires on traffic that looks legitimate | ¿Qué dice la solicitud completa y quién la envía? / What does the full request say and who sends it? | 01 Falsos positivos *(planeado / planned)* |
| Alerta de DDoS con tráfico aparentemente normal / DDoS alert with seemingly normal traffic | ¿Cuánto se aleja de la línea base y qué ocurrió en el negocio? / How far is it from baseline and what happened in the business? | 02 DDoS *(planeado / planned)* |
| Mucho tráfico "sospechoso" en ABP y no sé si es dañino / Lots of "suspicious" ABP traffic, unsure if harmful | ¿Qué protege esa política y cómo es su tráfico normal? / What does that policy protect and what does its normal traffic look like? | [03 ABP](../03-bot-protection-abp/abp-despliegue-por-fases.es.md) · [EN](../03-bot-protection-abp/abp-phased-rollout.en.md) |
| Voy a activar ABP en un sitio nuevo / I am enabling ABP on a new site | ¿Tengo línea base e inventario de tráfico legítimo? / Do I have a baseline and a legitimate-traffic inventory? | [03 ABP](../03-bot-protection-abp/abp-despliegue-por-fases.es.md) |
| Necesito crear una excepción / I need to create an exception | ¿Puedo resolverlo sin apagar los logs? / Can I solve it without turning logging off? | [05 Excepciones](../05-excepciones-y-tuning/excepciones-y-visibilidad.es.md) · [EN](../05-excepciones-y-tuning/exceptions-and-visibility.en.md) |
| La aplicación falla (página en blanco, error 5xx, error TLS) y se sospecha del WAF / The app fails (blank page, 5xx, TLS error) and the WAF is suspected | ¿La solicitud llega al origen? / Does the request reach the origin? | [06 WAF vs. origen](../06-waf-vs-origen/waf-vs-origen.es.md) · [EN](../06-waf-vs-origen/waf-vs-origin.en.md) |
| Error de certificado tras una migración o rotación / Certificate error after a migration or rotation | ¿Qué certificado está sirviendo realmente el sitio? / Which certificate is the site actually serving? | [06 WAF vs. origen](../06-waf-vs-origen/waf-vs-origen.es.md) |
| Un inicio de sesión muestra muchos intentos sospechosos / A login shows many suspicious attempts | ¿Está el dominio de autenticación detrás del proxy? / Is the authentication domain behind the proxy? | 04 ATO *(planeado / planned)* |
| Hay que explicar un incidente a un área no técnica / An incident must be explained to a non-technical audience | ¿Qué pasó, qué riesgo hay y qué se hizo? / What happened, what is the risk and what was done? | 07 Reportes *(planeado / planned)* |
