# console-place-locations — Delta Spec (add-console-zone-selector)

## MODIFIED Requirements

### Requirement: Guardado mono-sede sincroniza espejo y sede

En un lugar con una sola sede activa, guardar cambios de las secciones 03/04 SHALL escribir las columnas planas de `places` **y** la fila de la sede principal con los mismos valores (dirección, lat/lng, teléfono, WhatsApp) más la zona como `zone_id` (selector de zona canónica; el campo `zone` texto es legado y no recibe escrituras). Al cargar el lugar, el campo de zona SHALL hidratarse desde la sede principal resolviendo nombre canónico si hay `zone_id` y texto legado si no (la zona no es columna de `places`); guardar sin tocar el campo MUST NOT alterar ni el `zone_id` ni el texto legado persistidos. El flujo de edición del staff MUST NOT cambiar respecto al actual.

#### Scenario: Edición de dirección mono-sede

- **WHEN** el staff corrige la dirección de un lugar mono-sede y guarda
- **THEN** `places.address` y `place_locations.address` de la sede principal quedan con el valor nuevo

#### Scenario: Zona canónica en mono-sede

- **WHEN** el staff selecciona "Cuba" en el selector de zona de la sección 03 y guarda
- **THEN** la sede principal queda con `zone_id` de Cuba en `city_zones` y su campo `zone` texto no cambia

#### Scenario: Zona precargada al abrir

- **WHEN** el staff abre un lugar mono-sede cuya sede principal tiene zona (por `zone_id` o por texto legado) y guarda sin tocar la zona
- **THEN** el selector muestra al cargar el nombre canónico (o el legado marcado como tal) y tras guardar la sede conserva su `zone_id` y su texto legado exactamente como estaban
