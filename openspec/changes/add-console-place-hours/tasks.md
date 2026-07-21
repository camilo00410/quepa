# Tasks: add-console-place-hours

Todo el trabajo vive en `console/lugar.html` (autocontenido: HTML + CSS + JS vanilla). Referencias: `design.md` (D1–D7) y `specs/console-place-hours/spec.md`.

## 1. Estructura y estilos de la sección

- [x] 1.1 Verificar que el `lugar.html` local corresponde al desplegado en `console.quepa.co` (`git status`/`git log` en `quepa-landing`); si divergen, resolver antes de editar
- [x] 1.2 Agregar la sección `· 05 · HORARIOS` en `.detail-form` después de `· 04 · CONTACTO Y RESERVAS`, con cabecera que incluye chip de fuente y "verificado hace X" (D6)
- [x] 1.3 Maquetar la grilla semanal: fila por día `LUN..DOM`, franjas como pares de inputs de texto `HH:MM` en fuente mono + botón quitar, botón "+ franja" por día, estado "Cerrado" para días sin franjas, indicador visual de cruce de medianoche y presentación "Abierto 24 horas" (D1)
- [x] 1.4 Maquetar el estado null: "· Sin horario registrado" + botón "Registrar horario" que despliega la grilla vacía (D2)
- [x] 1.5 Agregar el campo de nota (`hours_note`) con input de texto y `help` explicando que captura excepciones ("Domingos y festivos cerrado")
- [x] 1.6 Estilos con los patrones y tokens existentes del archivo (`.section`, `.field-d`, mono, `--ink-*`); revisar el breakpoint ≤960px para que la grilla no desborde

## 2. Estado y renderizado

- [x] 2.1 Extender `state` con el modelo de horarios editable (`hoursGrid` por día `mon..sun`, `hoursNote`, `hoursSource`, `hoursVerifiedAt`) y los flags `hoursDirty` / `noteDirty` (D4)
- [x] 2.2 En `loadPlace()`, poblar el estado desde `data.hours`/`hours_note`/`hours_source`/`hours_verified_at` tolerando columnas ausentes (base sin `0007` → estado null) y render inicial de grilla + cabecera de procedencia con `relTime()`
- [x] 2.3 Wire de la grilla: agregar/quitar franja y editar horas actualizan el estado, marcan `hoursDirty` y re-renderizan (incluido el indicador de medianoche en vivo); editar la nota marca `noteDirty`
- [x] 2.4 En modo `isNew`, iniciar la sección en estado null igual que un lugar sin horario

## 3. Validación y serialización

- [x] 3.1 Implementar la validación de guardado (D5): regex `HH:MM` con `24:00` solo como cierre, apertura ≠ cierre, solapamientos por día (cruce de medianoche válido); retorna el primer error con día y franja para el toast
- [x] 3.2 Implementar el serializador canónico (D3): 7 claves `mon..sun` en orden fijo, franjas ordenadas por apertura, `[]` para días cerrados; grilla completamente vacía → `null` (D2)

## 4. Guardado

- [x] 4.1 Integrar en `gather()`/`save()` según los flags: `hoursDirty` → `hours` + `hours_note` + `hours_source='manual'` + `hours_verified_at=now()` (o los tres en null si la grilla quedó vacía); solo `noteDirty` → únicamente `hours_note`; sin flags → payload sin columnas de horarios (D4)
- [x] 4.2 Ejecutar la validación 3.1 dentro de `save()` antes del update/insert; si falla, abortar con el toast de error existente sin escribir
- [x] 4.3 Tras guardado exitoso con `hoursDirty`, actualizar la cabecera a `MANUAL · hace un momento` y limpiar los flags; el insert de lugar nuevo aplica las mismas reglas (D6)
- [x] 4.4 Confirmar que un fallo del update/insert (p. ej. columnas de `0007` inexistentes) cae en el toast de error sin toast de éxito ni cambio de cabecera (D7)

## 5. Verificación manual

- [x] 5.1 Servir local (`python3 -m http.server 8000` desde `quepa-landing`, abrir `/console/lugar.html?id=<uuid>` con sesión staff) y probar contra un lugar con `hours` de Google: grilla correcta, chip `GOOGLE`, guardar sin tocar horarios no incluye columnas de horarios en el payload (verificar en la pestaña Network)
- [x] 5.2 Probar la edición: corregir una franja → guardar → recargar y confirmar en la base `hours_source='manual'` y `hours_verified_at` nuevos; caso nota-solo → únicamente `hours_note` cambió
- [x] 5.3 Probar los casos límite: franja `18:00–02:00` (indicador de medianoche, guarda bien), `00:00–24:00` (24 horas), formato inválido y solapamiento (bloquean con mensaje), vaciar toda la grilla (queda `hours=null` y "Sin horario registrado")
- [x] 5.4 Probar lugar nuevo con y sin horarios, y un lugar de los lotes sin horario (estado null estable)
- [x] 5.5 Probar responsive: la grilla en viewport ≤960px y el layout de una columna ≤1100px sin desbordes
