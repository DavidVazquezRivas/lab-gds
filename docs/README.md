# Especificación de Interfaces de Integración — Laboratorio GDS

Este documento define los contratos de interfaz (API Contracts) para el sistema de Distribución Global (GDS). El objetivo es estandarizar la comunicación entre el GDS, el Servicio Local y el Proveedor Externo.

---

## Tabla de contenidos

- [Resumen](#resumen)
- [Normas generales](#normas-generales)
- [DTOs (Objetos de Datos)](#dtos-objetos-de-datos)
  - [ClientDTO](#clientdto)
  - [RoomRequestDTO](#roomrequestdto)
- [Local Web Service (BD PULL)](#local-web-service-bd-pull)
  - [Consultar Disponibilidad](#local-consultar-disponibilidad)
  - [Realizar Reserva](#local-realizar-reserva)
- [Provider Web Service (BD PUSH)](#provider-web-service-bd-push)
  - [Consultar Disponibilidad](#provider-consultar-disponibilidad)
  - [Realizar Reserva](#provider-realizar-reserva)
  - [Configurar Inventario](#provider-configurar-inventario)
- [GDS (Orquestador y Base de Datos Central)](#gds-orquestador-y-base-de-datos-central)
- [Ejemplos](#ejemplos)
- [Contribuciones y contacto](#contribuciones-y-contacto)
- [Licencia](#licencia)

---

## Resumen

Esta especificación describe los endpoints y los modelos de datos para integrar los servicios de Local (BD PULL), de un proveedor externo (BD PUSH), y el GDS que orquesta las peticiones. Está orientada a desarrolladores que implementen/consuman estas APIs.

## Normas generales

- Formato de fecha: ISO 8601 (YYYY-MM-DD)
- Formato de moneda: decimal con 2 decimales
- Codificación: UTF-8
- Content-Type: application/json

---

## DTOs (Objetos de Datos)

Los objetos descritos se reutilizan en las llamadas para mantener compatibilidad entre servicios.

### ClientDTO

```json
{
  "first_name": "string",
  "last_name": "string",
  "email": "string",
  "phone": "string"
}
```

### RoomRequestDTO

```json
{
  "check_in_date": "YYYY-MM-DD",
  "nights": 1,
  "room_type": "string" // Ejemplos: "INDIVIDUAL", "DOBLE", "SUITE"
}
```

---

## Local Web Service (Inventario Interno — BD PULL)

Este servicio expone la disponibilidad y permite crear reservas en la base de datos local (BD PULL).

### Local — Consultar Disponibilidad

- Endpoint: `POST /api/local/search`
- Request: `RoomRequestDTO`
- Response (200 OK):

```json
{
  "source": "LOCAL",
  "results": [
    {
      "hotel_id": 123,
      "hotel_name": "Hotel Interno",
      "room_type": "DOBLE",
      "available_count": 5,
      "price_per_night": 89.9
    }
  ]
}
```

### Local — Realizar Reserva

- Endpoint: `POST /api/local/book`
- Request:

```json
{
  "hotel_id": 123,
  "check_in_date": "2025-12-10",
  "room_type": "DOBLE",
  "client": {
    "first_name": "Juan",
    "last_name": "Pérez",
    "email": "juan@example.com",
    "phone": "+34123456789"
  }
}
```

- Response (201 Created):

```json
{
  "status": "CONFIRMED",
  "reservation_id": "local-RES-0001",
  "local_timestamp": "2025-12-01T12:34:56Z"
}
```

---

## Provider Web Service (CRS Externo — BD PUSH)

Este servicio representa un proveedor externo al que el GDS debe adaptarse.

### Provider — Consultar Disponibilidad

- Endpoint: `POST /api/provider/availability`
- Request: `RoomRequestDTO`
- Response (200 OK):

```json
{
  "source": "PROVIDER_XYZ",
  "offers": [
    {
      "provider_hotel_code": "PRV123",
      "description": "Hotel Externo - Hab. Doble",
      "room_category": "DOBLE",
      "is_available": true,
      "rate": 120.0
    }
  ]
}
```

### Provider — Realizar Reserva

- Endpoint: `POST /api/provider/reservation`
- Request:

```json
{
  "provider_hotel_code": "PRV123",
  "dates": {
    "start": "2025-12-10",
    "end": "2025-12-12"
  },
  "room_category": "DOBLE",
  "guest_details": {
    "first_name": "Ana",
    "last_name": "García",
    "email": "ana@example.com",
    "phone": "+34111222333"
  }
}
```

- Response (200 OK / 201 Created):

```json
{
  "booking_status": "SUCCESS",
  "provider_booking_ref": "PRV-BOOK-0001",
  "message": "Reserva confirmada en sistema externo"
}
```

> Nota: Es crítico obtener el `provider_booking_ref` y guardarlo como `hotel_reserva_id` en la tabla del GDS para trazabilidad.

### Provider — Configurar Inventario (Gestión)

- Endpoint: `PUT /api/provider/inventory`
- Request:

```json
{
  "hotel_code": "PRV123",
  "date": "2025-12-10",
  "room_type": "DOBLE",
  "total_quantity": 20,
  "price": 120.0
}
```

- Response (200 OK):

```json
{
  "status": "UPDATED",
  "current_stock": 20
}
```

---

## GDS (Orquestador y Base de Datos Central)

El GDS es el orquestador que:

- Consume `Local` y `Provider` simultáneamente.
- Unifica los resultados de disponibilidad y los presenta a la OTA.
- Registra reservas (tanto Local como Provider) en su tabla de `RESERVAS`.

### Estructura lógica de tabla `RESERVAS` (sugerida)

| Campo            | Origen       | Descripción                                                  |
| ---------------- | ------------ | ------------------------------------------------------------ |
| id_reserva       | GDS          | Identificador interno (autoincremental)                      |
| datos_cliente    | OTA          | JSON con nombre/ contacto                                    |
| hotel            | OTA          | Nombre o ID del hotel reservado                              |
| hotel_reserva_id | API Response | `reservation_id` (Local) O `provider_booking_ref` (Provider) |
| source           | API Response | `LOCAL` o `PROVIDER_XYZ`                                     |
| created_at       | GDS          | Timestamp de creación                                        |

### Flujo de disponibilidad (OTA)

1. El usuario solicita disponibilidad con fecha y tipo de habitación.
2. El GDS llama concurrentemente a `Local` y a `Provider`.
3. Unifica listas JSON y muestra opciones al usuario.
4. Cuando el usuario confirma, GDS realiza la reserva en la fuente correspondiente y registra el resultado.

---

## Ejemplos

- Ejemplo de una búsqueda y reserva completa: ver `Local` y `Provider` request/response en las secciones anteriores.

### Archivos DTO

Los ejemplos de request/response JSON se encuentran en la carpeta `dto/` del repositorio. Ahí encontrarás plantillas para cada endpoint (`local/`, `provider/`) y las DTOs usadas por el GDS.

Ver `dto/README.md` para más detalles.

---

## Contribuciones y contacto

Si quieres contribuir a este documento, abre un issue o PR en este repositorio con los detalles de la propuesta. Para consultas técnicas contacta a `devops@example.com` (placeholder).

---

## Licencia

Este repositorio usa licencia MIT (o la que prefieran). Añade aquí el texto de la licencia según corresponda.

---

_Documento generado para proporcionar una especificación clara y coherente de las interfaces de integración para el Laboratorio GDS._
