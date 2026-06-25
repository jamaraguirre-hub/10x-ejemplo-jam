# Documento Gerencial — Grupo 2: Contactos Duplicados
**Proceso de Limpieza de Base de Contactos — Marketing Cloud Engagement**
**Fecha:** Junio 2026 | **Responsable:** Seguros Bolívar

---

## 1. Descripción del Problema

La base de contactos de Marketing Cloud Engagement acumula registros duplicados: distintos contactos que comparten el mismo correo electrónico y/o el mismo número de móvil. Esta situación genera:

- **Envíos duplicados** al mismo destinatario real, afectando la experiencia del cliente.
- **Métricas infladas**: aperturas, clics y conversiones se contabilizan más de una vez.
- **Costos innecesarios** por volumen de envíos y almacenamiento de contactos.
- **Riesgo de compliance**: múltiples registros dificultan honrar las preferencias de comunicación (opt-out).

---

## 2. Criterio de Identificación

Se considera duplicado todo par (o grupo) de contactos que comparten **simultáneamente**:

| Campo | Criterio |
|-------|----------|
| Correo electrónico | Igual valor, ignorando mayúsculas/minúsculas y espacios |
| Número de móvil | Igual valor, ignorando espacios |

> Si un contacto tiene correo pero no móvil, la deduplicación aplica por correo.
> Si un contacto tiene móvil pero no correo, la deduplicación aplica por móvil.
> Contactos sin ningún canal de contacto **no entran** al proceso de duplicados.

---

## 3. Reglas de Protección (Exclusiones)

Los siguientes contactos **NO serán eliminados**, independientemente de ser duplicados:

| # | Regla | Justificación |
|---|-------|---------------|
| 1 | **Unsubscribed** | Contactos que ejercieron su derecho de baja — protección legal y de compliance |
| 2 | **Enviado en últimos 90 días** | Contacto con actividad de envío reciente (`_Sent` consolida todas las BUs) — riesgo de interrupción de comunicación activa |
| 3 | **El "mejor" registro del grupo** | De cada grupo de duplicados, siempre se conserva 1 contacto según la prioridad definida |

### Nota sobre la protección de 90 días
La vista `_Sent` en la Business Unit padre consolida los envíos de **todas las Business Units hijas**, por lo que la protección aplica de forma transversal a toda la estructura multibu. Se eligieron 90 días como ventana conservadora que cubre campañas mensuales y comunicaciones bimestrales.

---

## 4. Criterio de Conservación (¿Cuál se queda?)

De cada grupo de duplicados se conserva **1 contacto** según la siguiente prioridad:

| Prioridad | Criterio | Descripción |
|-----------|----------|-------------|
| 1° | **Registro más completo** | Tiene correo electrónico válido Y número de móvil válido |
| 2° | **Registro más reciente** | Mayor fecha de creación/modificación (`DateJoined DESC`) |
| 3° | **Mayor SubscriberID** | Desempate final por ID de sistema |

Los demás registros del grupo (N − 1) son candidatos a eliminación, sujeto a las reglas de protección.

---

## 5. Tratamiento por Estado del Contacto

| Estado | Cantidad en base | Tratamiento |
|--------|-----------------|-------------|
| **Active** | 4,275,761 | Aplica proceso completo: se conserva 1, se borran N−1 duplicados no protegidos |
| **Held** | 442,996 | Mismo tratamiento: se conserva 1, se borran N−1 duplicados no protegidos |
| **Bounced** (soft/technical) | 292,227 | Mismo tratamiento: se conserva 1, se borran N−1 duplicados no protegidos |
| **Unsubscribed** | 1 | **Excluidos completamente** del proceso |

> Los contactos **Bounced Hard** sin canal válido son tratados en el **Grupo 1** (correo y celular inválidos), por lo que no se duplica el esfuerzo aquí.

---

## 6. Top 5 Grupos de Duplicados más Grandes

Los grupos con mayor número de registros duplicados identificados son:

| Posición | Email / Móvil (referencia) | Registros en grupo | Registros a borrar |
|----------|---------------------------|-------------------|-------------------|
| 1 | (grupo más grande) | Por confirmar con query de diagnóstico | N − 1 |
| 2 | (segundo grupo) | Por confirmar | N − 1 |
| 3 | (tercer grupo) | Por confirmar | N − 1 |
| 4 | (cuarto grupo) | Por confirmar | N − 1 |
| 5 | (quinto grupo) | Por confirmar | N − 1 |

> Para obtener el Top 5 exacto, ejecutar la siguiente consulta diagnóstica en Query Studio:
>
> ```sql
> SELECT TOP 5
>     UPPER(LTRIM(RTRIM(ISNULL(s2.EmailAddress,'')))) AS Email,
>     LTRIM(RTRIM(ISNULL(t2.Mobile,''))) AS Mobile,
>     COUNT(*) AS TotalEnGrupo,
>     COUNT(*) - 1 AS ABorrar
> FROM _Subscribers s2
> LEFT JOIN (
>     SELECT SubscriberKey, MAX(Mobile) AS Mobile
>     FROM _SMSMessageTracking
>     WHERE CHARINDEX('#', SubscriberKey) = 0
>     AND LEN(SubscriberKey) <= 50
>     GROUP BY SubscriberKey
> ) t2 ON t2.SubscriberKey = s2.SubscriberID
> LEFT JOIN (
>     SELECT DISTINCT SubscriberKey
>     FROM _Sent
>     WHERE EventDate >= DATEADD(DAY, -90, GETDATE())
>     AND CHARINDEX('#', SubscriberKey) = 0
>     AND LEN(SubscriberKey) <= 50
> ) sent90 ON sent90.SubscriberKey = s2.SubscriberKey
> WHERE (
>     (s2.EmailAddress IS NOT NULL AND LTRIM(RTRIM(s2.EmailAddress)) <> '')
>     OR (t2.Mobile IS NOT NULL AND LTRIM(RTRIM(t2.Mobile)) <> '')
> )
> AND s2.Status <> 'unsubscribed'
> AND sent90.SubscriberKey IS NULL
> GROUP BY
>     UPPER(LTRIM(RTRIM(ISNULL(s2.EmailAddress,'')))),
>     LTRIM(RTRIM(ISNULL(t2.Mobile,'')))
> HAVING COUNT(*) > 1
> ORDER BY COUNT(*) DESC
> ```

---

## 7. Cifras del Proceso

| Métrica | Valor |
|---------|-------|
| Duplicados brutos identificados (sin protección) | 834,349 |
| Protegidos por envío reciente (últimos 90 días) | 402,683 |
| Protegidos por estado Unsubscribed | < 10 |
| **Total a eliminar (neto)** | **431,666** |
| Lotes necesarios (límite 850,000 por lote) | **1 lote** |
| Límite API por lote | 850,000 contactos |

---

## 8. Arquitectura del Proceso Automatizado

```
Automation Studio — Grupo 2 Duplicados
│
├── Paso 1 — Query Activity
│   └── Pobla: DE_Grupo2_Borrar_Duplicados
│       Regla: duplicados con protección 90 días + unsubscribed
│
├── Paso 2 — Query Activity
│   └── Pobla: DE_G2_Borrar_Lote_1 (filas 1 a 850,000)
│
├── Paso 3 — Query Activity
│   └── Pobla: DE_G2_Borrar_Lote_2 (filas 850,001 a 1,700,000)
│       [Quedará vacío con el volumen actual — protección futura]
│
└── Paso 4 — Script Activity (SSJS)
    └── Para cada lote:
        ├── Si lote vacío → log SKIPPED
        ├── Llama REST API: /contacts/v1/contacts/actions/delete
        │   deleteOperationType: ContactAndAttributes
        └── Registra resultado en DE_Log_Borrado_Contactos_P1
```

### Data Extensions involucradas

| DE | External Key | Propósito |
|----|-------------|-----------|
| `DE_Grupo2_Borrar_Duplicados` | `DE_Grupo2_Borrar_Duplicados` | Staging: todos los duplicados a borrar |
| `DE_G2_Borrar_Lote_1` | `DE_G2_Borrar_Lote_1` | Lote 1 — filas 1 a 850,000 |
| `DE_G2_Borrar_Lote_2` | `DE_G2_Borrar_Lote_2` | Lote 2 — filas 850,001+ (reserva) |
| `DE_Log_Borrado_Contactos_P1` | `DE_Log_Borrado_Contact_P1` | Log consolidado de todos los grupos |

---

## 9. Estructura de la DE de Staging (`DE_Grupo2_Borrar_Duplicados`)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| SubscriberKey | Text 50 | SubscriberID del contacto a borrar |
| EmailAddress | Text 254 | Correo del contacto |
| MobileNumber | Text 50 | Móvil del contacto |
| ModifiedDate | Date | Fecha de creación/modificación (DateJoined) |
| Reason | Text 100 | Valor fijo: `Grupo2_Duplicados` |

---

## 10. Impacto Esperado

| Indicador | Antes | Después |
|-----------|-------|---------|
| Contactos duplicados activos | 834,349+ duplicados | 0 duplicados en las combinaciones identificadas |
| Envíos duplicados al mismo destinatario | Presente | Eliminado |
| Precisión de métricas de campaña | Inflada | Real |
| Contactos totales en base | ~5M | Reducción de ~431,666 registros |

---

## 11. Riesgos y Mitigaciones

| Riesgo | Probabilidad | Mitigación |
|--------|-------------|------------|
| Eliminar contacto en campaña activa | Baja | Protección 90 días de envío reciente |
| Eliminar el contacto "correcto" del grupo | Muy baja | Criterio de prioridad documentado y validado en conteo |
| Volumen supere el lote | Baja | Query Lote 2 disponible como contingencia |
| Registros Unsubscribed eliminados | Nula | Excluidos explícitamente del proceso |

---

## 12. Trazabilidad y Auditoría

Cada ejecución queda registrada en `DE_Log_Borrado_Contactos_P1` con:
- Fecha de ejecución
- Grupo y Lote ejecutado
- Cantidad de registros procesados
- Operation ID devuelto por la API
- Estado: OK / SKIPPED / ERROR / EXCEPTION
- Mensaje de respuesta de la API

---

*Documento generado como parte del proceso de higiene de base de contactos — MCE Seguros Bolívar 2026.*
