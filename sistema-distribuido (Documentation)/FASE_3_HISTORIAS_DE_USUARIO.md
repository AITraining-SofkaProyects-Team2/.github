# Fase 3 - Historias de Usuario (Resumen)

## HU-06 - Búsqueda por ID de ticket

- Criterios de Aceptación:
  - Endpoint `GET /api/tickets/:id` devuelve el ticket cuando `:id` es un UUIDv4 válido existente.
  - Si `:id` no existe, retorna HTTP 404 con mensaje "Ticket no encontrado".
  - Si `:id` tiene formato inválido, retorna HTTP 400 con mensaje de validación.
  - Si se llama al endpoint raíz `GET /api/tickets/` (ID vacío), se debe comportar como listado paginado (HU-01).

- Notas de pruebas:
  - Se creó el test unitario `TC-031` (archivo `backend/reports-query/tests/HU-06.test.ts`) en estado RED, validando que `GET /api/tickets/` retorna el listado paginado (comportamiento cuando no se pasa ID).
