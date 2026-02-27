# Diseño de Casos de Prueba - Sistema de Gestión de Quejas ISP

## Proyecto: Taller Sistema Distribuido

Este documento contiene el diseño detallado de los casos de prueba para las historias de usuario del sistema de gestión de quejas distribuido. Cada caso de prueba está estructurado en formato Gherkin con particiones de equivalencia, valores límites y tablas de decisión cuando aplique.

---

## Índice de Tests

### UH-013: Cambio de Estado de Tickets desde la Lista

| Estado | ID Test | Historia | Descripción | Tipo |
|---------|----------|-------------|--------|------|
| GREEN | TC-013-001 | UH-013 | Validación de formato de ticketId - UUID válido | Unitario |
| GREEN | TC-013-002 | UH-013 | Validación de formato de ticketId - UUID inválido | Unitario |
| GREEN | TC-013-003 | UH-013 | Validación de estado válido - RECEIVED | Unitario |
| GREEN | TC-013-004 | UH-013 | Validación de estado inválido - valores fuera del dominio | Unitario |
| GREEN | TC-013-005 | UH-013 | Actualización exitosa de RECEIVED a IN_PROGRESS | Unitario |
| GREEN | TC-013-006 | UH-013 | Error 404 - Ticket no encontrado | Unitario |
| GREEN | TC-013-007 | UH-013 | Actualización idempotente - mismo estado | Unitario |
| GREEN | TC-013-008 | UH-013 | Manejo de errores de base de datos | Unitario |
| ⬜ | TC-013-009 | UH-013 | Request PATCH completa con body válido retorna 200 | integración de componentes |
| ⬜ | TC-013-010 | UH-013 | Request PATCH con body inválido retorna 400 | integración de componentes |
| ⬜ | TC-013-011 | UH-013 | Flujo E2E completo de cambio de estado | E2E |
| ⬜ | TC-013-012 | UH-013 | Múltiples cambios de estado consecutivos | E2E |

===TESTS EN FRONTEND===
| RED | TC-013-013 | UH-013 | Modal se abre al hacer clic en botón de cambio de estado | integración de componentes |
| RED | TC-013-014 | UH-013 | Modal muestra información correcta del ticket | integración de componentes |
| RED | TC-013-015 | UH-013 | Selector muestra todos los estados disponibles | integración de componentes |
| RED | TC-013-016 | UH-013 | Modal se cierra al hacer clic en Cancelar | integración de componentes |
| RED | TC-013-017 | UH-013 | Modal envía request correcta al confirmar cambio | integración de componentes |
| RED | TC-013-018 | UH-013 | Lista se actualiza después de cambio exitoso | integración de componentes |
| RED | TC-013-019 | UH-013 | Mensaje de error se muestra en caso de fallo | integración de componentes |
| RED | TC-013-020 | UH-013 | Modal se cierra automáticamente después de éxito | integración de componentes |
---

## Casos de Prueba Detallados

### UH-013: Cambio de Estado de Tickets desde la Lista

---

#### TC-013-001: Validación de formato de ticketId - UUID válido

- **ID del Test**: TC-013-001
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el endpoint acepta ticketIds con formato UUIDv4 válido.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente
  - El endpoint PATCH `/api/tickets/:ticketId/status` está disponible

**Pasos (Gherkin)**:
```gherkin
Given el endpoint PATCH /api/tickets/:ticketId/status está disponible
When se envía una request con ticketId "550e8400-e29b-41d4-a716-446655440000" (UUID válido)
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema valida el formato del ticketId exitosamente
  And procede con la lógica de actualización
```

**Partición de equivalencia**:
| Grupo | Valores de ticketId | Tipo |
|-------|---------------------|------|
| UUID v4 válido | "550e8400-e29b-41d4-a716-446655440000" | Válido |
| UUID v4 válido con minúsculas | "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11" | Válido |
| UUID v4 válido con mayúsculas | "A0EEBC99-9C0B-4EF8-BB6D-6BB9BD380A11" | Válido |
| UUID con formato incorrecto | "invalid-uuid-format" | Inválido |
| Cadena vacía | "" | Inválido |
| Cadena no-UUID | "abc123" | Inválido |

**Valores límites**:
- UUID con longitud exacta (36 caracteres con guiones)
- UUID con todos ceros: "00000000-0000-0000-0000-000000000000"
- UUID con todos F's: "ffffffff-ffff-ffff-ffff-ffffffffffff"

---

#### TC-013-002: Validación de formato de ticketId - UUID inválido

- **ID del Test**: TC-013-002
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el endpoint rechaza ticketIds con formato inválido retornando HTTP 400.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente
  - El endpoint PATCH `/api/tickets/:ticketId/status` está disponible

**Pasos (Gherkin)**:
```gherkin
Given el endpoint PATCH /api/tickets/:ticketId/status está disponible
When se envía una request con ticketId "invalid-uuid-format" (formato inválido)
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema retorna HTTP 400
  And el mensaje de error indica "Formato de ticketId inválido"
  And no se ejecuta la lógica de actualización
```

**Partición de equivalencia**:
| Grupo | Valores de ticketId | Tipo | Resultado Esperado |
|-------|---------------------|------|---------------------|
| Formato incorrecto | "invalid-uuid" | Inválido | HTTP 400 |
| Muy corto | "abc" | Inválido | HTTP 400 |
| Muy largo | "550e8400-e29b-41d4-a716-446655440000-extra" | Inválido | HTTP 400 |
| Sin guiones | "550e8400e29b41d4a716446655440000" | Inválido | HTTP 400 |
| Con caracteres especiales | "550e8400-e29b-41d4-a716-44665544000@" | Inválido | HTTP 400 |
| Null | null | Inválido | HTTP 400 |

**Valores límites**:
- 35 caracteres (uno menos del mínimo)
- 37 caracteres (uno más del máximo)
- Cadena vacía ("")

---

#### TC-013-003: Validación de estado válido - RECEIVED

- **ID del Test**: TC-013-003
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el endpoint acepta "RECEIVED" como estado válido.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente
  - Existe un ticket en la base de datos

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "IN_PROGRESS"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "RECEIVED" }
Then el sistema valida que "RECEIVED" es un estado válido
  And actualiza el ticket al estado "RECEIVED"
  And retorna HTTP 200
```

**Partición de equivalencia**:
| Grupo | Valor de status | Tipo |
|-------|-----------------|------|
| Estado válido | "RECEIVED" | Válido |
| Estado válido | "IN_PROGRESS" | Válido |
| Estado inválido | "CLOSED" | Inválido |
| Estado inválido | "RESOLVED" | Inválido |
| Estado inválido | "PENDING" | Inválido |
| Minúsculas | "received" | Inválido (case-sensitive) |
| Vacío | "" | Inválido |

**Valores límites**:
- Exactamente uno de los dos valores permitidos del dominio

---

#### TC-013-004: Validación de estado válido - IN_PROGRESS

- **ID del Test**: TC-013-004
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el endpoint acepta "IN_PROGRESS" como estado válido.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente
  - Existe un ticket en la base de datos

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema valida que "IN_PROGRESS" es un estado válido
  And actualiza el ticket al estado "IN_PROGRESS"
  And retorna HTTP 200
```

**Partición de equivalencia**:
| Grupo | Valor de status | Tipo |
|-------|-----------------|------|
| Estado válido 1 | "RECEIVED" | Válido |
| Estado válido 2 | "IN_PROGRESS" | Válido |
| Con espacios | " IN_PROGRESS " | Inválido |
| Con guión bajo diferente | "IN-PROGRESS" | Inválido |

**Valores límites**:
- Segundo de los dos valores permitidos del dominio

---

#### TC-013-005: Validación de estado inválido - valores fuera del dominio

- **ID del Test**: TC-013-005
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el endpoint rechaza estados que no pertenecen al dominio definido.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente
  - Existe un ticket en la base de datos

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "CLOSED" }
Then el sistema retorna HTTP 400
  And el mensaje de error indica "Estado inválido. Valores permitidos: RECEIVED, IN_PROGRESS"
  And no se actualiza el ticket en la base de datos
```

**Partición de equivalencia**:
| Grupo | Valor de status | Tipo | Código HTTP Esperado |
|-------|-----------------|------|----------------------|
| Estados válidos | "RECEIVED", "IN_PROGRESS" | Válido | 200 |
| Estado inexistente | "CLOSED" | Inválido | 400 |
| Estado inexistente | "RESOLVED" | Inválido | 400 |
| Estado inexistente | "CANCELLED" | Inválido | 400 |
| Cadena vacía | "" | Inválido | 400 |
| Null | null | Inválido | 400 |
| Undefined | undefined | Inválido | 400 |
| Número | 123 | Inválido | 400 |
| Boolean | true | Inválido | 400 |

**Valores límites**:
- Estado con un carácter de diferencia: "RECEIVEDD", "IN_PROGRES"
- Estado en minúsculas: "received", "in_progress"

**Tabla de Decisión**:
| # | Estado = "RECEIVED" | Estado = "IN_PROGRESS" | Estado = Otro | Resultado |
|---|---------------------|------------------------|---------------|-----------|
| 1 | ✓ | - | - | HTTP 200 |
| 2 | - | ✓ | - | HTTP 200 |
| 3 | - | - | ✓ | HTTP 400 |

---

#### TC-013-006: Actualización exitosa de RECEIVED a IN_PROGRESS

- **ID del Test**: TC-013-006
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que un ticket puede cambiar exitosamente de estado RECEIVED a IN_PROGRESS.
- **Precondiciones**: 
  - Existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
  - La base de datos está disponible

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
  And el campo processed_at del ticket es "2026-02-25T10:00:00.000Z"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema retorna HTTP 200
  And la respuesta contiene el ticket actualizado
  And el ticket tiene status = "IN_PROGRESS"
  And el campo processed_at se actualiza con la timestamp actual
  And el resto de campos del ticket permanecen sin cambios
```

**Partición de equivalencia**:
| Grupo | Estado Inicial | Estado Final | Tipo |
|-------|----------------|--------------|------|
| Transición válida forward | RECEIVED | IN_PROGRESS | Válido |
| Transición válida backward | IN_PROGRESS | RECEIVED | Válido |
| Sin cambio | RECEIVED | RECEIVED | Válido (idempotente) |
| Sin cambio | IN_PROGRESS | IN_PROGRESS | Válido (idempotente) |

**Valores límites**:
- Ticket recién creado (processed_at inicial)
- Ticket con múltiples actualizaciones previas

---

#### TC-013-007: Actualización exitosa de IN_PROGRESS a RECEIVED

- **ID del Test**: TC-013-007
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que un ticket puede cambiar exitosamente de estado IN_PROGRESS a RECEIVED (transición bidireccional).
- **Precondiciones**: 
  - Existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "IN_PROGRESS"
  - La base de datos está disponible

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "IN_PROGRESS"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "RECEIVED" }
Then el sistema retorna HTTP 200
  And la respuesta contiene el ticket actualizado
  And el ticket tiene status = "RECEIVED"
  And el campo processed_at se actualiza con la timestamp actual
```

**Partición de equivalencia**:
| Grupo | Dirección de Transición | Tipo |
|-------|-------------------------|------|
| Forward | RECEIVED → IN_PROGRESS | Válido |
| Backward | IN_PROGRESS → RECEIVED | Válido |
| Ninguna | RECEIVED → RECEIVED | Válido |
| Ninguna | IN_PROGRESS → IN_PROGRESS | Válido |

**Valores límites**:
- Primera actualización de estado del ticket
- Ticket actualizado múltiples veces entre estados

---

#### TC-013-008: Error 404 - Ticket no encontrado

- **ID del Test**: TC-013-008
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el sistema retorna HTTP 404 cuando se intenta actualizar un ticket que no existe.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente
  - No existe un ticket con el ID especificado en la base de datos

**Pasos (Gherkin)**:
```gherkin
Given no existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en la base de datos
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema retorna HTTP 404
  And el mensaje de error indica "Ticket no encontrado"
  And no se realiza ninguna operación en la base de datos
```

**Partición de equivalencia**:
| Grupo | Ticket Exists | Tipo | Código HTTP Esperado |
|-------|---------------|------|----------------------|
| Existe | true | Válido | 200 |
| No existe | false | Caso de error | 404 |

**Valores límites**:
- UUID válido pero no existente
- Primer ticket creado del sistema
- Último ticket eliminado

---

#### TC-013-009: Actualización idempotente - mismo estado

- **ID del Test**: TC-013-009
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que cambiar un ticket al mismo estado que ya tiene es una operación válida e idempotente.
- **Precondiciones**: 
  - Existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "IN_PROGRESS"

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "IN_PROGRESS"
  And el campo processed_at del ticket es "2026-02-25T10:00:00.000Z"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema retorna HTTP 200
  And la respuesta contiene el ticket
  And el ticket mantiene status = "IN_PROGRESS"
  And el campo processed_at se actualiza con la nueva timestamp actual
  And la operación es idempotente (múltiples llamadas tienen el mismo efecto)
```

**Partición de equivalencia**:
| Grupo | Estado Inicial | Estado Solicitado | Tipo |
|-------|----------------|-------------------|------|
| Idempotente RECEIVED | RECEIVED | RECEIVED | Válido |
| Idempotente IN_PROGRESS | IN_PROGRESS | IN_PROGRESS | Válido |
| Cambio real | RECEIVED | IN_PROGRESS | Válido |
| Cambio real | IN_PROGRESS | RECEIVED | Válido |

**Valores límites**:
- Una sola llamada idempotente
- Múltiples llamadas idempotentes consecutivas (2, 3, 5, 10)

---

#### TC-013-010: Actualización del campo processed_at

- **ID del Test**: TC-013-010
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el campo processed_at se actualiza correctamente con la timestamp actual en cada cambio de estado.
- **Precondiciones**: 
  - Existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000"
  - El sistema genera timestamps en formato ISO 8601

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
  And el campo processed_at del ticket es "2026-02-25T10:00:00.000Z" (valor antiguo)
  And la timestamp actual del sistema es "2026-02-26T15:30:45.123Z"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema retorna HTTP 200
  And el campo processed_at del ticket en la respuesta es "2026-02-26T15:30:45.123Z"
  And el campo processed_at en la base de datos es "2026-02-26T15:30:45.123Z"
  And el valor nuevo es mayor que el valor antiguo
```

**Partición de equivalencia**:
| Grupo | Situación del Timestamp | Tipo |
|-------|-------------------------|------|
| Primera actualización | processed_at = null o fecha inicial | Válido |
| Actualización subsecuente | processed_at = fecha pasada | Válido |
| Múltiples actualizaciones rápidas | processed_at actualizado hace < 1s | Válido |

**Valores límites**:
- Timestamp justo después de createdAt
- Múltiples actualizaciones en el mismo segundo
- Actualización después de días/semanas

---

#### TC-013-011: Manejo de errores de base de datos

- **ID del Test**: TC-013-011
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el sistema maneja correctamente errores de conexión o fallos en la base de datos.
- **Precondiciones**: 
  - Se puede simular un fallo en la base de datos (mediante mock o desconexión temporal)

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000"
  And la base de datos está experimentando problemas de conexión
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el body contiene { "status": "IN_PROGRESS" }
Then el sistema retorna HTTP 500
  And el mensaje de error es genérico "Error interno del servidor"
  And el error detallado se registra en los logs del servidor
  And no se muestra información sensible al cliente
```

**Partición de equivalencia**:
| Grupo | Tipo de Error DB | Código HTTP Esperado |
|-------|------------------|----------------------|
| Sin error | Operación exitosa | 200 |
| Error de conexión | Connection timeout | 500 |
| Error de deadlock | Database deadlock | 500 |
| Error de constraint | Constraint violation | 500 |
| Error de permisos | Permission denied | 500 |

**Valores límites**:
- Primera fallo tras 100 operaciones exitosas
- Fallo intermitente cada N requests

---

#### TC-013-012: Request PATCH completa con body válido retorna 200

- **ID del Test**: TC-013-012
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar el flujo completo de una request PATCH exitosa con body válido (Prueba de integración).
- **Precondiciones**: 
  - Existe un ticket en la base de datos
  - Todos los servicios están operativos

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
  And todos los servicios del Query Service están operativos
When se envía una request HTTP PATCH a /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el header Content-Type es "application/json"
  And el body es { "status": "IN_PROGRESS" }
Then el código de respuesta HTTP es 200
  And el header Content-Type de la respuesta es "application/json"
  And el body de la respuesta contiene el ticket completo actualizado en formato JSON
  And el campo status del ticket es "IN_PROGRESS"
```

**Partición de equivalencia**:
| Grupo | Componentes de la Request | Tipo |
|-------|---------------------------|------|
| Request completa válida | método PATCH + ruta válida + body válido | Válido |
| Método incorrecto | GET/POST/PUT | Inválido |
| Body incorrecto | Body malformado o vacío | Inválido |
| Header incorrecto | Sin Content-Type | Válido (tolerante) |

**Valores límites**:
- Body mínimo válido: `{ "status": "RECEIVED" }`
- Body con campos adicionales ignorados

---

#### TC-013-013: Request PATCH con body inválido retorna 400

- **ID del Test**: TC-013-013
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que requests con body malformado o inválido retornan HTTP 400.
- **Precondiciones**: 
  - El Query Service está ejecutándose correctamente

**Pasos (Gherkin)**:
```gherkin
Given el endpoint PATCH /api/tickets/:ticketId/status está disponible
When se envía una request PATCH con ticketId válido "550e8400-e29b-41d4-a716-446655440000"
  And el body es { "status": 12345 } (tipo de dato incorrecto)
Then el sistema retorna HTTP 400
  And el mensaje de error indica "Body de request inválido"
  And se especifica el problema: "El campo 'status' debe ser una cadena de texto"
```

**Partición de equivalencia**:
| Grupo | Formato de Body | Tipo | Código HTTP |
|-------|-----------------|------|-------------|
| Body válido | `{ "status": "RECEIVED" }` | Válido | 200/404 |
| Campo faltante | `{}` | Inválido | 400 |
| Tipo incorrecto | `{ "status": 123 }` | Inválido | 400 |
| Campo null | `{ "status": null }` | Inválido | 400 |
| JSON malformado | `{ status: "RECEIVED"` | Inválido | 400 |
| Texto plano | `"IN_PROGRESS"` | Inválido | 400 |
| Vacío | `` | Inválido | 400 |

**Valores límites**:
- Body con solo el campo requerido
- Body con campos adicionales

**Tabla de Decisión**:
| # | JSON Válido | Campo "status" presente | Tipo correcto | Valor válido | Resultado |
|---|-------------|-------------------------|---------------|--------------|-----------|
| 1 | ✓ | ✓ | ✓ | ✓ | 200/404 |
| 2 | ✓ | ✓ | ✓ | ✗ | 400 |
| 3 | ✓ | ✓ | ✗ | - | 400 |
| 4 | ✓ | ✗ | - | - | 400 |
| 5 | ✗ | - | - | - | 400 |

---

#### TC-013-014: Modal se abre al hacer clic en botón de cambio de estado

- **ID del Test**: TC-013-014
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el modal de cambio de estado se abre correctamente al hacer clic en el botón correspondiente (Prueba Frontend).
- **Precondiciones**: 
  - La lista de tickets está renderizada
  - Existe al menos un ticket visible en la lista

**Pasos (Gherkin)**:
```gherkin
Given un administrador está viendo la lista de tickets
  And existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" visible en la lista
  And el modal de cambio de estado no está visible
When el administrador hace clic en el botón "Cambiar Estado" del ticket
Then el modal de cambio de estado se abre y se muestra en pantalla
  And el fondo detrás del modal se oscurece (overlay)
  And el modal está centrado en la pantalla
  And el modal tiene foco para accesibilidad
```

**Partición de equivalencia**:
| Grupo | Acción del Usuario | Resultado Esperado |
|-------|--------------------|--------------------|
| Clic en botón | Click en "Cambiar Estado" | Modal se abre |
| Clic fuera | Click en otra parte | Modal no se abre |
| Teclado | Enter/Space en botón enfocado | Modal se abre |
| Modal ya abierto | Click en otro botón | Se cierra el anterior y abre nuevo |

**Valores límites**:
- Primer ticket de la lista
- Último ticket de la lista
- Ticket en página 2, 3, etc.

---

#### TC-013-015: Modal muestra información correcta del ticket

- **ID del Test**: TC-013-015
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el modal muestra correctamente el ID del ticket y su estado actual.
- **Precondiciones**: 
  - El modal de cambio de estado está abierto para un ticket específico

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
When el administrador abre el modal de cambio de estado para este ticket
Then el modal muestra el texto "Ticket ID: 550e8400-e29b-41d4-a716-446655440000"
  And el modal muestra el texto "Estado actual: RECEIVED" o una etiqueta equivalente
  And la información coincide exactamente con los datos del ticket
```

**Partición de equivalencia**:
| Grupo | Estado del Ticket | Tipo |
|-------|-------------------|------|
| Estado RECEIVED | RECEIVED | Válido |
| Estado IN_PROGRESS | IN_PROGRESS | Válido |

**Valores límites**:
- ID más corto posible (UUID mínimo)
- ID más largo (UUID estándar)

---

#### TC-013-016: Selector muestra todos los estados disponibles

- **ID del Test**: TC-013-016
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el selector (dropdown) del modal muestra todos los estados válidos del dominio.
- **Precondiciones**: 
  - El modal de cambio de estado está abierto

**Pasos (Gherkin)**:
```gherkin
Given el modal de cambio de estado está abierto
When el administrador visualiza el selector de estados
Then el selector contiene exactamente 2 opciones
  And una opción es "RECEIVED" con etiqueta legible "Recibido"
  And otra opción es "IN_PROGRESS" con etiqueta legible "En Progreso"
  And no hay opciones adicionales como "CLOSED" o "RESOLVED"
  And el estado actual del ticket está preseleccionado por defecto
```

**Partición de equivalencia**:
| Grupo | Estados Mostrados | Tipo |
|-------|-------------------|------|
| Correcto | RECEIVED, IN_PROGRESS | Válido |
| Incorrecto | RECEIVED, IN_PROGRESS, CLOSED | Inválido |
| Incompleto | Solo RECEIVED | Inválido |

**Valores límites**:
- Exactamente 2 opciones (ni más ni menos)
- Estado actual preseleccionado

---

#### TC-013-017: Modal se cierra al hacer clic en Cancelar

- **ID del Test**: TC-013-017
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el modal se cierra sin realizar cambios cuando se hace clic en Cancelar.
- **Precondiciones**: 
  - El modal de cambio de estado está abierto
  - Existe un ticket con estado "RECEIVED"

**Pasos (Gherkin)**:
```gherkin
Given el modal de cambio de estado está abierto para un ticket en estado "RECEIVED"
  And el administrador ha seleccionado "IN_PROGRESS" en el selector (sin confirmar)
When el administrador hace clic en el botón "Cancelar"
Then el modal se cierra y desaparece de la pantalla
  And no se envía ninguna request HTTP al backend
  And el estado del ticket permanece "RECEIVED" sin cambios
  And la lista de tickets no se actualiza
```

**Partición de equivalencia**:
| Grupo | Acción para Cerrar Modal | Request Enviada | Estado Modificado |
|-------|--------------------------|-----------------|-------------------|
| Clic en Cancelar | ✓ | ✗ | ✗ |
| Clic en overlay (fondo) | ✓ | ✗ | ✗ |
| Tecla ESC | ✓ | ✗ | ✗ |
| Clic en Confirmar | ✓ | ✓ | ✓ |

**Valores límites**:
- Cancelar sin cambiar selector
- Cancelar después de cambiar selector
- Cancelar múltiples veces

---

#### TC-013-018: Modal envía request correcta al confirmar cambio

- **ID del Test**: TC-013-018
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que al confirmar el cambio, el modal envía la request HTTP correcta al backend.
- **Precondiciones**: 
  - El modal de cambio de estado está abierto
  - El administrador ha seleccionado un nuevo estado

**Pasos (Gherkin)**:
```gherkin
Given el modal está abierto para el ticket "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
  And el administrador ha seleccionado "IN_PROGRESS" en el selector
When el administrador hace clic en el botón "Confirmar"
Then el frontend envía una request HTTP PATCH
  And la URL es /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And el header Content-Type es "application/json"
  And el body es { "status": "IN_PROGRESS" }
  And la request se envía antes de cerrar el modal
```

**Partición de equivalencia**:
| Grupo | Estado Seleccionado | Request Body Esperado |
|-------|---------------------|-----------------------|
| RECEIVED | RECEIVED | `{ "status": "RECEIVED" }` |
| IN_PROGRESS | IN_PROGRESS | `{ "status": "IN_PROGRESS" }` |

**Valores límites**:
- Primera confirmación del usuario
- Confirmación después de múltiples cancelaciones previas

---

#### TC-013-019: Lista se actualiza después de cambio exitoso

- **ID del Test**: TC-013-019
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que la lista de tickets se actualiza correctamente después de un cambio de estado exitoso, sin recargar la página completa.
- **Precondiciones**: 
  - La lista de tickets está renderizada
  - El modal está abierto y se confirma un cambio de estado
  - El backend retorna HTTP 200

**Pasos (Gherkin)**:
```gherkin
Given la lista muestra un ticket "550e8400-e29b-41d4-a716-446655440000" con estado "RECEIVED"
  And el administrador confirma el cambio a "IN_PROGRESS" en el modal
When el backend retorna HTTP 200 con el ticket actualizado
Then el modal se cierra automáticamente
  And la lista de tickets se actualiza sin recargar la página
  And el ticket "550e8400-e29b-41d4-a716-446655440000" ahora muestra estado "IN_PROGRESS"
  And se muestra un badge o etiqueta visual de "En Progreso"
  And no se pierde la posición de scroll en la lista
  And no se pierde el filtro o búsqueda aplicados
```

**Partición de equivalencia**:
| Grupo | Resultado del Backend | Actualización de UI |
|-------|----------------------|---------------------|
| HTTP 200 | Éxito | Lista actualizada |
| HTTP 400/404/500 | Error | Lista sin cambios |

**Valores límites**:
- Actualizar el primer ticket visible
- Actualizar el último ticket visible
- Actualizar un ticket en la segunda página

---

#### TC-013-020: Mensaje de error se muestra en caso de fallo

- **ID del Test**: TC-013-020
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que se muestra un mensaje de error descriptivo cuando falla el cambio de estado.
- **Precondiciones**: 
  - El modal está abierto
  - Se confirma un cambio de estado
  - El backend retorna un error (400, 404, o 500)

**Pasos (Gherkin)**:
```gherkin
Given el modal está abierto para un ticket
  And el administrador confirma un cambio de estado
When el backend retorna HTTP 404 con mensaje "Ticket no encontrado"
Then el modal permanece abierto
  And se muestra un mensaje de error dentro del modal o como notificación
  And el mensaje indica "Error: Ticket no encontrado" o equivalente
  And el mensaje se muestra con estilo visual de error (color rojo, icono de error)
  And el usuario puede reintentar o cancelar
```

**Partición de equivalencia**:
| Grupo | Código HTTP | Mensaje Esperado |
|-------|-------------|------------------|
| 400 | Bad Request | "Estado inválido. Valores permitidos: RECEIVED, IN_PROGRESS" |
| 404 | Not Found | "Ticket no encontrado" |
| 500 | Server Error | "Error interno del servidor. Intente nuevamente" |

**Valores límites**:
- Primer error tras 100 operaciones exitosas
- Múltiples errores consecutivos

**Tabla de Decisión**:
| # | Backend Status | Mostrar Error | Cerrar Modal | Actualizar Lista |
|---|----------------|---------------|--------------|------------------|
| 1 | 200 | ✗ | ✓ | ✓ |
| 2 | 400 | ✓ | ✗ | ✗ |
| 3 | 404 | ✓ | ✗ | ✗ |
| 4 | 500 | ✓ | ✗ | ✗ |

---

#### TC-013-021: Modal se cierra automáticamente después de éxito

- **ID del Test**: TC-013-021
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el modal se cierra automáticamente después de mostrar el mensaje de éxito, o permite cierre manual inmediato.
- **Precondiciones**: 
  - El cambio de estado fue exitoso
  - Se mostró el mensaje de confirmación

**Pasos (Gherkin)**:
```gherkin
Given el administrador confirmó un cambio de estado exitosamente
  And el backend retornó HTTP 200
  And se muestra el mensaje "Estado actualizado exitosamente"
When han transcurrido 3 segundos desde que se mostró el mensaje
Then el modal se cierra automáticamente
  And la notificación de éxito desaparece
Alternatively:
When el usuario hace clic en el botón X o fuera del modal antes de los 3 segundos
Then el modal se cierra inmediatamente
  And la notificación de éxito desaparece
```

**Partición de equivalencia**:
| Grupo | Cierre del Modal | Tipo |
|-------|------------------|------|
| Automático después de 3s | Timeout cumplido | Válido |
| Manual antes de 3s | Usuario cierra | Válido |
| Automático con error | No debe cerrar | Diferente flujo |

**Valores límites**:
- Exactamente a los 3 segundos
- Cierre manual a los 0.5s, 1s, 2s, 2.9s

---

#### TC-013-022: Flujo E2E completo de cambio de estado

- **ID del Test**: TC-013-022
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar el flujo end-to-end completo desde que el usuario abre el modal hasta que el estado se persiste en la base de datos (Prueba E2E).
- **Precondiciones**: 
  - Sistema completo desplegado y operativo
  - Base de datos con datos de prueba
  - Usuario administrador autenticado

**Pasos (Gherkin)**:
```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED" en la base de datos
  And un administrador autenticado está viendo la lista de tickets en el navegador
When el administrador localiza el ticket en la lista
  And hace clic en el botón "Cambiar Estado" del ticket
  And selecciona "IN_PROGRESS" en el selector del modal
  And hace clic en "Confirmar"
Then el frontend envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status con body { "status": "IN_PROGRESS" }
  And el Controller del Query Service recibe la request
  And el Service valida el ticketId y el nuevo estado
  And el Repository ejecuta UPDATE en la base de datos
  And la base de datos actualiza el registro con status = "IN_PROGRESS" y processed_at = NOW()
  And el Repository retorna el ticket actualizado
  And el Service retorna el ticket al Controller
  And el Controller retorna HTTP 200 con el ticket en JSON
  And el frontend actualiza la UI mostrando estado "IN_PROGRESS"
  And el modal se cierra automáticamente después de 3 segundos
  And se muestra el mensaje "Estado actualizado exitosamente"
  And consultando directamente la base de datos, el ticket tiene status = "IN_PROGRESS"
```

**Partición de equivalencia**:
| Grupo | Componentes Involucrados | Tipo |
|-------|--------------------------|------|
| Flujo completo exitoso | Frontend + Backend + DB | Válido |
| Error en Frontend | Frontend | Error |
| Error en Backend | Backend + DB | Error |
| Error en DB | DB | Error |

**Valores límites**:
- Primera transacción E2E del sistema
- 1000ª transacción E2E (carga sostenida)

---

#### TC-013-023: Múltiples cambios de estado consecutivos

- **ID del Test**: TC-013-023
- **ID de la Historia de Usuario**: UH-013
- **Descripción**: Verificar que el sistema maneja correctamente múltiples cambios de estado consecutivos en diferentes tickets.
- **Precondiciones**: 
  - Existen al menos 3 tickets en la lista
  - Todos los servicios están operativos

**Pasos (Gherkin)**:
```gherkin
Given existen 3 tickets en la lista:
  | ticketId | estado inicial |
  | ticket-1 | RECEIVED |
  | ticket-2 | IN_PROGRESS |
  | ticket-3 | RECEIVED |
When el administrador cambia el estado de "ticket-1" a "IN_PROGRESS"
  And espera a que se confirme la actualización
  And luego cambia el estado de "ticket-2" a "RECEIVED"
  And espera a que se confirme la actualización
  And luego cambia el estado de "ticket-3" a "IN_PROGRESS"
Then los 3 cambios se procesan exitosamente
  And la lista muestra:
  | ticketId | estado final |
  | ticket-1 | IN_PROGRESS |
  | ticket-2 | RECEIVED |
  | ticket-3 | IN_PROGRESS |
  And la base de datos refleja los 3 cambios correctamente
  And no hay inconsistencias entre UI y DB
```

**Partición de equivalencia**:
| Grupo | Cantidad de Cambios Consecutivos | Tipo |
|-------|----------------------------------|------|
| Un solo cambio | 1 | Válido |
| Pocos cambios | 2-5 | Válido |
| Muchos cambios | 10-50 | Válido |
| Cambios concurrentes | Simultáneos | Caso especial |

**Valores límites**:
- 2 cambios consecutivos (mínimo para "múltiples")
- 10 cambios consecutivos
- Cambios con 0s de espera entre ellos

---

*Documento generado el 26 de febrero de 2026*  
*Última actualización: 26/02/2026*
