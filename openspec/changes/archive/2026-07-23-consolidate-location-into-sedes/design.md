# Design: consolidate-location-into-sedes

## Context

`console/lugar.html` (~2300 líneas, autocontenido, supabase-js por CDN) edita ubicación con una regla bimodal por cardinalidad: mono-sede escribe en 03/04 y `saveSedes` copia a la sede `Principal`; multi-sede bloquea 03/04 (`applyCardinalityMode` + `MIRROR_FIELD_IDS`), `syncReflejo()` los convierte en reflejo de la sede ⭐, y `gather()` lee esos inputs sincronizados para armar el espejo de `places`. El contrato de datos (change `add-place-locations` del webhook) es: `place_locations` = verdad, columnas planas de `places` = espejo de la `is_primary`, mantenido por quien escribe. Desde el retiro del seeder (2026-07-22) el Console es el único guardián del espejo. `place_social_links.place_location_id` existe desde la migración 0009, reservado para redes por sede, hoy siempre null.

## Goals / Non-Goals

**Goals:**

- Una sola regla de edición de ubicación/contacto por sede: la grid de SEDES, siempre.
- Eliminar la sección 03 (incluido el mapa de preview) y los campos espejo de 04; ciudad/región suben a datos generales.
- Mantener intacto el contrato del espejo: `places.address/lat/lng/phone/whatsapp` = sede `is_primary`, calculado al guardar.
- Capturar el link corto de Google Maps por sede (`place_social_links` + `place_location_id`) y retirar `google_maps` del dropdown de redes de marca.

**Non-Goals:**

- Tocar `quepa-webhook` (el enrutado de links por sede en `get_place_details` es un change hermano, prerequisito de despliegue).
- Reubicar el mapa de preview (se elimina; si vuelve, es otro change).
- Captura por sede de otras plataformas (Instagram/TikTok/etc. siguen a nivel marca; el schema ya lo soportaría).
- Migraciones SQL (cero — el modelo ya soporta todo).
- Cambios en `console/lugares.html` (el chip "+N sedes" queda igual).

## Decisions

**D1 — Grid como única superficie; espejo derivado en memoria.** `gather()` deja de leer los inputs de 03/04 para el espejo: toma `address/lat/lng/phone/whatsapp` de `primarySede()` en el estado de la grid. Muere toda la maquinaria bimodal (`syncReflejo`, `applyCardinalityMode`, `MIRROR_FIELD_IDS`, la rama mono-sede de `saveSedes`, las notas de bloqueo). El orden de guardado se conserva (`places` → `saveSedes` → `saveSocialLinks`): `places` primero porque el alta necesita el `place_id` para las FKs; el estado de la grid ya está en memoria antes de escribir, así que el espejo sale correcto. *Alternativa rechazada:* dejar 03 como resumen solo-lectura — mantiene la duplicación visual que originó la confusión.

**D2 — Ciudad y región son identidad de marca, no ubicación.** Suben a la sección de datos generales, siguen obligatorios (alimentan `unique (name, city)`, la herencia `location.city ?? ciudad del lugar` y el selector de zonas por ciudad efectiva). El `city` por sede sigue nullable con placeholder "= ciudad del lugar".

**D3 — Mapa de preview eliminado, feedback textual.** `renderMap` y el centroide-como-referencia mueren con la sección. La confirmación de coordenadas queda en el resumen textual por sede ("4.79906, -75.80593" / "sin coordenadas") del detalle avanzado existente. El invariante "el centroide jamás se persiste en `lat`/`lng`" se conserva trivialmente: ya no hay código que lo lea.

**D4 — Alta con fila `Principal` pre-creada.** Un lugar nuevo (y un huérfano sin sedes activas — lugares creados entre despliegues) muestra la grid con una fila `Principal` marcada ⭐ lista para llenar. Reemplaza la síntesis desde 03/04 en `saveSedes`; la validación pasa a exigir al menos una sede activa con exactamente una principal (hoy `validateSedes` permite cero sedes).

**D5 — Link de Maps por sede: campo en la fila, persistencia en `place_social_links`.** Cada fila de sede gana "Link de Maps (corto — es lo que ve el cliente)". Guardar escribe `{platform: 'google_maps', url, place_location_id: sede.id, active: true}`; editar la URL actualiza la fila por `id`; vaciar el campo desactiva (`active=false`, coherente con la regla de redes). `is_primary` de estas filas queda `false` — la primacía la da la sede, no la fila. Orden de guardado ya correcto: `saveSedes` asigna `id` a sedes nuevas antes de que `saveSocialLinks` los necesite. *Nota de rotulado:* la nota de "link corto no trae coordenadas" del campo de URL-para-coords ahora remite a este campo.

**D6 — Adopción de filas `google_maps` existentes (todas con `place_location_id` null hoy).**
- Mono-sede: al cargar, la URL de la fila de marca aparece en el campo de la sede `Principal` con nota "se asignará a esta sede al guardar"; guardar hace `update` de esa misma fila (mismo `id`) poniendo `place_location_id`. Automático porque no hay ambigüedad posible, pero visible antes de ejecutarse.
- Multi-sede: la fila de marca aparece en el módulo de sedes como "link sin sede asignada" con su URL y un selector de sede; el staff decide y guardar actualiza la misma fila. Sin asignar, queda dormida (comportamiento del webhook de hoy: no se promueve a `maps_url`). Nunca se migra en silencio — misma filosofía que la transición de zonas legado.
- `google_maps` sale del dropdown de redes de marca; las filas existentes dejan de editarse desde la lista de redes (su casa nueva es el módulo de sedes).

**D7 — Orden de despliegue cross-repo.** El change hermano de `quepa-webhook` (enrutar filas con `place_location_id` al `locations[]` de `get_place_details`, coronando el `maps_url` de esa sede y excluyéndolas de `social_links` de marca) se despliega ANTES. Es inocuo hacia atrás: mientras el Console no escriba `place_location_id`, no cambia nada. Rollback del Console: revertir el deploy estático; las filas ya escritas con `place_location_id` siguen siendo entendidas por el webhook.

## Risks / Trade-offs

- [Refactor grande en un archivo de 2300 líneas sin tests] → checklist manual de regresión en tasks (alta, edición mono y multi, coronación con democión previa, desactivación/reactivación, zonas canónicas y legado, horarios intactos, redes de marca intactas), probado en desktop y mobile según convención del repo.
- [Guardado parcial: `places` actualizado y `saveSedes` falla → espejo adelantado a la verdad] → mismo perfil de hoy: error visible + reintento repara. No empeora.
- [Pérdida del mapa: el staff pierde la verificación visual de coordenadas] → aceptado por decisión de producto; el aviso "fuera del rango de Colombia" del change `console-maps-link-coords` sigue vivo como red de seguridad.
- [Links `google_maps` multi-sede que nadie asigna] → quedan visibles como "sin sede asignada" (no invisibles, no borrados); el webhook los ignora igual que hoy.
- [Cambio de hábito del staff (03 desaparece)] → la superficie única se autoexplica; actualizar `quepa-landing/CLAUDE.md` y avisar al equipo en el cierre.

## Open Questions

- Ninguna bloqueante para el Console. La semántica fina del lado webhook (¿el link de la sede principal corona también el `maps_url` de marca en mono-sede? ¿qué pasa con un link de marca residual?) se decide en el change hermano.
