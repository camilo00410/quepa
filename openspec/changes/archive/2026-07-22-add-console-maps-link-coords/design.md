# Design: add-console-maps-link-coords

## Context

`console/lugar.html` captura coordenadas en dos puntos: `f-lat`/`f-lng` (sección 03, espejo de la sede principal, alimentan `renderMap()`) y las celdas Lat/Lng de la grid de sedes (`sede.lat/lng`, tipo number). Ambos son inputs de float crudo que el staff no llena (0/544 en la curación de Pereira). El guardado ya funciona (espejo mono-sede / grid multi-sede, contrato `console-place-locations`) y el mapa ya distingue pin exacto vs pin `≈` de centroide de zona (`add-console-zone-selector`). Las URLs de Google Maps de escritorio contienen las coordenadas del viewport/lugar en formatos conocidos. Convención dura del repo: autocontenido, sin dependencias, sin llamadas a servicios externos.

## Goals / Non-Goals

**Goals:**

- Que capturar coordenadas sea "pega el link de Maps" — una acción que el staff ya hace al verificar direcciones.
- Sacar los floats de la vista sin sacarlos del sistema (siguen siendo el dato guardado y la válvula de escape).
- Feedback inmediato: el pin del mapa se mueve al pegar; error honesto si la URL no trae coordenadas.

**Non-Goals:**

- Resolver links cortos `maps.app.goo.gl` (exige fetch externo — fuera por autocontención; se pide la URL completa).
- Geocoding de direcciones, autollenado vía APIs, cambios de schema o del webhook.
- Persistir la URL pegada (para eso existe `google_maps` en redes sociales).

## Decisions

### D1 — Extracción por regex local, en orden de confianza

Parser puro (función única, compartida por 03 y la grid) que intenta, en orden: `!3d<lat>!4d<lng>` (el pin exacto del lugar en URLs de ficha), `@<lat>,<lng>` (centro del viewport), `q=<lat>,<lng>` y `ll=<lat>,<lng>` (formatos legacy). El primero que matchee gana — `!3d/!4d` va primero porque `@` es el centro de la cámara, no el pin, y pueden diferir. Números validados como float con 3+ decimales para descartar zooms y basura. Alternativa descartada: resolver el link corto siguiendo el redirect — rompe la convención de cero llamadas externas y mete CORS.

### D2 — El campo URL es efímero; los floats siguen siendo el dato

El input de URL no se guarda ni se repuebla al cargar: es un "conducto" que al pegar escribe `lat`/`lng` en el estado existente y se limpia solo (dejando la nota "coordenadas capturadas ✓"). Los flujos de guardado no cambian en una línea. Alternativa descartada: persistir la URL — no hay columna, duplicaría `place_social_links.google_maps` y convertiría un mecanismo de UI en dato.

### D3 — Floats colapsados como "avanzado", editables (válvula de escape)

Sección 03 y grid muestran el campo URL como vía primaria; los inputs de lat/lng quedan dentro de un `<details>`/toggle colapsado con el valor actual resumido (`4.79906, -75.80593` o "sin coordenadas"). Siguen editables: limpiar coordenadas, pegar valores que llegan por otra vía, o corregir a mano son casos reales. Alternativa descartada: solo-lectura total — obligaría a inventar acciones aparte para limpiar/corregir.

### D4 — Validación de rango con aviso, no bloqueo

Sanity check Colombia aproximado (lat -5…14, lng -82…-66): fuera de rango → aviso visible ("esas coordenadas caen fuera de Colombia — ¿pegaste la URL correcta?") pero el valor se acepta (hay lugares limítrofes y el staff manda). URL sin coordenadas extraíbles → nota honesta pidiendo la URL completa del navegador, estado intacto.

### D5 — Mono/multi sede: mismo conducto, mismos destinos de siempre

En mono-sede el campo URL de 03 escribe `f-lat`/`f-lng` (y el guardado espejo hace lo suyo); en la grid cada sede tiene su campo URL que escribe `sede.lat/lng` (+ `syncReflejo` si es la principal). En multi-sede los campos de 03 siguen bloqueados como espejo — el URL de 03 se deshabilita con la nota existente de "edita en SEDES".

## Risks / Trade-offs

- [Google cambia el formato de URL] → los 4 patrones cubren los formatos estables de años; si uno muere, el parser degrada a "no encontré coordenadas" (honesto, nunca datos malos) y se agrega el patrón nuevo.
- [Staff pega el link corto del celular] → nota explícita con el cómo ("ábrelo en el navegador y copia la URL completa"); es fricción de un toque.
- [El `@` del viewport puede no ser el pin exacto si el staff movió el mapa] → `!3d/!4d` tiene prioridad; el pin del mapa del Console da verificación visual inmediata para atrapar el resto.
- [Un `<details>` colapsado esconde un dato que a veces importa] → el resumen del valor actual queda visible en el summary, no hay que abrir para saber si hay coordenadas.

## Migration Plan

1. Deploy estático normal (Vercel). Sin dependencias de orden: no toca datos ni backend.
2. Smoke: pegar URL de ficha (`!3d/!4d`), URL de viewport (`@`), link corto (nota honesta), URL basura (nota), verificar pin del mapa y guardado mono y multi-sede.

Rollback: revertir el deploy estático.

## Open Questions

- ¿El aviso de "capturadas ✓" debería mostrar también la distancia al centroide de la zona seleccionada como sanity check ("a 12 km de Cuba — ¿seguro?")? Barato con los centroides ya cargados; se decide en implementación viendo el ruido real.
