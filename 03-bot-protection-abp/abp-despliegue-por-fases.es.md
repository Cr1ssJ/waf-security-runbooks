# ABP: despliegue por fases y clasificación de tráfico

| Campo | Valor |
|---|---|
| Plataforma | Imperva Cloud WAAP — Advanced Bot Protection (ABP) |
| Versión | 0.1 (borrador) |
| Última validación | Octubre 2026 |
| Audiencia | Ingenieros de WAF y analistas SOC |
| Idioma | Español · [English](abp-phased-rollout.en.md) |

---

## 1. Para qué sirve

ABP distingue a las personas de la automatización. El objetivo **no es bloquear bots**, sino bloquear los bots **dañinos** sin afectar a usuarios reales, buscadores legítimos ni integraciones propias.

Este runbook explica cómo desplegar ABP en fases, cómo clasificar el tráfico que aparece como sospechoso y qué errores evitar. Su idea central: **un bot no es sinónimo de ataque**; lo que importa es si ese tráfico automatizado es legítimo para *esa* aplicación.

## 2. Principios

1. Diagnosticar antes de actuar.
2. Construir una línea base de tráfico antes de crear reglas.
3. Avanzar siempre en el orden **Passive → Alert/CAPTCHA → Block**, sin saltar fases.
4. Validar los falsos positivos en un ambiente de pruebas antes de pasar a producción.
5. Identificar el tráfico legítimo (propio y de terceros) antes de activar condiciones de bloqueo.
6. Preferir señales de **comportamiento** sobre firmas fáciles de rotar.

## 3. Alcance de las políticas

| Aspecto | Qué hacer |
|---|---|
| Granularidad | Preferir políticas por **ruta o directorio** para delimitar bien el alcance. Para sitios internos con audiencia conocida y restringida, el alcance **por sitio** suele ser más simple y seguro. |
| Unicidad | Los directorios deben ser **únicos** entre todos los sitios de un mismo grupo de sitios (*website group*). Un directorio repetido genera conflictos. |
| Política por defecto | No asumas cómo cae el tráfico hacia la política por defecto: **compruébalo de forma empírica en modo Passive** antes de depender de ello. |
| Sitios agrupados | Dos sitios con tráfico distinto pueden compartir grupo por decisión operativa, pero eso limita lo que puedes afinar por separado. Documenta la decisión y reevalúala cuando los sitios maduren. |

## 4. Roadmap por pilares

Una forma ordenada de planificar ABP es dividirlo en pilares. Cada uno se recorre con las fases de la sección 5.

| Pilar | Qué busca | Lógica sugerida | Precauciones |
|---|---|---|---|
| **1. Higiene de tráfico** | Descartar solicitudes que ningún navegador real genera: sin User-Agent, rutas mal normalizadas, ausencia de token, origen en proveedores cloud. | Evaluar cada señal por separado en Passive y decidir después cómo combinarlas. | La ausencia de token es normal en integraciones sin navegador. Los proveedores cloud pueden alojar infraestructura **propia**: confirma un inventario de ASN legítimos antes de bloquear. |
| **2. Detección de manipulación** | Tráfico que altera su propio token o su interrogación (por ejemplo, checksums inválidos o manipulación del desafío). | `any` (OR): basta **una** señal. | Son señales de alta confianza, pero igual se validan en Passive primero. |
| **3. Detección de emulación de entorno** | Navegadores automatizados o que mienten sobre su identidad (impostores, *webdriver*, herramientas de evasión). | Combinar con otras señales; las de menor precisión se **etiquetan (Tag)** y se actúa sobre la etiqueta. | Algunas señales de manipulación de huella tienen falsos positivos conocidos en dispositivos poco comunes. Ejemplos de señales a revisar (confirma los nombres en tu consola): `browser_impostor`, `fingerprint_tampering_*`. |
| **4. Separación de ambientes** | Evitar que el tráfico de pruebas contamine producción. | Depende de la arquitectura. | Es **opcional** y puede quedar en pausa por decisión del dueño de la aplicación. |

> **Regla práctica:** usa `any` (OR) para detectar **manipulación** y `all` (AND) para **fraude correlacionado**. Así reduces los falsos positivos donde una sola señal no basta.

## 5. Fases de despliegue

El criterio para avanzar es la **evidencia**, no el calendario. La duración de cada fase depende del volumen de tráfico del sitio.

| Fase | Qué se hace | Criterio de salida |
|---|---|---|
| **0. Línea base** | Inventariar el tráfico normal con las cinco dimensiones (sección 6): User-Agent, ISP/ASN, rutas, horarios y comportamiento de sesión. Identificar integraciones propias y de terceros. | Existe un inventario documentado de tráfico legítimo y de tráfico desconocido. |
| **1. Passive** | Activar las condiciones sin ningún efecto sobre el tráfico. Solo se observa y se mide. | Se entiende qué volumen y qué tipo de tráfico marca cada condición. |
| **2. Alert / CAPTCHA** | Las condiciones de confianza media pasan a alertar o a desafiar con CAPTCHA. Se analizan los falsos positivos. | Los falsos positivos están explicados y tratados (excepción mínima, ajuste de regla o corrección en la aplicación). |
| **3. Block** | Solo las condiciones de **alta confianza**, ya validadas en pruebas, pasan a bloqueo. | El bloqueo no afecta a tráfico legítimo durante el periodo de observación acordado. |
| **4. Mantenimiento** | Revisiones periódicas: ¿siguen vigentes las excepciones?, ¿cambió la aplicación?, ¿aparecieron nuevas técnicas? | Revisión documentada. |

**Buenas prácticas de control:**

- Lleva un tracker con hojas separadas de **Pre-Producción** y **Producción**, más un resumen y una leyenda.
- Todo cambio se valida en preproducción antes de producción.
- Al comunicar a un cliente, formula la recomendación como: *"evaluar condicionantes adicionales según el comportamiento observado"*, y menciona siempre que se validará antes de llevarla a producción.

## 6. Las cinco dimensiones de análisis

Analiza siempre varias dimensiones **juntas**; ninguna por sí sola es concluyente.

| Dimensión | Qué mirar | Cómo interpretarlo |
|---|---|---|
| **User-Agent** | Familias de clientes HTTP, herramientas de ataque, cadenas vacías o aleatorias. | Es el primer filtro, pero se falsifica con facilidad. Nunca decidas solo por el UA. |
| **ISP / ASN** | Residencial, móvil, cloud, hosting, corporativo, anonimizadores. | Redes residenciales y móviles comparten IP (NAT): mucho volumen desde una IP puede ser mucha gente. Cloud y hosting indican automatización: ¿es la tuya o es ajena? |
| **Ruta** | Autenticación, pagos, administración, búsqueda, catálogo, archivos de sistema. | Cuanto más sensible la ruta, menor la tolerancia a señales de bot. |
| **Tiempo** | Horarios de oficina, ritmo uniforme, picos nocturnos, crecimiento lento. | Un ritmo perfectamente constante 24/7 suele ser un script. Un ciclo día/noche marcado sugiere personas. |
| **Comportamiento de sesión** | Cookies, tokens, cantidad de solicitudes por sesión, repetición exacta de peticiones. | Sesiones sin historial o con métricas en cero junto con señales de ML son indicadores fuertes. |

## 7. Patrones de lógica seguros (manejo de valores ausentes)

El motor de reglas de ABP está diseñado para **fallar en abierto**: cuando un dato falta, la condición vale `nil` y la regla **no se activa**. Esto evita bloqueos accidentales, pero también significa que una regla mal escrita puede **no bloquear lo que crees**.

```
; INCORRECTO: si no existe User-Agent, la condición vale nil y NO se activa
headers.user_agent == "curl"

; CORRECTO: cubre tanto la ausencia como la coincidencia
(any (not headers.user_agent?) (headers.user_agent starts_with "curl"))
```

```
; Orden importa: primero comprobar que existe, después leerlo
(all headers.x_custom? (matches headers.x_custom re"^abc"))
```

| Tema | Qué recordar |
|---|---|
| Cadenas literales | Exigen coincidencia de la cadena **completa**. Para rutas con partes variables usa expresiones regulares: `re"..."`. |
| Rutas | La ruta llega **decodificada**. Para detectar anomalías de codificación usa la señal de ruta no normalizada. |
| Negaciones | Para negar una combinación aplica De Morgan: «ni A ni B» equivale a «no A **y** no B», no a «no A **o** no B». |

```
; Ejemplo: ruta con identificador numérico variable
(matches request.path re"^/api/v[0-9]+/orders/[0-9]+$")
```

## 8. Excepciones para bots legítimos en varios sitios

Para permitir un bot legítimo (por ejemplo, un monitor de disponibilidad) combina **tres condiciones** y protege cada una contra valores ausentes:

```
(all
  headers.user_agent?
  headers.host?
  (in ip 203.0.113.0/24)
  (matches headers.user_agent re"^MonitorBot/")
  (headers.host == "app.example.com"))
```

- **IP ∧ User-Agent ∧ Host**: un atacante necesitaría falsificar las tres a la vez.
- Para buscadores, verifica además el **DNS inverso con confirmación directa**; el User-Agent solo no basta.
- Prefiere la acción **Tag** sobre Allow para conservar visibilidad. Ver [Excepciones sin perder visibilidad](../05-excepciones-y-tuning/excepciones-y-visibilidad.es.md).

## 9. Cookies y subdominios

- `Domain=ejemplo.com` y `Domain=.ejemplo.com` son funcionalmente idénticos (RFC 6265).
- Una cookie **sin** atributo `Domain` queda ligada a un solo *host* y **no** llega a otros subdominios. Si necesitas que el token de ABP circule entre subdominios, revisa este atributo.

## 10. Errores frecuentes

| Error | Consecuencia | Alternativa |
|---|---|---|
| Bloquear por país cuando la base de usuarios es internacional | Se bloquea a viajeros y clientes legítimos. | Controles por ASN y comportamiento. |
| Activar el bloqueo de centros de datos sin inventario | Puedes bloquear infraestructura propia alojada en cloud. | Confirmar primero un inventario de ASN legítimos. |
| Usar una señal de redes residenciales o proxies como bloqueo duro permanente y aislado | Puede enseñar al atacante a rotar identificadores. | Mitigación suave o combinada con otras señales (según orientación recibida de Imperva). |
| Basar reglas solo en User-Agent o huella digital | Se evade rotando el valor. | Reglas de comportamiento (métricas de sesión + señales de ML). |
| Dar por hecha la cascada hacia la política por defecto | Rutas sin cobertura o con cobertura inesperada. | Verificarlo en Passive. |
| Repetir directorios dentro de un mismo grupo de sitios | Conflicto entre políticas. | Directorios únicos. |
| Buscar una IP con notación CIDR en los filtros de la consola | Los filtros de la interfaz aceptan IP exacta o texto libre, no CIDR. | Usar CIDR solo dentro de las reglas, no en los filtros de búsqueda. |

## 11. Plantilla de investigación

```
Política / alcance:
Tipo de sitio (público / interno expuesto / autenticación / API):
Periodo observado:
Volumen total / sospechoso / mitigado (cantidad y %):

--- User-Agent (top 3) ---
Clasificación:
--- ISP / ASN (top 3) ---
Clasificación (residencial / móvil / cloud / hosting / anonimizador):
--- Rutas objetivo (top 3) ---
Sensibilidad:
--- Patrón temporal ---
Uniforme / con picos / horario laboral:
--- Comportamiento de sesión ---

Veredicto por condición:  [ ] Legítimo  [ ] Malicioso  [ ] Mixto  [ ] Sin datos
Veredicto general:
  [ ] Mayormente falsos positivos → ajustar
  [ ] Mixto → acción por condición
  [ ] Mayormente malicioso → la política funciona
  [ ] Escalar → datos insuficientes
Acciones recomendadas (formuladas como evaluación, validadas en preproducción):
```

## 12. Contexto de amenazas (para dar sentido a lo que ves)

| Técnica | Qué intenta | Señales típicas | Referencia |
|---|---|---|---|
| Relleno de credenciales (*credential stuffing*) | Probar combinaciones de usuario y contraseña robadas. | Muchas peticiones POST a autenticación, sin token, desde hosting o proxies residenciales. | OWASP Automated Threats: *Credential Stuffing* |
| Extracción masiva de datos (*scraping*) | Copiar precios, inventario o contenido. | GET repetitivo sobre catálogo o búsqueda, ritmo uniforme, un solo recurso muy golpeado. | OWASP Automated Threats: *Scraping* |
| Reconocimiento y escaneo | Buscar paneles de administración, archivos de configuración y endpoints de diagnóstico de frameworks. | Rutas de «diccionario», desde hosting, con User-Agent genérico o ausente. | OWASP Automated Threats: *Vulnerability Scanning* |
| Abuso de flujos de negocio | Automatizar reservas, compras o creación de cuentas. | Sesiones sin historial, huellas rotadas, picos fuera de horario. | OWASP API Security Top 10: *Unrestricted Access to Sensitive Business Flows* |

**Dato de campo:** los mismos proveedores de hosting se repiten entre incidentes distintos. Documenta el **ASN**, no solo la IP: el patrón es lo que permite anticipar.

## 13. Cómo cerrar un análisis

Al informar, usa lenguaje simple y estructura fija: **¿Qué se detectó? → Origen → Comportamiento observado → Contexto y riesgo → Estado y próximos pasos.** Presenta las conclusiones como **hipótesis de trabajo**, no como hechos confirmados, salvo que haya evidencia directa.
