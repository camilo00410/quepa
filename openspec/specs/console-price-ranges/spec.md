# console-price-ranges Specification

## Purpose

Captura del rango de precios real en COP en el detalle de lugar del Console (`· 02 ·` de `console/lugar.html`): campos desde/hasta con formato de miles, validación espejo del check de base (desde ≤ hasta), aviso no bloqueante de magnitud y convivencia con el picker $–$$$$ manual. El backend y la exposición al bot viven en la spec hermana `place-price-ranges` (quepa-webhook, migración `0012`). Origen: change `add-console-price-ranges` (2026-07-24).
## Requirements
### Requirement: Captura del rango de precios en el detalle de lugar
La sección `· 02 · ECONÓMICO Y SERVICIOS` de `console/lugar.html` SHALL ofrecer dos campos "Desde" y "Hasta" en COP que se hidratan de `places.price_min`/`price_max` al cargar y se persisten vía `gather()` al guardar. Los campos SHALL formatear con separador de miles es-CO en pantalla y MUST persistir enteros limpios. Un solo extremo diligenciado SHALL ser válido.

#### Scenario: Captura y guardado de rango completo
- **WHEN** el staff digita 35000 y 75000 y guarda
- **THEN** el update de `places` incluye `price_min = 35000` y `price_max = 75000` y los campos muestran "35.000" y "75.000"

#### Scenario: Hidratación al cargar
- **WHEN** se abre un lugar con `price_min = 180000` y `price_max = 250000`
- **THEN** los campos muestran "180.000" y "250.000" sin tocar el picker $–$$$$

#### Scenario: Solo un extremo
- **WHEN** el staff diligencia solo "Desde" y guarda
- **THEN** el guardado procede con `price_max = null` sin error

### Requirement: Validación espejo del check de base de datos
El Console MUST bloquear el guardado cuando ambos extremos existan y `desde > hasta`, con un error inline honesto en la sección, ANTES de llamar a Supabase.

#### Scenario: Rango invertido bloqueado
- **WHEN** el staff digita desde 75000 y hasta 35000 e intenta guardar
- **THEN** el guardado no llega a Supabase y aparece un error inline indicando que el mínimo no puede superar el máximo

### Requirement: Aviso no bloqueante de magnitud
Cuando un extremo tenga valor entre 1 y 999, el Console SHALL mostrar una nota visible indicando que el valor se guarda tal cual (posible olvido de miles), y MUST permitir guardar sin corregir ni multiplicar el valor.

#### Scenario: Valor sospechosamente pequeño
- **WHEN** el staff digita 35 en "Desde"
- **THEN** aparece la nota de magnitud y el guardado sigue disponible con `price_min = 35`

### Requirement: Convivencia con el bucket manual
Los campos de rango MUST NOT alterar la captura ni el valor del picker `price_range` 1–4, y el picker MUST NOT modificar los campos de rango. Un `help` en la sección SHALL comunicar la unidad implícita: hotel = por noche; toda otra categoría = por persona.

#### Scenario: Picker y rango independientes
- **WHEN** el staff cambia el bucket de $$ a $$$ sin tocar los campos de rango
- **THEN** el guardado persiste el nuevo bucket y deja `price_min`/`price_max` exactamente como estaban

#### Scenario: Ayuda de unidad para hoteles
- **WHEN** el lugar tiene categoría `hotel`
- **THEN** el texto de ayuda de la sección indica que el rango se interpreta por noche

