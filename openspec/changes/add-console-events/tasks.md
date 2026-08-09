# Tasks: add-console-events

Orden pensado para que C2 (administrar los 196 ya cargados) quede utilizable antes que C1 (captura nueva): la lista con series se puede probar contra datos reales desde el grupo 3, y el enriquecimiento masivo — que es lo que desbloquea los 0 flyers — llega en el grupo 5.

## 1. Andamiaje y navegación

- [x] 1.1 Crear `console/eventos.html` a partir del esqueleto de `console/lugares.html`: mismo `<head>` (Manrope + JetBrains Mono, favicon), mismo cliente Supabase con `storageKey: 'quepa-console-auth'`, misma guarda de sesión y mismo layout de sidebar + topbar
- [x] 1.2 Crear `console/evento.html` a partir del esqueleto de `console/lugar.html`, con soporte de `?id=<uuid>` y `?id=new`
- [x] 1.3 Agregar el ítem "Eventos" (con ícono de calendario). **Corregido contra el código real**: solo 4 páginas tienen sidebar (`lugares.html`, `lugar.html`, `mapa.html`, `descubrir.html`) — ahí va el `nav-item`; `welcome.html` usa tarjetas de módulo (`.mod`) y recibe una tarjeta; `index.html` es el login y no lleva navegación. Activo marcado en `eventos.html` y `evento.html`

## 2. Lectura y agrupación en series

- [x] 2.1 Carga de eventos desde `events` con sus etiquetas (query a `event_tag_links` con join a `event_tags`, batcheada sobre la página visible — mismo patrón que el chip "+N sedes" de `lugares.html`)
- [x] 2.2 Función pura de agrupación por clave `(name, city, start_time)`: devuelve series con sus `id`s, conteo de fechas, rango de fechas y flag de puntual. Verificar contra los datos reales: 196 filas deben colapsar a 72 renglones (52 puntuales + 20 series)
- [x] 2.3 Clasificación temporal por día calendario en América/Bogotá (`Intl.DateTimeFormat` con `timeZone`, mismo patrón que `bogotaClock`): Próximos · Hoy · Vencidos · Todos. **No portar** `isEventVigente` ni la lógica de cruce de medianoche (D5)
- [x] 2.4 Orden por `event_date` y luego `start_time` dentro de cada agrupación

## 3. UI de la lista

- [x] 3.1 Renglón de serie colapsado: nombre + hora de inicio + conteo de fechas + rango ("6 ago → 15 sep"), con control de expansión a las fechas
- [x] 3.2 Renglón de evento puntual: nombre + fecha + hora, sin control de expansión
- [x] 3.3 Fecha y hora de inicio visibles en TODO renglón y en los resultados de búsqueda (D-riesgo: 41 filas se llaman igual)
- [x] 3.4 Tabs de filtro temporal con conteos, estilo `.tab`/`.pill` de `lugares.html`; distinción visual de eventos inactivos
- [x] 3.5 Búsqueda por nombre/organizador que muestra fecha y hora en cada resultado
- [x] 3.6 Botón "+ Nuevo evento" → `evento.html?id=new`

## 4. Ficha: campos y validación

- [x] 4.1 Markup de secciones numeradas al estilo `lugar.html` (identidad · cuándo · dónde · contenido · contacto)
- [x] 4.2 Obligatorios con validación pre-save que bloquea antes de llamar a Supabase: `name`, `description`, `address`, `city`, `event_date`, `start_time`, y al menos una etiqueta
- [x] 4.3 Hora de fin **opcional**: sin `required`, vacío se muestra como "sin hora de cierre registrada", y `end_time <= start_time` NO es error — indicar "termina al día siguiente" (D7)
- [x] 4.4 Selector de ciudad poblado de la unión deduplicada de `places.city` y `city_zones.city` (dos queries al cargar), con opción "otra ciudad…" que revela texto libre + aviso visible del riesgo de invisibilidad (D4)
- [x] 4.5 Campo de etiquetas: reutilizar el componente `tag-input`/`renderTags`/`wireTagInputs` de `lugar.html` como base, pero con autocompletado contra `event_tags` y catálogo completo visible al abrir
- [x] 4.6 Copiar `normalizeTagSlug` verbatim desde `quepa-webhook/src/db/events.ts` con comentario apuntando al original; reutilizar por slug y crear en `event_tags` solo si no existe (D9)
- [x] 4.7 Opcionales: `maps_url`, `organizer`, `notes`, `ticket_url`, `lat`/`lng`; vacíos se persisten como `null`, nunca como cadena vacía
- [x] 4.8 `contact_phones` en formato de etiquetas (varios por evento), persistido como `text[]`
- [x] 4.9 Campo de flyer: validación de esquema HTTPS que bloquea el guardado, más previsualización de la imagen (D6)
- [x] 4.10 `priority` 0–2 con ayuda que explique que selecciona pero nunca reordena; **resolver el rótulo sin usar el lenguaje de "destacado"** (Open Question del design)

## 5. Alcance de escritura y guardado

- [x] 5.1 El alcance se fija al abrir: entrada por renglón de serie → alcance serie; entrada por fecha expandida → alcance fecha (D3)
- [x] 5.2 Indicador permanente del alcance vigente en la ficha, conmutable antes de guardar
- [x] 5.3 El botón de guardar declara el número de fechas que va a escribir, y el conteo se actualiza al conmutar el alcance
- [x] 5.4 `save()` con alcance serie: update sobre la lista de `id`s de la serie, tocando `updated_at`; con alcance fecha: update sobre un solo `id`. Toda escritura se resuelve por `id`, nunca por `name`
- [x] 5.5 Sincronización de `event_tag_links` al guardar (agregar los nuevos vínculos, quitar los removidos), respetando el alcance
- [x] 5.6 Alta de evento nuevo: insert en `events` + etiquetas, con manejo honesto del error de `unique (name, city, event_date, start_time)` si ya existe

## 6. Baja y reactivación

- [x] 6.1 Desactivar por fecha: `active = false` sobre un `id`, sin `delete` (D8)
- [x] 6.2 Desactivar la serie completa: `active = false` sobre todos los `id`s, con confirmación que nombra el conteo de fechas
- [x] 6.3 Reactivación simétrica en ambos alcances

## 7. Verificación y cierre

- [ ] 7.1 Prueba contra datos reales: abrir "La Ruta de la Cosecha" (41 fechas), poner un `flyer_url` con alcance serie y verificar que quedan las 41 filas y ninguna otra. **Parcial**: el conjunto de escritura ya se verificó en solo-lectura contra producción (la query de serie devuelve exactamente esas 41 de 196); falta la escritura real, que toca datos productivos
- [x] 7.2 Prueba de las dos series que comparten nombre a horas distintas (Unicentro Pereira 13:00 vs. 16:00; Sra. Pereira 17:00 vs. 19:00): confirmar que se listan y se editan por separado
- [ ] 7.3 Prueba de los 17 eventos sin `end_time`: abrir uno, editar otro campo y guardar sin diligenciar hora de fin. **Parcial**: confirmados los 17 en producción y que la ficha no los exige; falta el guardado real
- [ ] 7.4 Prueba de cruce de medianoche: capturar 21:00–03:00 y confirmar que guarda sin error. **Parcial**: la clasificación del Console coincide con la convención del webhook en los 196 eventos reales (25 ya cruzan medianoche); falta el guardado real
- [ ] 7.5 Prueba de etiquetas: escribir `"MUSICA / CONCIERTO  "` (variante de mayúsculas y espacios de la etiqueta existente) y confirmar que reutiliza `musica concierto` sin crear fila en `event_tags`. Ojo: `"MUSICA "` a secas normaliza a `musica`, que **no** es esa etiqueta y **sí** crea una nueva — correcto por diseño; contra ese caso la defensa es el catálogo a la vista, no la normalización (D9)
- [ ] 7.6 Prueba de baja: desactivar una fecha de una serie y confirmar en el agente que ese día ya no se ofrece y los demás sí
- [ ] 7.7 Verificar en browser desktop y mobile (convención del repo para cambios visuales)
- [x] 7.8 **Corregir o eliminar `openspec/changes/NOTA-eventos-console.md`**: afirma que `end_time` es obligatoria, lo cual contradice la migración `0015` y es la trampa más cara del change
- [x] 7.9 Actualizar `quepa-landing/CLAUDE.md` (sección Console) con el contrato del módulo: serie derivada y su clave, alcance de escritura, `end_time` opcional, ciudad por unión de catálogos, etiquetas por slug normalizado, flyer por URL y baja por desactivación
