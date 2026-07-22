# Proposal: add-console-zone-selector

## Why

El campo Zona/Barrio del Console es texto libre sin validación: cada variante que digita el staff ("Cuba", "cuba", "Barrio Cuba") fragmenta el matching por zona del agente, y digitar lat/lng a mano es el único modo de ubicar una sede. El backend define en `add-city-zones` (quepa-webhook, change hermano) el gazetteer `city_zones` (nombre canónico + centroide + aliases) y la FK `place_locations.zone_id` con fallback de lectura al texto legado. El Console es el escritor principal del catálogo (el seeder ya no ingresa datos nuevos), así que es donde la referencia canónica debe capturarse.

## What Changes

- **Zona/Barrio pasa de input libre a selector** poblado desde `city_zones` (zonas activas de la ciudad efectiva), tanto en la sección `· 03 · UBICACIÓN` (`f-barrio`, lugar mono-sede) como en la columna Zona de la grid `· 06 · SEDES` (por sede, con su `location.city ?? ciudad del lugar`). Incluye opción "Sin zona".
- **El guardado escribe `zone_id`**, no texto: el campo legado `zone` queda intacto como dato histórico (contrato del backend: deprecado, sin escrituras nuevas).
- **Estado legado visible**: una sede con `zone_id` null pero texto legado muestra ese texto marcado como no-canónico ("cuba · texto legado"); elegir una zona del selector asigna la FK sin tocar el texto. Nada se pierde ni se migra en silencio.
- **Centroide como pista visual, nunca como dato**: si la sede no tiene lat/lng propios, el mapa de la sección 03 se centra en el centroide de la zona seleccionada como referencia — el Console MUST NOT persistir el centroide en `lat`/`lng` de la sede (coordenadas exactas siguen siendo dato curado aparte).
- **El Console no crea zonas**: si el barrio no está en el gazetteer, el staff lo ve indicado (la zona nueva entra por el flujo curado del webhook); si el gazetteer no tiene zonas para la ciudad, el selector degrada deshabilitado con nota honesta, sin reabrir texto libre.

Fuera de alcance: crear/editar zonas desde el Console, geocoding de direcciones, cualquier uso del centroide en búsquedas (etapa 2b del backend), y cambios en `lugares.html`/`mapa.html`.

## Capabilities

### New Capabilities

- `console-zone-selector`: captura canónica de zona en el Console — selector desde `city_zones` por ciudad, escritura de `zone_id`, visualización del legado no-canónico, centroide como pista de mapa display-only y degradación honesta sin gazetteer.

### Modified Capabilities

- `console-place-locations`: el requisito "Guardado mono-sede sincroniza espejo y sede" cambia — la zona de la sede principal se guarda como `zone_id` (selector) en vez de texto de `f-barrio`, y la hidratación al cargar resuelve nombre canónico con fallback al texto legado.

## Impact

- **Código**: `console/lugar.html` (carga de zonas por ciudad con el cliente supabase-js existente, `f-barrio` → `<select>`, columna Zona de la grid → select por sede, ajuste de `saveSedes`/hidratación/`syncReflejo` a `zone_id`, mapa con centroide como centro por defecto). Autocontenido, sin dependencias nuevas — convención del repo.
- **Datos**: lectura de `city_zones` y escritura de `place_locations.zone_id` con la anon key (mismo canal actual; RLS deshabilitado en el schema — sin grants nuevos).
- **Dependencia dura cross-repo**: requiere la migración `0010` de `add-city-zones` aplicada y el gazetteer de la ciudad poblado ANTES de desplegar esto; sin la tabla, la carga de zonas fallaría (degradación con nota, pero el selector nacería inútil).
- **Dependencia de flujo OpenSpec**: el delta MODIFIED de `console-place-locations` asume ese spec sincronizado a `openspec/specs/` — archivar (con sync) `add-console-place-locations` antes de sincronizar/archivar este change.
- **Sin breaking**: el texto legado sigue visible y la lectura del agente ya tiene fallback (contrato del backend).
