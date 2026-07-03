# Reglas de Limpieza de Contactos — MCE Seguros Bolívar 2026

**Base inicial:** 7,895,021 contactos | **Total a borrar:** 884,976 | **Reducción proyectada:** 11.2%

---

## Grupo 1 — Email y Celular Inválidos
**723 contactos · 1 lote · ✅ Ejecutado**

### Criterios de borrado
| # | Regla |
|---|-------|
| 1 | Email sin `@` o sin dominio válido tras el punto |
| 2 | Email en lista negra de 31 valores genéricos conocidos (NO.TIENE@..., NOAPLICA@..., etc.) |
| 3 | Dominio con typo conocido (GMAL.COM, HOTMIL.COM, YAOO.COM, etc.) |
| 4 | Celular con longitud fuera de rango (< 10 o > 12 dígitos) |
| 5 | Celular de 10 dígitos que no inicia en `3` (Colombia) |
| 6 | Celular de 12 dígitos sin prefijo `57` o sin dígito `3` tras el prefijo |
| 7 | Hard Bounce confirmado + sin celular válido alternativo |

### Protecciones (no se borran)
- Contactos con email **Y** celular válidos
- Unsubscribed — excluidos completamente
- Bounced soft o technical — solo se aplica a Hard Bounce

### Cobertura multi-BU
`_Subscribers`, `_SMSMessageTracking` y `_Bounce` consolidan desde la BU padre ✅

---

## Grupo 2 — Contactos Duplicados
**432,473 contactos · 1 lote · 🔄 Ejecución diaria automática**

### Criterio de duplicado
Mismo correo electrónico (case-insensitive) **Y** mismo número de móvil.
Si falta un canal, agrupa por el disponible. Sin ningún canal → excluido del proceso.

### Prioridad — quién se conserva (1 por grupo)
| Prioridad | Criterio |
|-----------|----------|
| 1° | Registro más completo: tiene email **Y** móvil válidos |
| 2° | Más reciente: `DateJoined DESC` |
| 3° | Mayor `SubscriberID` (desempate de sistema) |

### Protecciones (no se borran)
- **Unsubscribed** — excluidos completamente del proceso
- **Envío reciente 90 días** — contactos con envío en `_Sent` en los últimos 90 días (en revisión si se ajusta a más días)
- **El "mejor" registro** del grupo siempre se conserva

### Tratamiento por estado
Active, Held y Bounced (soft/technical): se conserva 1, se borran los demás duplicados no protegidos.

### Cobertura multi-BU
`_Sent` desde BU padre consolida todas las BUs hijas (confirmado: 4.56M envíos en 60 días) ✅

---

## Grupo 3 — Contactos Inactivos
**1,127 contactos · 1 lote · 🔄 Ejecución diaria automática**

### Criterios de borrado (todos deben cumplirse)
| # | Regla | Valor |
|---|-------|-------|
| 1 | Recibió emails en últimos 12 meses | ≥ 3 envíos |
| 2 | Nunca abrió en 12 meses | `_Open` sin registros |
| 3 | Nunca hizo clic en 12 meses | `_Click` sin registros |
| 4 | Estado del contacto | Solo `active` |
| 5 | Antigüedad del contacto | Entre 18 y 30 meses desde `DateJoined` |

### Protecciones (no se borran)
- Contactos con **≤ 2 envíos** (sin oportunidad suficiente de interactuar)
- Contactos con **envío en últimos 90 días** — comunicación activa en curso
- Contactos con **apertura o clic** en cualquier BU en los últimos 12 meses
- Contactos `held` o `bounced` — excluidos
- Contactos creados hace **menos de 18 meses** (muy nuevos)
- Contactos creados hace **más de 30 meses** (históricos protegidos)

### Por qué se ajustaron las reglas
La versión inicial (sin mínimo de envíos ni protección 90 días) arrojaba 26,497 candidatos. Con los ajustes se redujo a 1,127 — cifra más conservadora y defendible ante el equipo.

### Cobertura multi-BU
`_Sent`, `_Open` y `_Click` consolidan interacciones de todas las BUs hijas ✅

---

## Grupo 4 — Contactos sin Canal ("Fantasma")
**450,653 contactos · 1 lote único · 📋 Pendiente ejecución**

Fuente: DE "Contacts without Channel Addresses" (External Key: `13157682-8EF4-477A-A7D6-BA5773ABC570`)

### Estructuras de ContactKey eliminadas
| Estructura | Cantidad |
|-----------|---------|
| Email como ContactKey (sin canal real) | 349,906 |
| Solo números — cédulas puras o celulares sin contexto | 88,872 |
| Estructura rota: múltiples IDs unidos por "y" | 5,547 |
| Documento con separador de guion (CCxxxx-x) | 4,712 |
| Otros alfanuméricos no clasificados | 1,244 |
| Estructura rota: separados por Pipe `\|` | 191 |
| ID Nativo Salesforce CRM (003) — piloto descontinuado | 181 |
| **Total a borrar** | **450,653** |

### Estructuras protegidas (no se borran)
| Estructura | Cantidad | Motivo |
|-----------|---------|--------|
| Documento Pegado sin Símbolos (CC, CE, NIT, TI, RC…) | 212,201 | Convención válida del negocio |
| Documento con Dos Puntos (CC:, CE:, PP:) | 46,599 | Formato del App capturado en su momento |
| UUID / GUID de Sistema Externo | 12,362 | Identificadores de integraciones activas |
| Texto Completo: CÉDULA DE... | 1,750 | Dato con referencia identificable |
| **Total protegidos** | **272,912** | |

### Criterio de borrado — lógica SQL
Se usa `IN` con lista explícita de categorías a borrar, **no** `NOT IN`, para que cualquier estructura nueva que aparezca en el futuro quede automáticamente protegida por defecto.

### Cobertura multi-BU
No aplica — estos contactos no tienen canal de contacto activo; nunca recibieron comunicaciones.

---

## Resumen ejecutivo

| Grupo | Descripción | A borrar | Estado |
|-------|-------------|---------|--------|
| 1 | Email y celular inválidos | 723 | ✅ Ejecutado |
| 2 | Duplicados | 432,473 | 🔄 Diario |
| 3 | Inactivos sin interacción | 1,127 | 🔄 Diario |
| 4 | Sin canal de contacto | 450,653 | 📋 Pendiente |
| **Total** | | **884,976** | |

## Lecciones operativas aplicadas

| Problema | Solución |
|---------|---------|
| `SubscriberID` en lugar de `SubscriberKey` en los lotes | Corregido — delete API requiere ContactKey real |
| `Rows.Retrieve()` no confiable con DEs grandes | Reemplazado por `Script.Util.WSProxy` |
| Lote apuntando a DE staging desactualizada | Verificar que el `FROM` del lote apunte a la DE staging vigente |
| Script corría sin ejecutar los queries previos | Siempre correr el automation completo desde el Paso 1 |
| `deleteListContentsWhenCompleted: true` | Cambiado a `false` en todos los scripts |

## Trazabilidad
Toda ejecución queda registrada en `DE_Log_Borrado_Contactos_P1` con: fecha, grupo, lote, cantidad, Operation ID, status y mensaje de respuesta de la API.

---
*MCE Seguros Bolívar · Corte 30/06/2026 · All Contacts actual: 7,544,122*
