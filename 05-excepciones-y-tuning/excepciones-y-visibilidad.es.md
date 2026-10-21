# Excepciones sin perder visibilidad: Allowlist y ACL

| Campo | Valor |
|---|---|
| Plataforma | Imperva Cloud WAAP (WAF y ABP) |
| Versión | 0.1 (borrador) |
| Última validación | Octubre 2026 |
| Audiencia | Ingenieros de WAF y analistas SOC |
| Idioma | Español · [English](exceptions-and-visibility.en.md) |

---

## 1. Para qué sirve

Una excepción es una **reducción deliberada de protección**. A veces es necesaria (una integración legítima dispara una regla), pero el mecanismo que elijas decide si después podrás ver lo que ocurre o si quedarás a ciegas.

> **Principio:** ante la duda, no crees una excepción. Alertar, ajustar la regla o corregir la aplicación suelen ser opciones más seguras.

## 2. El hecho que más se pasa por alto

> **El tráfico que coincide con una allowlist no genera eventos de seguridad: ni en la consola ni en el SIEM.**

Esto se confirmó en operación real. Valídalo en tu cuenta antes de depender de ello. Consecuencias:

- No puedes auditar después ese tráfico.
- Si el origen permitido se compromete, su tráfico malicioso es **invisible**.
- Las reglas de correlación del SIEM nunca se activan sobre él.

## 3. Comparación de mecanismos

> Basada en lo observado en nuestro entorno. Confirma el comportamiento de cada mecanismo en tu cuenta.

| Mecanismo | ¿Genera eventos? | ¿Visible en SIEM? | Cuándo usarlo |
|---|---|---|---|
| **Allowlist** | No | No | Último recurso, solo si no existe otra opción. |
| **Excepción basada en ACL** | Sí | Sí | **Opción por defecto** para cualquier excepción. |
| **Acción Allow / Ignore en una regla** | Con visibilidad reducida | Con visibilidad reducida | Solo si la ACL no resuelve el caso. |

## 4. Árbol de decisión

1. **¿Es realmente un falso positivo?** Revisa la solicitud completa, el origen y el contexto de la aplicación.
2. **¿Se resuelve ajustando la regla o corrigiendo la aplicación?** Si la regla es demasiado amplia, arréglala; no la rodees.
3. **¿Puedo usar una ACL?** Si sí, **detente aquí**.
4. **¿Es la allowlist la única opción?** Entonces aplica el alcance mínimo y documenta la zona ciega que estás aceptando.

## 5. Alcance: del más al menos restrictivo

```
1. IP + Ruta + Parámetro + Regla      ← ideal
2. IP + Ruta + Regla
3. IP + Regla
4. Rango de IP + Ruta + Regla
5. Ruta + Regla                       ← solo si el origen varía legítimamente
6. Solo la regla                      ← casi nunca es correcto
7. A nivel de todo el sitio           ← casi siempre es un error
```

> **Atención con la posición:** el alcance real de una excepción depende de **dónde** está definida. Una excepción en una política evaluada primero (por ejemplo, la política por defecto) puede aplicar a todo un grupo de sitios, aunque solo la necesitaras para uno.

## 6. Lista de verificación previa

```
Antes de elegir el mecanismo:
[ ] ¿Se puede resolver con una ACL?  Si sí: usarla. Fin.

Evidencia:
[ ] Solicitud(es) completa(s) revisada(s)
[ ] IP / ASN identificados y verificados
[ ] User-Agent confirmado como legítimo (en buscadores: DNS inverso con confirmación directa)
[ ] Ruta y parámetro documentados
[ ] Volumen y patrón evaluados
[ ] Justificación de negocio confirmada con el equipo de la aplicación

Riesgo:
[ ] Se entiende qué zona quedará sin visibilidad
[ ] Se evaluó el impacto si ese origen se compromete
[ ] Se listaron los controles que siguen aplicando
[ ] Alcance mínimo definido

Aprobación y cierre:
[ ] Cambio aprobado por el responsable de seguridad o el cliente
[ ] El cliente sabe que el tráfico excepcionado no aparecerá en los reportes
[ ] Registrado en el control de cambios
[ ] Fecha de revisión definida
```

## 7. Ciclo de vida

```
Creación ──► Revisión a 30 días ──► Revisión a 90 días ──► Auditoría anual ──► Mantener / Modificar / Eliminar
```

En cada revisión pregunta: ¿el origen sigue activo?, ¿cambió el caso de uso?, ¿el alcance sigue siendo el mínimo?, ¿puede convertirse ahora en ACL para recuperar visibilidad? **El objetivo de cada revisión es eliminar la allowlist o convertirla.**

## 8. Casos reales (anonimizados)

### Caso A — Un rastreador de buscador que explicaba gran parte del tráfico

**Situación.** Se investigaban errores 404 anómalos en rutas de consulta de un portal público. Una parte importante del tráfico era de un rastreador de buscador legítimo (verificado con DNS inverso), permitido mediante una excepción de la política por defecto.

**Aprendizaje.**

- Como esa política se evalúa **primero** en el grupo de sitios, la excepción no estaba limitada al sitio donde se necesitaba.
- Verificar el DNS inverso evitó clasificar mal el tráfico. Las IP restantes eran sobre todo redes residenciales y móviles, coherentes con usuarios reales.
- Se detectó además una IP de hosting con escaneo automatizado, **sin relación** con el problema original. Se recomendó tratarla en un caso aparte.

**Lección:** separa los hallazgos. Un problema de aplicación y un escáner pueden coexistir en las mismas rutas.

### Caso B — Reglas de detección de herramientas de hacking en modo Alert

**Situación.** Se desplegó una regla de detección de herramientas de hacking en varios sitios, en modo **Alert**, justamente para observar antes de bloquear.

**Resultado.** Aparecieron falsos positivos de clientes HTTP legítimos: utilidades de Windows, bibliotecas de Java, Postman, `python-requests` y clientes Ruby, usados por integraciones y equipos de desarrollo.

**Lección:** sin la fase de observación (Passive/Alert), estas integraciones se habrían bloqueado. La clasificación por User-Agent no distingue **intención**; solo describe la herramienta.

### Caso C — Una regla de redirección sin modo Alert

**Situación.** Las reglas de redirección ejecutan su acción de inmediato; no existe un equivalente a "modo Alert". Además, normalmente **tienen prioridad: se evalúan antes que las reglas de seguridad**. Por eso, una vez activa, una redirección puede actuar sobre el tráfico antes de que lo evalúen las reglas de seguridad (confirma el orden de evaluación en tu cuenta).

**Solución.** Crear una **regla sombra** en paralelo, con la misma condición y acción Alert, y mantener la **regla de redirección real apagada (deshabilitada)** mientras se valida con ella. Solo cuando la evidencia confirme que la condición afecta únicamente al tráfico esperado se enciende la redirección.

## 9. Plantilla de registro de cambio

```
Fecha:
Mecanismo (Allowlist / ACL / Allow en regla):
Regla o condición afectada:
Sitio / ruta:
Origen (IP / ASN / servicio):
Patrón que disparaba la detección:
Causa raíz:
Evidencia revisada:
Alcance del cambio:
Zona sin visibilidad aceptada:
Aprobado por:
Fecha de revisión:
```

## 10. Cómo explicarlo a un área no técnica

> «Una excepción permite que cierto tráfico pase sin ser bloqueado. Si además apaga los registros, ya no podremos ver qué hace ese tráfico. Por eso preferimos excepciones que lo dejan pasar **y** lo siguen registrando. Antes de llevar cualquier cambio a producción, lo validamos en un ambiente de pruebas.»
