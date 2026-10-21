# ¿Es el WAF o es el origen? Diagnóstico de errores, mTLS y certificados

| Campo | Valor |
|---|---|
| Plataforma | Imperva Cloud WAAP |
| Versión | 0.1 (borrador) |
| Última validación | Octubre 2026 |
| Audiencia | Ingenieros de WAF, equipos de infraestructura y soporte |
| Idioma | Español · [English](waf-vs-origin.en.md) |

---

## 1. Para qué sirve

Cuando una aplicación falla (página en blanco, errores 5xx, errores TLS), lo primero que se sospecha es el WAF. A veces es cierto; muchas veces no. Este runbook ayuda a **demostrar con evidencia** dónde se rompe la cadena, en lugar de presumirlo.

> **Principio:** se diagnostica antes de actuar y se valida de forma empírica. Las conclusiones se presentan como hipótesis de trabajo hasta tener evidencia directa.

## 2. Paso 1 — Delimitar el problema

| Pregunta | Por qué importa |
|---|---|
| ¿Afecta a todos o a un subconjunto? | Un problema de un subconjunto apunta a algo que ese grupo tiene en común. |
| ¿Usuarios externos sí, internos o por VPN no? | Los usuarios internos suelen llegar al origen por un camino distinto, sin pasar por el WAF. |
| ¿Solo usuarios autenticados? | Puede indicar un problema de autenticación en el origen, no de filtrado. |
| ¿Es constante o intermitente? | Lo intermitente sugiere varios nodos o balanceo con configuraciones distintas. |
| ¿Cuándo empezó y qué cambió? | Busca coincidencias con actualizaciones de la aplicación, rotación de certificados o cambios de configuración. |

## 3. Paso 2 — Seguir la solicitud salto por salto

```
Cliente  ──►  Imperva (WAF/CDN)  ──►  Balanceador / Origen  ──►  Aplicación
```

| Salto | Evidencia a recolectar |
|---|---|
| **Cliente** | Archivo HAR, código y cuerpo de la respuesta, tipo de contenido, tamaño. |
| **Imperva** | Evento de la solicitud: ¿fue **bloqueada** o **pasada al origen**? Confirma en tu consola el campo equivalente a *«passed to server»*. |
| **Origen / balanceador** | Registros del servidor: ¿llegó la solicitud?, ¿con qué código y tiempo de respuesta? |

**La pregunta decisiva:** *¿la solicitud llega al origen?* Si Imperva la pasó y los registros del origen muestran el error, el problema está del lado del origen.

> **Señal típica del origen:** una respuesta de **tamaño idéntico** y **tipo de contenido inesperado** para un recurso estático (por ejemplo, un error HTML pequeño donde debería haber un archivo JavaScript) apunta a un balanceador u origen, no al WAF.

## 4. Caso — Errores intermitentes solo para usuarios autenticados (NTLM)

**Síntoma.** Usuarios externos veían una página en blanco tras autenticarse; usuarios internos, no. El problema era intermitente y comenzó tras una actualización de la aplicación.

**Hipótesis iniciales (descartadas con evidencia).** Balanceo hacia nodos con despliegues inconsistentes, o bloqueo por parte del WAF. El proveedor de la aplicación indicaba que las solicitudes nunca llegaban a su servidor.

**Causa raíz confirmada.** El ajuste de Imperva *Origin Connection Reuse* **multiplexa** conexiones TCP hacia el origen. La autenticación **NTLM** necesita completar todo el *handshake* sobre **una sola conexión TCP**; al compartirla, la autenticación se rompía. Los registros de IIS mostraron cientos de errores 500 (código `sc-win32-status 1346`, `ERROR_BAD_IMPERSONATION_LEVEL`) **exclusivamente** en usuarios autenticados.

**Solución.** Desactivar *Origin Connection Reuse* para ese sitio. Se probó en vivo con soporte de Imperva, con varios usuarios y dispositivos, sin reproducir el problema.

**Recomendaciones.**

- Confirma con soporte de Imperva si este ajuste seguirá siendo configurable en tu versión; Imperva ha comunicado cambios planeados sobre él.
- A largo plazo, evaluar migrar la autenticación del balanceador de **NTLM a Kerberos**.

**Lección:** una causa que parecía del origen resultó ser una interacción entre el WAF y el protocolo de autenticación. La evidencia de los registros del origen fue la que cerró el caso.

## 5. mTLS: las tres preguntas obligatorias

Antes de tocar nada en un caso de mTLS, responde:

1. ¿El problema es **Cliente → Imperva**, **Imperva → Origen**, o ambos?
2. ¿El origen valida el certificado en el **handshake TLS** o mediante una **cabecera HTTP**?
3. ¿Hubo una **carga o rotación de certificado** reciente? Las fechas suelen coincidir con el inicio del incidente.

### Hechos que conviene tener claros

| Hecho | Consecuencia |
|---|---|
| La opción de **enviar los detalles del certificado del cliente al origen** solo **reenvía cabeceras**; **no** realiza un mTLS real. | No satisface a un origen que exige mTLS. |
| Para mTLS **Imperva → Origen**, el certificado se carga en la sección de mTLS de Imperva hacia el origen. | Sin esa carga, el origen no recibe un certificado de cliente válido. |
| Algunos balanceadores validan el certificado **en el handshake**, no por cabeceras. | Reenviar cabeceras no sirve. |
| Dos renegociaciones TLS seguidas de un cierre abrupto. | El servidor **exige** un certificado de cliente que el cliente **no presenta**. |
| En Windows, `CRYPT_E_NO_REVOCATION_CHECK`. | Suele ser un falso positivo: el sistema no alcanza la lista de revocación interna. |

### Comandos de apoyo

```
# Probar con certificado de cliente (en Windows, sin comprobación de revocación)
curl -v --ssl-no-revoke --cert cliente.pem --key cliente.key https://app.example.com/

# Ver la cadena de certificados que presenta el servidor
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts
```

## 6. Migración de certificados

**Síntoma.** Tras migrar un sitio de un certificado de cuenta a uno por sitio, SSL falla.

**Causa observada.** Si la opción *Imperva generated certificate* está **deshabilitada** (en la configuración general del sitio, sección SSL Support), el sitio recurre a un certificado personalizado y SSL se rompe.

**Qué hacer.**

- Revisa esa opción en la configuración del sitio. **No basta** con mirar el tipo de certificado en el listado de certificados.
- Para validar la compatibilidad entre cadenas, comprueba con una herramienta externa de análisis TLS que existan **dos rutas de certificación** cuando corresponda.

## 7. Cuidado con las métricas de rendimiento

Los «errores de cliente» del panel de rendimiento reflejan **comportamiento HTTP**, no compatibilidad de cifrados TLS. Si el handshake falla antes de que se procese HTTP, el WAF **no puede detectar** ese fallo del lado del cliente. No uses esas métricas para descartar un problema TLS.

## 8. Lista de validación del lado de Imperva

Útil para documentar formalmente que Imperva no es la causa (por ejemplo, ante un proveedor o un equipo que lo cuestiona):

```
[ ] La solicitud afectada aparece en los eventos de Imperva
[ ] Indica que fue pasada al origen (sin bloqueo)
[ ] No hay reglas de seguridad ni de ABP activadas sobre esa solicitud
[ ] Se revisaron reglas de redirección, reescritura y caché que apliquen a la ruta
[ ] Se confirmó el certificado que sirve el sitio y su configuración de SSL
[ ] Se revisó el ajuste de conexión hacia el origen (reutilización de conexiones)
[ ] Se comparó el comportamiento con y sin pasar por Imperva (si es posible)
[ ] Se contrastó con los registros del origen (código, tamaño, tiempo)
[ ] Se verificó dos veces en ventanas de tiempo distintas
```

## 9. Cómo reportarlo

Presenta el resultado como **análisis técnico propio**, con hipótesis de trabajo y evidencia adjunta. Estructura sugerida: **¿Qué se detectó? → Origen → Comportamiento observado → Contexto y riesgo → Estado y próximos pasos.** Si la solución depende de un ajuste, indica que se validará fuera de producción antes de aplicarse.
