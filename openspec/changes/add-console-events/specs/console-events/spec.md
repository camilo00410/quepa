# console-events

## ADDED Requirements

### Requirement: Lista de eventos agrupada en series

`console/eventos.html` SHALL listar los eventos agrupados por la clave de serie `(name, city, start_time)`, calculada en el cliente. Una clave con dos o más fechas SHALL presentarse como un renglón de serie colapsado, expandible a sus fechas; una clave con una sola fecha SHALL presentarse como renglón de evento puntual. El renglón de serie SHALL mostrar el conteo de fechas y el rango de fechas que abarca. El Console MUST NOT persistir el agrupamiento: no hay columna de serie y no se escribe ningún identificador de serie en la base.

#### Scenario: Serie recurrente colapsada
- **WHEN** la base tiene 41 eventos "La Ruta de la Cosecha" en Pereira a las 12:00, en 41 fechas distintas
- **THEN** la lista muestra UN renglón de serie con conteo 41 y el rango "6 ago → 15 sep", expandible a las 41 fechas

#### Scenario: Evento puntual
- **WHEN** un evento tiene una clave de serie con una sola fecha
- **THEN** la lista lo muestra como renglón simple, sin control de expansión

#### Scenario: Misma actividad a dos horas distintas son dos series
- **WHEN** existen eventos "Unicentro Pereira: El Portón de las Fiestas" en Pereira, unos a las 13:00 y otros a las 16:00
- **THEN** la lista muestra DOS renglones de serie separados, uno por hora de inicio

#### Scenario: Sin escritura de agrupamiento
- **WHEN** se carga o se guarda desde cualquier vista de la lista
- **THEN** ninguna petición a Supabase incluye un campo de serie, y el esquema de `events` no se modifica

### Requirement: La fecha y la hora son identidad visible

Todo renglón de la lista y toda ficha SHALL mostrar la fecha del evento y su hora de inicio junto al nombre. El Console MUST NOT presentar el nombre como identificador único en ninguna vista, y la búsqueda por texto SHALL mostrar fecha y hora en cada resultado. Toda operación de lectura o escritura sobre un evento SHALL resolverse por `id`, nunca por nombre.

#### Scenario: Búsqueda sobre un nombre repetido
- **WHEN** el staff busca "La Ruta de la Cosecha" y hay 41 filas con ese nombre
- **THEN** cada resultado muestra su fecha y hora de inicio, de modo que se distinguen entre sí

#### Scenario: Edición resuelta por id
- **WHEN** el staff guarda cambios en un evento
- **THEN** el update de Supabase filtra por `id`, y ninguna operación usa `name` como criterio de selección

### Requirement: Filtros temporales por día calendario

La lista SHALL ofrecer los filtros Próximos, Hoy, Vencidos y Todos, clasificando por `event_date` comparada contra el día actual en América/Bogotá, y SHALL ordenar por `event_date` y luego por `start_time`. El Console MUST NOT reimplementar la aritmética de vigencia de `quepa-webhook/src/lib/event-window.ts`: no calcula cruce de medianoche, no interpreta `end_time <= start_time` para efectos de vigencia y no evalúa vigencia al minuto.

#### Scenario: Clasificación por día
- **WHEN** hoy es 9 de agosto de 2026 y existen eventos del 6, del 9 y del 15 de agosto
- **THEN** el del 6 aparece en Vencidos, el del 9 en Hoy y el del 15 en Próximos

#### Scenario: Sin aritmética de vigencia duplicada
- **WHEN** un evento del día anterior tiene `start_time` 21:00 y `end_time` 03:00
- **THEN** el Console lo clasifica por su día calendario y no intenta determinar si el agente lo sigue ofreciendo en este instante

### Requirement: Alcance de escritura declarado y anclado al punto de entrada

Abrir un renglón de serie SHALL fijar el alcance en "serie" y abrir una fecha expandida SHALL fijarlo en "esa fecha". La ficha SHALL mostrar el alcance vigente de forma permanente y SHALL permitir conmutarlo antes de guardar. El control de guardado SHALL nombrar el número de fechas que va a escribir. Una escritura de alcance serie SHALL aplicar el cambio a todos los `id` de esa serie.

#### Scenario: Entrada por la serie
- **WHEN** el staff abre el renglón de serie de un evento con 41 fechas
- **THEN** la ficha indica alcance "serie · 41 fechas" y el botón de guardar declara que escribirá 41 fechas

#### Scenario: Entrada por una fecha
- **WHEN** el staff expande la serie y abre la fecha del 16 de agosto
- **THEN** la ficha indica alcance de esa sola fecha y el guardado escribe únicamente ese `id`

#### Scenario: Conmutar el alcance
- **WHEN** el staff abrió una fecha individual y cambia el alcance a serie antes de guardar
- **THEN** el control de guardado actualiza el conteo declarado y la escritura aplica a todos los `id` de la serie

#### Scenario: Escritura de serie
- **WHEN** el staff guarda un `flyer_url` con alcance serie sobre 41 fechas
- **THEN** las 41 filas quedan con ese `flyer_url` y ninguna otra fila se modifica

### Requirement: Hora de fin opcional con semántica de desconocida

El formulario MUST permitir guardar un evento sin `end_time`. Un `end_time` ausente SHALL mostrarse explícitamente como hora de cierre no registrada, y el Console MUST NOT deducir, sugerir ni completar una duración. El Console MUST NOT rechazar `end_time <= start_time`: esa relación significa que el evento cruza medianoche y SHALL indicarse como tal en la ficha.

#### Scenario: Guardado sin hora de cierre
- **WHEN** el staff guarda un evento con hora de inicio y sin hora de fin
- **THEN** el guardado procede con `end_time = null` y la ficha muestra que la hora de cierre no está registrada

#### Scenario: Edición de uno de los eventos sin cierre
- **WHEN** el staff abre uno de los 17 eventos que no tienen `end_time` y modifica otro campo
- **THEN** puede guardar sin diligenciar la hora de fin y `end_time` permanece `null`

#### Scenario: Cruce de medianoche aceptado
- **WHEN** el staff captura inicio 21:00 y fin 03:00
- **THEN** el guardado procede y la ficha indica que el evento termina al día siguiente, sin error de validación

### Requirement: Ciudad por selector anclado a las grafías existentes

El campo de ciudad SHALL ser un selector poblado con la unión deduplicada y ordenada de `places.city` y `city_zones.city`, leídas al cargar la página. El selector SHALL ofrecer una opción explícita de otra ciudad que revele un campo de texto libre acompañado de un aviso visible sobre el riesgo. El Console MUST NOT escribir en `places` ni en `city_zones`.

#### Scenario: Selección de ciudad conocida
- **WHEN** el staff abre la ficha de un evento nuevo
- **THEN** el selector ofrece las ciudades presentes en `places` o en `city_zones`, sin duplicados y ordenadas

#### Scenario: Ciudad fuera del catálogo
- **WHEN** el staff elige la opción de otra ciudad
- **THEN** aparece un campo de texto libre con un aviso visible de que una ciudad mal escrita deja el evento invisible para el agente, y el guardado sigue disponible

#### Scenario: Sin escritura en catálogos de geografía
- **WHEN** el staff guarda un evento con una ciudad escrita a mano
- **THEN** el valor se persiste únicamente en `events.city` y no se inserta ninguna fila en `places` ni en `city_zones`

### Requirement: Etiquetas contra el catálogo con slug normalizado

El campo de tipo de evento SHALL mostrar el catálogo completo de `event_tags` al abrir la ficha y SHALL autocompletar sobre él al escribir. Al confirmar una etiqueta, el Console SHALL normalizar el texto a slug con el mismo criterio que `normalizeTagSlug` de `quepa-webhook/src/db/events.ts` (descomposición NFD, eliminación de diacríticos, minúsculas, colapso de caracteres no alfanuméricos a espacio, recorte) y SHALL reutilizar la etiqueta existente cuando el slug coincida, creando una fila nueva en `event_tags` solo cuando no exista. Los vínculos SHALL persistirse en `event_tag_links`. Todo evento SHALL tener al menos una etiqueta para poder guardarse.

#### Scenario: Reutilización por slug
- **WHEN** el staff escribe "MUSICA " y ya existe la etiqueta con slug `musica concierto` de label "Música / concierto"
- **THEN** el autocompletado ofrece la existente y, al elegirla, se vincula sin crear una fila nueva en `event_tags`

#### Scenario: Variante tipográfica no duplica
- **WHEN** el staff escribe "Música" y existe una etiqueta cuyo slug normalizado es idéntico
- **THEN** se reutiliza la etiqueta existente y `event_tags` no gana filas

#### Scenario: Etiqueta nueva
- **WHEN** el staff escribe un texto cuyo slug normalizado no existe en el catálogo
- **THEN** se crea la fila en `event_tags` con el slug normalizado y el label tal como se escribió, y queda disponible para los siguientes eventos

#### Scenario: Catálogo completo a la vista
- **WHEN** el staff abre la ficha de un evento
- **THEN** puede ver el listado completo de etiquetas existentes sin necesidad de escribir

#### Scenario: Etiqueta obligatoria
- **WHEN** el staff intenta guardar un evento sin ninguna etiqueta
- **THEN** el guardado se bloquea con un error inline

### Requirement: Captura de campos obligatorios y opcionales

El formulario SHALL exigir `name`, `description`, `address`, `city`, `event_date` y `start_time` antes de guardar, y SHALL ofrecer como opcionales `end_time`, `maps_url`, `flyer_url`, `organizer`, `notes`, `contact_phones`, `ticket_url`, `lat` y `lng`. Los campos opcionales vacíos MUST NOT bloquear el guardado ni persistir cadenas vacías en lugar de `null`. `contact_phones` SHALL capturarse en formato de etiquetas, admitiendo varios valores por evento.

#### Scenario: Obligatorios incompletos
- **WHEN** el staff intenta guardar sin dirección
- **THEN** el guardado se bloquea con error inline y no se llama a Supabase

#### Scenario: Opcionales vacíos
- **WHEN** el staff guarda un evento con todos los opcionales vacíos
- **THEN** el guardado procede y esos campos quedan en `null` o como arreglo vacío, no como cadena vacía

#### Scenario: Varios números de contacto
- **WHEN** el staff agrega dos números en el campo de contacto
- **THEN** `contact_phones` se persiste como arreglo con ambos valores

### Requirement: Flyer capturado como URL pública HTTPS

El campo de flyer SHALL aceptar una URL y MUST validar que el esquema sea HTTPS antes de permitir el guardado, porque la API de WhatsApp la consume directo. El Console SHALL mostrar una previsualización de la imagen cuando la URL sea válida. Esta entrega MUST NOT incluir subida de archivos a Supabase Storage.

#### Scenario: URL válida con preview
- **WHEN** el staff pega una URL HTTPS de imagen
- **THEN** aparece la previsualización y el guardado persiste el valor en `flyer_url`

#### Scenario: Esquema no HTTPS rechazado
- **WHEN** el staff pega una URL con esquema `http://`
- **THEN** el guardado se bloquea con un error inline que explica que WhatsApp requiere HTTPS

#### Scenario: Flyer de serie completa
- **WHEN** el staff captura un flyer con alcance serie sobre 18 fechas
- **THEN** las 18 filas quedan con el mismo `flyer_url` en una sola operación de guardado

### Requirement: Prioridad como curaduría interna

La ficha SHALL capturar `priority` con los tres valores 0, 1 y 2, acompañada de un texto que indique que solo selecciona cuáles eventos entran a la tanda de 3 y que nunca reordena, ya que el orden siempre es por hora. El rótulo del campo MUST NOT usar el lenguaje de destacados pagados, que el MVP declara fuera de alcance.

#### Scenario: Captura de prioridad
- **WHEN** el staff marca un evento con prioridad 2
- **THEN** el guardado persiste `priority = 2` y la ayuda del campo explica que la prioridad selecciona pero no reordena

### Requirement: Baja por desactivación con alcance

La baja de un evento SHALL ejecutarse como `active = false` y MUST NOT usar `delete`. La operación SHALL respetar la misma granularidad de alcance que la edición: una fecha o la serie completa. La reactivación SHALL estar disponible de forma simétrica. La lista SHALL distinguir visualmente los eventos inactivos.

#### Scenario: Baja de una fecha
- **WHEN** el staff desactiva la fecha del 22 de agosto de una serie de 18 fechas
- **THEN** esa fila queda con `active = false`, las otras 17 permanecen activas y ninguna fila se elimina

#### Scenario: Baja de la serie completa
- **WHEN** el staff desactiva una serie de 12 fechas
- **THEN** las 12 filas quedan con `active = false` y ninguna se elimina

#### Scenario: Reactivación
- **WHEN** el staff reactiva un evento previamente desactivado
- **THEN** la fila vuelve a `active = true` y reaparece como activa en la lista

### Requirement: Navegación al módulo desde todo el Console

Las seis páginas de `console/` SHALL incluir un ítem "Eventos" en el sidebar que apunte a `console/eventos.html`, marcado como activo cuando la página vigente pertenezca al módulo.

#### Scenario: Ítem presente en todas las páginas
- **WHEN** el staff navega a cualquiera de las páginas del Console
- **THEN** el sidebar muestra el ítem "Eventos"

#### Scenario: Estado activo
- **WHEN** el staff está en `eventos.html` o en `evento.html`
- **THEN** el ítem "Eventos" aparece marcado como activo en el sidebar
