# Proposal: add-console-place-hours

## Why

El backend (`quepa-webhook`, change `add-place-hours`) ya definió el modelo de horarios en `places` (`hours` JSONB semanal, `hours_note`, `hours_source`, `hours_verified_at`) y reservó explícitamente el carril `hours_source='manual'` para curación humana — pero no existe ninguna herramienta que lo use: hoy corregir un horario mal parseado o registrar el de los ~600 lugares sin fuente (lotes Maps web) exige SQL a mano. El detalle de lugar del Quepa Console (`console/lugar.html`) es el editor natural del catálogo y no muestra ni permite editar horarios.

## What Changes

- Nueva sección `· 05 · HORARIOS` en `console/lugar.html`: grilla semanal editable (lunes–domingo, múltiples franjas por día, soporte de cruce de medianoche y 24h), campo `hours_note`, y metadatos de procedencia (`hours_source` + `hours_verified_at`) visibles en solo lectura.
- El guardado del lugar incluye los horarios **solo si el staff los tocó**: en ese caso escribe `hours` (o `null` si los borró todos), `hours_note`, `hours_source='manual'` y `hours_verified_at=now()`. Si la sección no se tocó, el payload de guardado no incluye ninguna columna de horarios (comportamiento actual intacto: no se pisa lo que puso el backfill).
- Semántica del modelo respetada en UI: `hours=null` se muestra como "Sin horario registrado" (estado DESCONOCIDO, nunca "cerrado"); `[]`/clave ausente = cerrado ese día; `cierre < apertura` = cruza medianoche (se señala visualmente, no es error); `00:00–24:00` = abierto 24 horas.
- Fuera de alcance (explícito): badge "abierto ahora" en el console (la aritmética de apertura vive server-side en `quepa-webhook/src/lib/place-hours.ts` con tests; no se duplica en JS vanilla sin tests), horarios en el listado `lugares.html`, y calendario de festivos.

## Capabilities

### New Capabilities

- `console-place-hours`: sección de horarios en el detalle de lugar del Quepa Console — visualización de la grilla semanal con su semántica (desconocido/cerrado/medianoche/24h), edición manual con marcado de procedencia `manual`, y guardado no destructivo (solo escribe si hubo edición).

### Modified Capabilities

<!-- Ninguna: raíz OpenSpec nueva en quepa-landing, no hay specs previos. El change add-place-hours de quepa-webhook no se modifica — este change solo consume su modelo de datos. -->

## Impact

- **Código**: `console/lugar.html` únicamente (HTML + CSS + JS vanilla autocontenidos, convención del repo: sin build step ni dependencias nuevas; supabase-js UMD ya está cargado).
- **Datos**: escrituras a `places.hours`, `places.hours_note`, `places.hours_source`, `places.hours_verified_at` vía supabase-js con la sesión de staff (mismo canal que el resto del formulario). Ninguna columna nueva: la migración `0007_place_hours.sql` vive en `quepa-webhook`.
- **Dependencia de despliegue**: si la migración `0007` aún no está aplicada en Supabase (tarea 1.3 de `add-place-hours` pendiente), la sección degrada a "Sin horario registrado" en lectura, pero **guardar una edición de horarios fallaría** (columna inexistente). La UI debe manejar ese error de forma visible; el orden sano es migración primero.
- **Contrato cross-repo**: el formato JSONB de `hours` es el definido por `quepa-webhook` (D1 de `add-place-hours`). Cualquier cambio de formato se decide allá; el console es consumidor.
- **Riesgo conocido preexistente**: un lugar presente en `data/*.json` de `quepa-webhook` con `hours` non-null propios pisaría la edición manual del console en un futuro `pnpm seed --apply` (el merge protector solo protege contra `null`). Mostrar `hours_source` en la sección mitiga informando qué se está tocando.
