# Delta: console-place-locations

## ADDED Requirements

### Requirement: La grid de sedes es la única superficie de edición

La grid de la sección SEDES SHALL ser la única vía de edición de ubicación y contacto por sede (dirección, zona, ciudad, lat/lng, teléfono, WhatsApp) en todo lugar, sin importar su cardinalidad — no existe modo bimodal ni secciones espejo editables. Las ediciones SHALL aplicarse por `id` de fila (renombrar un label es un update normal, nunca baja+alta). Al guardar, las columnas espejo de `places` (`address`, `lat`, `lng`, `phone`, `whatsapp`) SHALL derivarse en memoria de la sede marcada `is_primary` en la grid; el Console MUST NOT leerlas de otros inputs del formulario. Ciudad y región del lugar SHALL editarse en la sección de datos generales como identidad de la marca y MUST NOT duplicarse en superficies de ubicación.

#### Scenario: Edición mono-sede en la grid

- **WHEN** el staff corrige la dirección de la única sede de un lugar en la grid y guarda
- **THEN** `place_locations.address` de esa sede y `places.address` quedan con el valor nuevo

#### Scenario: Edición multi-sede sin secciones bloqueadas

- **WHEN** el staff abre un lugar con 3 sedes activas
- **THEN** no existe ninguna sección de ubicación/contacto bloqueada ni nota de reflejo — la grid es la única superficie, editable directamente

#### Scenario: Espejo derivado de la principal al guardar

- **WHEN** el staff cambia la dirección y el teléfono de la sede principal en la grid y guarda
- **THEN** `places.address` y `places.phone` quedan con los valores nuevos de esa sede

#### Scenario: Rename de label seguro

- **WHEN** el staff renombra la sede "Centro" a "Plaza de Bolívar" y guarda
- **THEN** la misma fila (`id` intacto) tiene el label nuevo y no aparece ninguna fila adicional

## MODIFIED Requirements

### Requirement: Sección de sedes en el detalle de lugar

`console/lugar.html` SHALL mostrar una sección SEDES con las sedes del lugar desde `place_locations` — label, zona, dirección, ciudad, lat/lng, teléfono, WhatsApp — señalando la principal y colapsando las inactivas. La sección SHALL cargar con el mismo cliente supabase-js del resto del formulario y degradar a un estado vacío honesto si la consulta falla (error visible, nunca datos inventados).

#### Scenario: Cadena con sedes visibles

- **WHEN** el staff abre el detalle de Frisby (6 sedes activas)
- **THEN** la sección lista las 6 sedes con sus datos reales y la principal (Cuba) marcada

#### Scenario: Lugar mono-sede

- **WHEN** el staff abre un lugar con solo su sede `Principal`
- **THEN** la sección muestra esa única sede editable en la grid, sin que exista otra superficie de edición de ubicación en el formulario

### Requirement: Alta de lugar crea la sede Principal

Crear un lugar desde el Console SHALL presentar en la grid una fila `Principal` pre-creada (`is_primary`, activa) lista para llenar, y el guardado SHALL insertarla en `place_locations` tras el insert de `places`, en el mismo flujo. Un lugar existente sin sedes activas (huérfano) SHALL recibir la misma fila pre-creada al abrirse. La validación SHALL exigir al menos una sede activa con exactamente una principal antes de guardar. Si el insert de la sede falla, el Console SHALL mostrar el error de forma visible e indicar reintentar el guardado — MUST NOT fallar en silencio.

#### Scenario: Lugar nuevo con sede

- **WHEN** el staff crea un lugar, llena dirección y teléfono en la fila `Principal` pre-creada de la grid y guarda
- **THEN** existe una fila en `place_locations` con `label='Principal'`, `is_primary=true` y esos mismos datos, y `places.address`/`phone` los reflejan

#### Scenario: Lugar huérfano se repara al abrir

- **WHEN** el staff abre un lugar sin ninguna sede activa
- **THEN** la grid muestra una fila `Principal` pre-creada para llenar y el guardado la persiste

## REMOVED Requirements

### Requirement: Guardado mono-sede sincroniza espejo y sede

**Reason**: Muere la regla bimodal — las secciones 03/04 ya no existen como superficie de edición; la grid es la única vía en toda cardinalidad.
**Migration**: Cubierto por "La grid de sedes es la única superficie de edición" (espejo derivado de la `is_primary` al guardar). La hidratación y preservación de zona (`zone_id`/texto legado) queda cubierta por los requirements por sede de `console-zone-selector`.

### Requirement: Multi-sede edita en la grid con espejo sincronizado

**Reason**: La distinción multi-sede pierde sentido: la grid es la única superficie siempre, no solo con 2+ sedes; ya no hay secciones en solo lectura que sincronizar.
**Migration**: Cubierto por "La grid de sedes es la única superficie de edición". La garantía de updates por `id` de fila (rename de label seguro) se conserva en ese requirement implícitamente y en "Desactivación suave con guardas", que no cambia.
