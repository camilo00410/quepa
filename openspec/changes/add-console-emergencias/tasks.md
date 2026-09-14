# Tasks — add-console-emergencias

## 1. Lista · `console/emergencias.html` (la cola de verificación — desbloquea al equipo)

- [ ] 1.1 Scaffold de la página sobre el patrón de `eventos.html`: guard `ensureSession()`, sidebar, tipografía Manrope + JetBrains Mono, tokens de marca
- [ ] 1.2 Query a `emergency_resources` con orden `verified_at asc` y filtro por defecto `active = true`; render de columnas nombre · tipo · dirección/zona · semáforo · "Actualizado por" · acciones
- [ ] 1.3 Semáforo de frescura en cliente: constantes espejo `AGING_HOURS = 24` / `STALE_HOURS = 48` con comentario de procedencia (`quepa-webhook/src/lib/resource-freshness.ts`), formato de hora América/Bogotá estilo eventos
- [ ] 1.4 Acción "✓ Confirmar" por fila: `update({ verified_at })` y nada más, sin diálogo, refresco de la fila en sitio sin reordenar la lista
- [ ] 1.5 Filtros: tipo (chips de los 5), estado (activos · inactivos · todos), ciudad, búsqueda por nombre — estilo chips de `lugares.html`/`eventos.html`
- [ ] 1.6 Trazabilidad: query batcheada `.in('resource_id', ids)` a `emergency_resource_activity` sobre las filas visibles; degradación a "—" + aviso discreto si la vista falla, sin romper la cola
- [ ] 1.7 Encabezado de contexto leyendo `city_emergencies` activa (ciudad, kind) y contador `N/60` de activos del área (`area_cities`) con aviso de cercanía y consecuencia al desbordar (amputa `entidad` y `linea`); oculto sin emergencia activa
- [ ] 1.8 Desactivar/reactivar desde la fila con confirmación que nombra el recurso (`active = false`/`true`; ningún control de DELETE en todo el módulo)

## 2. Ficha · `console/emergencia.html?id=<uuid>|new`

- [ ] 2.1 Scaffold de ficha sobre el patrón de `evento.html`: carga por `id`, modo `new`, guardado por `id`
- [ ] 2.2 Formulario con los campos del schema: nombre, tipo (chips de los 5 en orden editorial), dirección, zona, `hours_text` (texto libre, sin estructura), `receiving[]`/`not_receiving[]` (chips de texto), `contact_phones[]` (múltiple), `contact_url`, `maps_url`, `how_to_access`, `source`, `notes`, `lat`/`lng`
- [ ] 2.3 Selector de ciudad: unión `places.city` ∪ `city_zones.city` con las ciudades del `area_cities` de la emergencia activa primero; escape "otra ciudad…" con aviso visible de invisibilidad por typo
- [ ] 2.4 Chips de anotación rápida "capacidad completa" / "temporalmente cerrado" que insertan frase estándar al inicio de `notes` (texto plano editable, sin taxonomía nueva)
- [ ] 2.5 Guardado: insert/update sobre `emergency_resources`; traducción del conflicto `unique (name, city, type)` a mensaje inline claro; "Verificado por" en ficha desde la vista de actividad
- [ ] 2.6 Desactivar/reactivar desde la ficha, simétrico y con confirmación nominal

## 3. Navegación y documentación

- [ ] 3.1 Ítem "Emergencias" en el sidebar de `lugares.html`, `lugar.html`, `mapa.html`, `descubrir.html`, `eventos.html` y `evento.html`; tarjeta `.mod` en `welcome.html`
- [ ] 3.2 Sección nueva en `quepa-landing/CLAUDE.md` con el contrato del módulo (espejo de las secciones de sedes/eventos): constantes espejo 24/48, "confirmar solo escribe `verified_at`", nunca DELETE, `notes` como casa del estado operativo, dependencia de la migración `0019`

## 4. Verificación

- [ ] 4.1 Confirmar que la migración `0019_emergency_audit.sql` (change `add-emergency-audit-log` de quepa-webhook) está aplicada en Supabase ANTES de publicar — la vista `emergency_resource_activity` debe responder
- [ ] 4.2 Smoke test con cuenta de staff real: confirmar un recurso (semáforo verde, `changed_fields = {verified_at}` en el historial), editarlo, desactivarlo y reactivarlo, crear uno nuevo y verlo alcanzable por el bot
- [ ] 4.3 Probar degradación: simular fallo de la vista (nombre errado en local) y verificar que la cola sigue operable con "—" en trazabilidad
- [ ] 4.4 Probar en browser desktop y mobile (convención del repo para cambios visuales); ráfaga de confirmaciones seguidas sin que la lista se reordene bajo el cursor
