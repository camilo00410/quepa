# Proposal: add-console-place-locations

## Why

El backend (`quepa-webhook`, change `add-place-locations`, migración `0008` ya aplicada) partió el modelo en marca (`places`) + sedes (`place_locations`), con las columnas planas de `places` como espejo de la sede principal. El Quepa Console quedó ciego a ese modelo y eso abre dos grietas **activas hoy**: (1) un lugar creado desde `lugar.html` inserta en `places` sin crear su sede — rompe el invariante "todo lugar tiene una sede principal" para cada alta de staff (el backfill fue one-time); (2) una edición de dirección/contacto en el Console actualiza el espejo pero no la fila de sede, así que el scoring por zona de `search_places` (que lee `place_locations`) trabaja con datos viejos hasta el próximo seed. Además el staff no tiene forma de ver ni corregir las sedes de las cadenas ya consolidadas (4 marcas, 17 sedes).

## What Changes

- Nueva sección `· 06 · SEDES` en `console/lugar.html`: grid de sedes del lugar (label, zona, dirección, ciudad, lat/lng, teléfono, WhatsApp, principal ⭐, activa) cargada de `place_locations`, con el patrón ya probado por la sección de redes sociales (tabla hija, borrado suave).
- **Fix grieta 1**: crear un lugar desde el Console también crea su sede `Principal` (`is_primary=true`) con los datos de las secciones 03/04, en el mismo flujo de guardado (mismo patrón que `saveSocialLinks`).
- **Modelo de edición por cardinalidad**: en lugares mono-sede (el 95%: staff no cambia su flujo) las secciones `· 03 · UBICACIÓN` y `· 04 · CONTACTO` siguen editables y al guardar escriben `places` **y** la sede `Principal` (espejo mantenido en el punto de escritura). En lugares multi-sede, 03/04 pasan a solo lectura (reflejo de la sede principal, con nota "edítala en SEDES") y la verdad se edita en la grid; al editar la sede principal en la grid, el guardado sincroniza las columnas espejo de `places`.
- Ediciones de sede por `id` de fila (rename de label seguro — el Console no depende del match por label como el seed). Cambio de principal con democión previa (dos updates en orden: el índice único parcial `place_locations_one_primary` rechaza dos primarias). Desactivación suave (`active=false` + `is_primary=false`), nunca delete; la UI impide desactivar la sede principal.
- El campo `f-barrio` (hoy deshabilitado con placeholder "—", sin JS) se habilita como la **zona** de la sede correspondiente.
- `console/lugares.html`: chip "+N sedes" en el listado para lugares multi-sede (cosmético, una query batcheada).

Fuera de alcance (explícito): horarios por sede (los horarios siguen a nivel marca, sección 05 intacta — decisión del change de backend), multi-pin de sedes en `mapa.html` (sinergia futura cuando las sedes tengan coordenadas), edición de sedes en creación de lugar (un lugar nuevo nace mono-sede; las sedes extra se agregan después), y cualquier cambio de RLS/grants (el schema tiene RLS deshabilitado — hardening pendiente preexistente, nota en migración `0005` del webhook).

## Capabilities

### New Capabilities

- `console-place-locations`: gestión de sedes en el Quepa Console — visualización de la grid, creación de la sede Principal en altas de lugar, sincronización espejo↔sede en el guardado según cardinalidad, edición segura por id (rename, cambio de principal con democión, desactivación suave) y zona por sede.

### Modified Capabilities

<!-- Ninguna: console-place-hours (sección 05) no se toca; sus columnas siguen a nivel places. -->

## Impact

- **Código**: `console/lugar.html` (sección nueva + flujo de guardado + habilitar `f-barrio`; HTML/CSS/JS vanilla autocontenido, convención del repo: sin build step ni dependencias) y `console/lugares.html` (chip de conteo).
- **Datos**: lecturas/escrituras a `place_locations` vía supabase-js UMD con la anon key (mismo canal que `places` y `place_social_links`; RLS deshabilitado en todo el schema — sin grants nuevos).
- **Contrato cross-repo**: el modelo de datos es el de `quepa-webhook` (`add-place-locations` D1/D2): identidad de sede `(place_id, label)`, exactamente una `is_primary` por lugar (índice único parcial), columnas planas de `places` = espejo de la principal. Cualquier cambio de modelo se decide allá; el Console es consumidor/escritor.
- **Dependencia de despliegue**: la migración `0008` ya está aplicada (verificado). Sin ella la sección degradaría a "sin sedes" en lectura y fallaría al guardar — no aplica.
- **Riesgo conocido preexistente (documentado en el webhook)**: un `pnpm seed --apply` futuro puede pisar campos de sedes editados en el Console si esa sede existe en `data/*.json` con valores propios. Matiz a favor: una sede *agregada* desde el Console (label ausente del JSON) está protegida del seed sin `--allow-clear`. Mostrar la procedencia no existe para sedes (no hay columna source) — mitigación: ninguna en esta entrega, riesgo aceptado igual que en dirección/teléfono planos.
