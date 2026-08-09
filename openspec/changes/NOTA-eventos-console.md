# Nota: el esquema de eventos ya está fijado (contrato con el webhook)

**Para quien construya el módulo Eventos del Quepa Console (C1/C2 de
`requirements-mvp-quepa-1.0.md`).**

El change hermano `add-events-agent` en `quepa-webhook` **ya creó la migración
`0013_events.sql`**. Ese esquema es el contrato: el Console escribe, el agente
lee. No hay que diseñarlo de nuevo ni "mejorarlo" desde este lado — un cambio
de forma acordado aquí y no allá rompe al agente en silencio.

## Lo que el Console debe respetar

> **Corrección (2026-08-09, change `add-console-events`).** Esta nota decía que
> `end_time` era obligatoria. **Ya no lo es**: la migración `0015` la volvió
> nullable porque la programación oficial de las Fiestas trae 17 eventos con una
> sola hora, y exigirla obligaba a inventarles el cierre o a dejarlos invisibles.
> El párrafo de abajo queda corregido; el resto de la nota sigue vigente.

**Tabla `events`.** Obligatorios en la base (no solo en el formulario):
`name`, `description`, `address`, `city`, `event_date`, `start_time`.
Opcionales: **`end_time`**, `maps_url`, `flyer_url`, `organizer`, `notes`,
`contact_phones` (text[]), `ticket_url`, `lat`, `lng`.

- **Una sola descripción.** Alimenta la frase breve del top (A8) y la ficha
  emotiva (A9). No agregar un segundo campo de descripción: el equipo de datos
  ya llena decenas de eventos por día de programación.
- **`end_time` es OPCIONAL** (migración `0015`). El formulario **no** debe
  exigirla: hoy hay 17 eventos reales sin ella. `null` significa hora de cierre
  **DESCONOCIDA**, jamás "ya terminó" — misma doctrina que `places.hours`. El
  evento sigue vigente hasta el final de su día calendario y el agente dice en
  voz alta que no tiene la hora. **Nunca deducir una duración.** Y la convención
  `end_time <= start_time` ⇒ **cruza medianoche** (rumba 9 p.m.–3 a.m.) sigue
  igual: el formulario no debe rechazar ese caso como "hora inválida", es el
  caso normal de las verbenas.
- **`priority`** (smallint 0–2: normal / destacado / imperdible). Es curaduría
  humana y **solo selecciona** cuáles eventos entran a la tanda de 3; nunca
  reordena (el orden siempre es por hora) ni cruza el día. El agente jamás la
  muestra al usuario.
- **Identidad = `(name, city, event_date, start_time)`.** El mismo nombre se
  repite legítimamente entre días ("Verbena popular" cada noche): la edición
  debe hacerse por `id`, no por nombre.

**Etiquetas (`event_tags` + `event_tag_links`).** Catálogo compartido, no
texto libre por evento:

- `slug` normalizado (minúsculas, sin tildes, sin espacios sobrantes) para
  comparar; `label` con la forma escrita para mostrar. "Música", "musica" y
  "MUSICA " son **la misma** etiqueta.
- El campo del formulario autocompleta con las existentes y crea la nueva solo
  si no existe. Debe poderse ver el listado completo al cargar un evento — sin
  eso el catálogo degenera en 40 etiquetas que dicen lo mismo y el filtro del
  agente deja de servir.
- Las **diez etiquetas precargadas** ya vienen en la migración. No duplicarlas.

## Lo que NO hay que hacer

- **No correr `embed:sync` ni generar embeddings de eventos.** Los eventos no
  llevan búsqueda semántica por decisión de diseño: un evento cargado queda
  visible en la consulta siguiente, sin ritual de sincronización. Esa es una
  ventaja operativa deliberada, no un pendiente.
- **No escribir en el gazetteer** ni en tablas de lugares desde este módulo.

## Flyer

`flyer_url` debe quedar como **URL pública HTTPS** (Supabase Storage sirve):
la API de WhatsApp la consume directo. El agente la envía como mensaje de
imagen aparte, después del texto de la ficha.
