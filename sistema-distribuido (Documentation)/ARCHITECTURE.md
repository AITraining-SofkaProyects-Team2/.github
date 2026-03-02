# Reporte de Arquitectura de Software
**Proyecto:** Taller Sistema Distribuido - Gestión de Quejas ISP  
**Fecha:** 24 de febrero de 2026  
**Analizado por:** SoftwareArchitect Agent  

---

## 1. Resumen Ejecutivo
El proyecto implementa una arquitectura orientada a eventos con el objetivo de ser eficiente frente a usos intencivos:
```
          frontend (React) 
      ┌──────────┘└─────────┐ 
      ↓                     ↓ 
Producer (Express)    Reports-Query 
      ↓                     ↑ 
  RabbitMQ                  │ 
      ↓                     │ 
Consumer (worker)           │ 
      │                     │ 
      └───→ PostgreSQL ←────┘ 
```

La separación de responsabilidades y el uso de patrones (Singleton para conexiones, Facade para mensajería, Strategy para priorización) está presente en el diseño. 

El sistema completo está estructurado en tres servicios principales (Producer, Consumer, Reports-Query) y un frontend, una base de datos PostgreSQL y un Broker para la mensajería entre servicios. Se utiliza Docker Compose para orquestación local.

---

## 2. Stack Tecnológico Identificado
- Lenguaje: TypeScript 
- Frontend: React + Vite  
- Backend: Node.js + Express (Producer), Node.js worker (Consumer y Reports-Query)  
- Mensajería: RabbitMQ (rabbitmq:3-management-alpine)  
- Persistencia: Postgres (postgres:15-alpine)  
- Testing: Vitest  
- Orquestación local: Docker Compose

---

## 3. Evolución de la Infraestructura: De Monorepo a Multi-repo

El sistema nació diseñado bajo una arquitectura de microservicios, inicialmente gestionados dentro de un **Monorepo** (un único repositorio con carpetas independientes para cada servicio). Para mejorar el orden, la autonomía y el ciclo de vida de cada componente, se realizó la transición a una estructura de **Multi-repo** dentro de la organización `AITraining-SofkaProyects-Team2`.

### Actividad 1.1: Debate Arquitectónico

#### 3.1 Análisis del "Monorepo Inicial" (Dolores)
Aunque los microservicios ya estaban divididos lógicamente en carpetas (`backend/producer`, `backend/consumer`, etc.), el uso de un único repositorio compartido presentaba los siguientes inconvenientes operativos:
*   **Acoplamiento en el Pipeline**: Cada cambio en un servicio disparaba (o podía disparar) tests y validaciones de otros componentes no relacionados, ralentizando el feedback de CI.
*   **Gestión de Versiones y Tags**: Era complejo versionar y etiquetar individualmente cada microservicio (ej. desplegar la v1.2 del Producer mientras el Consumer seguía en v1.0).
*   **Dificultad de Orden y Acceso**: El crecimiento de la base de código bajo un solo árbol de directorios dificultaba la organización de permisos, revisiones de código y el enfoque de los desarrolladores en un solo dominio.
*   **Dependencia de Despliegue**: Los ciclos de despliegue solían estar atados a una única rama principal, dificultando la entrega continua independiente para cada servicio.

#### 3.2 Transición hacia Multi-repo y Clean Architecture (Beneficios)
La migración a repositorios independientes, manteniendo el enfoque de **Clean Architecture** interno, ha aportado los siguientes beneficios:
*   **Independencia de Ciclo de Vida**: Cada microservicio tiene ahora su propio Pipeline de CI/CD, permitiendo despliegues granulares y tags de versión totalmente independientes.
*   **Enfoque y Orden Organizacional**: La separación en la organización de GitHub permite una gestión de incidencias, documentación y tableros de proyecto específica por servicio.
*   **Resiliencia y Aislamiento**: Al estar en repositorios físicos distintos, se garantiza un aislamiento total en el desarrollo, evitando dependencias accidentales entre servicios mediante rutas de archivos locales.
*   **Testabilidad Optimizada**: El reporte de cobertura y las pruebas unitarias son ahora específicos para cada artefacto, facilitando el cumplimiento de los estándares de QA (>75% en Consumer, >90% en Producer).

#### 3.3 Nueva Estructura Multirepositorio
Los microservicios se han distribuido en los siguientes repositorios independientes:
*   **Frontend**: `Semana-03-frontend` (React + Vite)
*   **Producer**: `Semana-3-microservicio-Producer` (API Gateway / Mensajería)
*   **Consumer**: `Semana-3-microservicio-Consumer` (Worker / Lógica de Negocio)
*   **Reports-Query**: `Semana-3-microservicio-Reports-query` (Lectura / Dashboard)


---
<<<<<<< HEAD
## 4. Manejo Centralizado de Errores (Reports-Query)

El microservicio Reports-Query implementa un **patrón Chain of Responsibility** para el manejo de errores HTTP, garantizando que cada tipo de error se mapee a un código HTTP semántico correcto.

### Arquitectura del manejo de errores

```
Controlador (TicketsController)
    ↓
Servicio (TicketQueryService) → Lanza error específico
    ↓
asyncHandler → Captura el error y lo propaga
    ↓
errorHandler (middleware) → Evalúa tipo y mapea a HTTP
    ↓
Cliente ← Respuesta HTTP + JSON
```

### Flujo detallado

#### 1. **Servicio lanza error específico**
El servicio valida parámetros y lanza errores tipados:

```typescript
// src/services/TicketQueryService.ts
async findById(ticketId: string): Promise<Ticket> {
  if (!UUID_V4_REGEX.test(ticketId)) {
    throw new InvalidUuidFormatError();  // ❌ Error tipado
  }
  
  const ticket = await this.repository.findById(ticketId);
  if (ticket === null) {
    throw new TicketNotFoundError();     // ❌ Error tipado
  }
  
  return ticket;
}
```

#### 2. **Controlador delega sin try-catch**
El controlador NO intercepta errores, los deja propagarse:

```typescript
// src/controllers/ticketsController.ts
async getTicketById(req: Request, res: Response): Promise<void> {
  const { ticketId } = req.params;
  const ticket = await this.queryService.findById(ticketId);  // ← Error aquí
  res.status(200).json(ticket);
}
```

#### 3. **asyncHandler captura y propaga**
El wrapper captura promesas rechazadas y las envía al middleware:

```typescript
// src/utils/asyncHandler.ts
export const asyncHandler = (fn) => {
  return (req, res, next) => {
    fn(req, res, next).catch(next);  // ← Captura y propaga con next()
  };
};
```

#### 4. **errorHandler mapea a HTTP**
El middleware centralizado evalúa el tipo de error y retorna el código correcto:

```typescript
// src/middlewares/errorHandler.ts
export function errorHandler(err: Error, _req: Request, res: Response, _next: NextFunction): void {
  // Evaluación secuencial (Chain of Responsibility)
  
  // 400 — Validación con cuerpo personalizado
  if (err instanceof ValidationError) {
    res.status(400).json(err.responseBody);
    return;
  }

  // 400 — Formato de UUID inválido
  if (err instanceof InvalidUuidFormatError) {
    res.status(400).json({ error: err.message });
    return;
  }

  // 404 — Recurso no encontrado
  if (err instanceof TicketNotFoundError) {
    res.status(404).json({ error: err.message });
    return;
  }

  // N — Errores con propiedad status inyectada (extensible)
  if ('status' in err && typeof (err as any).status === 'number') {
    res.status((err as any).status).json({ error: err.message });
    return;
  }

  // 500 — Error desconocido / fallback seguro
  res.status(500).json({ error: 'Internal server error' });
}
```

### Tipos de errores y mapeo HTTP

| Tipo de Error | Código HTTP | Ejemplo |
|---|:---:|---|
| `ValidationError` | 400 | Parámetros de query inválidos (`status=INVALID`) |
| `InvalidUuidFormatError` | 400 | UUID malformado en `/api/tickets/not-a-uuid` |
| `InvalidTicketStatusError` | 400 | Status no permitido al actualizar |
| `TicketNotFoundError` | 404 | Ticket no existe: `/api/tickets/550e8400-e29b-41d4-a716-446655440000` |
| Error desconocido | 500 | Cualquier error no capturado por los tipos anteriores |

### Ejemplo completo: GET /api/tickets/:ticketId

**Caso 1: UUID válido, ticket existe → 200**
```
GET /api/tickets/550e8400-e29b-41d4-a716-446655440000
→ TicketQueryService.findById() valida UUID ✓
→ Busca en DB ✓
→ Retorna ticket
→ Respuesta: 200 + JSON del ticket
```

**Caso 2: UUID inválido → 400**
```
GET /api/tickets/invalid-uuid
→ InvalidUuidFormatError lanzado
→ asyncHandler lo captura
→ errorHandler verifica tipo
→ Respuesta: 400 + { error: "Invalid UUID format" }
```

**Caso 3: UUID válido, ticket no existe → 404**
```
GET /api/tickets/550e8400-e29b-41d4-a716-446655440000
→ TicketQueryService valida UUID ✓
→ Busca en DB, no encuentra
→ TicketNotFoundError lanzado
→ asyncHandler lo captura
→ errorHandler verifica tipo
→ Respuesta: 404 + { error: "Ticket not found" }
```

### Ventajas de esta arquitectura

✅ **Controladores limpios** — Sin try-catch anidados, solo lógica de negocio  
✅ **Manejo centralizado** — Un único lugar para mapear errores a HTTP  
✅ **Extensible** — Agregar nuevos tipos de error es simple (agregar otro `if` al errorHandler)  
✅ **Semántico** — Cada error → código HTTP correcto  
✅ **Fallback seguro** — Errores desconocidos → 500, nunca falla silenciosamente  
✅ **Testeable** — Cada tipo de error se puede testear independientemente

---
## 5. Endpoints de servicios

### Producer - `POST /complaints`

url = `/complaints`  
http = POST  
códigos de estado:
- ~~`201 Created`~~ => `202 Accepted`: ticket recibido y en proceso
- `500`: error interno
- `503`: servicio no disponible (RabbitMQ down)

Cuerpo de la petición (JSON):
```json
{
  "ticketId": "1234567890",
  "status": "RECEIVED",
  "message": "Accepted for processing",
  "createdAt": "2024-01-01T00:00:00Z"
}
```

**Descripción:**  
El endpoint `POST /complaints` del Producer pasa de responder `201 Created` a `202 Accepted` cuando la creación implica encolado asíncrono del evento.

**Motivo:**  
La operación principal del Producer es aceptar la petición y encolar un evento para el Consumer; la persistencia la realiza el Consumer. Responder `202` permite que el Producer confirme recepción sin bloquearse por la confirmación del broker.

**Comportamiento y contrato del endpoint después del cambio:**
- Endpoint: `POST /complaints`
- Código HTTP: `202 Accepted`
- Cuerpo de respuesta (JSON):
  - `ticketId` (string): identificador único del ticket generado.
  - `status` (string): valor inicial `RECEIVED` (indica que el ticket fue aceptado para procesamiento).
  - `message` (string): texto breve — ejemplo: `Accepted for processing`.
  - `createdAt` (string, ISO-8601): timestamp de creación del ticket.

---

### Reports-query

### Antes de la refactorización
| Endpoint | 200 | 400 | 404 | 405 | 500 |
|----------|:---:|:---:|:---:|:---:|:---:|
| `GET /health` | ✅ | — | — | — | — |
| `GET /api/tickets` | ✅ | ❌ Sin validación de params | — | — | ✅ |
| `GET /api/tickets/metrics` | ✅ | — | — | — | ✅ |
| `GET /api/tickets/line/:lineNumber` | ✅ | ❌ Error genérico → 500 | — | — | ✅ |
| `GET /api/tickets/:ticketId` | ✅ | ✅ InvalidUuidFormatError | ✅ TicketNotFoundError | — | ✅ |
| `POST /api/tickets` | — | — | — | ❌ Sin handler | — |
| Rutas desconocidas | — | — | ❌ Sin catch-all | — | — |

### Después de la refactorización
| Endpoint | 200 | 400 | 404 | 405 | 500 |
|----------|:---:|:---:|:---:|:---:|:---:|
| `GET /health` | ✅ | — | — | — | — |
| `GET /api/tickets` | ✅ | ✅ Validación completa | — | — | ✅ |
| `GET /api/tickets/metrics` | ✅ | — | — | — | ✅ |
| `GET /api/tickets/line/:lineNumber` | ✅ | ✅ ValidationError | — | — | ✅ |
| `GET /api/tickets/:ticketId` | ✅ | ✅ InvalidUuidFormatError | ✅ TicketNotFoundError | — | ✅ |
| `POST/PUT/DELETE /api/*` | — | — | — | ✅ 405 + `Allow: GET` | — |
| Rutas desconocidas | — | — | ✅ 404 JSON | — | — |


### `GET /health`
![reports-get-health-200](./evidencias%20postman/reports-get-health-200.png)

### `GET /api/tickets`
#### 200 OK 
![reports-get-api-tickets-200](./evidencias%20postman/reports-get-api-tickets-200.png)

#### 400 Bad Request
![reports-get-api-tickets-400](./evidencias%20postman/reports-get-api-tickets-400.png)

#### 500 Internal Server Error
![reports-get-api-tickets-500](./evidencias%20postman/reports-get-api-tickets-500.png)


### `GET /api/tickets/metrics`
#### 200 OK
![reports-get-api-tickets-metrics-200](./evidencias%20postman/reports-get-api-tickets-metrics-200.png)

#### 500 Internal Server Error
![reports-get-api-tickets-metrics-500](./evidencias%20postman/reports-get-api-tickets-metrics-500.png)

### `GET /api/tickets/line/:lineNumber`
#### 200 OK
![reports-get-api-tickets-line-200](./evidencias%20postman/reports-get-api-tickets-line-200.png)

#### 400 Bad Request
![reports-get-api-tickets-line-400](./evidencias%20postman/reports-get-api-tickets-line-400.png)

#### 500 Internal Server Error
![reports-get-api-tickets-metrics-500](./evidencias%20postman/reports-get-api-tickets-metrics-500.png)

### `GET /api/tickets/:ticketId`
#### 200 OK
![reports-get-api-tickets-id-200](./evidencias%20postman/reports-get-api-tickets-id-200.png)

#### 400 Bad Request
![reports-get-api-tickets-id-400](./evidencias%20postman/reports-get-api-tickets-id-400.png)

#### 404 Not Found
![reports-get-api-tickets-id-404](./evidencias%20postman/reports-get-api-tickets-id-404.png)

#### 500 Internal Server Error 
![reports-get-api-tickets-id-500](./evidencias%20postman/reports-get-api-tickets-id-500.png)

### `POST/PUT/DELETE /api/*`
#### 405 Method Not Allowed
![reports-post-api-_](./evidencias%20postman/reports-post-api-_.png)

![reports-delete-api-_](./evidencias%20postman/reports-delete-api-_.png)

![reports-put-api-_](./evidencias%20postman/reports-put-api-_.png)

### Rutas desconocidas
#### 404 Not Found
![ruta no encontrada](./evidencias%20postman/rutaNoEncontrada.png)
=======
## 4. Endpoints de Servicios (Contrato Actual)

### Producer Microservice
Servicio de entrada para nuevas quejas ISP.

*   **Endpoint**: `POST /complaints`
*   **Códigos de Estado**:
    *   `202 Accepted`: Ticket validado y encolado en RabbitMQ.
    *   `400 Bad Request`: Fallo en validación (email, tipo, descripción faltante en 'OTHER').
    *   **503 Service Unavailable**: Broker de mensajería no disponible.
*   **Payload**: `{ lineNumber, email, incidentType, description }`

### Consumer Microservice (Worker)
Procesa mensajes asíncronos y persiste en PostgreSQL.

*   **Responsabilidad**: Cálculo de prioridad, determinación de estado y persistencia.
*   **Manejo de Errores**: Reintento exponencial (hasta 3 veces) antes de enviar a **Dead Letter Queue (DLQ)**.

### Reports-query Microservice
Servicio para consulta y generación de reportes de tickets.

*   **Endpoints**:
    *   `GET /api/tickets`: Listado con filtros y paginación.
    *   `GET /api/tickets/metrics`: Agregaciones operacionales.
    *   `GET /api/tickets/line/:lineNumber`: Histórico por línea cliente.
    *   `GET /api/tickets/:ticketId`: Detalle de un ticket específico.
    *   `PATCH /api/tickets/:ticketId/status`: Actualización manual de estado (Gestión operativa).
*   **Validación**: Esquema estricto para filtros de consulta, retornando `400 Bad Request` en parámetros inválidos.

---

## 5. Diseño de Base de Datos y Comunicación

*   **PostgreSQL**: Tabla central `tickets` compartida por Consumer (Escritura) y Reports-query (Lectura).
*   **RabbitMQ**: Comunicación unidireccional `Producer → Consumer` mediante colas durables.
*   **Health Checks**: Cada microservicio expone su estado interno (conexión a broker/DB y métricas operacionales) para observabilidad.
>>>>>>> 1d1e95a9680320cb73e979b4a036048710a0d986
