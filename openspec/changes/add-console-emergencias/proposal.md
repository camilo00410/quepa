# Proposal: add-console-emergencias

## Why

El carril de emergencia está completo **salvo la superficie de administración**: el backend (`add-emergency-mode` en quepa-webhook, migraciones `0016` y `0018`) está desplegado por el terremoto de Pereira, el agente responde con `search_help_points`, y el catálogo ya vive — **46 recursos del área metropolitana cargados a mano** con SQL/REST y service role. Dejar el Console fuera fue una decisión explícita de alcance de la v1 (D10 de ese change: "carga y re-confirmación SQL/REST asistida, sin Console"). Esta es la entrega que la salda.

La matemática de la operación es la que manda: `verified_at` degrada a "envejecido" a las **24 horas** y "vencido" a las **48** (`resource-freshness.ts`), y el bot dice la vejez en voz alta junto a cada dato. Con 46 recursos, mantener el catálogo verde son **decenas de re-confirmaciones diarias que hoy exigen SQL con service role** — nadie del equipo de analistas puede confirmar un albergue, corregir un horario ni dar de baja un punto de acopio cerrado. El caso de uso #1 no es capturar el recurso #47: es re-confirmar los 46 que ya existen, todos los días, mientras dure la emergencia.

## What Changes

**Dos páginas nuevas**, espejo del par `eventos.html` / `evento.html`:

- `console/emergencias.html` — la lista, concebida como **cola de verificación**.
- `console/emergencia.html?id=<uuid>|new` — ficha de captura y edición.
- Ítem "Emergencias" en el sidebar de todas las páginas de `console/` que lo tienen, y tarjeta en `welcome.html` (el sidebar está duplicado por archivo; se actualizan todas).

**La cola de verificación es el concepto central de la lista.** El orden por defecto es por vejez de `verified_at` (lo más vencido arriba), con semáforo visual — 🟢 < 24 h · 🟡 24–48 h · 🔴 > 48 h — **espejo de los umbrales de `quepa-webhook/src/lib/resource-freshness.ts`**, duplicados a conciencia con comentario de procedencia (mismo trato que `normalizeTagSlug` en eventos: si allá cambian, acá también, y no hay constraint que atrape la divergencia).

- **"✓ Confirmar" es la acción primaria y cuesta un clic**: sella `verified_at = now()` desde la fila, sin abrir la ficha. El trigger del change hermano registra quién y cuándo — el front no loguea nada.
- Filtros: tipo (los 5), estado (activos por defecto · inactivos · todos), ciudad y búsqueda por nombre. Columnas: nombre, tipo, dirección/zona, semáforo con hora de última verificación, "Actualizado por" y acciones (confirmar · editar · desactivar).
- **Contador contra el tope editorial**: el agente entrega máximo **60 recursos por área** (`RESOURCE_CAP`, recalibrado tres veces; el área de Pereira va en 46) y al desbordar amputa primero `entidad` y `linea`. La lista muestra `46/60` para que la carga masiva no se coma el cap sin que nadie lo vea.

**Ficha: los campos de `emergency_resources`, sin inventar ninguno.**

- **Tipo: los 5 del enum cerrado como chips** — `acopio · albergue · atencion · entidad · linea` — en ese orden, que es el orden de urgencia editorial con que el agente entrega. **No se expande el enum** (decisión de exploración): "alimentación", "agua", "atención veterinaria" son recursos `atencion` con la granularidad en `receiving[]` y la descripción.
- `receiving[]` / `not_receiving[]` como listas de chips de texto (qué reciben un acopio, qué ofrecen un albergue); `contact_phones[]` múltiple; `maps_url` pegado tal cual; `how_to_access`, `source`, `notes`, `zone`, `lat`/`lng` opcionales.
- **`hours_text` es texto humano libre, y así se queda**: sin aritmética de apertura ni estructura tipo `places.hours` — doctrina de `0016`: este dato cambia a diario y la falsa precisión de un "abierto ahora" calculado es peor que el texto honesto.
- **"Capacidad completa" / "temporalmente cerrado" se anotan en `notes`** (decisión de exploración): no hay columna de estado operativo y no se crea. `search_help_points` ya entrega `notes` al agente, que las dice — un albergue lleno *mencionado con advertencia* es mejor que uno desaparecido, porque la gente igual pregunta por él.
- **Ciudad: selector anclado a las grafías existentes** (misma unión `places.city` ∪ `city_zones.city` del precedente de eventos, con escape "otra ciudad…" + aviso visible). El modo de falla es idéntico y silencioso: los recursos se recuperan por SQL puro contra `area_cities` de la emergencia — un typo en la ciudad saca el recurso del área sin síntoma. El recurso conserva su **ciudad física**; la cobertura la declara la emergencia, nunca se re-etiqueta el recurso (doctrina de `0018`).

**Baja: desactivar (`active=false`), nunca borrar** — doctrina de eventos, no de lugares (`lugares.html` sí borra físicamente; este módulo no hereda eso). Un recurso viejo **jamás se oculta solo**: la baja es un juicio humano, el dato vencido se muestra con su edad. Reactivar es simétrico.

**Trazabilidad sin escribirla desde el front.** Las columnas "Actualizado por" / "Verificado por" se leen de la vista **`emergency_resource_activity`** que crea el change hermano `add-emergency-audit-log` (quepa-webhook, migración `0019`): el trigger audita todo INSERT/UPDATE con el email del staff sellado desde el JWT; las escrituras de service role figuran como `sistema/manual`. El Console solo lee.

**No se toca el esquema desde este change.** La migración vive en el repo webhook. Tampoco se toca el carril del agente: ni tool, ni prompt, ni enum.

**Fuera de alcance (explícito):**

- **Administrar `city_emergencies`** (activar/apagar la emergencia, editar `notice_text`/`narrative`/`area_cities`): sigue siendo acto de admin por SQL. El módulo la **lee** para el encabezado de contexto ("EMERGENCIA · PEREIRA · terremoto") y para saber qué área contar contra el cap.
- **Expandir el enum de tipos** o crear columna de estado operativo.
- **Uploader de imágenes**, igual que en eventos.
- **Roles o permisos**: los analistas usan las cuentas de staff existentes del Console (decisión de exploración); todo autenticado administra.
- **RLS sobre tablas existentes**: la única RLS nueva es la de la tabla de historial y vive en el change del webhook.
- **Embeddings**: los recursos no llevan por diseño (`0016`) — propagación cero, una fila corregida es visible en la consulta siguiente. No hay `embed:sync` que correr.

## Capabilities

### New Capabilities

- `console-emergency-resources`: administración de `emergency_resources` en el Quepa Console — cola de verificación ordenada por vejez con semáforo 24/48 h y confirmación de un clic, ficha con los 5 tipos cerrados y estado operativo en notas, baja por desactivación, ciudad anclada a grafías existentes, contador contra el cap editorial de 60, y trazabilidad leída de la vista de auditoría del backend.

### Modified Capabilities

<!-- Ninguna: lugares, sedes, zonas, precios, redes y eventos no cambian de requisitos. -->

## Impact

- **Código**: `console/emergencias.html` y `console/emergencia.html` (nuevos); ítem de navegación en las páginas existentes con sidebar (`lugares.html`, `lugar.html`, `mapa.html`, `descubrir.html`, `eventos.html`, `evento.html`) y tarjeta en `welcome.html`.
- **Datos**: lectura y escritura sobre `emergency_resources` (insert/update; jamás delete). Lectura de `emergency_resource_activity`, `city_emergencies`, `places.city` y `city_zones.city`. Ninguna escritura fuera de `emergency_resources`.
- **Dependencia de despliegue**: la migración `0019_emergency_audit.sql` del change `add-emergency-audit-log` (quepa-webhook) debe estar aplicada **antes** de publicar esta versión del Console — mismo patrón que la dependencia de sedes en `consolidate-location-into-sedes`. Sin ella, la vista no existe y las columnas de trazabilidad fallarían.
- **Riesgo de duplicación de reglas**: los umbrales 24/48 h viven con el código del bot en `resource-freshness.ts`; el Console los duplica como constantes comentadas. La clasificación se hace contra el reloj del cliente en América/Bogotá — una divergencia de minutos con el `verified_display` server-side es aceptable y del mismo carácter que la diferencia ya documentada del módulo de eventos.
- **Calendario**: la emergencia está activa **hoy**; cada día sin módulo son ~46 re-confirmaciones por SQL. La cola de verificación (lista + confirmar + desactivar) es lo que desbloquea al equipo; la ficha de alta puede llegar horas después sin bloquear a nadie.
- **Docs**: sección nueva en `quepa-landing/CLAUDE.md` (contrato del módulo, espejo de las secciones de sedes/eventos).
