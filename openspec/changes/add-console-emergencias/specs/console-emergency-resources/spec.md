# console-emergency-resources

## ADDED Requirements

### Requirement: Cola de verificación ordenada por vejez
La lista (`console/emergencias.html`) SHALL ordenar por defecto los recursos por `verified_at` ascendente (lo más vencido arriba) y SHALL mostrar por recurso un semáforo de frescura de tres estados — verde < 24 h, ámbar 24–48 h, rojo > 48 h — derivado en el cliente con constantes espejo de `resource-freshness.ts`, junto a la hora de última verificación en América/Bogotá.

#### Scenario: Lo vencido es lo primero que se ve
- **WHEN** existe un recurso verificado hace 3 días y otro verificado hoy
- **THEN** el vencido aparece arriba con semáforo rojo y el reciente abajo con semáforo verde

#### Scenario: Umbral de envejecimiento
- **WHEN** un recurso fue verificado hace 30 horas
- **THEN** su semáforo es ámbar y la etiqueta muestra cuándo fue la última verificación

### Requirement: Confirmación de un clic que solo sella verified_at
Cada fila SHALL ofrecer una acción "✓ Confirmar" que actualice únicamente `verified_at = now()` en `emergency_resources`, sin diálogo de confirmación y sin modificar ningún otro campo, refrescando la fila en sitio sin reordenar la lista bajo el cursor.

#### Scenario: Re-confirmación en ráfaga
- **WHEN** el analista pulsa "✓ Confirmar" en una fila vencida
- **THEN** `verified_at` queda en el momento actual, el semáforo pasa a verde en sitio, y ningún otro campo del recurso cambia (el historial del backend registra `changed_fields = {verified_at}` con el email del analista)

### Requirement: Filtros y búsqueda de la lista
La lista SHALL ofrecer filtros por tipo (los 5 del enum), estado (activos por defecto · inactivos · todos), ciudad y búsqueda por nombre, y SHALL mostrar las columnas: nombre, tipo, dirección/zona, semáforo de verificación, "Actualizado por" y acciones (confirmar · editar · desactivar/reactivar).

#### Scenario: Filtrar albergues activos
- **WHEN** el analista filtra por tipo `albergue`
- **THEN** la lista muestra solo albergues activos, ordenados por vejez de verificación

#### Scenario: Ver los dados de baja
- **WHEN** el analista cambia el filtro de estado a "inactivos"
- **THEN** aparecen los recursos con `active = false`, visualmente marcados como fuera de las recomendaciones del bot

### Requirement: Ficha de captura y edición sobre el schema existente
La ficha (`console/emergencia.html?id=<uuid>|new`) SHALL capturar los campos de `emergency_resources` sin inventar ninguno: tipo como chips de los 5 valores del enum en su orden editorial (`acopio · albergue · atencion · entidad · linea`), `hours_text` como texto libre sin aritmética de apertura, `receiving[]`/`not_receiving[]` y `contact_phones[]` como listas editables, y los opcionales `address`, `zone`, `maps_url`, `contact_url`, `how_to_access`, `source`, `notes`, `lat`/`lng`. Una violación del `unique (name, city, type)` SHALL producir mensaje inline claro, no el error crudo de PostgREST.

#### Scenario: Alta visible de inmediato para el agente
- **WHEN** el analista crea un acopio nuevo con nombre, ciudad del área, dirección y qué reciben
- **THEN** la fila queda en `emergency_resources` y es alcanzable por `search_help_points` en la consulta siguiente, sin ningún paso de embeddings

#### Scenario: Duplicado honesto
- **WHEN** el analista intenta crear un recurso con nombre, ciudad y tipo ya existentes
- **THEN** la ficha muestra un mensaje inline que nombra el conflicto y no guarda

### Requirement: Estado operativo como anotación en notes
La ficha SHALL ofrecer chips de anotación rápida ("capacidad completa", "temporalmente cerrado") que insertan una frase estándar al inicio de `notes`, editable como texto libre. El módulo MUST NOT introducir columna, enum ni taxonomía nueva de estado operativo: lo persistido es texto plano en `notes`, que el agente ya recibe vía `search_help_points`.

#### Scenario: Albergue lleno sin desaparecer
- **WHEN** el analista marca "capacidad completa" en un albergue y guarda
- **THEN** `notes` inicia con la frase estándar, el recurso sigue activo, y el bot puede mencionarlo con la advertencia en vez de omitirlo

### Requirement: Baja por desactivación, nunca borrado
El módulo SHALL dar de baja recursos únicamente con `active = false`, con confirmación que nombra el recurso; reactivar SHALL ser simétrico. El módulo MUST NOT ofrecer ningún control de borrado físico (a diferencia de `lugares.html`).

#### Scenario: Punto de acopio que cerró
- **WHEN** el analista desactiva un acopio y confirma el diálogo que lo nombra
- **THEN** la fila queda `active = false`, sale de las recomendaciones del bot en la consulta siguiente, y conserva su historial

#### Scenario: Sin camino al DELETE
- **WHEN** un analista recorre lista y ficha de un recurso
- **THEN** no encuentra ninguna acción que ejecute un borrado físico

### Requirement: Ciudad anclada a grafías existentes
La ficha SHALL capturar la ciudad con un selector poblado de la unión `places.city` ∪ `city_zones.city`, listando primero las ciudades del `area_cities` de la emergencia activa, con escape "otra ciudad…" de texto libre acompañado de aviso visible sobre el riesgo de invisibilidad por typo. El módulo MUST NOT escribir en `city_emergencies`, `places` ni `city_zones`.

#### Scenario: Recurso del área metropolitana
- **WHEN** el analista abre el selector de ciudad durante la emergencia de Pereira
- **THEN** las ciudades del área aparecen primero y elegir una garantiza la grafía que `area_cities` reconoce

#### Scenario: Ciudad fuera del catálogo
- **WHEN** el analista elige "otra ciudad…" y escribe a mano
- **THEN** el aviso de riesgo es visible antes de guardar

### Requirement: Trazabilidad leída de la vista de auditoría
La lista SHALL poblar "Actualizado por" (y la ficha "Verificado por") desde la vista `emergency_resource_activity` con una query batcheada sobre las filas visibles. Si la vista no responde, las columnas SHALL degradar a "—" con aviso discreto y el resto del módulo MUST seguir operable.

#### Scenario: Quién tocó qué
- **WHEN** ana@quepa.co editó un recurso y luego otro analista lo confirmó
- **THEN** la lista muestra a ana@quepa.co como último editor y la ficha muestra al otro analista como último verificador

#### Scenario: Migración de auditoría ausente
- **WHEN** la vista `emergency_resource_activity` no existe aún en la base
- **THEN** las columnas de trazabilidad muestran "—" con un aviso, y la cola de verificación, la confirmación y el CRUD siguen funcionando

### Requirement: Contador contra el cap editorial
La lista SHALL mostrar el conteo de recursos activos del área de la emergencia activa contra el tope de 60 (`RESOURCE_CAP`), con aviso al acercarse y, al desbordar, la consecuencia concreta (el agente amputa primero `entidad` y `linea`). El contador MUST ser informativo: nunca bloquea un alta. Sin emergencia activa, el contador se oculta.

#### Scenario: Área cerca del tope
- **WHEN** el área de la emergencia acumula 58 recursos activos
- **THEN** la lista muestra `58/60` con aviso de cercanía al tope

#### Scenario: Tope desbordado sin bloqueo
- **WHEN** el analista crea el recurso 61 del área
- **THEN** el alta se guarda y la lista muestra el desborde con la consecuencia editorial explícita

### Requirement: Navegación integrada
El ítem "Emergencias" SHALL aparecer en el sidebar de todas las páginas de `console/` que lo tienen, y `welcome.html` SHALL incluir una tarjeta del módulo.

#### Scenario: Acceso desde cualquier página
- **WHEN** el analista está en `lugares.html`, `eventos.html`, `mapa.html` o `descubrir.html`
- **THEN** el sidebar ofrece "Emergencias" y navega a `emergencias.html`
