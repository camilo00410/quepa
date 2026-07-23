# console-zone-selector Specification

## Requirements

### Requirement: Selector de zona canónica por ciudad

El punto de captura de zona en `console/lugar.html` SHALL ser la columna Zona de la grid de sedes — un selector por sede, único punto de captura (no existe selector de zona a nivel de lugar) — poblado con las zonas activas de `city_zones` para la ciudad efectiva de la sede (`location.city ?? ciudad del lugar`), con opción "Sin zona", cacheado por ciudad en la sesión de edición. Guardar SHALL escribir `place_locations.zone_id` — el Console MUST NOT escribir el campo `zone` texto ni insertar filas en `city_zones`.

#### Scenario: Asignar zona canónica

- **WHEN** el staff selecciona "Cuba" en la zona de una sede de Pereira y guarda
- **THEN** la sede queda con `zone_id` de la fila Cuba de `city_zones` y su campo `zone` texto no cambia

#### Scenario: Quitar la zona

- **WHEN** el staff selecciona "Sin zona" y guarda
- **THEN** la sede queda con `zone_id=null` y el texto legado (si existía) vuelve a ser lo que se muestra

### Requirement: Estado legado visible y transición explícita

Una sede con `zone_id` null y texto legado en `zone` SHALL mostrar ese texto en el selector como opción actual marcada como no-canónica ("texto legado"). Elegir una zona del gazetteer SHALL asignar `zone_id` dejando el texto legado intacto en la base. El Console MUST NOT migrar texto legado a `zone_id` de forma automática o silenciosa — la transición la decide el staff sede por sede.

#### Scenario: Sede solo-legado

- **WHEN** el staff abre una sede con `zone='cuba'` y `zone_id` null
- **THEN** el selector muestra «"cuba" · texto legado» como estado actual, y al elegir "Cuba" del gazetteer y guardar la sede queda con `zone_id` asignado y `zone='cuba'` intacto

### Requirement: Degradación honesta sin gazetteer

Si la consulta de zonas falla o la ciudad no tiene zonas en `city_zones`, los selectores SHALL deshabilitarse con una nota que explique que las zonas se curan en el backend. La sede SHALL conservar su `zone_id`/legado tal cual y el guardado del resto del formulario MUST NOT bloquearse. El Console MUST NOT reabrir la captura de zona como texto libre.

#### Scenario: Ciudad sin zonas

- **WHEN** el staff edita un lugar de una ciudad sin filas en `city_zones`
- **THEN** el selector aparece deshabilitado con la nota, la zona existente no se altera y el lugar se puede guardar normalmente
