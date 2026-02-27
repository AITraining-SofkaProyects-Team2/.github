# Documento de Historias de Usuario

## Proyecto: Taller Sistema Distribuido - Gestión de Quejas ISP

Este documento contiene las historias de usuario del sistema de gestión de quejas distribuido para un ISP. Cada historia de usuario sigue los principios INVEST (Independiente, Negociable, Valiosa, Estimable, Pequeña y Testeable) para asegurar que sean de alta calidad y útiles para los equipos de desarrollo.

---

## UH-13: Cambio de Estado de Tickets desde la Lista

### Solicitud del Usuario
El usuario solicita la capacidad de cambiar el estado de un ticket directamente desde la lista de tickets. Actualmente, la lista de tickets es de solo lectura y no permite realizar ninguna acción sobre los tickets visualizados. Los administradores necesitan poder actualizar el estado de los tickets para reflejar el progreso en la resolución de las quejas reportadas.

### Descripción General de la Historia de Usuario
Como administrador del sistema de quejas ISP, necesito poder cambiar el estado de un ticket desde la vista de listado para actualizar el progreso de resolución sin necesidad de navegar a otras pantallas o sistemas externos. Esta funcionalidad debe permitir transiciones entre todos los estados válidos del ciclo de vida del ticket (RECEIVED, IN_PROGRESS) mediante una interfaz modal intuitiva que solicite confirmación antes de aplicar el cambio.

### Requerimientos del Usuario

- **REQ-013-001**: El sistema debe permitir a los administradores cambiar el estado de cualquier ticket visible en la lista de tickets.
- **REQ-013-002**: El cambio de estado debe realizarse mediante una interfaz modal que muestre claramente el estado actual y el nuevo estado seleccionado.
- **REQ-013-004**: El sistema debe mostrar un mensaje de confirmación claro cuando el cambio de estado se complete exitosamente.
- **REQ-013-005**: El sistema debe mostrar un mensaje de error descriptivo cuando el cambio de estado falle por cualquier motivo.

### Requerimientos Funcionales

- **FUNC-013-001**: El Query Service debe exponer un endpoint PATCH `/api/tickets/:ticketId/status` que acepte el nuevo estado en el body de la request.
- **FUNC-013-002**: El endpoint debe validar que el `ticketId` proporcionado tenga formato UUIDv4 válido.
- **FUNC-013-003**: El endpoint debe verificar que el ticket existe en la base de datos antes de intentar actualizar el estado.
- **FUNC-013-004**: El endpoint debe validar que el nuevo estado sea uno de los valores válidos del dominio: `RECEIVED` o `IN_PROGRESS`.
- **FUNC-013-005**: El sistema debe permitir transiciones bidireccionales entre estados: `RECEIVED` ↔ `IN_PROGRESS`.
- **FUNC-013-006**: El frontend debe mostrar un botón o acción en cada fila de la lista de tickets que permita iniciar el cambio de estado.
- **FUNC-013-007**: Al hacer clic en el botón de cambio de estado, el sistema debe abrir un modal que muestre:
  - El ID del ticket
  - El estado actual del ticket
  - Un selector (dropdown) con los estados disponibles
  - Botones de "Confirmar" y "Cancelar"
- **FUNC-013-008**: Al confirmar el cambio en el modal, el frontend debe enviar una request PATCH al endpoint del Query Service con el nuevo estado.
- **FUNC-013-009**: Si la actualización es exitosa (HTTP 200), el frontend debe:
  - Actualizar el estado del ticket en la lista sin recargar toda la página
  - Mostrar un mensaje de confirmación tipo toast/notification
- **FUNC-013-010**: Si la actualización falla, el frontend debe:
  - Mantener el modal abierto
  - Mostrar el mensaje de error recibido del backend
- **FUNC-013-011**: El Repository debe implementar el método `updateStatus(ticketId: string, newStatus: TicketStatus): Promise<Ticket>` que ejecute la actualización en la base de datos.

### Requerimientos No Funcionales

- **NFUNC-013-001**: El endpoint de actualización de estado debe responder en menos de 500ms en el percentil 95.
- **NFUNC-013-002**: El endpoint debe implementar manejo de errores robusto siguiendo el patrón Chain of Responsibility establecido en el proyecto.
- **NFUNC-013-003**: La interfaz de usuario del modal debe ser accesible (WCAG 2.1 nivel AA) incluyendo navegación por teclado.
- **NFUNC-013-004**: El cambio de estado debe ser idempotente - cambiar un ticket al mismo estado que ya tiene no debe generar error.
- **NFUNC-013-005**: El Service y Repository deben seguir los principios SOLID establecidos en el proyecto, especialmente:
  - Responsabilidad Única (SRP): El servicio solo coordina, el repositorio solo persiste
  - Inversión de Dependencias (DIP): Uso de interfaces para abstraer implementaciones
- **NFUNC-013-006**: La implementación debe incluir cobertura de pruebas del 70% para la lógica de negocio relacionada con el cambio de estado.
- **NFUNC-013-007**: En caso de error de base de datos, el sistema debe retornar un error HTTP 500 con un mensaje genérico al cliente
- **NFUNC-013-008**: El modal debe implementarse como un componente React reutilizable que pueda ser usado en otros contextos del sistema.

### Criterios de Aceptación

#### Escenario 1: Cambio de estado exitoso
```gherkin
Given un administrador autenticado está viendo la lista de tickets
  And existe un ticket con ticketId "abc-123" en estado "RECEIVED"
When el administrador hace clic en el botón de cambiar estado del ticket
  And selecciona "IN_PROGRESS" en el modal
  And hace clic en "Confirmar"
Then el sistema envía PATCH /api/tickets/abc-123/status con body { "status": "IN_PROGRESS" }
  And el backend actualiza el ticket en la base de datos
  And el backend retorna HTTP 200 con el ticket actualizado
  And el frontend actualiza el estado del ticket en la lista
  And el frontend muestra el mensaje "Estado actualizado exitosamente"
  And el modal se cierra automáticamente después de 3 segundos
```

#### Escenario 2: Ticket no encontrado
```gherkin
Given un administrador autenticado está viendo la lista de tickets
When el administrador intenta cambiar el estado de un ticket con ID "invalid-uuid"
Then el backend retorna HTTP 404 con mensaje "Ticket no encontrado"
  And el frontend muestra el mensaje de error en el modal
  And el modal permanece abierto
```

#### Escenario 3: Estado inválido
```gherkin
Given un administrador autenticado está viendo la lista de tickets
  And existe un ticket con ticketId "abc-123"
When el administrador intenta cambiar el estado a "CLOSED" (valor no válido en el dominio actual)
Then el backend retorna HTTP 400 con mensaje "Estado inválido. Valores permitidos: RECEIVED, IN_PROGRESS"
  And el frontend muestra el mensaje de error
  And el modal permanece abierto
```

#### Escenario 4: Cancelar cambio de estado
```gherkin
Given un administrador autenticado abre el modal de cambio de estado
When el administrador hace clic en "Cancelar"
Then el modal se cierra sin realizar cambios
  And el estado del ticket permanece sin modificaciones
  And no se envía ninguna request al backend
```

#### Escenario 5: Cambio idempotente
```gherkin
Given un ticket con ticketId "abc-123" en estado "IN_PROGRESS"
When el administrador cambia el estado a "IN_PROGRESS" (mismo estado actual)
Then el backend acepta la operación y retorna HTTP 200
  And el campo processed_at se actualiza con la nueva timestamp
  And el frontend muestra el mensaje de confirmación
```

#### Métodos bajo prueba (TDD)

| Método | Estado del test |
|--------|-----------------|
| `TicketQueryService.updateTicketStatus(ticketId: string, newStatus: TicketStatus): Promise<Ticket>` | ✅ TC-013-001 (GREEN) — Implementado y validando UUID válido |
| | ✅ TC-013-002 (GREEN) — Validando rechazo de UUID inválido |
| | ✅ TC-013-003 (GREEN) — Validando aceptación y dominio de estados |
| | ✅ TC-013-004 (GREEN) — Validando rechazo de estados fuera del dominio |
| | ✅ TC-013-005 (GREEN) — Validando existencia del ticket antes de actualizar |
| `TicketsController.updateTicketStatus(req, res)` | ✅ TC-013-006 (GREEN) — Controlador mapea TicketNotFoundError a HTTP 404 |
| | ✅ TC-013-007 (GREEN) — Validando operación idempotente con mismo estado |
| `TicketRepository.updateStatus(ticketId, status)` | ✅ TC-013-008 (GREEN) — Validando envoltorio de errores de BD en DatabaseError |

### Notas Técnicas

#### Arquitectura del Endpoint
- **Ruta**: `PATCH /api/tickets/:ticketId/status`
- **Controller**: `TicketsController.updateTicketStatus(req, res)`
- **Service**: `TicketQueryService.updateTicketStatus(ticketId, newStatus)`
- **Repository**: `TicketRepository.updateStatus(ticketId, newStatus)`

#### Esquema del Body de la Request
```typescript
{
  "status": "RECEIVED" | "IN_PROGRESS"
}
```

#### Esquema de la Response (200 OK)
```typescript
{
  "ticketId": "uuid",
  "lineNumber": "string",
  "email": "string",
  "type": "enum",
  "description": "string | null",
  "status": "enum",          // Estado actualizado
  "priority": "enum",
  "createdAt": "ISO 8601",
  "processedAt": "ISO 8601"  // Timestamp actualizado
}
```

#### Posibles Errores
- **400 Bad Request**: 
  - Estado inválido
  - ticketId con formato incorrecto
  - Body de request malformado
- **404 Not Found**: Ticket no existe
- **500 Internal Server Error**: Error de base de datos u otro error inesperado

#### Actualización de Base de Datos
```sql
UPDATE tickets 
SET status = $1, processed_at = NOW() 
WHERE ticket_id = $2 
RETURNING *;
```

### Dependencias
- Esta historia depende de:
  - **HU-01**: Lista de tickets existente donde se agregará la funcionalidad

### Pruebas Requeridas

#### Pruebas Unitarias (Backend)
- `TC-013-001`: Validación de formato de ticketId — UUID válido
- `TC-013-002`: Validación de formato de ticketId — UUID inválido
- `TC-013-003`: Validación de estado válido (RECEIVED)
- `TC-013-004`: Validación de estado inválido — valores fuera del dominio
- `TC-013-005`: Actualización exitosa de RECEIVED a IN_PROGRESS
- `TC-013-006`: Error 404 — Ticket no encontrado
- `TC-013-007`: Actualización idempotente — mismo estado
- `TC-013-008`: Manejo de errores de base de datos

#### Pruebas de Integración (Backend)
- `TC-013-009`: Request PATCH completa con body válido retorna 200
- `TC-013-010`: Request PATCH con body inválido retorna 400

#### Pruebas de Componente (Frontend)
- `TC-013-013`: Modal se abre al hacer clic en botón de cambio de estado
- `TC-013-014`: Modal muestra información correcta del ticket (ID, estado actual)
- `TC-013-015`: Selector muestra todos los estados disponibles
- `TC-013-016`: Modal se cierra al hacer clic en Cancelar sin realizar cambios
- `TC-013-017`: Modal envía request correcta al confirmar cambio
- `TC-013-018`: Lista se actualiza correctamente después de cambio exitoso
- `TC-013-019`: Mensaje de error se muestra en caso de fallo
- `TC-013-020`: Modal se cierra automáticamente después de éxito

#### Pruebas E2E
- `TC-013-011`: Flujo E2E completo de cambio de estado
- `TC-013-012`: Múltiples cambios de estado consecutivos en diferentes tickets


---

