# Design: add-console-events

## Context

El carril de eventos está desplegado y respondiendo por WhatsApp. Lo único que falta es la superficie de edición: el Console no tiene UI de eventos (verificado — todas las coincidencias de "event" en `console/*.html` son `addEventListener`).

Estado real de la base al 9 de agosto de 2026, medido contra producción:

| Dato | Valor |
|---|---|
| Eventos cargados | 196, todos `active`, todos Pereira |
| Rango de fechas | 2026-08-06 → 2026-09-15 (3 ya vencidos) |
| Pico de programación | 14–30 de agosto (9 a 18 eventos por día) |
| `end_time` presente | 179 de 196 (**17 sin hora de cierre, legítimos**) |
| `organizer` | 196 de 196 |
| `flyer_url` · `maps_url` · `ticket_url` · `contact_phones` | **0 de 196** |
| `priority` | 122 normal · 60 destacado · 14 imperdible (ya curado) |
| Etiquetas | catálogo de 10, 267 vínculos (~1.4 por evento) |
| Identidades duplicadas | **0** — 196 filas, 196 identidades distintas |

Restricciones que enmarcan todo el diseño:

- **El esquema es contrato desplegado.** `0013` y `0015` están en producción y el agente lee de ahí. Este change no toca la base.
- **El Console es estático y autocontenido**: HTML + JS inline, sin build ni bundler, hablando directo a Supabase con la publishable key. Convención del repo — no se introduce build step.
- **Sin embeddings, la ciudad es el único filtro.** `searchVigentEvents` hace `ilike('city', …)`. Los lugares tienen barrido textual de rescate; los eventos no tienen nada.
- **La batería de QA corre del 10 al 14 de agosto.** Lo que desbloquea al equipo es administrar y enriquecer, no capturar.

## Goals / Non-Goals

**Goals:**

- Que el equipo pueda ver, editar y dar de baja los 196 eventos ya cargados, sin editar el equivocado.
- Que llenar los tres huecos grandes (`flyer_url`, `maps_url`, `contact_phones`) cueste órdenes de magnitud menos que una edición por fila.
- Que el formulario no pueda producir un evento inválido ni invisible para el agente: sin inventar horas de cierre, sin typos de ciudad, sin etiquetas duplicadas.
- Cero migraciones y cero conceptos nuevos que el agente tenga que aprender.

**Non-Goals:**

- Uploader a Supabase Storage (ver D6).
- Embeddings de eventos — decisión de diseño de `add-events-agent` D2, revertirla no es completarla.
- Escribir en `places`, `place_locations` o `city_zones` desde este módulo.
- Roles, permisos o auditoría por usuario: hoy todo el equipo administra.
- Importador masivo de programación.
- Consolidar el carril de eventos con el de lugares en el Console (el backend mantiene el espejo a propósito; acá aplica la misma disciplina).

## Decisions

### D1 · La serie es DERIVADA en el cliente, no una columna

Agrupar en memoria por la clave de serie. No hay `series_id`, no hay migración, no hay backfill.

*Alternativas consideradas:* (a) columna `series_id` con migración y backfill de las 196 filas; (b) no tener concepto de serie y listar 196 filas planas.

*Por qué derivada:* la evidencia dice que la serie ya existe en los datos y no necesita persistirse — las filas de una serie son idénticas salvo `event_date` (`address`, `description`, `organizer`, `end_time` y `priority` coinciden en las 5 series más grandes; 16 de 20 series tienen descripción idéntica en todas sus fechas). Una columna agregaría una migración a un esquema en producción a un día de QA, y un concepto que el agente no consume. La opción (b) se descarta por el mismo dato: 144 de 196 filas viven dentro de 20 series, y una lista plana hace que 41 renglones se vean idénticos.

*Costo aceptado:* renombrar una fila la saca de su serie. Es visible en la lista (aparece como puntual) y reversible renombrando de vuelta. No hay integridad que mantener porque no hay estado persistido.

### D2 · La clave de serie es `(name, city, start_time)` — con `start_time`

*Alternativa considerada:* agrupar por `(name, city)` solamente.

*Por qué con hora:* medido, 2 de 70 nombres corren a dos horas distintas y son series genuinamente separadas:

```
"Unicentro Pereira: El Portón de las Fiestas…"  @ 13:00 → 4 fechas
                                                 @ 16:00 → 2 fechas
"Donde la Belleza Florece — Sra. Pereira 2026"  @ 17:00 → 1 fecha
                                                 @ 19:00 → 2 fechas
```

Sin `start_time` en la clave, esas dos parejas se funden (70 renglones en vez de 72) y **una edición a nivel serie escribiría sobre dos series distintas** — exactamente el error que el concepto existe para evitar. `city` va en la clave aunque hoy todo sea Pereira: es parte de la identidad del evento y omitirla rompe silenciosamente cuando entre la ciudad #2.

*Costo aceptado:* si una serie cambia de hora a mitad de corrida, se parte en dos renglones. Es correcto — son dos horarios distintos — y queda visible.

### D3 · El alcance de escritura lo fija el punto de entrada, y siempre se declara

Abrir el renglón de la serie edita la serie (escribe los N ids). Expandir y abrir una fecha edita esa fecha (escribe un id). El alcance se muestra de forma permanente en la ficha y es conmutable antes de guardar; el botón de guardar dice sobre cuántas fechas va a escribir.

*Alternativa considerada:* default siempre a la fecha individual, con escalada explícita a serie.

*Por qué por punto de entrada:* el default seguro y el default útil apuntan a lados opuestos, y elegir uno solo pierde. Con default-a-fecha, llenar el flyer de La Ruta de la Cosecha vuelve a costar 41 ediciones — que es justo lo que esta entrega existe para eliminar. Con default-a-serie siempre, un typo corregido se propaga a 41 filas. Anclar el alcance al renglón que el usuario abrió hace que la intención ya esté expresada en el clic, y la declaración permanente evita que sea implícita.

*Riesgo residual:* está en D-riesgos abajo; es el modo de falla principal de esta entrega.

### D4 · Ciudad: selector de la unión `places.city ∪ city_zones.city`, con escape friccionado

Dos queries al cargar, unión deduplicada y ordenada (11 valores hoy), más una opción "otra ciudad…" que revela texto libre con aviso visible.

*Alternativas consideradas:* (a) solo `places.city` — 6 valores, le faltan Armenia, Salento, Filandia, Marsella y La Virginia que el gazetteer sí tiene; (b) solo `city_zones.city` — 9 valores, le faltan Buenavista y Apía que sí tienen lugares; (c) lista fija en el HTML — sería el séptimo bloque duplicado entre archivos y se desactualiza sola; (d) texto libre — el statu quo, que es el modo de falla.

*Por qué la unión:* es literalmente "las grafías que Quepa ya usa en alguna parte", se automantiene sin código y ninguna de las dos tablas por sí sola es superconjunto de la otra. El modo de falla que se bloquea es grave y **silencioso**: un evento con la ciudad mal escrita no lanza error, no aparece roto, simplemente nunca lo encuentra el agente.

*Por qué el escape existe:* sin él, el día que abran la ciudad #2 nadie puede cargar eventos hasta que alguien edite y publique HTML. Con fricción y aviso, el typo deja de ser accidente de tipeo y pasa a ser acto deliberado.

### D5 · El Console NO reimplementa la aritmética de vigencia

La lista clasifica por **día calendario** (`event_date` vs. hoy en América/Bogotá): Próximos · Hoy · Vencidos. No se porta `isEventVigente`, ni el cruce de medianoche, ni el caso `end == start`.

*Alternativa considerada:* portar `event-window.ts` a JS del Console para que "vencido" coincida al minuto con lo que ve el agente.

*Por qué no:* esa aritmética vive con tests en `quepa-webhook/src/lib/event-window.ts` y tiene tres casos sutiles (cruce de medianoche, `end <= start` = 24h, null = hasta fin del día). Una segunda copia en JS vanilla sin tests se desincroniza callada, y el Console no gana nada real con la precisión al minuto: es una lista de administración, no la respuesta al usuario.

*Costo aceptado y explícito:* una verbena de ayer 9 p.m.–3 a.m. aparece como "vencida" en el Console desde la medianoche, mientras el agente correctamente la sigue ofreciendo hasta las 3 a.m. La lista muestra el estado como día, no como verdad al minuto, y no afirma nada sobre lo que el agente está haciendo en este instante.

### D6 · Flyer por URL pública, uploader después

Campo de texto con validación de esquema HTTPS y preview de la imagen. No se monta Storage.

*Alternativa considerada:* uploader a un bucket de Supabase Storage con lectura pública.

*Por qué URL primero:* no existe subida de imágenes en ninguna de las 6 páginas del Console, así que el uploader exige bucket nuevo, política de lectura pública, política de subida con la publishable key y validación de peso/formato para la API de WhatsApp. Son 0 de 196 los eventos con flyer y el pico de programación empieza en cinco días. La columna destino es la misma (`flyer_url`), así que el uploader entra después escribiendo el mismo campo — sin migrar datos ni reescribir la ficha.

*Restricción dura:* `flyer_url` debe quedar como URL pública HTTPS porque la consume directo la API de WhatsApp. La validación rechaza esquemas no-HTTPS.

### D7 · `end_time` opcional, y el vacío se dice en voz alta

El campo no es requerido. Vacío se muestra como "sin hora de cierre registrada", no como celda muda.

*Por qué:* 17 de los 196 eventos no la tienen y son legítimos — la programación oficial trae una sola hora. Un `required` deja esos 17 sin poder editarse o empuja al staff a inventarles un cierre, que es dato fabricado. `0015` ya fijó la semántica: null = **desconocida**, jamás "ya terminó" (misma doctrina que `places.hours`).

**`end_time <= start_time` NO es error de validación**: es el cruce de medianoche de las verbenas, y es el caso normal. La ficha lo refleja como tal ("termina al día siguiente") en vez de bloquear.

*Nota:* `openspec/changes/NOTA-eventos-console.md` afirma en negrita lo contrario. Quedó desactualizada con `0015` y hay que corregirla o borrarla como parte de esta entrega — es la trampa más cara del change.

### D8 · Baja por desactivación, nunca `delete`

`active = false`, con la misma granularidad de alcance que la edición: una fecha o la serie completa. Reactivar es simétrico.

*Por qué:* cumple C2 criterio 2 — el agente filtra `.eq('active', true)` y el evento sale de las recomendaciones en la consulta siguiente — y conserva el registro de lo que estaba programado. Las dos operaciones reales son distintas: "llueve el sábado" desactiva una fecha, "se cayó el patrocinador" desactiva la serie. Un `delete` duro no distingue y no se deshace. (`lugar.html` sí tiene delete duro para lugares; acá gana la doctrina de desactivación que ya usan sedes y redes sociales.)

### D9 · Etiquetas: el catálogo visible es la defensa primaria; la normalización es el respaldo

El campo muestra las etiquetas existentes al abrir, autocompleta al escribir, y solo crea una nueva cuando el slug normalizado no existe. La normalización replica `normalizeTagSlug` de `quepa-webhook/src/db/events.ts` (NFD, quitar diacríticos, minúsculas, colapsar no-alfanuméricos, trim).

*Riesgo reconocido:* esto es una **segunda copia de una función que vive en el backend**, y no hay red de seguridad de base de datos — el `unique` es sobre `slug`, así que una normalización divergente inserta una etiqueta nueva sin conflicto, en vez de fallar. Por eso el orden importa: la defensa que más pesa es que **el catálogo completo esté a la vista** (son 10 etiquetas), de modo que el humano reutilice antes de escribir. La normalización atrapa lo que se escape de eso. La función se copia verbatim con comentario apuntando al original.

### D10 · El ítem de nav entra en las 6 páginas

*Alternativa considerada:* agregarlo solo a las páginas que el equipo usa a diario.

*Por qué las 6:* el sidebar ya está duplicado por archivo (convención existente del Console, no se introduce un partial ni un build step para esto). Seis ediciones idénticas cuestan minutos; una navegación que aparece y desaparece según la página cuesta confianza en la herramienta.

## Risks / Trade-offs

- **Editar la serie creyendo que se edita el día (o al revés).** Es el modo de falla principal: 41 renglones se llaman igual, todo el equipo administra, y la operación destructiva y la útil comparten formulario. → Mitigación: el alcance se ancla al punto de entrada (D3), se muestra de forma permanente, y el botón de guardar nombra el conteo de fechas afectadas. Ninguna escritura de serie ocurre sin que el conteo esté a la vista en el momento del clic.
- **Elegir la fila equivocada dentro de una serie.** Buscar "La Ruta de la Cosecha" devuelve 41 resultados indistinguibles si la lista muestra solo el nombre. → Mitigación: fecha y hora de inicio son parte de la identidad visible en todo renglón, en lista y en ficha. El nombre nunca identifica solo.
- **Divergencia de la normalización de slugs (D9).** Sin backstop en la base. → Mitigación: catálogo completo visible como defensa primaria, función copiada verbatim, y son 10 etiquetas — un duplicado se nota a simple vista en la próxima carga.
- **El escape de ciudad reintroduce el typo (D4).** → Mitigación: fricción deliberada y aviso visible. Se acepta a cambio de no bloquear la ciudad #2 detrás de un deploy.
- **"Vencido" en el Console no coincide al minuto con el agente (D5).** → Mitigación: es diferencia conocida y acotada a eventos que cruzan medianoche; la lista habla de días, no afirma qué está ofreciendo el agente ahora mismo.
- **La publishable key permite escritura sobre `events` desde el navegador**, igual que hoy sobre `places`. → No es riesgo nuevo ni se resuelve en este change: es la postura vigente del Console. Se deja registrado para que quede como decisión consciente y no como descuido heredado.

## Migration Plan

No hay migración de base. `0013` y `0015` ya están aplicadas en producción — verificado contra la base (196 filas, 179 con `end_time`, catálogo de 10 etiquetas, 267 vínculos). El despliegue es publicación estática del Console, sin orden de deploy respecto al webhook y sin ventana de degradación.

*Rollback:* revertir el commit del Console. Las dos páginas nuevas no tienen consumidores y el ítem de nav desaparece con ellas. Ningún dato queda en estado intermedio, porque toda escritura es un update completo de fila.

## Open Questions

- **Cómo se rotula `priority` en la ficha.** La restricción está fijada (curaduría interna; solo selecciona, nunca reordena; el agente jamás la muestra; "destacados pagados" está fuera del alcance del MVP), pero la palabra exacta no. "Destacado" arrastra el modelo mental equivocado. Se resuelve al escribir el markup.
- **Si el orden por defecto de la lista es por fecha o por serie.** Por fecha responde "qué pasa esta semana"; por serie responde "qué me falta llenar". Ambas son necesidades reales del equipo esta semana; puede terminar siendo un toggle.
- **Qué pasa con `lat`/`lng` de eventos.** Están en el esquema y ninguna tool las consume todavía (`add-events-agent` D10). Capturarlas ahora es barato y evita una segunda pasada, pero no aporta nada hasta que exista el pin de evento. Queda como campo opcional de baja prioridad dentro de esta entrega.
