# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Two static landing pages for **Quepa** — an AI conversational agent on WhatsApp that recommends places — plus the internal staff panel. No build step, no package manager, no dependencies. Each HTML file is fully self-contained: SVGs inlined, CSS in `<style>`, JS in `<script>`, fonts from Google Fonts CDN.

- `index.html` — B2C landing (intended domain: `quepa.co`)
- `web-comercios/index.html` — B2B landing for merchants (intended domain: `comercios.quepa.co`)
- `console/` — **Quepa Console**, panel interno de staff (login Supabase, catálogo de lugares). Ver sección abajo.

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
# then open http://localhost:8000/
# and    http://localhost:8000/web-comercios/
```

The `/subscribe` endpoint on `webhook.quepa.co` whitelists `http://localhost:8000` and `http://127.0.0.1:8000` only when the webhook is running with `NODE_ENV !== 'production'` — see `quepa-webhook/src/index.ts`. To exercise the form against the real production endpoint locally, set `WAITLIST_ENDPOINT` temporarily to a tunneled URL (e.g., ngrok pointing at a local webhook instance).

## Deploy

Vercel (`vercel.json`) and Netlify (`netlify.toml`) are both pre-configured to publish the repo root as a static site with security headers and `must-revalidate` caching on `*.html`. No build command. To get `quepa.co` + `comercios.quepa.co` as separate subdomains on either platform, create **two** projects pointing at the same repo with different root directories (`/` and `/web-comercios`).

## Waitlist endpoint (B2C)

The B2C form posts JSON `{ email, city?, source, company }` to `WAITLIST_ENDPOINT` defined in `index.html` (~line 1110). It currently points at `https://webhook.quepa.co/subscribe`, which is the Fastify endpoint in the sibling repo `quepa-webhook` (writes to a Supabase `waitlist` table, dedupes by email, rate-limits 5/min/IP).

- `source` is `"hero"` or `"foot"` depending on which form was used.
- `company` is a **honeypot** — an off-screen input the user never sees. The backend silently discards any submission where it's non-empty. Do not remove the input, do not stop sending it in the payload.
- If `WAITLIST_ENDPOINT` is empty, the form runs in demo mode (logs to console, simulates success).

If you change the endpoint URL, also update the CORS whitelist on the webhook side (`quepa-webhook/src/index.ts`, `CORS_ORIGINS`).

## B2B placeholder still pending

`DEMO_URL` in `web-comercios/index.html` (~line 1275) is `https://cal.com/quepa/demo` — replace with the real Cal.com / Calendly link when it's ready. The script rewrites the `href` of every CTA matched by its selector to this constant on page load, so changing CTA markup may silently break the rewrite — verify the selector after structural edits.

## Cross-links

`https://quepa.co` ↔ `https://comercios.quepa.co` are hardcoded in both files (nav CTAs and footer). Search for these strings if changing domains.

## Editing conventions

- Keep files self-contained — do not introduce a build step, bundler, or external JS/CSS files unless explicitly asked.
- Brand tokens (colors `#0A0A0A`, `#D4F542`, `#25D366`; fonts Hanken Grotesk + JetBrains Mono) are duplicated in both files' `<style>` blocks. When changing brand values, update both.
- **Console typography diverges from the landings on purpose** (decisión 2026-07-23): las 6 páginas de `console/` usan **Manrope** (400–800) + JetBrains Mono; las landings B2C/B2B mantienen Hanken Grotesk. No "normalizar" el Console de vuelta a Hanken.
- Comments and copy are in Spanish (es_CO). Match the existing voice when adding text.

## Known gaps (deliberate, per README)

These are tracked as roadmap items, not oversights — don't "fix" them unless the user asks:

- No `og:image` meta tag on either page (a 1200×630 PNG with the Quepa lockup is planned).
- No analytics (GTM / Plausible) wired in.
- No `sitemap.xml` or `robots.txt`.
- Fonts loaded from Google Fonts CDN, not self-hosted.

## Version

v1.0 · Mayo 2026 (per README)
