# Quepa — Landings + Console

Sitios estáticos de Quepa. Cada HTML es **autocontenido**: SVGs inlineados, CSS en `<style>`, JS en `<script>`, fuentes desde Google Fonts CDN. No hay build step ni dependencias.

## Estructura

```
quepa-landing/
├── index.html              ← Landing principal (quepa.co) · Quepa Canchas · v2.0 · sep 2026
├── b2c/index.html          ← Landing B2C anterior (v1.0, mayo 2026), conservada tal cual
├── web-comercios/index.html← Redirección a quepa.co (la landing de comercios se movió a la raíz)
├── console/                ← Quepa Console: panel interno de staff (login Supabase, catálogo)
├── openspec/               ← Changes y specs del Console
├── vercel.json · netlify.toml
└── README.md               ← este archivo
```

## Landing principal — Quepa Canchas (`index.html`)

Landing B2B para dueños de canchas: **Quepa automatiza las reservas por WhatsApp**. Una sola conversión — **agendar una reunión** por WhatsApp. Sin formularios y sin precios expuestos (el precio se ve en la reunión).

Las dos constantes que importan están en el `<script>` al final del archivo:

| Constante | Estado | Qué hace |
|---|---|---|
| `WA_VENTAS` | Configurada (`573142751611`) | Todos los botones "Agenda una reunión" y "Escríbenos" abren WhatsApp con un mensaje ya escrito que termina en `[web·<sección>]`, para saber desde dónde llegó cada prospecto. Si se vacía, los CTA caen a `mailto:hola@quepa.co`. |
| `PANEL_URL` | **Vacía a propósito** | Botón **"Ingresar"** de la barra superior (acceso de clientes actuales). No hace nada hasta que se ponga aquí la URL del panel real, p. ej. `https://panel.quepa.co`. |

Notas:

- Los chats de WhatsApp de demostración **no son scrolleables por el usuario** (ni rueda, ni trackpad, ni dedo): solo los mueve la animación.
- Toda animación respeta `prefers-reduced-motion`.
- Cualquier botón nuevo de agendar debe llevar la clase `js-wa` y un `data-cta="<sección>"`.

## Landing B2C archivada (`b2c/index.html`)

La landing de consumidor v1.0 (mayo 2026, con el refinamiento de motion y tipografía de ago 2026). Se conserva tal cual: ahí sigue viviendo el formulario de waitlist que hace POST a `https://webhook.quepa.co/subscribe` con `{ email, city?, source, company }` (`company` es honeypot — no quitarlo). Sus cross-links a `comercios.quepa.co` ahora redirigen a la raíz.

## Correr en local

```bash
python3 -m http.server 8000
# http://localhost:8000/        → landing Canchas
# http://localhost:8000/b2c/    → landing B2C archivada
# http://localhost:8000/console/→ Quepa Console
```

El endpoint `/subscribe` de `webhook.quepa.co` solo permite `http://localhost:8000` y `http://127.0.0.1:8000` cuando el webhook corre con `NODE_ENV !== 'production'` — ver `quepa-webhook/src/index.ts`.

## Deploy

Vercel (`vercel.json`) y Netlify (`netlify.toml`) publican la raíz del repo como sitio estático, con headers de seguridad y `must-revalidate` en `*.html`. Sin build command.

Si existía un proyecto aparte para `comercios.quepa.co` con raíz `/web-comercios`, ahora ese subdominio redirige a `quepa.co`.

## Cosas que valdría la pena agregar más adelante

- **Open Graph image (`og:image`)** — un PNG 1200×630 con el lockup Quepa para previews en WhatsApp/Twitter/LinkedIn.
- **Google Tag Manager / Plausible** para analítica.
- **Sitemap.xml** y **robots.txt** para SEO.
- **Hosteo de fuentes propio** (Hanken Grotesk + JetBrains Mono con `@font-face`) para no depender de Google Fonts CDN.

## Versión

- Landing principal (Canchas): v2.0 · sep 2026
- Landing B2C archivada: v1.0 · mayo 2026
