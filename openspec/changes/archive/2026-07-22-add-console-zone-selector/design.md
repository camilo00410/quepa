# Design: add-console-zone-selector

## Context

`console/lugar.html` captura zona en dos puntos: `f-barrio` (sección 03, espejo de la sede principal en mono-sede) y la columna Zona de la grid de sedes (multi-sede). Ambos son inputs de texto libre que hoy escriben `place_locations.zone`. El backend (`add-city-zones` en quepa-webhook) introduce `city_zones` (gazetteer curado por ciudad: nombre canónico, centroide, aliases) y `place_locations.zone_id` (FK nullable), con el texto `zone` deprecado-pero-intacto y fallback de lectura en el agente. Contratos que este change debe respetar: el Console **jamás inserta zonas** y **jamás escribe más el texto legado**; la regla espejo↔sede por cardinalidad (D1 de `add-console-place-locations`) sigue igual. Restricción del repo: archivos autocontenidos, sin build ni dependencias.

## Goals / Non-Goals

**Goals:**

- Capturar zona por referencia canónica (`zone_id`) en los dos puntos de edición, sin cambiar el flujo de trabajo del staff.
- Hacer visible el estado legado (texto sin FK) y su transición uno-a-uno a canónico, sin migraciones silenciosas.
- Darle al staff la "idea de dónde queda" con el centroide — sin contaminar las coordenadas reales de la sede.

**Non-Goals:**

- CRUD de zonas desde el Console (el gazetteer se cura en el webhook).
- Geocoding de direcciones o autocompletar lat/lng con el centroide.
- Cambios en el listado (`lugares.html`) o el mapa global (`mapa.html`).

## Decisions

### D1 — `<select>` nativo poblado por ciudad efectiva, cacheado por sesión de edición

Una query a `city_zones` (activas, de la ciudad) por ciudad tocada, cacheada en `state.zonesByCity`. `f-barrio` usa la ciudad del lugar (`f-city`); cada fila de la grid usa `location.city ?? ciudad del lugar` (caso AMCO). Select nativo (sin datalist: el valor debe ser una FK, no texto; sin librerías: convención del repo). Opciones: "— Sin zona —" (null) + zonas ordenadas alfabéticamente. Si cambia `f-city`, los selects se repueblan y las selecciones que ya no apliquen caen a "Sin zona" con aviso.

### D2 — El estado legado es una opción visible, no un valor editable

Sede con `zone_id` null y texto legado: el select muestra una opción extra deshabilitada-como-actual `"cuba" · texto legado` seleccionada, para que el staff vea qué había. Elegir una zona canónica asigna `zone_id` (el texto legado queda intacto en la base, invisible desde entonces en la UI porque la FK gana — misma regla de resolución del backend). Volver a "Sin zona" pone `zone_id=null` y el legado reaparece. Así la transición es explícita, sede por sede, hecha por humanos.

### D3 — El guardado escribe solo `zone_id`; el espejo sigue la cardinalidad existente

`saveSedes()` cambia `zone: texto` por `zone_id: uuid|null` en sus tres ramas (alta de Principal, mono-sede que copia de 03, upsert de grid). `syncReflejo()` y la hidratación de `loadPlace()` (fix del 2026-07-22) pasan a resolver: nombre canónico si la sede tiene `zone_id`, texto legado si no — la misma regla única del backend, implementada una vez en un helper del archivo. `gather()` no cambia (la zona nunca fue columna de `places`).

### D4 — Centroide display-only: centra el mapa, jamás se persiste

Si la sede no tiene `lat`/`lng` propios y se selecciona zona, `renderMap()` centra en el centroide con un marcador diferenciado de "zona aproximada". Los inputs `f-lat`/`f-lng` quedan como están (vacíos). Alternativa descartada: prefill de lat/lng con el centroide — persistiría una aproximación como si fuera dato exacto, sin flag que las distinga, y envenenaría la etapa 2b de proximidad.

### D5 — Degradación honesta sin gazetteer

Si la query de zonas falla o la ciudad no tiene zonas: selects deshabilitados con nota ("Sin zonas para {ciudad} en el gazetteer — se curan en el webhook"), la sede conserva lo que tenga (FK o legado) y el guardado NO se bloquea (guarda el resto de campos; `zone_id` queda como estaba). No se reabre texto libre: sería regresar la fragmentación que este change elimina.

## Risks / Trade-offs

- [El staff necesita una zona que no existe] → nota con el contrato ("se cura en el webhook") + la sede puede guardarse sin zona; el gazetteer crece por el flujo curado. Fricción aceptada: es la garantía de canonicidad.
- [Ciudades sin gazetteer al momento del deploy] → D5 degrada sin bloquear; secuencia de despliegue exige gazetteer de Pereira/AMCO poblado primero (donde está el catálogo real).
- [Cambiar `f-city` invalida selecciones de zona] → repoblar + caída explícita a "Sin zona" con aviso; caso raro (la ciudad de un lugar casi nunca cambia).
- [Dos cargas extra por lugar (zonas por ciudad)] → una query liviana cacheada por ciudad; mismo patrón de queries hijas ya usado (sedes, redes).

## Migration Plan

1. Pre-requisito: `add-city-zones` desplegado (migración `0010` + gazetteer Pereira/AMCO poblado + backfill corrido).
2. Deploy estático normal del Console (Vercel).
3. Smoke: lugar mono-sede con `zone_id` (select muestra canónico), lugar solo-legado (opción "texto legado", asignar canónica, verificar `zone_id` en Supabase y texto intacto), sede sin coordenadas con zona (mapa centrado en centroide, `f-lat`/`f-lng` siguen vacíos), ciudad sin zonas (degradación con nota).

Rollback: revertir el deploy estático; `zone_id` ya escrito queda válido (el backend lo resuelve igual).

## Open Questions

- ¿Mostrar el conteo de sedes-solo-legado en algún lugar del Console como cola de curación visible? (Barato: chip en el detalle; puede ser follow-up.)
- Presentación exacta del marcador "zona aproximada" en el mapa (estilo/copy) — decidir en implementación con el mapa real.
