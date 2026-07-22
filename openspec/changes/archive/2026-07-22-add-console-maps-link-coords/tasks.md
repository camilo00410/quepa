# Tasks: add-console-maps-link-coords

## 1. Parser

- [x] 1.1 Función pura de extracción en `console/lugar.html`: patrones en orden `!3d…!4d…` → `@lat,lng` → `q=` → `ll=`; floats con 3+ decimales; retorna `{lat, lng}` o null (D1)
- [x] 1.2 Sanity check de rango Colombia (lat -5…14, lng -82…-66) que produce aviso sin bloquear (D4)

## 2. Sección 03 (mono-sede / espejo)

- [x] 2.1 Campo "URL de Google Maps" sobre los inputs de coordenadas: al pegar, extrae → escribe `f-lat`/`f-lng` → `renderMap()` → limpia el campo con nota "coordenadas capturadas ✓" (D2, D5)
- [x] 2.2 Colapsar `f-lat`/`f-lng` como detalle avanzado con resumen del valor actual, editables (D3); respetar el modo espejo multi-sede (URL y avanzado deshabilitados con la nota existente)
- [x] 2.3 Nota honesta para URL no extraíble (link corto / búsqueda / basura) sin tocar valores (D4)

## 3. Grid de sedes

- [x] 3.1 Campo URL por sede en la fila (compacto): extrae → `sede.lat/lng` → `syncReflejo()` si es principal → feedback en la celda (D5)
- [x] 3.2 Celdas Lat/Lng de la grid al mismo patrón colapsado/avanzado con resumen (D3)

## 4. Verificación y docs

- [x] 4.1 Verificado 2026-07-22 en producción por el usuario: captura de coordenadas del Parque Consotá pegando la URL de Maps (4.799063,-75.8059273 hoy sirviendo los links del bot); degradación con link corto probada en conversación
- [x] 4.2 Actualizar `quepa-landing/CLAUDE.md` (sección Console): captura de coordenadas por URL de Maps, floats como avanzado, la URL no se persiste
