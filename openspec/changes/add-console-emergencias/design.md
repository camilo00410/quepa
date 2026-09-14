# Design — add-console-emergencias

## Context

El Console es HTML autocontenido que habla directo a Supabase (anon key + sesión de staff, guard `ensureSession()` por página); no hay backend intermedio ni build step. El precedente arquitectónico exacto es el módulo de eventos: par lista+ficha, tablas sin embeddings, propagación cero, baja por `active=false`. Del lado del backend, `emergency_resources` ya existe (migración `0016`), el agente ya la consume (`search_help_points`), la frescura ya tiene semántica server-side (`resource-freshness.ts`: 24 h envejecido, 48 h vencido, el bot dice la edad en voz alta) y el change hermano `add-emergency-audit-log` (quepa-webhook, migración `0019`) aporta el trigger de auditoría y la vista `emergency_resource_activity`.

La restricción operativa que ordena el diseño: la emergencia está activa, hay 46 recursos cargados, y la ventana de 24/48 h convierte la **re-confirmación** en la operación dominante — decenas de veces al día, a veces con varios analistas simultáneos.

## Goals / Non-Goals

**Goals:**

- Que confirmar un recurso cueste **un clic desde la lista** y quede auditado sin que el front escriba una sola línea de log.
- Que lo vencido sea imposible de no ver (orden por vejez + semáforo).
- Que el alta/edición respete el schema existente sin inventar campos ni taxonomías.
- Que el módulo degrade con honestidad si la vista de auditoría aún no existe.

**Non-Goals:**

- Administrar `city_emergencies` (solo lectura para contexto), expandir el enum de tipos, columna de estado operativo, uploader, roles, RLS de tablas existentes, embeddings. Todo declarado en el proposal.
- Reimplementar `verified_display` ni la lógica del bot: el Console clasifica con sus propias constantes espejo, asumiendo divergencias de minutos.

## Decisions

### D1 · Módulo propio, espejo de eventos — no extensión de `lugares.html`

Los recursos de emergencia no son lugares: sin marca→sedes, sin precios, sin Stars, sin embeddings, y con doctrina de baja **opuesta** (`lugares.html` borra físicamente; aquí jamás existe un control de DELETE). Un módulo propio mantiene las doctrinas separadas y hereda del par de eventos lo que sí aplica: escritura por `id`, desactivación con confirmación nominal, reactivación simétrica.

### D2 · La lista ES la cola de verificación

Orden por defecto `verified_at asc` (lo más vencido arriba); el semáforo se deriva en el cliente con constantes espejo de `resource-freshness.ts` (`AGING_HOURS = 24`, `STALE_HOURS = 48`), duplicadas con comentario de procedencia — mismo trato que `normalizeTagSlug` en eventos: si allá cambian, acá también, y ningún constraint atrapa la divergencia; la defensa es el comentario y la nota en `CLAUDE.md`. La hora se muestra en América/Bogotá con el estilo de eventos ("hoy 3:40 p.m." / "ayer" / fecha corta).

### D3 · "Confirmar" escribe SOLO `verified_at` — es contrato, no detalle

El botón de fila hace `update({ verified_at: now })` y nada más. Razón: el trigger del change hermano calcula `changed_fields`, y la vista cuenta como verificación los updates cuyo `changed_fields` contenga `verified_at`. Un "confirmar" que además bumpee otros campos contaminaría el historial y el "Verificado por". Sin diálogo de confirmación: la acción es segura, idempotente y su costo debe ser un clic (los diálogos se reservan para desactivar). La fila se refresca en sitio; no se recarga la lista completa (el orden por vejez movería la fila bajo el cursor del analista a mitad de ráfaga).

### D4 · Estado operativo: frases estándar que viven en `notes`, sin taxonomía oculta

"Capacidad completa" y "temporalmente cerrado" son anotaciones, no estados (decisión de exploración). La ficha ofrece **chips de anotación rápida** que insertan una frase estándar al inicio de `notes` ("CAPACIDAD COMPLETA — " / "CERRADO TEMPORALMENTE — "), editable como cualquier texto. Lo persistido es texto plano en `notes` — que `search_help_points` ya entrega y el bot ya dice — y quitarlo es borrar la frase. Alternativa rechazada: un pseudo-enum en el front que "parsee" notes — taxonomía escondida en texto libre, frágil y mentirosa.

### D5 · Ciudad por selector anclado, con las ciudades del área primero

Mismo selector-unión del precedente de eventos (`places.city` ∪ `city_zones.city`, escape "otra ciudad…" con aviso visible), porque el modo de falla es idéntico: la recuperación es SQL puro contra `area_cities` y un typo saca el recurso del área sin síntoma. Ajuste propio: las ciudades del `area_cities` de la emergencia activa se listan **primero** en el selector — son el caso del 100 % hoy. El recurso guarda su ciudad física; la cobertura es de la emergencia (doctrina `0018`), y este módulo jamás escribe `city_emergencies`.

### D6 · Contador de cap informativo, nunca bloqueante

La lista muestra `N/60` — recursos activos del área de la emergencia activa contra `RESOURCE_CAP` — con aviso al acercarse y, al desbordar, la consecuencia concreta: el agente amputa primero `entidad` y `linea`. Es información editorial, no validación: el Console señala, nunca bloquea (mismo criterio que el aviso "¿va en miles?" de precios).

### D7 · Trazabilidad: una query batcheada a la vista, con degradación honesta

"Actualizado por" / "Verificado por" salen de `emergency_resource_activity` con un solo `.in('resource_id', ids)` sobre la página visible (precedente: chip de sedes en `lugares.html`). Si la query a la vista falla — migración `0019` no aplicada, o cualquier error —, las columnas muestran "—" con un aviso discreto y **la cola sigue funcionando**: la trazabilidad degrada, la operación de emergencia no.

### D8 · Concurrencia: last-write-wins, con el historial como red

Varios analistas en ráfaga es el caso esperado. No se monta locking optimista: la escritura por `id` con last-write-wins es el comportamiento de todo el Console, y el trigger de auditoría conserva ambos lados de cualquier pisada — reconstruible, no prevenible. El refresco en sitio de D3 minimiza la ventana de mirar datos viejos.

## Risks / Trade-offs

- **[La vista no existe al publicar (orden de despliegue violado)]** → Dependencia declarada en proposal + tarea de verificación explícita antes de publicar; y D7 hace que aun así el módulo no se caiga.
- **[Divergencia de umbrales 24/48 con `resource-freshness.ts`]** → Comentario de procedencia junto a las constantes + nota en `CLAUDE.md`. Divergencia de minutos por reloj de cliente: aceptada, mismo carácter que la diferencia documentada del módulo de eventos.
- **[`unique (name, city, type)` al crear]** → El error de PostgREST es críptico; la ficha lo traduce a mensaje inline claro ("ya existe un `<tipo>` con ese nombre en esa ciudad") — precedente del check de rango de precios.
- **[Frases estándar en `notes` degeneran en variantes]** → Los chips insertan siempre la misma frase; pero `notes` es texto libre y el bot lo lee tal cual — la consistencia perfecta no es alcanzable ni necesaria, la frase estándar solo baja la fricción.
- **[Sin emergencia activa el contador no tiene área]** → El contador se oculta y el módulo sigue operable (los recursos existen independientes del interruptor); el encabezado de contexto simplemente no se muestra.

## Open Questions

Ninguna bloqueante. Diferida: administración de `city_emergencies` desde el Console (hoy acto de admin por SQL, a propósito).
