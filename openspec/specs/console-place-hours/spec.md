# console-place-hours

Sección de horarios en el detalle de lugar del Quepa Console (`console/lugar.html`): visualización de la grilla semanal con la semántica del modelo de `quepa-webhook`, edición manual con procedencia `manual`, y guardado no destructivo.

## Requirements

### Requirement: Visualización del horario semanal con la semántica del modelo

La sección de horarios SHALL renderizar el JSONB `places.hours` como grilla semanal lunes–domingo (claves `mon..sun`), respetando la semántica del modelo: `hours = null` es horario DESCONOCIDO y nunca se presenta como cerrado; `[]` o clave ausente es cerrado ese día; un intervalo con cierre menor que apertura cruza medianoche y se señala visualmente sin tratarse como error; `[["00:00","24:00"]]` se presenta como abierto 24 horas.

#### Scenario: Lugar con horario conocido

- **WHEN** se carga un lugar cuyo `hours` tiene intervalos (p. ej. `{"mon": [["12:00","15:00"],["18:00","22:30"]], "wed": []}`)
- **THEN** el lunes muestra las dos franjas `12:00–15:00` y `18:00–22:30`, el miércoles muestra "Cerrado", y los días con clave ausente muestran "Cerrado"

#### Scenario: Lugar sin horario registrado

- **WHEN** se carga un lugar con `hours` null (o la columna no existe aún en la base)
- **THEN** la sección muestra el estado "Sin horario registrado" con la acción de registrar horario, y en ningún elemento de la UI se presenta el lugar como "cerrado"

#### Scenario: Intervalo que cruza medianoche

- **WHEN** un día tiene el intervalo `["18:00","02:00"]`
- **THEN** la franja se muestra con un indicador de cruce de medianoche y no se marca como inválida

#### Scenario: Abierto 24 horas

- **WHEN** un día tiene el intervalo `["00:00","24:00"]`
- **THEN** ese día se presenta como abierto 24 horas

### Requirement: Procedencia y verificación visibles en solo lectura

La sección SHALL mostrar `hours_source` (chip `GOOGLE`/`OSM`/`MANUAL`, o `—` si es null) y `hours_verified_at` en formato relativo, sin ofrecer ningún control para editarlos directamente. La única transición de procedencia posible desde el console SHALL ser a `manual`, ejecutada por el guardado de una edición de la grilla.

#### Scenario: Horario proveniente del backfill de Google

- **WHEN** se carga un lugar con `hours_source = 'google'` y `hours_verified_at` de hace 3 días
- **THEN** la cabecera de la sección muestra el chip `GOOGLE` y "hace 3 d", y no existe control para cambiar la fuente a mano

#### Scenario: Lugar sin procedencia

- **WHEN** se carga un lugar con `hours_source` null
- **THEN** la cabecera muestra `—` sin fecha de verificación

### Requirement: Edición de la grilla semanal

La sección SHALL permitir agregar múltiples franjas por día, editar sus horas de apertura y cierre como texto `HH:MM`, y quitar franjas individuales. Un día sin franjas SHALL presentarse como cerrado dentro de un horario con contenido.

#### Scenario: Agregar una franja

- **WHEN** el staff usa "+ franja" en un día y escribe apertura `08:00` y cierre `17:00`
- **THEN** la franja queda en la grilla y el día deja de mostrarse como cerrado

#### Scenario: Quitar una franja

- **WHEN** el staff quita la única franja de un día
- **THEN** el día vuelve a mostrarse como "Cerrado"

### Requirement: Validación bloqueante al guardar

Al guardar con la grilla tocada, el sistema SHALL validar cada franja: horas con formato `HH:MM` válido (`24:00` admitido solo como cierre), apertura distinta de cierre, y sin solapamientos entre franjas del mismo día (el cruce de medianoche es válido). Si la validación falla, el guardado SHALL abortar sin escribir nada y mostrar un error que nombre el día y la franja inválida. El sistema SHALL NOT auto-corregir valores.

#### Scenario: Hora con formato inválido

- **WHEN** una franja tiene apertura `25:00` o texto no `HH:MM` y el staff guarda
- **THEN** no se escribe nada en la base y aparece un error visible indicando el día y la franja con el problema

#### Scenario: Franjas solapadas en el mismo día

- **WHEN** un día tiene `["12:00","16:00"]` y `["15:00","20:00"]` y el staff guarda
- **THEN** el guardado aborta con un error que señala el solapamiento en ese día

#### Scenario: Cruce de medianoche no es error

- **WHEN** un día tiene únicamente `["18:00","02:00"]` y el staff guarda
- **THEN** la validación lo acepta y el guardado procede

### Requirement: Guardado no destructivo con marcado de procedencia

El payload de guardado SHALL incluir columnas de horarios solo según lo tocado: con la grilla tocada incluye `hours` serializado (las 7 claves `mon..sun` en orden fijo, franjas ordenadas por apertura), `hours_note`, `hours_source = 'manual'` y `hours_verified_at` con el instante del guardado; con solo la nota tocada incluye únicamente `hours_note`; sin nada tocado no incluye ninguna columna de horarios. Una grilla que queda sin franjas en todos los días SHALL serializarse como `hours = null` con `hours_source = null` y `hours_verified_at = null` (horario desconocido, nunca "cerrado los 7 días").

#### Scenario: Guardado sin tocar la sección

- **WHEN** el staff edita la descripción del lugar y guarda sin tocar horarios ni nota
- **THEN** el payload del update no contiene `hours`, `hours_note`, `hours_source` ni `hours_verified_at`, y los horarios existentes en la base quedan intactos

#### Scenario: Edición manual de la grilla

- **WHEN** el staff corrige una franja de un horario que venía con `hours_source = 'google'` y guarda
- **THEN** el update escribe el `hours` completo serializado, `hours_source = 'manual'` y `hours_verified_at` del momento, y la cabecera pasa a mostrar `MANUAL · hace un momento`

#### Scenario: Solo la nota tocada conserva la procedencia

- **WHEN** el staff corrige un typo en la nota de un horario con `hours_source = 'google'` y guarda sin tocar la grilla
- **THEN** el update incluye únicamente `hours_note`; `hours`, `hours_source` y `hours_verified_at` no van en el payload

#### Scenario: Vaciar la grilla limpia el horario

- **WHEN** el staff quita todas las franjas de todos los días y guarda
- **THEN** el update escribe `hours = null`, `hours_source = null` y `hours_verified_at = null`, y la sección muestra "Sin horario registrado"

### Requirement: Horarios en la creación de un lugar nuevo

En el modo de lugar nuevo, la sección SHALL iniciar en estado "Sin horario registrado" y permitir registrar horarios antes de crear; el insert SHALL incluir las columnas de horarios solo si la grilla o la nota fueron tocadas, con las mismas reglas de serialización y procedencia del guardado.

#### Scenario: Crear lugar con horario

- **WHEN** el staff crea un lugar nuevo y registra franjas antes de guardar
- **THEN** el insert incluye `hours` serializado, `hours_source = 'manual'` y `hours_verified_at` del momento

#### Scenario: Crear lugar sin horario

- **WHEN** el staff crea un lugar nuevo sin tocar la sección de horarios
- **THEN** el insert no incluye ninguna columna de horarios

### Requirement: Errores de guardado visibles

Si el update o insert con columnas de horarios falla (p. ej. la migración `0007` no está aplicada y las columnas no existen), el sistema SHALL mostrar el error en el toast existente y SHALL NOT reportar éxito ni actualizar la cabecera de procedencia.

#### Scenario: Guardar horarios sin la migración aplicada

- **WHEN** el staff guarda una edición de horarios contra una base sin las columnas de `0007`
- **THEN** aparece el toast de error con el mensaje de PostgREST, no hay toast de éxito y la cabecera de la sección no cambia
