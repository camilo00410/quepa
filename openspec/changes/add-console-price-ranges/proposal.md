# Proposal: add-console-price-ranges

## Why

El census del equipo ya recolecta "Rango de precios desde/hasta" en COP, y el backend (change hermano `add-price-ranges` en quepa-webhook, migración `0012`) crea `places.price_min`/`price_max` para guardarlo. El Console es la única superficie de edición del catálogo — sin campos de captura, el dato rico se queda en la spreadsheet.

## What Changes

- La sección `· 02 · ECONÓMICO Y SERVICIOS` de `console/lugar.html` gana dos campos "Desde / Hasta" en COP junto al picker $–$$$$ (que queda intacto: bucket manual, decisión de producto 2026-07-23).
- Formato con separador de miles al escribir/blur; se persiste el entero limpio.
- Validación dura espejo del check de DB: `desde > hasta` bloquea el guardado con error inline honesto (la base lo rechazaría de todos modos).
- Aviso **no bloqueante** de magnitud (valor entre 1 y 999 → "¿seguro? parece en miles"), mismo criterio que coordenadas fuera de Colombia.
- Texto de ayuda con la unidad implícita: hotel = por noche; el resto = por persona (para que el equipo capture consistente).
- `gather()` incluye `price_min`/`price_max`; la carga hidrata ambos campos.
- Fuera de alcance: mostrar el rango en la tabla de `lugares.html` (sigue la celda "$$") y cualquier cambio al filtro por bucket.

## Capabilities

### New Capabilities
- `console-price-ranges`: captura y edición del rango de precios COP por lugar en el detalle del Console — campos desde/hasta, validación espejo del check DB, aviso de magnitud no bloqueante y convivencia con el bucket manual.

### Modified Capabilities

<!-- Ninguna: picker, filtros y demás secciones del Console no cambian de requisitos. -->

## Impact

- **Código**: solo `console/lugar.html` (CSS de los campos, markup en · 02 ·, `state`, hidratación, `gather()`, validación).
- **Dependencia dura**: la migración `0012` del webhook debe estar aplicada ANTES de publicar esta versión del Console (si no, el update a Supabase falla por columnas inexistentes). Mismo patrón de orden que `consolidate-location-into-sedes`.
- **Docs**: nota en `quepa-landing/CLAUDE.md` (sección Console) con el contrato de captura y la unidad implícita.
