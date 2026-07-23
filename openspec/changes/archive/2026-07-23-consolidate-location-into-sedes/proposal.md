# Proposal: consolidate-location-into-sedes

## Why

La edición de ubicación en `console/lugar.html` es bimodal y confunde al staff: en mono-sede se edita en las secciones 03/04 (que copian a la sede Principal), en multi-sede se edita en la grid de sedes (y 03/04 se bloquean como reflejo). Son dos superficies que muestran el mismo dato con reglas de escritura que cambian según la cardinalidad — el equipo de datos no sabe cuál editar ni por qué a veces está bloqueada. La grid de sedes ya captura todo lo físico; 03/04 solo sobreviven como espejo editable-a-veces.

## What Changes

- **La grid de SEDES pasa a ser la única superficie de edición** de ubicación y contacto por sede (dirección, zona, ciudad, lat/lng, teléfono, WhatsApp), en mono y multi-sede por igual. Muere la regla bimodal D1 (`syncReflejo`, `applyCardinalityMode`, `MIRROR_FIELD_IDS`, la rama mono-sede de `saveSedes`).
- **La sección `· 03 · UBICACIÓN` desaparece**, incluido el mapa de preview (se elimina, no se reubica). Ciudad y región — identidad de la marca (`places.city`/`region`, no espejo) — suben a la sección de datos generales.
- **La sección 04 pierde teléfono y WhatsApp** (campos espejo, ya existen por sede en la grid) y conserva lo de marca: website, booking_url y redes sociales.
- **El espejo de `places`** (`address`, `lat/lng`, `phone`, `whatsapp`) se calcula de la sede `is_primary` en memoria al guardar — el contrato con `quepa-webhook` (RPC `search_places` y links del agente leen columnas planas) no cambia.
- **Alta de lugar**: la grid arranca con una fila `Principal` pre-creada para llenar (reemplaza la regla D4 de sintetizarla desde 03/04).
- **Cada sede gana un campo "link corto de Google Maps"** persistido en `place_social_links` con `place_location_id` (columna reservada desde la migración 0009, hoy siempre null). `google_maps` sale del dropdown de redes sociales de marca. Filas `google_maps` existentes sin sede: en mono-sede se auto-asignan a la Principal al guardar (sin ambigüedad posible); en multi-sede quedan visibles como "sin sede asignada" y el staff las asigna.
- **BREAKING (flujo de staff, no de datos)**: el lugar mono-sede deja de editarse en 03/04; toda edición de ubicación pasa por la grid.

## Capabilities

### New Capabilities

- `console-sede-maps-link`: captura por sede del link corto de Google Maps (el que se comparte al cliente), persistido en `place_social_links.place_location_id`; retiro de `google_maps` del dropdown de redes de marca y adopción de las filas existentes.

### Modified Capabilities

- `console-place-locations`: la grid es la única vía de edición siempre (desaparecen los requirements de guardado mono-sede vía 03/04 y de secciones bloqueadas en multi-sede); el alta pre-crea la fila `Principal` en la grid; el espejo de `places` se deriva de la sede principal al guardar.
- `console-maps-link-coords`: la captura de coordenadas por URL queda solo por sede en la grid (muere el campo de 03); las referencias al pin del mapa se eliminan (ya no hay mapa); la nota del link corto ahora redirige al campo nuevo de link de Maps de la sede.
- `console-zone-selector`: muere el selector espejo `f-barrio` de 03 (queda solo la columna Zona de la grid); el requirement del centroide como referencia visual del mapa se retira junto con el mapa (la prohibición de persistir centroides en `lat`/`lng` se conserva como invariante).

## Impact

- **Código**: `console/lugar.html` (refactor mayor del formulario: remover 03, mover ciudad/región, remover mapa y su render, rework de `gather()`/`save()`/`saveSedes()`, campo maps-link por sede, dropdown de redes sin `google_maps`). `console/lugares.html` no cambia (el chip "+N sedes" queda igual).
- **Base de datos**: cero migraciones — el modelo ya soporta todo (`place_social_links.place_location_id` existe desde 0009).
- **Cross-repo / orden de despliegue**: requiere un change hermano en `quepa-webhook` desplegado ANTES — `get_place_details` debe enrutar las filas `google_maps` con `place_location_id` a su entrada en `locations[]` (coronando el `maps_url` de esa sede) y excluirlas de `social_links` de marca. Hasta que el Console no escriba filas por sede, ese change es inocuo hacia atrás.
- **Docs**: actualizar `quepa-landing/CLAUDE.md` (secciones de sedes y redes) al cerrar la implementación.
