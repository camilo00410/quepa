# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Two static landing pages for **Quepa** — an AI conversational agent on WhatsApp that recommends places — plus the internal staff panel. No build step, no package manager, no dependencies. Each HTML file is fully self-contained: SVGs inlined, CSS in `<style>`, JS in `<script>`, fonts from Google Fonts CDN.

- `index.html` — B2C landing (intended domain: `quepa.co`)
- `web-comercios/index.html` — B2B landing for merchants (intended domain: `comercios.quepa.co`)
- `console/` — **Quepa Console**, panel interno de staff (login Supabase, catálogo de lugares). Ver sección abajo.

## Quepa Console: sedes (`place_locations`)

El detalle de lugar (`console/lugar.html`, sección `· 06 · SEDES`) gestiona las sedes del modelo marca→sedes definido en `quepa-webhook` (change `add-place-locations`, migración `0008`). Contrato que el Console debe respetar:

- `places` = la marca (descripción, vibe, 1 embedding); `place_locations` = sedes físicas. Las columnas planas de `places` (`address`, `lat/lng`, `phone`, `whatsapp`) son **espejo de la sede `is_primary`**.
- **Regla "el espejo lo mantiene quien escribe"**: lugar mono-sede → las secciones 03/04 son editables y el guardado sincroniza la sede `Principal`; lugar multi-sede (2+ activas) → 03/04 quedan en solo lectura (reflejo de la principal) y la grid de sedes es la única vía de escritura, que a su vez arrastra el espejo.
- Crear un lugar crea también su sede `Principal` (si el insert de la sede falla, reintentar el guardado lo repara — el flujo deja `state.isNew=false` tras el insert del lugar).
- Cambio de principal: **democión primero, coronación después** (el índice único parcial `place_locations_one_primary` rechaza dos primarias). Sedes se **desactivan** (`active=false` + `is_primary=false`), nunca se borran; la UI impide desactivar la principal.
- Labels únicos por lugar (case/espacios-insensible, **incluidas las inactivas** — el `unique (place_id, label)` de la base las cubre). Ediciones por `id` de fila: renombrar un label es un update normal.
- Riesgo conocido: un `pnpm seed --apply` en `quepa-webhook` puede pisar campos de sedes que existan en `data/*.json`; una sede agregada solo desde Console (label fuera del JSON) está protegida sin `--allow-clear`.

`console/lugares.html` muestra un chip "+N sedes" (query batcheada a `place_locations` sobre la página visible).

## Quepa Console: redes sociales (`place_social_links`)

El detalle de lugar (`console/lugar.html`, sección `· 04 · CONTACTO Y RESERVAS`) gestiona redes sociales del modelo normalizado definido en `quepa-webhook` (change `add-place-social-links`, migración `0009`). Contrato que el Console debe respetar:

- Cargar y guardar redes en `place_social_links`, no como columnas nuevas en `places`.
- Plataformas iniciales permitidas: `instagram`, `facebook`, `tiktok`, `youtube`, `google_maps`, `whatsapp_channel`, `website`, `other`.
- Quedan fuera por ahora: X/Twitter, LinkedIn, TripAdvisor y Threads.
- No existe `priority`/`order`; solo `is_primary` como marca booleana de cuenta principal/oficial.
- Quitar una red desde Console desactiva la fila (`active=false`), no la borra físicamente.
- Al crear un lugar nuevo, primero se inserta `places` y luego las redes con el `place_id` resultante.

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
- Comments and copy are in Spanish (es_CO). Match the existing voice when adding text.

## Known gaps (deliberate, per README)

These are tracked as roadmap items, not oversights — don't "fix" them unless the user asks:

- No `og:image` meta tag on either page (a 1200×630 PNG with the Quepa lockup is planned).
- No analytics (GTM / Plausible) wired in.
- No `sitemap.xml` or `robots.txt`.
- Fonts loaded from Google Fonts CDN, not self-hosted.

## Version

v1.0 · Mayo 2026 (per README)
