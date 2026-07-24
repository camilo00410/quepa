# Design: add-console-price-ranges

## Context

`console/lugar.html` es autocontenido (CSS/JS inline, Supabase directo) y su sección `· 02 ·` captura hoy solo el bucket `price_range` 1–4 vía picker. El backend agrega `places.price_min`/`price_max` (COP, nullable, check `min <= max`) en la migración `0012` del change hermano. La grid de SEDES y las redes ya establecieron el patrón: validación honesta, avisos no bloqueantes, y `gather()` como único punto de armado del update.

## Goals / Non-Goals

**Goals:**
- Capturar desde/hasta en COP con la menor fricción posible para el equipo de datos.
- Espejar en UI el check de DB para que el guardado nunca se estrelle contra Supabase por rango invertido.
- Dejar clarísima la unidad implícita (hotel por noche / resto por persona) en el punto de captura.

**Non-Goals:**
- Derivar el bucket desde el rango (sigue manual).
- Mostrar el rango en `lugares.html` o filtrar por él.
- Validar magnitudes contra la realidad (eso es curaduría, no UI).

## Decisions

1. **Dos inputs texto con formato de miles, no `type=number`.** Al blur se formatea "35.000"; al guardar se parsea a entero limpio (regex de dígitos). `type=number` pelea con el separador es-CO y con el cero a la izquierda; el patrón texto+parse ya se usa en coordenadas.

2. **Validación dura solo para `desde > hasta`.** Bloquea el guardado con error inline en la sección (patrón `sedes-error`). Razón: la DB lo rechazaría igual; mejor un mensaje claro que un error críptico de PostgREST. Un solo extremo (solo desde o solo hasta) es válido y se guarda.

3. **Aviso no bloqueante de magnitud.** Valor entre 1 y 999 dispara nota "¿va en miles? 35 se guarda como $35, no $35.000" pero deja guardar — criterio idéntico a coordenadas fuera del rango de Colombia. No se auto-multiplica nunca (el Console no corrige datos, los señala).

4. **La unidad no se captura: se comunica.** Un `help` bajo los campos fija "hotel = por noche · resto = por persona". Si la categoría es `hotel`, el help lo refleja dinámicamente (leer `state`/select de categoría ya presente en la página).

5. **`gather()` es el único punto de escritura**, igual que el resto de campos planos de `places`: agrega `price_min`/`price_max` al objeto del update. Hidratación en la misma función que hoy setea `state.price_range`.

## Risks / Trade-offs

- [Console publicado antes de la migración 0012] → el update falla por columna inexistente. Mitigación: dependencia dura documentada en proposal y CLAUDE.md; mismo protocolo de orden que `consolidate-location-into-sedes`.
- [El equipo digita 35 en vez de 35.000] → aviso no bloqueante en captura + el dato es visible/corregible; el webhook nunca lo "arregla".
- [Confusión bucket vs rango] → ambos campos conviven en la misma sección con el help de unidad; el spec del webhook fija que el display del bot prefiere el rango.

## Migration Plan

1. Esperar confirmación de que la migración `0012` está aplicada (change hermano).
2. Publicar `lugar.html` con la captura.
3. El equipo llena rangos al ritmo del census (sin backfill masivo).

Rollback: revertir `lugar.html`; los valores ya guardados quedan en la base sin consumidores rotos.
