# Design: add-console-place-hours

## Context

`console/lugar.html` es una página autocontenida (HTML + CSS + JS vanilla, supabase-js UMD por CDN) que edita filas de `places` directo contra Supabase con la sesión de staff. El formulario junta secciones `01 Identidad · 02 Económico · 03 Ubicación · 04 Contacto`; `gather()` arma un payload de columnas explícitas y `save()` hace `update(payload)` — hoy no incluye ninguna columna de horarios, por lo que guardar nunca los pisa.

El modelo de datos viene de `quepa-webhook` (change `add-place-hours`, D1): `hours` JSONB con claves `mon..sun`, cada día una lista de intervalos `["HH:MM","HH:MM"]`; `[]` o clave ausente = cerrado; `cierre < apertura` = cruza medianoche; `[["00:00","24:00"]]` = 24 horas; `hours` null = **desconocido** (jamás "cerrado"). Columnas hermanas: `hours_note` (texto libre), `hours_source` (`google`/`osm`/`manual`), `hours_verified_at`. El backfill del webhook respeta `hours_source='manual'`: no lo pisa.

Convenciones del repo que restringen el diseño: sin build step, sin dependencias nuevas, tokens de marca duplicados por archivo, copy en español.

## Goals / Non-Goals

**Goals:**

- Ver el horario semanal de un lugar con la semántica correcta del modelo (desconocido ≠ cerrado, medianoche, 24h).
- Editar horarios manualmente marcando procedencia: `hours_source='manual'` + `hours_verified_at=now()`.
- Guardado no destructivo: si el staff no tocó la sección, el payload no incluye columnas de horarios.
- Ver procedencia (`hours_source`) y fecha de verificación antes de decidir editar.

**Non-Goals:**

- Badge "abierto ahora" o cualquier aritmética de apertura: vive server-side en `quepa-webhook/src/lib/place-hours.ts` con tests; no se duplica en JS vanilla sin tests.
- Horarios en el listado `lugares.html` o en otras páginas del console.
- Festivos / excepciones estructuradas: `hours_note` es texto libre y basta.
- Validar contra el parser del webhook (`hours-parser.ts`): el console emite el formato canónico directamente.

## Decisions

### D1 — Sección `· 05 · HORARIOS` en el form principal, con los patrones visuales existentes

Nueva `.section` después de `· 04 · CONTACTO Y RESERVAS` en `.detail-form`. Reutiliza `.field-d`, `.label`, `.help` y la tipografía mono para las horas. La grilla es una fila por día (`LUN..DOM` ↔ claves `mon..sun`), cada franja son dos inputs de texto `HH:MM` + botón de quitar, y un botón "+ franja" por día. Sin franjas, el día muestra "Cerrado".

Se descartó `<input type="time">`: no puede representar `24:00` (cierre de "abierto 24 horas") y su UI nativa estorba más de lo que aporta para horas escritas a mano. Inputs de texto con validación propia (regex) mantienen el formato canónico visible tal cual se persiste.

### D2 — Grilla vacía = `null` (desconocido), nunca "cerrado los 7 días"

Si el staff quita todas las franjas de todos los días, el serializador emite `hours = null` (+ `hours_source = null`, `hours_verified_at = null`), no `{mon: [], ...}`. Razón: un lugar cerrado 7/7 no pertenece activo al catálogo — el caso real de "borré todo" es "no sé el horario / quiero limpiarlo", y el estado seguro del modelo es desconocido. Esto además hace innecesario un botón aparte de "quitar horario": vaciar la grilla ES quitar el horario. Días individuales sin franjas dentro de un horario con contenido sí serializan `[]` (cerrado ese día).

Cuando `hours` es null, la sección muestra "· Sin horario registrado" con un botón "Registrar horario" que despliega la grilla vacía en modo edición.

### D3 — Serialización canónica: las 7 claves siempre, en orden `mon..sun`

Al serializar un horario non-null se emiten las siete claves en orden fijo, con `[]` para los días cerrados. Semánticamente "clave ausente" y `[]` son equivalentes (D1 del webhook), pero el formato explícito y ordenado hace la columna legible en inspección directa de la base y estable ante comparaciones estructurales. Los intervalos de cada día se ordenan por hora de apertura.

### D4 — Dos flags de dirty: la grilla y la nota marcan procedencia distinto

- **Grilla tocada** (`hoursDirty`): el payload incluye `hours` (serializado), `hours_note`, `hours_source: 'manual'` y `hours_verified_at: now()` (o los tres primeros en null si la grilla quedó vacía, ver D2).
- **Solo la nota tocada** (`noteDirty` sin `hoursDirty`): el payload incluye únicamente `hours_note`. Corregir un typo en la nota no re-atribuye a `manual` un horario que vino de Google ni re-sella `hours_verified_at`.
- **Nada tocado**: el payload no incluye ninguna columna de horarios (idéntico al comportamiento actual).

Se descartó un solo flag por sección: sellaría `manual` sobre datos de `google` por ediciones que no tocan el horario, corrompiendo la procedencia que el backfill usa para decidir qué respetar.

### D5 — Validación al guardar, bloqueante y con mensaje concreto

Reglas: cada hora cumple `^([01]\d|2[0-3]|24):[0-5]\d$`, `24:00` solo como cierre, apertura ≠ cierre, y las franjas de un mismo día no se solapan (evaluadas en el día declarado; el cruce de medianoche `cierre < apertura` es válido y se señala visualmente con un indicador junto a la franja, no como error). Si falla, `save()` aborta con el toast de error existente nombrando el día y la franja. No se auto-corrige nada: mismo principio del proyecto — datos duros no se adivinan.

### D6 — Procedencia visible, solo lectura, actualizada tras guardar

Cabecera de la sección con chip de fuente (`GOOGLE` / `OSM` / `MANUAL` / `—`) y "verificado hace X" usando el helper `relTime()` existente. Tras un guardado con `hoursDirty`, la cabecera pasa a `MANUAL · hace un momento` sin recargar. No hay control para editar `hours_source` a mano: la única transición posible desde el console es a `manual`, y la ejecuta el guardado.

### D7 — Sin manejo especial de la migración ausente

Si la migración `0007` no está aplicada, la lectura degrada sola (`select('*')` no trae las claves → la sección muestra "Sin horario registrado") y un guardado con horarios falla con el error de PostgREST en el toast existente (`e.message`), visible y sin fingir éxito. No se agrega detección de columnas: el orden sano de despliegue es migración primero, y el fallo visible ya cumple el requisito de no fallar en silencio.

## Risks / Trade-offs

- [Un `pnpm seed --apply` futuro pisa una edición manual si el lugar vive en `data/*.json` con `hours` non-null propios] → riesgo preexistente del console (aplica igual a descripción o teléfono); el chip `MANUAL` visible informa qué se está tocando, y el backfill del webhook sí respeta `manual`. Resolverlo de raíz es trabajo del lado seed, no de este change.
- [Grilla vacía = null puede sorprender a quien quería decir "cerrado todos los días"] → caso irreal para lugares activos del catálogo; el estado null muestra "Sin horario registrado" inmediatamente al guardar, feedback suficiente para notarlo.
- [Inputs de texto libres admiten formatos basura hasta el guardado] → la validación D5 bloquea el guardado con mensaje concreto; no llega nada inválido a la base.
- [El copy de días y la serialización viven duplicados respecto al webhook (contrato cross-repo implícito)] → el formato es estable y está documentado en ambos CLAUDE.md; cualquier evolución del formato se decide en `quepa-webhook` y este editor se adapta.

## Migration Plan

1. Editar `console/lugar.html` (sección + estilos + JS). Sin migraciones propias: las columnas ya existen en `0007` de `quepa-webhook`.
2. Probar local (`python3 -m http.server` + sesión staff real) contra un lugar con horario de Google, uno sin horario y uno nuevo.
3. Deploy por el flujo normal de Vercel del console. Si la `0007` no está aplicada aún en Supabase, la sección degrada en lectura y el guardado de horarios falla visible (D7) — coordinar con el despliegue pendiente de `add-place-hours` (tarea 1.3).

Rollback: revertir el archivo; ninguna otra pieza depende de la sección.

## Open Questions

- ¿El console debería poder editar `hours_note` como campo estructurado de festivos más adelante? Hoy texto libre alcanza; si `add-place-hours` evoluciona a calendario de festivos, este editor se rediseña con ese change.
