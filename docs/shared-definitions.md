# Definiciones compartidas — Laboratorio GDS

Este documento reúne las definiciones, enumeraciones y convenciones que deben compartirse entre el GDS, el Servicio Local y los Proveedores externos. Sirve para estandarizar nombres, valores y comportamientos a lo largo de todas las APIs.

---

## Normas globales

- Fechas y timestamps: ISO 8601 (`YYYY-MM-DD` para fechas, `YYYY-MM-DDThh:mm:ssZ` o con offset para timestamps). Recomendado: UTC (sufijo `Z`).
- Codificación: UTF-8.
- Content-Type de payloads: `application/json`.
- Moneda: usar código ISO 4217 (p.ej. `EUR`, `USD`) y formato decimal con 2 decimales.
- Separador decimal: punto (`.`).

---

## Tipos de habitación (Room Types)

Use únicamente las siguientes cadenas para `room_type` o `room_category` en requests/responses:

| Código | Descripción |
|---|---|
| SINGLE | Habitación individual (1 adulto)
| DOUBLE | Habitación doble (2 adultos o 2 camas)
| TWIN | Habitación con dos camas separadas
| TRIPLE | Habitación triple (3 adultos)
| SUITE | Suite (espacio ampliado, con uno o más ambientes)
| APARTMENT | Apartamento (cocina, sala área común)
| FAMILY | Habitación familiar (para familias)
| ECONOMY | Habitación económica
| DELUXE | Habitación de categoría superior

> Tip: Evitar sinónimos o variantes como "INDIVIDUAL" o "DOBLE" si el resto de integraciones usan los códigos en inglés; acuerden internamente el código a usar (ya sea español o inglés). Si hay que soportar ambas, aplicar un mapeo interno en el GDS.

---

## Estados de ofertas (Offer Status)

Valores para describir la disponibilidad de una oferta o tarifa:

| Código | Descripción |
|---|---|
| AVAILABLE | La oferta está disponible para venta inmediata
| UNAVAILABLE | La oferta no está disponible
| ON_HOLD | La oferta está retenida (ej. en proceso de reserva)
| SOLD_OUT | Se ha agotado el inventario
| INVALID | Oferta inválida (datos incompletos) |

---

## Estados de reserva (Reservation Status)

Valores para describir el estado de una reserva en todas las plataformas:

| Código | Descripción |
|---|---|
| PENDING | Reserva iniciada pero no confirmada (p. ej. esperando respuesta del proveedor)
| CONFIRMED | Reserva confirmada correctamente
| CANCELLED | Reserva cancelada
| FAILED | Reserva fallida (error de procesamiento)
| CHECKED_IN | Huésped realizó check-in
| CHECKED_OUT | Huésped realizó check-out
| NO_SHOW | No se presentó el huésped

> Nota: `reservation_id` (proveniente del Local) o `provider_booking_ref` (proveniente del Provider) debe guardarse en el campo `hotel_reserva_id` del GDS para trazabilidad.

---

## Estados de pago (Payment Status)

| Código | Descripción |
|---|---|
| PAID | Pagado en su totalidad
| UNPAID | No pagado
| PARTIALLY_PAID | Pago parcial
| REFUNDED | Reintegrado total
| PARTIALLY_REFUNDED | Reintegro parcial

---

## Campos y mapeos comunes

- `hotel_reserva_id`: Identificador de reserva del origen (Local: `reservation_id`; Provider: `provider_booking_ref`). Almacenar siempre el identificador original del origen.
- `source`: Indicar la fuente de la reserva/oferta: `LOCAL`, `PROVIDER_XYZ`, `PROVIDER_ABC`.
- `rate` o `price_per_night`: Valor decimal (2 decimales) con la moneda indicada en `currency`.
- `currency`: Código ISO 4217 (p.ej. `EUR`).
- `occupancy`: JSON con `adults` (integer) y `children` (integer, opcional), o separado en `max_adults`/`max_children` según convenga.

---

## Reglas de negocio recomendadas

- Cuando el GDS consulta a varias fuentes, las tarifas deben normalizarse a un formato interno antes de unificarlas.
- Las operaciones que alteran inventario (p. ej. reservas) deben realizarse con confirmación y, si el proveedor no devuelve un ID, el sistema debe reportar un error.
- Los códigos de error en las respuestas deben contener `code` y `message` para facilitar la trazabilidad.

Ejemplo de error estándar:

```json
{
  "code": "ERR_PROVIDER_001",
  "message": "Proveedor no responde",
  "details": {}
}
```

---

## Propiedades recomendadas en respuestas de API

- `status` (p.ej., `SUCCESS`, `ERROR`, `UPDATED`) — resumen del resultado.
- `code` — código interno o de proveedor (si aplica).
- `message` — mensaje legible por desarrolladores.
- `data` — objeto con la carga útil (por ejemplo, `reservation_id`, `provider_booking_ref`).

---

## Ejemplos rápidos

- Offer (Provider)

```json
{
  "source": "PROVIDER_XYZ",
  "offers": [
    {
      "provider_hotel_code": "PRV123",
      "room_category": "DOUBLE",
      "is_available": true,
      "rate": 120.00,
      "currency": "EUR",
      "status": "AVAILABLE"
    }
  ]
}
```

- Reservation (Local Success):

```json
{
  "status": "CONFIRMED",
  "reservation_id": "local-RES-0001",
  "hotel_reserva_id": "local-RES-0001",
  "timestamp": "2025-12-01T12:34:56Z"
}
```

- Reservation (Provider Success):

```json
{
  "status": "CONFIRMED",
  "provider_booking_ref": "PRV-BOOK-0001",
  "hotel_reserva_id": "PRV-BOOK-0001",
  "timestamp": "2025-12-01T12:34:56Z"
}
```

---

## Aclaraciones y mantenimiento

- Este documento debe consensuarse entre las partes y mantenerse en sync con cambios de esquema y versiones de la API.
- Para ampliar los tipos de habitación, habilitar un mapeo en el GDS y registrar nuevas opciones en este documento.

---

_Termina definición compartida._
