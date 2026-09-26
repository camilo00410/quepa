# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Static pages for **Quepa** plus the internal staff panel. No build step, no package manager, no dependencies. Each HTML file is fully self-contained: SVGs inlined, CSS in `<style>`, JS in `<script>`, fonts from Google Fonts CDN.

- `index.html` — **Quepa Canchas**, landing principal (intended domain: `quepa.co`) · v2.0 · sep 2026. Es B2B: le habla al dueño de la cancha, no al jugador. Ver sección abajo.
- `b2c/index.html` — landing B2C anterior (v1.0 mayo 2026 + refinamiento ago 2026), **archivada tal cual**. Ahí sigue el formulario de waitlist.
- `web-comercios/index.html` — stub de redirección a `https://quepa.co/` (la landing de comercios se movió a la raíz). Ya no es una landing.
- `console/` — **Quepa Console**, panel interno de staff (login Supabase, catálogo de lugares). Ver sección abajo.

El producto cambió de dirección en sep 2026 (de "recomendamos lugares" a "automatizamos las reservas de tu cancha por WhatsApp"): la landing de Canchas v2.0 se portó desde el repo hermano `Dangk20/quepa` y la B2C pasó a `b2c/` sin tocarla. La voz del bot y de la marca sigue viniendo de `quepa-webhook/context-company-quepa.md`.

## Landing principal: Quepa Canchas (`index.html`)

Una sola conversión: **agendar una reunión por WhatsApp**. Sin formularios (es la misma promesa que se le hace al dueño) y **sin precios expuestos** — el precio se ve en la reunión. Contrato del archivo:

- `WA_VENTAS` (configurada, `573142751611`) — todos los CTA `.js-wa` se reescriben en el load a `https://wa.me/<WA_VENTAS>?text=…`. Si se vacía, caen a `mailto:hola@quepa.co` con el mismo mensaje.
- Cada CTA lleva `data-cta="<sección>"` (hero, nav, como-funciona, montaje, final, footer, flotante) y el mensaje termina en `[web·<sección>]` para que ventas sepa de dónde llegó el prospecto. **Cualquier botón nuevo de agendar debe llevar `js-wa` + `data-cta`** o queda muerto.
- `PANEL_URL` está **vacía a propósito**: es el botón "Ingresar" de la barra superior (acceso de clientes actuales). Mientras esté vacía el click se cancela con `preventDefault()`. Cuando exista el panel real (p. ej. `https://panel.quepa.co`) se pone ahí.
- Los chats de WhatsApp de demostración **no son scrolleables por el usuario** (ni rueda, ni trackpad, ni dedo): solo los mueve la animación. Es deliberado — no "arreglarlo".
- El guion de la conversación del hero vive en `QP_GUION` (pasos `day`/`in`/`out`/`tap`/`sys`/`loc`, con `wait` opcional en ms). Editar copy del demo = editar ese array, no el DOM.
- Toda animación respeta `prefers-reduced-motion`.

## Quepa Console: sedes (`place_locations`)

El detalle de lugar (`console/lugar.html`, sección `· 03 · SEDES`) gestiona las sedes del modelo marca→sedes definido en `quepa-webhook` (change `add-place-locations`, migración `0008`). Contrato que el Console debe respetar (actualizado por el change `consolidate-location-into-sedes`):

- `places` = la marca (descripción, vibe, 1 embedding); `place_locations` = sedes físicas. Las columnas planas de `places` (`address`, `lat/lng`, `phone`, `whatsapp`) son **espejo de la sede `is_primary`**.
- **La grid de SEDES es la ÚNICA superficie de edición** de ubicación y contacto por sede, en mono y multi-sede por igual (ya no existe la regla bimodal ni secciones espejo editables). Al guardar, `gather()` deriva el espejo de `places` de la sede ⭐ en memoria. Ciudad y región son **identidad de la marca** y se editan en `· 01 · IDENTIDAD` — no son ubicación.
- **Alta y huérfanos**: un lugar nuevo (o uno sin sedes activas) arranca con la fila `Principal` ⭐ pre-creada en la grid; la validación exige al menos una sede activa con exactamente una principal. Si el insert de la sede falla, reintentar el guardado lo repara (`state.isNew=false` queda tras el insert del lugar).
- Cambio de principal: **democión primero, coronación después** (el índice único parcial `place_locations_one_primary` rechaza dos primarias). Sedes se **desactivan** (`active=false` + `is_primary=false`), nunca se borran; la UI impide desactivar la principal.
- Labels únicos por lugar (case/espacios-insensible, **incluidas las inactivas** — el `unique (place_id, label)` de la base las cubre). Ediciones por `id` de fila: renombrar un label es un update normal.
- **Zona canónica (`city_zones`, change `add-console-zone-selector` / backend `add-city-zones`, migración `0010`)**: la zona se captura por **selector por sede** poblado de `city_zones` (activas de la ciudad efectiva — `location.city ?? ciudad del lugar`) y se guarda como `place_locations.zone_id`. El Console **jamás** escribe el campo legado `zone` (texto deprecado, solo fallback de lectura: canónico si hay FK, legado si no — regla en `fillZoneSelect`), **jamás** inserta filas en `city_zones` (el gazetteer se cura en el webhook: `pnpm zones:discover` → `data/zones/*.json` → `pnpm zones:apply`). Una sede solo-legado muestra `"…" · texto legado` como opción no seleccionable; la transición a FK la hace el staff eligiendo del selector. El **centroide** de zona NUNCA se persiste en `lat`/`lng` (el mapa de preview se eliminó; el invariante queda trivial — el Console ya no lee centroides). Sin zonas para la ciudad: selector deshabilitado con tooltip, el guardado del resto no se bloquea, y no se reabre texto libre.
- **Coordenadas por URL de Maps (change `add-console-maps-link-coords`)**: la vía primaria de captura de lat/lng es pegar la URL **completa** de Google Maps en el campo de coordenadas de cada sede; el Console extrae con regex local en orden `!3d/!4d` → `@` → `q=` → `ll=` — **cero llamadas externas**. La URL no se persiste; los floats siguen siendo el dato guardado, editables como detalle colapsado. Un link corto ahí produce nota honesta (no trae coordenadas y su casa es el campo "Link de Maps"). Fuera del rango aproximado de Colombia se acepta con aviso, nunca se bloquea.
- **Link de Maps por sede (change `consolidate-location-into-sedes`)**: cada sede tiene un campo "Link de Maps" para el link **corto** de compartir (lo que ve el cliente). Se persiste en `place_social_links` como `{platform: 'google_maps', place_location_id: <sede>, is_primary: false}` (`saveMapsLinks`, corre después de `saveSedes` para tener los ids). Vaciar el campo desactiva la fila, nunca la borra. Filas `google_maps` de marca sin sede (`place_location_id` null): en mono-sede se adoptan a la `Principal` al guardar (nota visible antes); en multi-sede aparecen como "link sin sede asignada" con selector — nunca se migran en silencio ni se borran. **Dependencia**: el change hermano de `quepa-webhook` que enruta `place_location_id` en `get_place_details` debe estar desplegado ANTES de publicar esta versión del Console.

`console/lugares.html` muestra un chip "+N sedes" (query batcheada a `place_locations` sobre la página visible).

## Quepa Console: eventos (`events`, `event_tags`, `event_tag_links`)

`console/eventos.html` (lista) y `console/evento.html` (ficha) administran la programación que el bot ofrece cuando alguien pregunta qué hacer (change `add-console-events`; backend `add-events-agent` en quepa-webhook, migraciones `0013` y `0015` — ya aplicadas). **Este módulo no tocó el esquema**: es solo front. Contrato:

- **La serie es DERIVADA, no una columna.** Se agrupa en el cliente por `(name, city, start_time)` y **no se persiste nada** de ese agrupamiento. La hora va en la clave a propósito: una misma actividad puede correr a dos horas distintas y son series separadas (medido: 2 de 70 nombres) — sin ella, una edición de serie escribiría sobre dos. Al 2026-08-09: 196 filas colapsan a 72 renglones (52 puntuales + 20 series). Renombrar una fila la saca de su serie; es el costo aceptado y la ficha lo avisa antes de guardar.
- **El alcance de escritura lo fija el punto de entrada** y se muestra permanente: abrir el renglón de serie edita las N fechas, abrir una fecha edita una. Es conmutable y el botón de guardar **siempre dice cuántas fechas va a escribir**. Con alcance serie, `event_date` queda **deshabilitada** y no viaja en el payload: escribirla pondría las N filas el mismo día y chocaría con el `unique (name, city, event_date, start_time)`.
- **`end_time` es OPCIONAL** (migración `0015`): hay 17 eventos reales sin ella. Vacío = hora de cierre DESCONOCIDA, jamás "ya terminó"; se muestra explícito, nunca mudo. `end_time <= start_time` **no es error** — es cruce de medianoche y la ficha lo dice ("termina al día siguiente").
- **El Console NO reimplementa la vigencia.** La aritmética (cruce de medianoche, `end == start` = 24h, null = hasta fin del día) vive con tests en `quepa-webhook/src/lib/event-window.ts`. Acá la lista solo clasifica por **día calendario** en América/Bogotá. Diferencia conocida y visible en el pie: una verbena que cruza medianoche puede figurar "vencida" en el Console y seguir viva para el agente.
- **Ciudad por selector, poblado de la unión `places.city` ∪ `city_zones.city`** (11 valores hoy). Ninguna de las dos contiene a la otra: `places` tiene Buenavista y Apía sin gazetteer, `city_zones` tiene Armenia, Salento, Filandia, Marsella y La Virginia sin lugares. Hay escape "otra ciudad…" con texto libre **y aviso visible**, porque el modo de falla es grave y silencioso: eventos se busca con `ilike('city', …)` y **sin embeddings no tiene la red textual de rescate de los lugares** — un typo deja el evento invisible sin ningún error. El Console **jamás escribe** en `places` ni en `city_zones`.
- **Etiquetas contra el catálogo, nunca texto libre.** `normalizeTagSlug` está **copiada verbatim** de `quepa-webhook/src/db/events.ts` (verificado: mismo slug en todos los casos probados) — si allá cambia, acá también, porque **no hay constraint que atrape la divergencia** (el unique es sobre el slug ya normalizado). La defensa primaria es que **el catálogo completo esté a la vista como chips**: `"MUSICA / CONCIERTO  "` reutiliza `musica concierto`, pero `"MUSICA "` normaliza a `musica` y **sí** crea una etiqueta nueva — correcto por diseño, y contra eso solo sirve que el humano vea la existente. Al menos una etiqueta es obligatoria.
- **Flyer por URL pública HTTPS**, con preview y validación que bloquea el guardado. **No hay uploader**: no existe subida de archivos en ninguna página del Console y montarla exige bucket de Storage + políticas. La columna destino es la misma, así que el uploader entra después sin migrar nada.
- **`priority` se rotula "Prioridad de selección"** (Normal · Preferente · Imperdible), **nunca "destacado"**: los destacados pagados están fuera del alcance del MVP. Solo decide *cuáles* entran a la tanda de 3 cuando sobran; **nunca reordena** (el orden es siempre por hora) y el usuario jamás la ve.
- **Baja por desactivación (`active=false`), nunca `delete`** — a diferencia de `lugar.html`, que sí borra lugares. Respeta el alcance: una fecha ("llueve el sábado") o la serie completa ("se cayó el patrocinador"), con confirmación que nombra el conteo. Reactivar es simétrico.
- Navegación: el `nav-item` de Eventos está en las **4** páginas con sidebar (`lugares`, `lugar`, `mapa`, `descubrir`); `welcome.html` usa tarjetas `.mod` y lleva una tarjeta; `index.html` es el login y no lleva navegación.

## Quepa Console: redes sociales (`place_social_links`)

El detalle de lugar (`console/lugar.html`, sección `· 04 · CONTACTO Y RESERVAS`) gestiona redes sociales del modelo normalizado definido en `quepa-webhook` (change `add-place-social-links`, migración `0009`). La sección 04 es solo **datos de marca**: website, booking y redes — teléfono/WhatsApp se editan por sede en SEDES. Contrato que el Console debe respetar:

- Cargar y guardar redes en `place_social_links`, no como columnas nuevas en `places`.
- Plataformas del dropdown de marca: `instagram`, `facebook`, `tiktok`, `youtube`, `whatsapp_channel`, `website`, `other`. **`google_maps` NO está en el dropdown**: el link corto de Maps se captura por sede en la grid de SEDES (ver sección de sedes) y las filas `google_maps` se filtran de la lista de redes al cargar.
- Quedan fuera por ahora: X/Twitter, LinkedIn, TripAdvisor y Threads.
- No existe `priority`/`order`; solo `is_primary` como marca booleana de cuenta principal/oficial.
- Quitar una red desde Console desactiva la fila (`active=false`), no la borra físicamente.
- Al crear un lugar nuevo, primero se inserta `places`, luego las sedes (`saveSedes`), luego los links de Maps por sede (`saveMapsLinks`) y por último las redes de marca — las sedes van antes porque los links necesitan sus ids.

## Quepa Console: rango de precios (`places.price_min`/`price_max`)

La sección `· 02 · ECONÓMICO Y SERVICIOS` de `console/lugar.html` captura el rango de precios real en COP (change `add-console-price-ranges`; backend `add-price-ranges` en quepa-webhook, migración `0012` — aplicada 2026-07-23). Contrato:

- Campos "Rango en pesos · desde/hasta": texto con separador de miles en pantalla (`fmtMiles`), enteros limpios en la base (`parseCOP`). Un solo extremo diligenciado es válido.
- `desde > hasta` **bloquea el guardado** con error inline (espejo del check `places_price_range_coherent` — mejor mensaje claro que error críptico de PostgREST). Valores 1–999 solo generan **aviso no bloqueante** ("¿va en miles?"); el Console señala, nunca multiplica ni corrige.
- El picker $–$$$$ (`price_range`) es independiente y sigue **manual** (decisión 2026-07-23): no se deriva del rango ni al revés.
- La unidad **no se captura, se comunica**: hotel = por noche; el resto = por persona (la nota bajo los campos cambia con la categoría). El bot construye el texto final (`price_display`) en el webhook.

## Running locally

There is no dev server config. Use any static server from the repo root, e.g.:

```bash
python3 -m http.server 8000
# http://localhost:8000/         → landing Canchas
# http://localhost:8000/b2c/     → landing B2C archivada (la del formulario)
# http://localhost:8000/console/ → Quepa Console
```

The `/subscribe` endpoint on `webhook.quepa.co` whitelists `http://localhost:8000` and `http://127.0.0.1:8000` only when the webhook is running with `NODE_ENV !== 'production'` — see `quepa-webhook/src/index.ts`. To exercise the form against the real production endpoint locally, set `WAITLIST_ENDPOINT` temporarily to a tunneled URL (e.g., ngrok pointing at a local webhook instance).

## Deploy

Vercel (`vercel.json`) and Netlify (`netlify.toml`) are both pre-configured to publish the repo root as a static site with security headers and `must-revalidate` caching on `*.html`. No build command. Si existe un proyecto aparte para `comercios.quepa.co` con root `/web-comercios`, ese subdominio ahora solo sirve el stub que redirige a `quepa.co` — se puede dejar así o apagar el proyecto.

## Waitlist endpoint (B2C archivada)

El formulario vive ahora en **`b2c/index.html`** (~line 1110), no en la raíz: la landing de Canchas **no tiene formulario**. El origen (`https://quepa.co`) no cambió, así que el whitelist CORS del webhook sigue sirviendo.

The B2C form posts JSON `{ email, city?, source, company }` to `WAITLIST_ENDPOINT` defined in `b2c/index.html`. It currently points at `https://webhook.quepa.co/subscribe`, which is the Fastify endpoint in the sibling repo `quepa-webhook` (writes to a Supabase `waitlist` table, dedupes by email, rate-limits 5/min/IP).

- `source` is `"hero"` or `"foot"` depending on which form was used.
- `company` is a **honeypot** — an off-screen input the user never sees. The backend silently discards any submission where it's non-empty. Do not remove the input, do not stop sending it in the payload.
- If `WAITLIST_ENDPOINT` is empty, the form runs in demo mode (logs to console, simulates success).

If you change the endpoint URL, also update the CORS whitelist on the webhook side (`quepa-webhook/src/index.ts`, `CORS_ORIGINS`).

## Placeholder pendiente

`PANEL_URL` en `index.html` — URL del panel de clientes, ver la sección de la landing principal. El viejo `DEMO_URL` de Cal.com desapareció con la landing B2B v1.0 (la conversión es WhatsApp, no agenda web).

## Cross-links

- `index.html` (Canchas) no enlaza a otros dominios de Quepa: solo `wa.me`, `mailto:hola@quepa.co` y el futuro `PANEL_URL`.
- `b2c/index.html` tiene hardcodeado `https://comercios.quepa.co` (nav CTA y footer), que hoy redirige a la raíz. Se deja así: la B2C es un archivo histórico.
- `web-comercios/index.html` apunta a `https://quepa.co/` en `<meta refresh>`, `<link rel=canonical>` y `location.replace`. Si cambia el dominio, los tres.

## Editing conventions

- Keep files self-contained — do not introduce a build step, bundler, or external JS/CSS files unless explicitly asked.
- Brand tokens (colors `#0A0A0A`, `#D4F542`, `#25D366`, papel `#F3F1EC`; fonts Hanken Grotesk + JetBrains Mono) están duplicados en el `<style>` de cada landing. Al cambiar valores de marca, actualizar todos los archivos.
- El acento verde-amarillo en la landing de Canchas se reserva a CTA y destacados; el dominante es noche/papel. El rojo `--loss` es **fuera de la paleta de marca** a propósito (solo pérdidas en "Lo que pasa hoy").
- **Console typography diverges from the landings on purpose** (decisión 2026-07-23): las 6 páginas de `console/` usan **Manrope** (400–800) + JetBrains Mono; las landings B2C/B2B mantienen Hanken Grotesk. No "normalizar" el Console de vuelta a Hanken.
- Comments and copy are in Spanish (es_CO). Match the existing voice when adding text.

## Known gaps (deliberate, per README)

These are tracked as roadmap items, not oversights — don't "fix" them unless the user asks:

- No `og:image` meta tag on either page (a 1200×630 PNG with the Quepa lockup is planned).
- No analytics (GTM / Plausible) wired in.
- No `sitemap.xml` or `robots.txt`.
- Fonts loaded from Google Fonts CDN, not self-hosted.

## Version

- Landing principal (Quepa Canchas): **v2.0 · sep 2026**
- Landing B2C archivada (`b2c/`): v1.0 · mayo 2026 (+ refinamiento ago 2026)
