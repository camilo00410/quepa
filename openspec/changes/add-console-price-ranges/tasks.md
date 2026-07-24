# Tasks: add-console-price-ranges

## 1. UI de captura

- [x] 1.1 Markup en `· 02 ·` de `console/lugar.html`: fila "Rango en pesos" con inputs "Desde"/"Hasta" junto al picker (labels mono consistentes con la sección) + `help` de unidad implícita (hotel = por noche · resto = por persona, dinámico según categoría)
- [x] 1.2 CSS de los campos (reutilizar `.field-d`/inputs existentes; nota de aviso estilo `zone-note`) y error inline estilo `sedes-error`
- [x] 1.3 Formato de miles es-CO al blur y parseo a entero limpio (patrón texto+parse como coordenadas)

## 2. Estado y guardado

- [x] 2.1 `state.price_min`/`state.price_max`: hidratar al cargar junto a `state.price_range`
- [x] 2.2 `gather()`: incluir ambos campos en el update de `places`
- [x] 2.3 Validación dura pre-save: `desde > hasta` bloquea con error inline; un solo extremo es válido
- [x] 2.4 Aviso no bloqueante de magnitud (1–999) al blur, sin auto-corrección

## 3. Verificación y cierre

- [x] 3.1 Confirmar que la migración `0012` del webhook está aplicada antes de publicar (dependencia dura)
- [ ] 3.2 Prueba manual contra Supabase: guardar rango completo, solo-desde, rango invertido (bloqueado) e hidratación al recargar; verificar que el picker no se ve afectado
- [x] 3.3 Verificar en browser desktop y mobile (convención del repo para cambios visuales)
- [x] 3.4 Actualizar `quepa-landing/CLAUDE.md` (sección Console): contrato de captura de rangos, unidad implícita y orden de deploy respecto a `0012`
