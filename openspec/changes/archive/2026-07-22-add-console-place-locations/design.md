# Design: add-console-place-locations

## Context

`console/lugar.html` es un archivo autocontenido (HTML+CSS+JS vanilla, supabase-js UMD con anon key) con secciones `· 01 ·` a `· 05 ·`. `gather()` arma el payload de `places` con las 5 columnas espejo (`address`, `lat`, `lng`, `phone`, `whatsapp`); `save()` hace insert (lugar nuevo) o update, y luego `saveSocialLinks()` — el patrón de tabla hija (load por `place_id`, upsert, desactivación suave) ya existe en este archivo desde `add-place-social-links`. El backend definió (webhook `add-place-locations`): sedes en `place_locations`, identidad `(place_id, label)`, una sola `is_primary` por lugar (índice único parcial), columnas planas de `places` como espejo de la principal, sedes se desactivan (nunca delete). La sección 03 tiene un input `f-barrio` deshabilitado y sin JS — placeholder muerto que este change enciende.

## Goals / Non-Goals

**Goals:**

- Cerrar la grieta activa: altas de lugar desde Console crean su sede `Principal`; ediciones de staff mantienen `places` y `place_locations` coherentes sin depender del seed.
- Dar al staff visibilidad y edición de las sedes de cadenas (grid CRUD suave) sin cambiar el flujo de trabajo del lugar mono-sede.
- Respetar el contrato del backend: espejo, una primaria, democión antes de coronar, soft-delete.
- Cero build step, cero dependencias nuevas (convención del repo).

**Non-Goals:**

- Horarios por sede (sección 05 intacta), multi-pin en `mapa.html`, RLS/hardening, edición de sedes múltiples durante la creación del lugar, y cambios al modelo de datos (se decide en el webhook).

## Decisions

### D1 — Modelo de edición por cardinalidad (write-through donde escribe el staff)

- **Mono-sede** (una sede activa): secciones 03/04 editables como hoy; `save()` escribe `places` y además sincroniza la fila `Principal` (update por id con los mismos 5 campos + zona de `f-barrio`). El staff no nota ningún cambio de flujo.
- **Multi-sede** (2+ activas): 03/04 se muestran deshabilitadas con nota "Este lugar tiene N sedes — edita en · 06 · SEDES"; la grid es la única vía de escritura. Al guardar, las columnas espejo de `places` se toman de la fila marcada principal en la grid.

Alternativas descartadas: grid-siempre (rompe el flujo del 95% de los lugares por el 5%), solo-lectura (no cierra la grieta 2: la edición de 03/04 seguiría desincronizando las sedes).

Regla unificadora: **el espejo lo mantiene quien escribe** — en mono-sede escriben 03/04 (y arrastran la sede), en multi-sede escribe la grid (y arrastra el espejo).

### D2 — Ediciones de sede por `id` de fila, no por label

El Console tiene los `id` reales de `place_locations` (a diferencia del seed, que matchea por label): updates con `.eq('id', ...)`. Renombrar un label es un update normal — sin baja+alta, sin riesgo de fila duplicada. Las sedes nuevas son `insert` (no upsert): un conflicto `(place_id, label)` significa label repetido y debe aflorar como error de validación, no resolverse en silencio.

### D3 — Cambio de principal: demote-first, en dos updates secuenciales

El índice `place_locations_one_primary` rechaza dos primarias. Secuencia: (1) update sede actual `is_primary=false`; (2) update sede nueva `is_primary=true`; (3) sincronizar espejo de `places`. Sin transacciones en PostgREST: si (2) falla, el lugar queda momentáneamente sin primaria — estado que el índice permite y que el propio guardado reintentable corrige; la UI muestra el error y mantiene el botón de guardar activo. Es el mismo orden que usa el seed (`applyLocationsPlan`).

### D4 — Alta de lugar crea la sede Principal en el mismo flujo

En `save()` con `state.isNew`: tras el `insert` de `places` (que ya devuelve `id`), insertar `place_locations` con `label='Principal'`, `is_primary=true` y los valores de 03/04 — mismo lugar del flujo donde ya vive `saveSocialLinks(data.id)`. Si el insert de la sede falla, el lugar queda creado sin sede (estado igual al bug actual, no peor): toast de error explícito pidiendo reintentar guardado.

### D5 — Desactivación suave con guardas de UI

Desactivar una sede = `active=false` + `is_primary=false` (una sede inactiva no puede ocupar el slot del índice). La UI impide desactivar la principal (primero hay que coronar otra) y muestra las inactivas colapsadas ("N sedes inactivas") con opción de reactivar. Nunca `delete` de sedes; el delete de lugar existente (botón actual) arrastra las sedes por el `on delete cascade` de la FK — comportamiento aceptado, sin cambios.

### D6 — Validaciones en la grid

Label obligatorio y único (case/espacios-insensible, mismo criterio que el seed) dentro del lugar; exactamente una principal activa; lat/lng numéricos o vacíos; zona opcional. Los errores bloquean el guardado con el mismo mecanismo de toasts que el resto del formulario.

### D7 — Chip "+N sedes" en `lugares.html` con una query batcheada

Tras cargar la página de resultados, una sola query `place_locations` con `.in('place_id', ids)` filtrando activas; conteo por lugar en memoria; chip solo cuando N>1. Sin joins en la query principal (no cambiar su forma actual).

## Risks / Trade-offs

- **[Sin transacciones: guardado multi-paso puede fallar a medias]** → orden elegido para que cada estado intermedio sea recuperable reintentando el guardado (D3/D4); errores siempre visibles con toast, nunca silenciosos.
- **[Seed puede pisar ediciones de sedes del Console]** → riesgo preexistente documentado en el webhook (tasks 5.5 de `add-place-locations`); sedes agregadas por Console (label fuera del JSON) están protegidas sin `--allow-clear`. Aceptado en esta entrega.
- **[Dos escritores del espejo (Console y seed)]** → ambos aplican la misma regla (espejo = sede principal), así que convergen; una divergencia solo puede venir de ediciones SQL manuales.
- **[lugar.html crece (~1.500 → ~1.900 líneas)]** → el patrón social-links mantiene la sección aislada (estado propio, render propio, save propio); no se refactoriza el archivo en este change.

## Migration Plan

1. Deploy estático normal (Vercel/Netlify) — la migración `0008` ya está aplicada, no hay orden de deploy.
2. Smoke manual post-deploy: abrir un lugar mono-sede (Principal visible, 03/04 editables), una cadena (grid con sedes, 03/04 bloqueadas), crear un lugar de prueba (verificar fila `Principal` en Supabase) y desactivarlo.

Rollback: revertir el deploy estático; los datos escritos son compatibles con el modelo (no hay formato nuevo).

## Open Questions

- ¿Backfill de sedes para lugares creados por Console entre la migración `0008` y este deploy? (Si hubo altas en esa ventana, les falta su `Principal`; detectarlas es un query simple — `places` sin fila en `place_locations`. Se decide al implementar; puede ser una acción manual única.)
