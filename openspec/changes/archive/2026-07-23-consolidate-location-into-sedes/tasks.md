# Tasks: consolidate-location-into-sedes

## 1. Reestructura del formulario (lugar.html)

- [x] 1.1 Mover ciudad y región a la sección de datos generales (identidad de marca, siguen obligatorios) y verificar que la ciudad efectiva de los selectores de zona (`location.city ?? ciudad del lugar`) siga resolviendo
- [x] 1.2 Eliminar la sección `· 03 · UBICACIÓN` completa: campos espejo (`f-address`, `f-lat`, `f-lng`, `f-mapsurl`, `f-barrio`), mapa de preview (`renderMap` y su markup/estilos) y el selector espejo de zona
- [x] 1.3 Quitar teléfono y WhatsApp de la sección 04 (quedan website, booking_url y redes de marca); retirar `whatsappPill` o recablearlo a la sede principal si se conserva
- [x] 1.4 Eliminar la maquinaria bimodal: `syncReflejo`, `applyCardinalityMode`, `MIRROR_FIELD_IDS`, notas de bloqueo (`ubicacionLockNote`/`contactoLockNote`) y la rama mono-sede de `saveSedes`

## 2. Espejo y guardado

- [x] 2.1 Rework de `gather()`: `address/lat/lng/phone/whatsapp` del payload de `places` se derivan de `primarySede()` en el estado de la grid (null-safe cuando la fila está vacía)
- [x] 2.2 Alta y huérfanos: pre-crear la fila `Principal` (⭐, activa) en la grid al abrir un lugar nuevo o sin sedes activas; eliminar la síntesis D4 en `saveSedes`
- [x] 2.3 Endurecer `validateSedes`: exigir al menos una sede activa con exactamente una principal (ya no se permite cero sedes)
- [x] 2.4 Verificar el orden de guardado intacto (`places` → `saveSedes` → `saveSocialLinks`) y que el reintento tras fallo parcial siga reparando

## 3. Link de Maps por sede

- [x] 3.1 Agregar el campo "Link de Maps" por fila de sede activa, rotulado como el link corto que ve el cliente, sin extracción de coordenadas
- [x] 3.2 Persistencia en `saveSocialLinks` (o helper nuevo): insert con `place_location_id` de la sede, update por `id` al editar, `active=false` al vaciar; sedes nuevas obtienen su `id` de `saveSedes` antes
- [x] 3.3 Quitar `google_maps` del dropdown de redes de marca y de la edición en la lista de redes
- [x] 3.4 Adopción de filas existentes sin sede: mono-sede → precargar URL en la `Principal` con nota "se asignará al guardar" y update de la misma fila; multi-sede → bloque "link sin sede asignada" con selector de sede; sin asignar queda visible y dormida
- [x] 3.5 Actualizar la nota del link corto en el campo URL-para-coordenadas para que remita al campo "Link de Maps" de la sede

## 4. Verificación manual (desktop y mobile)

- [x] 4.1 Alta de lugar: fila `Principal` pre-creada, guardado crea `places` + sede + espejo coherentes
- [x] 4.2 Mono-sede: editar dirección/zona/coords/teléfono en la grid actualiza `place_locations` y el espejo de `places`; zona canónica y texto legado se preservan al guardar sin tocar
- [x] 4.3 Multi-sede: edición directa en la grid, coronación con democión previa (espejo pasa a la nueva principal), desactivación/reactivación con guardas, rename de label por `id`
- [x] 4.4 Maps-link: captura por sede, edición, vaciado (desactiva), adopción mono-sede y multi-sede, fila sin asignar persiste visible
- [x] 4.5 Regresión de lo que NO debía cambiar: horarios, redes de marca (sin google_maps), chip "+N sedes" en lugares.html, aviso de coordenadas fuera de Colombia

## 5. Cierre

- [x] 5.1 Actualizar `quepa-landing/CLAUDE.md` (secciones de sedes y redes: regla única de superficie, maps-link por sede, adopción)
- [x] 5.2 Confirmar que el change hermano de `quepa-webhook` (enrutado de `place_location_id` en `get_place_details`) está desplegado ANTES de publicar el Console
