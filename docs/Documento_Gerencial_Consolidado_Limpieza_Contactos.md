# Documento Gerencial Consolidado — Proceso de Limpieza de Base de Contactos
**Marketing Cloud Engagement — Seguros Bolívar**
**Fecha:** Junio 2026 | **Estado:** Grupos 1-3 en ejecución | Grupo 4 con proceso construido y validado

---

## 1. Resumen Ejecutivo

El proceso de higiene de base de contactos en MCE se organiza en **4 grupos independientes**, cada uno con su propio criterio de identificación, reglas de protección y automation de borrado. A la fecha:

| Grupo | Descripción | Contactos a borrar | Estado |
|-------|-------------|--------------------|--------|
| **Grupo 1** | Email y celular inválidos | 723 | ✅ Validado |
| **Grupo 2** | Duplicados (mismo email + mismo móvil) | 432,473 | 🔄 En ejecución (corrigiendo fuente del lote) |
| **Grupo 3** | Inactivos — sin apertura ni clic en 12 meses | 1,127 | ✅ Validado, pendiente ejecutar |
| **Grupo 4** | Contactos sin canal de contacto ("fantasma") | 450,653 | ✅ Proceso construido y validado, pendiente ejecutar |
| **Total identificado** | | **884,976** | |

---

## 2. Arquitectura Común del Proceso

Los 3 primeros grupos comparten la misma arquitectura de automation:

```
Automation Studio — Grupo N
│
├── Paso 1 — Query Activity (Overwrite)
│   └── Identifica candidatos → pobla DE staging del grupo
│
├── Paso 2 — Query Activity (Overwrite)
│   └── Pobla Lote 1 (filas 1 a 850,000) desde la DE staging
│
├── Paso 3 — Query Activity (Overwrite)
│   └── Pobla Lote 2 (filas 850,001 a 1,700,000) — reserva
│
└── Paso 4 — Script Activity (SSJS)
    └── Por cada lote:
        ├── Verifica si tiene datos (WSProxy, no Rows.Retrieve)
        ├── Si vacío → log "Skipped"
        ├── Llama REST API: POST /contacts/v1/contacts/actions/delete
        │   deleteOperationType: ContactAndAttributes
        └── Registra resultado en DE_Log_Borrado_Contactos_P1
```

**Log centralizado:** `DE_Log_Borrado_Contactos_P1` (External Key: `DE_Log_Borrado_Contact_P1`) registra todas las ejecuciones de los 4 grupos con: fecha, grupo, lote, cantidad, operation ID, status y mensaje.

---

## 3. Grupo 1 — Email y Celular Inválidos

### Criterio de identificación
Contactos donde el correo electrónico es inválido (formato incorrecto, dominio mal escrito, o valor en lista de correos genéricos conocidos) **Y** el celular es inválido (longitud o prefijo incorrecto para Colombia), o registros con bounce duro sin celular válido.

### Regla especial
Se incluyen también los **Hard Bounce** sin celular válido, ya que un correo con rebote duro permanente es, en la práctica, un canal inválido.

### Cifra final
**723 contactos** a eliminar — 1 solo lote.

### Cobertura multi-BU
✅ Cubre todas las BUs hijas: `_Subscribers`, `_SMSMessageTracking` y `_Bounce` consolidan desde la BU padre.

> **Nota:** `_SMSMessageTracking` está vacía en este ambiente, por lo que la validación de celular recae en su totalidad sobre el criterio de correo y los Hard Bounce.

---

## 4. Grupo 2 — Contactos Duplicados

### Criterio de identificación
Mismo correo electrónico (case-insensitive) **y** mismo número de móvil. Si falta uno de los dos canales, se agrupa por el disponible.

### Reglas de protección (NUEVAS y vigentes)

| # | Regla | Detalle |
|---|-------|---------|
| 1 | **Unsubscribed** | Excluidos completamente del proceso |
| 2 | **Envío reciente — 90 días** | Contactos con envío en `_Sent` en los últimos 90 días no se tocan (protección conservadora, en revisión por el equipo si se ajusta a más días) |
| 3 | **El "mejor" registro se conserva** | 1° más completo (email + móvil válidos) → 2° más reciente (DateJoined DESC) → 3° mayor SubscriberID |

### Tratamiento por estado
Active, Held y Bounced (soft/technical) reciben el mismo tratamiento: se conserva 1 por grupo, se borran los demás duplicados no protegidos. Unsubscribed queda excluido.

### Cifra final
**432,473 contactos** a eliminar — 1 lote (por debajo del límite de 850,000).

### Cobertura multi-BU
✅ `_Sent` desde la BU padre consolida los envíos de todas las BUs hijas (confirmado: 4.56M envíos en 60 días a nivel consolidado).

### Incidente operativo y corrección aplicada
Durante la primera ejecución real, el automation tomó datos de una DE staging anterior (`DE_Grupo2_Borrar_Duplicados`, con 754,768 registros desactualizados) en lugar de la DE vigente (`Grupo2_Borrar_Duplicados_Email_Movil`, con 432,473 registros validados). **Causa raíz:** el query del Lote 1 y Lote 2 referenciaban el nombre de la DE antigua. **Corrección:** se actualizó el `FROM` de ambos queries de lote para apuntar a la DE correcta. Adicionalmente se reforzó el script de borrado para verificar el contenido de los lotes vía WSProxy en lugar de `Rows.Retrieve()`, que no es confiable con volúmenes grandes.

---

## 5. Grupo 3 — Contactos Inactivos

### Criterio de identificación (reglas ajustadas tras validación con el equipo)

| # | Regla | Valor final |
|---|-------|------------|
| 1 | Ventana de interacción (apertura/clic) | 12 meses (365 días) — sin cambio |
| 2 | Mínimo de envíos sin interacción | **≥ 3 envíos** (nuevo — antes bastaba 1) |
| 3 | Protección por envío reciente | **90 días** (nuevo) |
| 4 | Antigüedad del contacto (DateJoined) | Entre 18 y 30 meses — sin cambio |
| 5 | Estado | Solo `active` |

### Por qué se ajustaron las reglas
La primera versión (sin mínimo de envíos ni protección de 90 días) arrojaba 26,497 candidatos; al revisar el criterio con el equipo se identificó que contactos con solo 1-2 envíos no tenían suficiente oportunidad real de interactuar, y que comunicaciones muy recientes no debían interrumpirse. Con los ajustes aplicados (≥3 envíos + protección 90 días) el número se redujo a una cifra mucho más conservadora y defendible.

### Cifra final
**1,127 contactos** a eliminar — validados con muestra Top 3 cruzada manualmente. 1 solo lote.

### Cobertura multi-BU
✅ Confirmado: `_Sent`, `_Open` y `_Click` desde la BU padre consolidan interacciones de todas las BUs hijas. Si un contacto abrió o hizo clic en cualquier BU hija, queda protegido correctamente.

---

## 6. Grupo 4 — Contactos sin Canal de Contacto ("Fantasma")

### Descripción
Contactos existentes en Contact Builder que **no tienen correo ni móvil registrado** — no visibles desde `_Subscribers` vía Query Studio. Se identifican y clasifican desde la DE "Contacts without Channel Addresses" por la estructura de su `ContactKey`.

| Dato | Valor |
|------|-------|
| Fuente | DE "Contacts without Channel Addresses" |
| External Key fuente | `13157682-8EF4-477A-A7D6-BA5773ABC570` |
| Universo total de la DE | 723,565 clasificados |
| **Contactos a borrar** | **450,653** |
| Contactos protegidos | 272,912 |
| Lotes necesarios | 1 lote único (< 850,000) |

### Clasificación por estructura de ContactKey

| Familia | Cantidad | Decisión |
|---------|----------|----------|
| Estructura de Correo (email como ContactKey sin canal real) | 349,906 | 🗑️ Borrar |
| Solo Números (cédulas puras / celulares) | 88,872 | 🗑️ Borrar |
| Estructura Rota: Múltiples IDs unidos por "y" | 5,547 | 🗑️ Borrar |
| Documento con separador de Guion | 4,712 | 🗑️ Borrar |
| Otros Alfanuméricos No Clasificados | 1,244 | 🗑️ Borrar |
| Estructura Rota: Datos separados por Pipe | 191 | 🗑️ Borrar |
| ID Nativo Salesforce CRM (003) | 181 | 🗑️ Borrar |
| **Documento Pegado sin Símbolos (CC, CE, NIT, TI, RC...)** | **212,201** | ✅ Proteger |
| **Documento con separador de Dos Puntos (CC:, CE:, PP:)** | **46,599** | ✅ Proteger |
| **UUID / GUID de Sistema Externo** | **12,362** | ✅ Proteger |
| **Texto Completo: CÉDULA DE...** | **1,750** | ✅ Proteger |

### Criterio de protección
La compañía administra ContactKeys con la estructura **tipo de documento + número de documento**. Las categorías protegidas representan identificadores válidos en esa convención (documentos con prefijo, con separador de dos puntos, o GUIDs de sistemas externos). Las categorías borradas corresponden a estructuras rotas, correos sin canal, numéricos sin contexto o IDs de sistemas que ya no están activos (Salesforce CRM — piloto de 2 meses, ya descontinuado).

### Criterio de identificación — WHERE clause
```sql
WHERE Sub.Familia_Estructura IN (
    'Estructura de Correo (Normales o Basura tipo www@)',
    'Solo Números (Cédulas puras, celulares con espacio o ceros adelante)',
    'Estructura Rota: Múltiples IDs unidos por y',
    'Documento con separador de Guion (Ej. CCxxxx-x)',
    'Otros Alfanuméricos No Clasificados',
    'ID Nativo de Salesforce CRM (Contact/003)',
    'Estructura Rota: Datos separados por Pipe (|)'
)
```
> Se usa `IN` con la lista explícita de categorías a borrar (no `NOT IN`) para que cualquier categoría nueva que aparezca en el futuro quede automáticamente protegida.

### Cobertura multi-BU
Estos contactos no tienen canal de contacto activo, por lo que la protección de envío reciente no aplica — nunca han recibido comunicaciones.

---

## 7. Lecciones Operativas Aplicadas a los 4 Grupos

| Lección | Ajuste realizado |
|---------|------------------|
| `Rows.Retrieve()` falla o no es confiable con DEs grandes | Reemplazado por verificación de existencia vía `Script.Util.WSProxy` |
| Riesgo de leer de una DE staging desactualizada | Validar explícitamente que cada query de lote referencia la DE staging vigente del grupo |
| `ParseJSON` sobre respuesta vacía o nula causa error no controlado | Guards explícitos antes de parsear cualquier respuesta HTTP |
| Ejecutar solo el script sin correr el automation completo deja datos viejos en los lotes | Siempre ejecutar el automation completo desde el Paso 1 |
| `deleteListContentsWhenCompleted: true` borra el lote antes de confirmar resultado | Cambiado a `false` en todos los scripts |

---

## 8. Cifras Consolidadas

| Grupo | A borrar | % del total a borrar | Estado de ejecución |
|-------|----------|----------------------|---------------------|
| Grupo 1 — Inválidos | 723 | 0.08% | ✅ Ejecutado y validado |
| Grupo 2 — Duplicados | 432,473 | 48.9% | 🔄 Corrección de fuente aplicada, pendiente re-ejecución |
| Grupo 3 — Inactivos | 1,127 | 0.13% | ✅ Validado, pendiente ejecución |
| Grupo 4 — Sin canal | 450,653 | 50.9% | ✅ Proceso construido y validado, pendiente ejecución |
| **Total** | **884,976** | **100%** | |

> Base total de contactos en MCE: **7,895,021** (Contact Builder, corte 06/29/2026).
> Los 4 grupos representan una limpieza del **11.2%** del total de la base.
> Tras ejecutar los 4 grupos la base quedaría en aproximadamente **7,010,045 contactos**.

---

## 9. Trazabilidad y Auditoría

Toda ejecución de los Grupos 1-4 queda registrada en `DE_Log_Borrado_Contactos_P1` con: fecha, grupo, lote, cantidad de registros, Operation ID de la API, status (OK/Skipped/Error/Exception) y mensaje de respuesta — permitiendo reconstruir el historial completo del proceso ante cualquier auditoría posterior.

---

*Documento generado como parte del proceso de higiene de base de contactos — MCE Seguros Bolívar 2026.*
