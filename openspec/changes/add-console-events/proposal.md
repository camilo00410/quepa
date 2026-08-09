# Proposal: add-console-events

## Why

El carril de eventos ya está completo **salvo la superficie de edición**: el backend (`add-events-agent` en quepa-webhook, migraciones `0013` y `0015`) está desplegado, el agente responde eventos, y la programación de las Fiestas de la Cosecha **ya está cargada — 196 eventos de Pereira, del 6 de agosto al 15 de septiembre de 2026**. El Console no tiene una sola línea de UI de eventos: hoy nadie del equipo puede corregir una hora, desactivar un evento cancelado ni completar un dato faltante.

Y faltan datos que el agente sí sabe usar. De los 196 cargados: **0 tienen flyer** (A9 lo manda por WhatsApp), **0 tienen maps_url**, **0 tienen números de contacto**. El trabajo real de esta entrega no es capturar el evento #197 — es administrar y enriquecer los 196 que ya viven, con la batería de QA corriendo del 10 al 14 de agosto y el pico de la programación entre el 14 y el 30.

Cubre C1 (formulario) y C2 (administración) de `requirements-mvp-quepa-1.0.md`, en ese orden de prioridad invertido: C2 primero, porque es lo que está bloqueado.

## What Changes

**Dos páginas nuevas**, espejo del par `lugares.html` / `lugar.html` que el Console ya usa:

- `console/eventos.html` — lista, filtros y estado.
- `console/evento.html?id=<uuid>|new` — ficha de captura y edición.
- Ítem "Eventos" en el sidebar de las 6 páginas de `console/` (el sidebar está duplicado por archivo; se actualizan todas para no dejar navegación inconsistente).

**La serie es el concepto central de la lista, y es DERIVADA — no hay columna ni migración.** Se agrupa en el cliente por `(name, city, start_time)`. Hoy eso colapsa 196 filas en 72 renglones (52 eventos puntuales + 20 series recurrentes que contienen 144 filas). Medido sobre los datos reales, las filas de una serie son idénticas salvo `event_date`: `address`, `description`, `organizer`, `end_time` y `priority` coinciden en las 5 series más grandes, y 16 de las 20 tienen descripción idéntica en todas sus fechas.

- La lista muestra series colapsadas y expandibles a sus fechas; los eventos puntuales van como fila simple.
- **El nombre nunca identifica por sí solo**: fecha y hora de inicio son parte de la identidad visible en todo renglón. "La Ruta de la Cosecha" son 41 filas legítimas (una por día, 41 días seguidos a las 12:00) — una lista que muestre solo el nombre es una máquina de editar la fila equivocada. Buscar por nombre devuelve 41 resultados idénticos si no se muestran fecha y hora.
- La edición puede hacerse **a nivel serie** (escribe los N ids) o **a nivel fecha** (escribe uno). El formulario declara en todo momento sobre cuántas fechas va a escribir, antes de guardar. Es el pago operativo del concepto: ponerle flyer a La Ruta de la Cosecha pasa de 41 ediciones a una.
- Renombrar una fila la saca de su serie. Es consecuencia aceptada de que la serie sea derivada; queda visible y es reversible.
- Filtros temporales por defecto: **Próximos · Hoy · Vencidos · Todos**, ordenados por fecha y hora. Una lista plana de 196 filas no es usable, y 3 eventos ya vencieron.

**Campos, con las reglas duras del contrato:**

- **`end_time` es OPCIONAL.** 17 de los 196 no la tienen y son legítimos (la programación oficial trae una sola hora). Null = **desconocida**, jamás "ya terminó" — misma doctrina que `places.hours`. Un formulario que la exija deja esos 17 inevitables de solo lectura o empuja al staff a inventarles un cierre. El campo vacío se muestra explícitamente como "sin hora de cierre registrada", no mudo.
- **`end_time <= start_time` = cruza medianoche** (verbena 9 p.m.–3 a.m.) y el formulario **no lo rechaza como hora inválida**: es el caso normal de las fiestas.
- **Ciudad: selector** poblado de la unión de `places.city` ∪ `city_zones.city` (11 valores hoy: Pereira, Dosquebradas, Manizales, Santa Rosa de Cabal, Buenavista, Apía, Armenia, Filandia, La Virginia, Marsella, Salento), más una opción explícita "otra ciudad…" que abre texto libre **con aviso visible**. Ninguna de las dos tablas sola sirve: `places` tiene Buenavista y Apía que el gazetteer no tiene, `city_zones` tiene Armenia, Salento, Filandia, Marsella y La Virginia que `places` no. El modo de falla que se bloquea es grave y silencioso: eventos se busca con `ilike('city', …)` y **sin embeddings no tiene la red de rescate textual que sí tienen los lugares** — un typo hace el evento invisible para el agente, sin síntoma. El escape existe para que abrir la ciudad #2 no requiera tocar código, pero es un acto deliberado con fricción, no un accidente de tipeo.
- **Etiquetas contra el catálogo `event_tags`**, no texto libre: autocompleta con las existentes, muestra el listado completo, y crea la nueva solo si el **slug normalizado** (minúsculas, sin tildes, sin espacios sobrantes) no existe. "Música", "musica" y "MUSICA " son la misma etiqueta. Las 10 precargadas ya están en la base y no se duplican. Escribe en `event_tag_links`.
- **`priority`** (0–2) se captura como **curaduría interna**, etiquetada de forma que no se lea como destacado pagado — el MVP declara los destacados pagados fuera de alcance, y el agente jamás la muestra al usuario. Se explica en el campo que solo **selecciona** cuáles entran a la tanda de 3 y que nunca reordena (el orden siempre es por hora).
- **Flyer por URL pública HTTPS** (campo de texto con preview y validación de esquema), **no uploader**. Ver "Fuera de alcance".
- Resto de opcionales: `maps_url`, `organizer`, `notes`, `contact_phones` (formato etiqueta, varios), `ticket_url`, `lat`/`lng`.
- La edición se hace **por `id`**, nunca por nombre: la identidad es `(name, city, event_date, start_time)` y el mismo nombre se repite legítimamente entre días.

**Baja de eventos: desactivar (`active=false`), nunca borrar.** Cumple C2 criterio 2 — el agente filtra por `active` y el evento sale de las recomendaciones de inmediato — y conserva el registro de lo que se programó. Se puede desactivar **una fecha** ("llueve el sábado") o **la serie completa** ("se cayó el patrocinador"), con la misma declaración de alcance que la edición. Reactivar es simétrico.

**No se toca el esquema.** Las migraciones `0013` y `0015` ya están aplicadas en producción y cubren todo lo anterior. Esta entrega es solo front del Console.

**Fuera de alcance (explícito):**

- **Uploader a Supabase Storage.** No existe subida de imágenes en ninguna de las 6 páginas del Console, y montarla exige bucket, lectura pública y política de subida con el anon key. El campo de URL desbloquea los 196 flyers esta semana y la columna es la misma, así que el uploader entra después sin romper nada ni migrar datos.
- **Embeddings de eventos.** Los eventos no llevan búsqueda semántica por decisión de diseño (`add-events-agent`, D2): un evento guardado queda visible en la consulta siguiente. **No se corre `embed:sync`** — agregarles embeddings sería revertir una decisión, no completar una.
- **Escribir en `places`, `place_locations` o el gazetteer `city_zones`** desde este módulo. La lista de ciudades se **lee**, nunca se escribe.
- Roles, permisos o auditoría por usuario: por ahora todo el equipo administra.
- Carga masiva / importador de programación: los 196 ya están cargados y la próxima programación se evaluará con ese caso en la mano.

## Capabilities

### New Capabilities

- `console-events`: administración y captura de eventos en el Quepa Console — lista con series derivadas y filtros temporales, ficha con edición a nivel serie o fecha, etiquetas contra catálogo normalizado, selector de ciudad anclado a las grafías existentes, hora de fin opcional con semántica de desconocida, y baja por desactivación.

### Modified Capabilities

<!-- Ninguna: lugares, sedes, zonas, precios y redes no cambian de requisitos. -->

## Impact

- **Código**: `console/eventos.html` y `console/evento.html` (nuevos); ítem de navegación en las 6 páginas existentes de `console/` (`index.html`, `welcome.html`, `lugares.html`, `lugar.html`, `mapa.html`, `descubrir.html`).
- **Datos**: lectura y escritura sobre `events`, `event_tags`, `event_tag_links`. Lectura de `places.city` y `city_zones.city`. Ninguna escritura fuera de las tres tablas de eventos.
- **Dependencia de despliegue**: **ninguna migración nueva**. `0013` y `0015` ya están aplicadas en producción — verificado contra la base: 196 filas, 179 con `end_time` y 17 sin ella, catálogo de 10 etiquetas y 267 vínculos.
- **Riesgo de duplicación de reglas**: la aritmética de vigencia (cruce de medianoche, `end == start` = 24 h, null = hasta fin del día calendario) vive con tests en `quepa-webhook/src/lib/event-window.ts`. El Console **no la reimplementa**: la lista filtra por `event_date` y muestra el estado del día, sin recalcular vigencia al minuto. Cualquier necesidad de precisión fina se resuelve mostrando el estado como difuso, no duplicando la regla.
- **Docs**: nota en `quepa-landing/CLAUDE.md` (sección Console) con el contrato de captura. **Corregir o eliminar `openspec/changes/NOTA-eventos-console.md`**: afirma en negrita que `end_time` es obligatoria, lo cual quedó desactualizado con la migración `0015` y es exactamente la trampa que dejaría 17 eventos reales sin poder editarse.
- **Calendario**: la batería de QA corre del 10 al 14 de agosto de 2026 y el pico de la programación es del 14 al 30. C2 (lista, edición, baja) es lo que desbloquea al equipo; C1 (captura nueva) puede llegar después sin dejar a nadie bloqueado.
