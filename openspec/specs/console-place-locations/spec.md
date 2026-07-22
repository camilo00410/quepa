# console-place-locations Specification

## Requirements

### Requirement: Sección de sedes en el detalle de lugar

`console/lugar.html` SHALL mostrar una sección `· 06 · SEDES` con las sedes del lugar desde `place_locations` — label, zona, dirección, ciudad, lat/lng, teléfono, WhatsApp — señalando la principal y colapsando las inactivas. La sección SHALL cargar con el mismo cliente supabase-js del resto del formulario y degradar a un estado vacío honesto si la consulta falla (error visible, nunca datos inventados).

#### Scenario: Cadena con sedes visibles

- **WHEN** el staff abre el detalle de Frisby (6 sedes activas)
- **THEN** la sección lista las 6 sedes con sus datos reales y la principal (Cuba) marcada

#### Scenario: Lugar mono-sede

- **WHEN** el staff abre un lugar con solo su sede `Principal`
- **THEN** la sección muestra esa única sede sin ruido y las secciones 03/04 permanecen editables como siempre

### Requirement: Alta de lugar crea la sede Principal

Crear un lugar desde el Console SHALL insertar también su sede `Principal` (`is_primary=true`, `active=true`) con los valores de dirección, lat/lng, teléfono, WhatsApp y zona de las secciones 03/04, tras el insert de `places` y en el mismo flujo de guardado. Si el insert de la sede falla, el Console SHALL mostrar el error de forma visible e indicar reintentar el guardado — MUST NOT fallar en silencio.

#### Scenario: Lugar nuevo con sede

- **WHEN** el staff crea un lugar con dirección y teléfono y guarda
- **THEN** existe una fila en `place_locations` con `label='Principal'`, `is_primary=true` y esos mismos datos

### Requirement: Guardado mono-sede sincroniza espejo y sede

En un lugar con una sola sede activa, guardar cambios de las secciones 03/04 SHALL escribir las columnas planas de `places` **y** la fila de la sede principal con los mismos valores (dirección, lat/lng, teléfono, WhatsApp) más la zona (`f-barrio`, habilitado). Al cargar el lugar, el campo de zona SHALL hidratarse desde la `zone` de la sede principal (la zona no es columna de `places`); guardar sin tocar el campo MUST NOT vaciar la `zone` persistida. El flujo de edición del staff MUST NOT cambiar respecto al actual.

#### Scenario: Edición de dirección mono-sede

- **WHEN** el staff corrige la dirección de un lugar mono-sede y guarda
- **THEN** `places.address` y `place_locations.address` de la sede principal quedan con el valor nuevo

#### Scenario: Zona habilitada

- **WHEN** el staff escribe "Cuba" en el campo de barrio/zona y guarda
- **THEN** la sede principal queda con `zone='Cuba'` (el campo deja de estar deshabilitado)

#### Scenario: Zona precargada al abrir

- **WHEN** el staff abre un lugar mono-sede cuya sede principal tiene `zone='Cuba'` y guarda sin tocar la zona
- **THEN** el campo Zona/Barrio muestra "Cuba" al cargar y la sede conserva `zone='Cuba'` tras guardar

### Requirement: Multi-sede edita en la grid con espejo sincronizado

En un lugar con 2+ sedes activas, las secciones 03/04 SHALL mostrarse en solo lectura reflejando la sede principal, con una nota que dirige a la sección SEDES; la grid SHALL ser la única vía de edición de datos de sede (updates por `id` de fila — renombrar un label es un update normal, nunca baja+alta). Al guardar, las columnas espejo de `places` SHALL tomarse de la sede marcada como principal.

#### Scenario: Secciones bloqueadas en cadena

- **WHEN** el staff abre un lugar con 3 sedes activas
- **THEN** los campos de 03/04 están deshabilitados con la nota, y editar la dirección de una sede en la grid y guardar actualiza esa fila de `place_locations`

#### Scenario: Editar la sede principal sincroniza el espejo

- **WHEN** el staff cambia la dirección de la sede principal en la grid y guarda
- **THEN** `places.address` queda con la dirección nueva

#### Scenario: Rename de label seguro

- **WHEN** el staff renombra la sede "Centro" a "Plaza de Bolívar" y guarda
- **THEN** la misma fila (`id` intacto) tiene el label nuevo y no aparece ninguna fila adicional

### Requirement: Cambio de principal con democión previa

Marcar otra sede como principal SHALL ejecutar primero la democión de la actual (`is_primary=false`) y después la coronación de la nueva (`is_primary=true`), respetando el índice único parcial; después SHALL sincronizar las columnas espejo de `places` con la nueva principal. Si la secuencia falla a medias, el error SHALL ser visible y el guardado reintentable — MUST NOT dejar el error en silencio.

#### Scenario: Coronar otra sede

- **WHEN** el staff marca la sede "Unicentro" como principal (antes lo era "Cuba") y guarda
- **THEN** solo "Unicentro" queda `is_primary=true` y `places.address` pasa a la dirección de Unicentro

### Requirement: Desactivación suave con guardas

Las sedes SHALL desactivarse (`active=false` + `is_primary=false`), nunca eliminarse; la UI SHALL impedir desactivar la sede principal (exige coronar otra primero) y SHALL permitir reactivar una sede inactiva. La validación de la grid SHALL exigir label obligatorio y único por lugar (case/espacios-insensible) y exactamente una principal activa antes de guardar.

#### Scenario: Desactivar una sede secundaria

- **WHEN** el staff desactiva la sede "Uniplaza" y guarda
- **THEN** la fila queda `active=false`, sigue existiendo en la base y aparece bajo "sedes inactivas" con opción de reactivar

#### Scenario: La principal no se puede desactivar

- **WHEN** el staff intenta desactivar la sede principal
- **THEN** la UI lo bloquea con un mensaje indicando coronar otra sede primero

#### Scenario: Label duplicado bloqueado

- **WHEN** el staff agrega una sede con label "cuba" cuando ya existe "Cuba"
- **THEN** la validación bloquea el guardado con error visible

### Requirement: Conteo de sedes en el listado

`console/lugares.html` SHALL mostrar un chip "+N sedes" en los lugares con más de una sede activa, resuelto con una única consulta batcheada sobre los lugares visibles — MUST NOT agregar joins ni cambiar la forma de la consulta principal del listado.

#### Scenario: Chip en cadena

- **WHEN** el listado muestra a Frisby (6 sedes activas)
- **THEN** su fila incluye un chip "+5 sedes" (o equivalente) y los lugares mono-sede no muestran chip
