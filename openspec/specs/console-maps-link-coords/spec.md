# console-maps-link-coords Specification

## Requirements

### Requirement: Captura de coordenadas pegando la URL de Google Maps

`console/lugar.html` SHALL ofrecer un campo "URL de Google Maps" como vía primaria de captura de coordenadas **por sede en la grid de sedes** (escribe `sede.lat/lng`) — es el único punto de captura; no existe un campo a nivel de lugar. Al pegar una URL, el Console SHALL extraer lat/lng localmente por regex en orden de confianza (`!3d…!4d…` primero, luego `@lat,lng`, `q=`, `ll=`) — MUST NOT hacer llamadas externas ni agregar dependencias. La extracción exitosa SHALL reflejarse de inmediato en el resumen textual de coordenadas de la sede.

#### Scenario: URL de ficha de lugar

- **WHEN** el staff pega `https://www.google.com/maps/place/X/@4.7990,-75.8059,17z/data=…!3d4.799063!4d-75.805927…` en el campo de una sede
- **THEN** la sede queda con lat 4.799063 y lng -75.805927 (los del `!3d/!4d`, no los del `@`) y el resumen de coordenadas muestra los valores nuevos

#### Scenario: El campo URL es efímero

- **WHEN** el staff pega una URL válida, guarda y recarga el lugar
- **THEN** las coordenadas persisten en `lat`/`lng` por el flujo de guardado existente y el campo URL aparece vacío (la URL no se persiste)

### Requirement: Floats como detalle avanzado, no como vía primaria

Los inputs numéricos de lat/lng SHALL quedar colapsados como detalle avanzado con el valor actual visible en el resumen ("4.79906, -75.80593" o "sin coordenadas"), y SHALL seguir siendo editables (corregir, limpiar o ingresar coordenadas de otra fuente). El flujo de guardado MUST NOT cambiar: los floats siguen siendo el dato que se persiste.

#### Scenario: Válvula de escape manual

- **WHEN** el staff abre el detalle avanzado de una sede, borra los valores y guarda
- **THEN** la sede queda sin coordenadas (null) y el resumen muestra "sin coordenadas"

### Requirement: Degradación honesta con URLs no extraíbles

Una URL sin coordenadas extraíbles (link corto `maps.app.goo.gl`, URL de búsqueda, texto arbitrario) SHALL producir una nota clara que pida la URL completa del navegador y señale que el link corto va en el campo "Link de Maps" de la sede (donde sí es el dato correcto), y MUST NOT modificar las coordenadas existentes. Coordenadas extraídas fuera del rango aproximado de Colombia SHALL aceptarse con un aviso visible de verificación — el staff decide.

#### Scenario: Link corto del celular

- **WHEN** el staff pega `https://maps.app.goo.gl/AbC123` en el campo de URL-para-coordenadas de una sede
- **THEN** aparece la nota indicando que ese link corto no trae coordenadas (pegar la URL completa del navegador) y que el link corto pertenece al campo "Link de Maps" de la sede, y `lat`/`lng` quedan como estaban

#### Scenario: Coordenadas sospechosas

- **WHEN** la URL pegada extrae lat 40.4168, lng -3.7038 (Madrid)
- **THEN** los valores se escriben pero con aviso visible de que caen fuera de Colombia, para que el staff verifique
