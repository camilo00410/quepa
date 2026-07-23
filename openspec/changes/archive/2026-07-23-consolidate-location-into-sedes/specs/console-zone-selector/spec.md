# Delta: console-zone-selector

## MODIFIED Requirements

### Requirement: Selector de zona canónica por ciudad

El punto de captura de zona en `console/lugar.html` SHALL ser la columna Zona de la grid de sedes — un selector por sede, único punto de captura (no existe selector de zona a nivel de lugar) — poblado con las zonas activas de `city_zones` para la ciudad efectiva de la sede (`location.city ?? ciudad del lugar`), con opción "Sin zona", cacheado por ciudad en la sesión de edición. Guardar SHALL escribir `place_locations.zone_id` — el Console MUST NOT escribir el campo `zone` texto ni insertar filas en `city_zones`.

#### Scenario: Asignar zona canónica

- **WHEN** el staff selecciona "Cuba" en la zona de una sede de Pereira y guarda
- **THEN** la sede queda con `zone_id` de la fila Cuba de `city_zones` y su campo `zone` texto no cambia

#### Scenario: Quitar la zona

- **WHEN** el staff selecciona "Sin zona" y guarda
- **THEN** la sede queda con `zone_id=null` y el texto legado (si existía) vuelve a ser lo que se muestra

## REMOVED Requirements

### Requirement: Centroide como referencia visual, nunca persistido

**Reason**: El mapa de preview del detalle de lugar se elimina por decisión de producto — no queda superficie donde mostrar el centroide como referencia aproximada.
**Migration**: El invariante duro se conserva y queda trivialmente garantizado: el Console ya no lee centroides de `city_zones` y MUST NOT escribirlos jamás en `lat`/`lng` de sedes ni lugares (regla del backend `add-city-zones`, que no cambia). La captura de coordenadas exactas sigue por URL de Maps o floats del detalle avanzado (`console-maps-link-coords`).
