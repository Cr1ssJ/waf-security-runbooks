# Imperva WAAP Runbooks

> **ES:** Guías operativas para ingenieros de WAF que trabajan con Imperva Cloud WAAP: cómo investigar, desplegar y mantener protección sin romper el tráfico legítimo.
> **EN:** Operational runbooks for WAF engineers working with Imperva Cloud WAAP: how to investigate, deploy and maintain protection without breaking legitimate traffic.

**Estado / Status:** v0.1 (borrador para revisión / draft for review) · **Última validación / Last validated:** octubre 2026 / October 2026

---

## Español

### ¿Qué es este repositorio?

Una colección de runbooks basada en operación real de Imperva Cloud WAAP (WAF, Advanced Bot Protection, ATO y DDoS) para clientes enterprise. No reescribe la documentación del proveedor: documenta la **lógica de decisión** y los **flujos de investigación** que se usan a diario para distinguir amenazas reales de ruido, desplegar controles por fases y comunicar el riesgo a personas que no son del área técnica.

### ¿Para quién es?

- Ingenieros y analistas de WAF o seguridad de aplicaciones.
- Analistas SOC que reciben alertas de tráfico web.
- Equipos de seguridad que reciben reportes de un proveedor de servicios gestionados y quieren entenderlos.

### Principios que guían todos los runbooks

| # | Principio | En una frase |
|---|---|---|
| 1 | Evaluar antes de remediar | Primero se entiende qué ocurre; después se decide qué cambiar. |
| 2 | Línea base primero | Sin saber cómo es el tráfico normal, cualquier regla es una apuesta. |
| 3 | Passive/Alert → acción activa (CAPTCHA o Block) | Primero se observa sin aplicar ninguna acción; después se actúa. Nunca se salta ese paso. |
| 4 | Validar fuera de producción | Los falsos positivos se descubren en pruebas, no con usuarios reales. |
| 5 | Inventario legítimo antes de bloquear | Hay que saber qué tráfico propio y de terceros es válido antes de activar un bloqueo. |
| 6 | Preservar la visibilidad | Una excepción que apaga los logs es una zona ciega. |
| 7 | Evidencia sobre suposiciones | Se demuestra dónde está el problema; no se presume. |

### Índice

| # | Runbook | Estado | Español | English |
|---|---|---|---|---|
| 00 | Guía rápida: síntoma → runbook | Disponible | [Abrir](00-guia-rapida/sintoma-a-runbook.md) | (mismo archivo / same file) |
| 01 | Falsos positivos | Planeado | — | — |
| 02 | Alertas de DDoS | Planeado | — | — |
| 03 | ABP: despliegue por fases y clasificación de tráfico | Disponible | [Abrir](03-bot-protection-abp/abp-despliegue-por-fases.es.md) | [Open](03-bot-protection-abp/abp-phased-rollout.en.md) |
| 04 | ATO (Account Takeover) | Planeado | — | — |
| 05 | Excepciones sin perder visibilidad | Disponible | [Abrir](05-excepciones-y-tuning/excepciones-y-visibilidad.es.md) | [Open](05-excepciones-y-tuning/exceptions-and-visibility.en.md) |
| 06 | ¿Es el WAF o es el origen? | Disponible | [Abrir](06-waf-vs-origen/waf-vs-origen.es.md) | [Open](06-waf-vs-origen/waf-vs-origin.en.md) |
| 07 | Reportes para áreas no técnicas | Planeado | — | — |

### Política de anonimización

Todo el contenido proviene de experiencia real, pero **no contiene datos de clientes**:

- Sin nombres de clientes, personas, dominios, subdominios, IP reales, identificadores de cuenta o de sitio, números de caso ni capturas con datos reales.
- Los casos se describen como **patrones** ("un sitio interno expuesto a internet", "un balanceador de carga en el origen").
- Los ejemplos usan dominios `example.com` y rangos de IP reservados para documentación (RFC 5737).

### Aviso importante

Estos runbooks reflejan experiencia operativa y **no representan la recomendación oficial del proveedor**. Los nombres de campos, flags, opciones y rutas de la consola pueden cambiar entre versiones: valida siempre en tu cuenta y en la documentación oficial antes de aplicar un cambio en producción.

---

## English

### What is this repository?

A set of runbooks based on real-world operation of Imperva Cloud WAAP (WAF, Advanced Bot Protection, ATO and DDoS) for enterprise customers. It does not rewrite vendor documentation: it documents the **decision logic** and **investigation workflows** used day to day to tell real threats from noise, roll out controls in phases and explain risk to non-technical stakeholders.

### Who is it for?

- WAF and application security engineers.
- SOC analysts handling web traffic alerts.
- Security teams receiving reports from a managed service provider who want to understand them.

### Guiding principles

| # | Principle | In one sentence |
|---|---|---|
| 1 | Evaluate before remediating | Understand what is happening first; decide what to change afterwards. |
| 2 | Baseline first | Without knowing what normal traffic looks like, every rule is a guess. |
| 3 | Passive/Alert → active action (CAPTCHA or Block) | Observe first without applying any action; then act. Never skip that step. |
| 4 | Validate outside production | False positives should surface in testing, not with real users. |
| 5 | Legitimate inventory before blocking | Know which first- and third-party traffic is valid before enabling a block. |
| 6 | Preserve visibility | An exception that turns logging off is a blind spot. |
| 7 | Evidence over assumptions | Prove where the problem is; do not presume it. |

### Anonymization policy

All content comes from real experience but **contains no customer data**: no customer, person, domain, real IP, account/site identifier, case number or screenshot with real data. Cases are described as **patterns**, and examples use `example.com` and IP ranges reserved for documentation (RFC 5737).

### Important notice

These runbooks reflect operational experience and **do not represent the vendor's official recommendations**. Field names, flags, options and console paths may change between versions: always validate in your own account and in the official documentation before changing production.

---

## Contribuir / Contributing

Las contribuciones son bienvenidas. Abre primero un *issue* para acordar el alcance. / Contributions are welcome; please open an issue first to agree on scope.

## Licencia / License

Contenido bajo [CC BY 4.0](LICENSE.md) (propuesta; confirmar al publicar / proposed; confirm on publishing).

## Autor / Author

**Cristian Jiménez** — WAF Engineer · [LinkedIn](https://www.linkedin.com/in/cristian-jimenez1603/) · [GitHub](https://github.com/Cr1ssJ)
