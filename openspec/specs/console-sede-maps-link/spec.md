# console-sede-maps-link Specification

## Requirements

### Requirement: Link corto de Google Maps por sede

Cada fila de sede activa en la grid SHALL ofrecer un campo "Link de Maps" para el link corto de Google Maps (`maps.app.goo.gl/…` o URL de compartir) — el link que el bot comparte al cliente, rotulado para distinguirlo del campo de URL-para-coordenadas. Guardar SHALL persistirlo en `place_social_links` como `{platform: 'google_maps', url, place_location_id: <id de la sede>, active: true, is_primary: false}` — la primacía la da la sede, no la fila. Editar la URL SHALL actualizar la misma fila por `id`; vaciar el campo SHALL desactivar la fila (`active=false`), nunca borrarla. El Console MUST NOT intentar extraer coordenadas de este campo.

#### Scenario: Capturar link corto en una sede

- **WHEN** el staff pega `https://maps.app.goo.gl/AbC123` en el campo "Link de Maps" de la sede "Cuba" y guarda
- **THEN** existe una fila en `place_social_links` con `platform='google_maps'`, esa URL y `place_location_id` de la sede Cuba, y `lat`/`lng` de la sede no cambian

#### Scenario: Sede nueva con link en el mismo guardado

- **WHEN** el staff agrega una sede nueva con su link de Maps y guarda todo junto
- **THEN** la sede se inserta primero y la fila de `place_social_links` queda con el `place_location_id` de la sede recién creada

#### Scenario: Vaciar el campo desactiva

- **WHEN** el staff borra la URL del campo "Link de Maps" de una sede y guarda
- **THEN** la fila correspondiente queda `active=false` y sigue existiendo en la base

### Requirement: google_maps fuera del dropdown de redes de marca

El dropdown de plataformas de la sección de redes sociales MUST NOT ofrecer `google_maps` para filas nuevas — su casa es el módulo de sedes. Las filas `google_maps` existentes MUST NOT editarse desde la lista de redes de marca; se gestionan desde el módulo de sedes según su estado de asignación.

#### Scenario: Dropdown sin Google Maps

- **WHEN** el staff agrega una red social nueva a un lugar
- **THEN** el dropdown ofrece instagram, facebook, tiktok, youtube, whatsapp_channel, website y other — sin `google_maps`

### Requirement: Adopción de filas google_maps existentes sin sede

Una fila `google_maps` activa con `place_location_id` null SHALL adoptarse así: en un lugar mono-sede, su URL SHALL aparecer en el campo "Link de Maps" de la sede `Principal` con una nota visible de que se asignará a esa sede al guardar, y el guardado SHALL actualizar la misma fila (mismo `id`) poniendo `place_location_id` — automático porque no hay ambigüedad, pero visible antes de ejecutarse. En un lugar multi-sede, la fila SHALL mostrarse en el módulo de sedes como "link sin sede asignada" con su URL y un selector de sede; asignar SHALL actualizar la misma fila. El Console MUST NOT reasignar en silencio ni borrar filas no asignadas — sin asignar, la fila queda visible y dormida (el webhook la ignora para `maps_url` en multi-sede, comportamiento vigente).

#### Scenario: Mono-sede auto-asigna al guardar

- **WHEN** el staff abre un lugar mono-sede con una fila `google_maps` de marca (sin sede) y guarda
- **THEN** esa misma fila queda con `place_location_id` de la sede `Principal` y no se crea ninguna fila nueva

#### Scenario: Multi-sede asigna manualmente

- **WHEN** el staff abre una cadena con un link `google_maps` de marca sin sede, lo asigna a la sede "Galicia" y guarda
- **THEN** la misma fila queda con `place_location_id` de Galicia

#### Scenario: Link sin asignar queda visible

- **WHEN** el staff abre una cadena con un link `google_maps` sin sede y guarda sin asignarlo
- **THEN** la fila permanece activa, sin `place_location_id`, y sigue apareciendo como "link sin sede asignada" en visitas posteriores
