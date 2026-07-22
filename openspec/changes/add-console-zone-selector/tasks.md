# Tasks: add-console-zone-selector

## 0. Pre-requisito cross-repo

- [ ] 0.1 Verificar que `add-city-zones` (quepa-webhook) está desplegado: migración `0010` aplicada, gazetteer Pereira/AMCO poblado, backfill corrido — sin esto no arrancar

## 1. Carga de zonas

- [ ] 1.1 `console/lugar.html`: query de `city_zones` activas por ciudad (name, lat, lng) con el cliente supabase-js existente, cache `state.zonesByCity`, carga para la ciudad del lugar + ciudades propias de sedes (caso AMCO)
- [ ] 1.2 Manejo de error/vacío (D5): flag por ciudad para degradar los selectores con nota, sin bloquear el guardado

## 2. Selectores

- [ ] 2.1 `f-barrio` (sección 03) pasa de input a `<select>`: "— Sin zona —" + zonas ordenadas + opción no-canónica «"…" · texto legado» cuando aplique (D2); respeta el modo espejo/deshabilitado multi-sede existente
- [ ] 2.2 Columna Zona de la grid de sedes → select por sede con su ciudad efectiva, mismo tratamiento del legado
- [ ] 2.3 Repoblar selects si cambia `f-city`, con caída explícita a "Sin zona" + aviso cuando la selección ya no aplica (D1)

## 3. Guardado e hidratación

- [ ] 3.1 Helper único de resolución (canónico si `zone_id`, legado si no) usado por hidratación de `loadPlace()`, `syncReflejo()` y render de la grid (D3)
- [ ] 3.2 `saveSedes()`: las tres ramas (alta de Principal, mono-sede, upsert de grid) escriben `zone_id` y dejan de escribir `zone` texto
- [ ] 3.3 Guardar sin tocar la zona conserva `zone_id` y legado intactos (regresión del bug del 2026-07-22, ahora sobre la FK)

## 4. Mapa

- [ ] 4.1 `renderMap()`: sede sin lat/lng con zona seleccionada → centrar en el centroide con marcador/copy de "zona aproximada" (D4); `f-lat`/`f-lng` jamás se rellenan con el centroide

## 5. Verificación y docs

- [ ] 5.1 Prueba manual desktop y mobile (convención del repo): mono-sede canónico, sede solo-legado (transición a FK verificando en Supabase texto intacto), multi-sede (grid + espejo), ciudad sin gazetteer (degradación), mapa con centroide
- [ ] 5.2 Actualizar `quepa-landing/CLAUDE.md` (sección Console): contrato de zona canónica — el Console escribe `zone_id`, nunca `zone` texto, nunca inserta en `city_zones`; centroide display-only
