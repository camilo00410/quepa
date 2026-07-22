# Proposal: add-console-maps-link-coords

## Why

Digitar latitud/longitud como floats a mano es un UX que nadie va a ejecutar: la curación de `pereira.json` capturó **0/544** coordenadas, y el 48% que sí existe en la base (894/1855 sedes) vino del pipeline de scraping, no del staff. Con el seeder fuera del flujo de altas, el Console es la única puerta de entrada del catálogo — si capturar coordenadas sigue siendo digitar floats, la cobertura solo puede decaer y la etapa 2b de proximidad (haversine con el pin de WhatsApp) nace muerta. La observación clave (exploración 2026-07-22): la URL de compartir de Google Maps — que el staff ya usa para verificar direcciones — **trae las coordenadas adentro** (`…/@4.799063,-75.805927,17z`).

## What Changes

- **Nuevo campo "URL de Google Maps"** como vía primaria de captura de coordenadas en `console/lugar.html`: al pegar una URL completa de Maps, el Console extrae `lat`/`lng` con regex (patrones `@lat,lng`, `!3d…!4d…`, `q=lat,lng`, `ll=lat,lng`) y los escribe en el estado existente (`f-lat`/`f-lng` en la sección 03; `sede.lat/lng` en la grid de sedes). Sin llamadas externas, sin dependencias — solo parsing local (convención del repo).
- **Los inputs de floats se degradan a "detalle avanzado"**: colapsados por defecto, siguen siendo editables como válvula de escape (coordenadas que llegan por otra vía, o limpiar el valor). El flujo visible queda: pegar URL → ver el pin moverse en el mapa.
- **Degradación honesta**: URL sin coordenadas extraíbles (links cortos `maps.app.goo.gl`, URLs de búsqueda) → nota clara pidiendo la URL completa del navegador, sin tocar los valores existentes. Sanity check de rango (Colombia aprox.) con aviso si el resultado cae fuera.
- **La URL no se persiste**: es mecanismo de captura, no dato. Guardar la URL de Maps como red social (`google_maps` en `place_social_links`) ya existe y es otra cosa.

Fuera de alcance: resolver links cortos (requiere fetch externo — bloqueado por la convención de autocontención), geocoding de direcciones, cambios de schema o backend (los campos `lat`/`lng` y su flujo de guardado quedan intactos), y autollenado de coordenadas vía census/Google (candidato a change futuro del webhook).

## Capabilities

### New Capabilities

- `console-maps-link-coords`: captura de coordenadas de sede pegando la URL de Google Maps — extracción local por regex, floats como detalle avanzado colapsado, validación de rango y degradación honesta con links no extraíbles.

### Modified Capabilities

<!-- Ninguna: la grid de sedes sigue mostrando/guardando lat/lng igual (console-place-locations intacta); cambia solo el mecanismo de captura, que se especifica completo en la capability nueva. -->

## Impact

- **Código**: solo `console/lugar.html` (campo URL + parser regex + colapso de floats en sección 03 y grid de sedes). Autocontenido, cero dependencias.
- **Datos**: ninguno — se escribe en los mismos `lat`/`lng` de siempre por los flujos de guardado existentes (espejo mono-sede y grid multi-sede, contrato de `console-place-locations`).
- **Sinergia**: el pin `≈` de zona (change `add-console-zone-selector`) sigue siendo el fallback cuando no hay coordenadas; este change alimenta el caso exacto.
- **Sin breaking**: sin migraciones, sin cambios cross-repo, sin tocar el webhook.
