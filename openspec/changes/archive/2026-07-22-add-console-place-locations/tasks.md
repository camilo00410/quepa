# Tasks: add-console-place-locations

## 1. Sección SEDES (lectura)

- [x] 1.1 `console/lugar.html`: HTML+CSS de la sección `· 06 · SEDES` (grid: label, zona, dirección, ciudad, lat/lng, teléfono, WhatsApp, ⭐ principal, activa; inactivas colapsadas), siguiendo el patrón visual de la sección de redes sociales
- [x] 1.2 Carga de sedes por `place_id` (orden: principal primero, luego label) en el estado de la página; estado vacío honesto y error visible si la consulta falla
- [x] 1.3 Detección de cardinalidad (`monoSede` = 1 sede activa) como base del modelo de edición (D1)

## 2. Grieta 1: alta de lugar crea la sede Principal

- [x] 2.1 En `save()` con `state.isNew`: tras el insert de `places`, insertar la sede `Principal` (`is_primary=true`) con los valores de 03/04 + zona; error visible con reintento si falla (D4)
- [x] 2.2 Habilitar `f-barrio` (quitar `disabled`, label "Zona/Barrio") y conectarlo al estado de la sede principal

## 3. Guardado mono-sede (espejo mantenido por 03/04)

- [x] 3.1 En `save()` de lugar existente mono-sede: además del update de `places`, update por `id` de la sede principal con dirección/lat/lng/teléfono/WhatsApp/zona (D1)
- [x] 3.2 Caso borde: lugar existente SIN sede (creado por Console entre la 0008 y este deploy) — el guardado crea la `Principal` en vez de actualizar; verificación manual con el query `places` sin fila en `place_locations` (Open Question del design)
- [x] 3.3 Hidratar `f-barrio` desde la `zone` de la sede principal en `loadPlace()` — bug reportado 2026-07-22: el campo cargaba vacío tras recargar (la zona sí se guardaba) y el siguiente guardado mono-sede pisaba `zone` a null con el campo vacío

## 4. Multi-sede (la grid es la verdad)

- [x] 4.1 Modo solo-lectura de 03/04 cuando hay 2+ sedes activas, con nota "edita en · 06 · SEDES"; los valores mostrados reflejan la sede principal
- [x] 4.2 Edición de campos de sede en la grid con update por `id` (rename de label incluido); sedes nuevas con `insert` (conflicto de label = error de validación visible, D2)
- [x] 4.3 Validaciones de grid (D6): label obligatorio y único case/espacios-insensible, exactamente una principal activa, lat/lng numéricos; bloqueo del guardado con toasts
- [x] 4.4 Cambio de principal: democión de la actual → coronación de la nueva → espejo de `places` desde la nueva principal (D3); errores a medias visibles y reintentables
- [x] 4.5 Desactivar/reactivar sede (`active` + `is_primary=false` al desactivar); la UI impide desactivar la principal (D5)
- [x] 4.6 Sincronización del espejo al guardar multi-sede: columnas planas de `places` tomadas de la fila principal de la grid

## 5. Listado

- [x] 5.1 `console/lugares.html`: chip "+N sedes" con una query batcheada `.in('place_id', ids)` sobre los lugares visibles, solo cuando N>1 (D7)

## 6. Verificación y docs

- [x] 6.1 Prueba manual en browser desktop y mobile (convención del repo): mono-sede (flujo intacto), Frisby (grid 6 sedes, 03/04 bloqueadas, coronar otra principal y revertir), crear lugar de prueba (verificar fila `Principal` en Supabase) y desactivar sede secundaria — verificado 2026-07-22 en producción, incluido el ciclo zona→guardar→recargar del fix 3.3
- [x] 6.2 Verificar en Supabase si existen lugares creados por Console sin sede (ventana post-0008) y crearles su `Principal` desde el propio Console (tarea 3.2) — verificado 2026-07-21 (solo lectura): 1853 places · 1853 sedes · 1853 primarias, cero huérfanos; el camino de reparación queda para altas futuras pre-deploy
- [x] 6.3 Documentar la sección y el contrato espejo↔sede en `quepa-landing/CLAUDE.md` (sección del Console; crearla si no existe — hoy el archivo solo documenta las landings)
