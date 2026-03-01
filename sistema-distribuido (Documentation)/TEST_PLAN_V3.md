# 🧪 Plan de Pruebas V3 — API REST & Pruebas de Integración

**Proyecto:** Sistema Distribuido de Gestión de Quejas ISP  
**Versión del documento:** 3.0  
**Fecha de creación:** 27 de febrero de 2026  
**Basado en:** [TEST_PLAN.md](./TEST_PLAN.md), [ARCHITECTURE.md](./ARCHITECTURE.md), [REQUERIMIENTOS_COMPLETOS.md](./REQUERIMIENTOS_COMPLETOS.md), [TESTS_DESIGNS.md](./TESTS_DESIGNS.md), [CALIDAD.md](./CALIDAD.md), [FASE_3_HISTORIAS_RIESGOS.md](./FASE_3_HISTORIAS_RIESGOS.md), [UHs_INVEST.md](./UHs_INVEST.md)

---

## Tabla de Contenidos

1. [Introducción y Propósito](#1-introducción-y-propósito)
2. [Alcance del Plan de Pruebas](#2-alcance-del-plan-de-pruebas)
3. [Estrategia de Calidad](#3-estrategia-de-calidad)
4. [Niveles de Prueba](#4-niveles-de-prueba)
5. [Catálogo de Endpoints de la API REST](#5-catálogo-de-endpoints-de-la-api-rest)
6. [Diseño de Pruebas de Integración — Endpoints Nuevos (Reports-Query)](#6-diseño-de-pruebas-de-integración--endpoints-nuevos-reports-query)
7. [Diseño de Pruebas de Integración — Endpoints Antiguos (Producer)](#7-diseño-de-pruebas-de-integración--endpoints-antiguos-producer)
8. [Matriz de Trazabilidad: Historias ↔ Pruebas de Integración](#8-matriz-de-trazabilidad-historias--pruebas-de-integración)
9. [Gestión de Riesgos](#9-gestión-de-riesgos)
10. [Herramientas y Entorno de Pruebas](#10-herramientas-y-entorno-de-pruebas)
11. [Calendario de Pruebas](#11-calendario-de-pruebas)
12. [Criterios de Entrada y Salida](#12-criterios-de-entrada-y-salida)
13. [Registro de Ejecución Manual](#13-registro-de-ejecución-manual)
14. [Principios INVEST Aplicados a las Historias de Prueba](#14-principios-invest-aplicados-a-las-historias-de-prueba)
15. [Apéndice: Valores de Referencia del Dominio](#15-apéndice-valores-de-referencia-del-dominio)

---

## 1. Introducción y Propósito

### 1.1 Propósito del Documento

Este documento consolida el **Plan de Pruebas V2** del sistema distribuido de gestión de quejas ISP, con enfoque específico en:

- **Pruebas de integración** para todos los endpoints de la API REST (nuevos y existentes).
- **Estrategia de calidad** integral del proyecto.
- **Gestión de riesgos** diferenciando riesgos de proyecto vs. riesgos de producto.
- **Registro de ejecución manual** con resultado (Pasó/Falló).
- **Aplicación de principios INVEST** en las historias relacionadas con pruebas.

Este plan complementa al [TEST_PLAN.md](./TEST_PLAN.md) original (enfocado en pruebas unitarias y metodología TDD) y centraliza todo lo referente a pruebas de integración de la API.

### 1.2 Diferencia con TEST_PLAN.md

| Aspecto | TEST_PLAN.md (V1) | TEST_PLAN_V2.md (Este documento) |
|---------|-------------------|----------------------------------|
| **Enfoque** | Pruebas unitarias, metodología TDD (RED-GREEN-REFACTOR) | Pruebas de integración de API REST |
| **Alcance** | Lógica de negocio aislada | Endpoints HTTP completos (request → response) |
| **Servicios** | Query Service (lógica interna) | Producer + Reports-Query (endpoints HTTP) |
| **Riesgos** | Referencia a matriz general | Gestión detallada (Proyecto vs. Producto) |
| **Ejecución** | Automatizada (Vitest) | Automatizada + registro manual |

### 1.3 Documentos de Referencia

| Documento | Relación |
|-----------|----------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Arquitectura, stack tecnológico, estructura de servicios |
| [REQUERIMIENTOS_COMPLETOS.md](./REQUERIMIENTOS_COMPLETOS.md) | Requerimientos funcionales y no funcionales |
| [FASE_3_HISTORIAS_RIESGOS.md](./FASE_3_HISTORIAS_RIESGOS.md) | Historias de usuario y matriz de riesgos original |
| [TESTS_DESIGNS.md](./TESTS_DESIGNS.md) | Diseño detallado de casos de prueba (UH-013) |
| [CALIDAD.md](./CALIDAD.md) | Pirámide de pruebas y estrategia de calidad |
| [DEUDA_TECNICA.md](./DEUDA_TECNICA.md) | Deuda técnica activa y pagada |
| [UHs_INVEST.md](./UHs_INVEST.md) | Historias de usuario con principios INVEST |

---

## 2. Alcance del Plan de Pruebas

### 2.1 Dentro del Alcance

| Elemento | Descripción |
|----------|-------------|
| **Reports-Query Service** | Todos los endpoints `GET /api/tickets`, `GET /api/tickets/:id`, `GET /api/tickets/metrics`, `PATCH /api/tickets/:ticketId/status` |
| **Producer Service** | Endpoint `POST /complaints` (antiguo), endpoint `POST /api/v1/tikets` (legacy) |
| **Pruebas de integración HTTP** | Request completa → Controller → Service → Repository (mock de BD) → Response |
| **Validación de contratos** | Esquemas de request/response, códigos HTTP, headers |
| **Combinación de filtros** | Query params combinados en endpoints de consulta |
| **Gestión de errores** | Respuestas de error (400, 404, 500, 503) |
| **Paginación** | Parámetros `page`, `pageSize`, límites |
| **Ordenamiento** | Parámetros `sortBy`, `sortOrder` |

### 2.2 Fuera del Alcance

| Elemento | Razón |
|----------|-------|
| Pruebas E2E completas (Frontend → RabbitMQ → Consumer) | Se cubren en `TEST_PLAN.md` y suite E2E existente |
| Pruebas de rendimiento / carga | Requiere herramientas especializadas (fase futura) |
| Pruebas de seguridad / penetración | No hay autenticación en esta fase |
| Pruebas de UI / componentes frontend | Se cubren en diseño de pruebas de `TESTS_DESIGNS.md` |
| Comunicación con RabbitMQ | Nivel E2E, fuera de integración HTTP |

### 2.3 Arquitectura de Servicios Bajo Prueba

```
                  ┌─────────────────────────────────────────────┐
                  │              Pruebas de Integración          │
                  │                                             │
  HTTP Request ──►│  Controller ──► Service ──► Repository      │──► HTTP Response
                  │       ▲            ▲            ▲            │
                  │       │            │            │(mock)      │
                  │   Validación   Lógica de    Base de Datos   │
                  │   de entrada   negocio      (mock/test DB)  │
                  └─────────────────────────────────────────────┘
```

**Servicios y puertos:**

| Servicio | Puerto | Rol en Pruebas de Integración |
|----------|:------:|-------------------------------|
| **Producer** | 3000 | Endpoint `POST /complaints` — validación de requests, respuesta HTTP |
| **Reports-Query** | 3002 | Endpoints de consulta y actualización de tickets |
| **PostgreSQL** | 5432 | Mocked en pruebas de integración; real en E2E |
| **RabbitMQ** | 5672 | Mocked en pruebas de integración del Producer |

---

## 3. Estrategia de Calidad

### 3.1 Objetivos de Calidad

| Objetivo | Métrica | Meta |
|----------|---------|------|
| **Cobertura de código** | % de líneas cubiertas en servicios backend | ≥ 100% en lógica de negocio |
| **Cobertura de endpoints** | % de endpoints con pruebas de integración | 100% |
| **Cobertura de códigos HTTP** | % de códigos de estado probados por endpoint | ≥ 90% |
| **Tiempo de respuesta** | Latencia p95 por endpoint | < 500ms para 50-80 tickets |
| **Tasa de defectos** | Defectos encontrados post-release | < 2 por sprint |
| **Regresiones** | Tests que fallan tras cambios | 0 regresiones no detectadas |

### 3.2 Pirámide de Pruebas — Distribución Objetivo

```
            ╱╲
           ╱E2E╲          ~5%   (flujo crítico completo)
          ╱──────╲
         ╱Integra-╲       ~25%  (endpoints HTTP, contratos)    ◄── FOCO DE ESTE DOCUMENTO
        ╱──ción────╲
       ╱────────────╲
      ╱  Unitarias   ╲    ~70%  (lógica de negocio aislada)
     ╱────────────────╲
```

### 3.3 Enfoque de Pruebas de Integración

Las pruebas de integración de API validan la **interacción entre capas** dentro de cada servicio:

1. **Request HTTP** → Llega al framework Express.
2. **Router** → Enruta al controller correcto.
3. **Controller** → Parsea parámetros, invoca al service.
4. **Service** → Aplica lógica de negocio, invoca al repository.
5. **Repository** → Interactúa con la fuente de datos (mock en integración).
6. **Response** → Controller serializa y devuelve la respuesta HTTP.

**Se utilizará `supertest`** para realizar requests HTTP contra la aplicación Express sin levantar un servidor real, permitiendo pruebas rápidas y deterministas.

### 3.4 Principios de Calidad del Proyecto

| Principio | Aplicación |
|-----------|------------|
| **Fail Fast** | Validaciones tempranas en controller y service; errores claros |
| **Defense in Depth** | Validación en múltiples capas (controller, service, repository) |
| **Shift Left** | Detección de defectos lo antes posible en el ciclo de desarrollo |
| **Testing as Documentation** | Los tests describen el comportamiento esperado de cada endpoint |
| **Deterministic Tests** | Mocks de BD y servicios externos para resultados predecibles |

---

## 4. Niveles de Prueba

### 4.1 Resumen de Niveles

| Nivel | Qué valida | Herramienta | Responsable | Documento |
|-------|-----------|-------------|-------------|-----------|
| **Unitario** | Lógica de negocio aislada (Strategy, Validators, Handlers) | Vitest + vi.mock | Desarrollador | TEST_PLAN.md |
| **Integración de Componentes** | Endpoint HTTP completo: request → controller → service → repository (mock) → response | Vitest + supertest | Desarrollador | **Este documento** |
| **Integración de Sistema** | Comunicación entre microservicios (Producer → RabbitMQ → Consumer → DB) | Docker Compose + scripts | QA / Desarrollador | Este documento (sección 7) |
| **E2E** | Flujo completo desde Frontend hasta persistencia en BD | Docker Compose + Vitest | QA | TEST_PLAN.md |

### 4.2 Detalle del Nivel de Integración de Componentes (API)

**Objetivo:** Verificar que cada endpoint de la API REST responda correctamente ante diferentes combinaciones de inputs, validando:

- Códigos de estado HTTP correctos
- Estructura de respuesta (JSON schema)
- Manejo de errores y mensajes
- Headers de respuesta
- Paginación y metadatos
- Combinación de query params (filtros)

**Configuración de prueba:**

```typescript
// Ejemplo de setup para pruebas de integración
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { app } from '../src/app'; // Aplicación Express

describe('GET /api/tickets', () => {
  it('debe retornar 200 con lista paginada', async () => {
    const response = await request(app)
      .get('/api/tickets')
      .query({ page: 1, pageSize: 10 });
    
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('data');
    expect(response.body).toHaveProperty('pagination');
  });
});
```

---

## 5. Catálogo de Endpoints de la API REST

### 5.1 Reports-Query Service (Nuevos Endpoints)

| ID | Método | Ruta | Descripción | HU Relacionada |
|----|--------|------|-------------|----------------|
| EP-RQ-01 | `GET` | `/api/tickets` | Listado paginado con filtros y ordenamiento | HU-01 a HU-05, HU-08 |
| EP-RQ-02 | `GET` | `/api/tickets/:id` | Búsqueda por ID de ticket (UUIDv4) | HU-06 |
| EP-RQ-03 | `GET` | `/api/tickets/metrics` | Métricas agregadas (totales, distribuciones) | HU-09 |
| EP-RQ-04 | `PATCH` | `/api/tickets/:ticketId/status` | Cambio de estado de un ticket | UH-013 |

#### EP-RQ-01: GET /api/tickets

**Query Parameters:**

| Parámetro | Tipo | Obligatorio | Valores Válidos | Default |
|-----------|------|:-----------:|-----------------|---------|
| `page` | number | No | ≥ 1 | 1 |
| `pageSize` | number | No | 1-20 | 10 |
| `status` | string | No | `RECEIVED`, `IN_PROGRESS` | — |
| `priority` | string | No | `HIGH`, `MEDIUM`, `LOW`, `PENDING` | — |
| `incidentType` | string | No | `NO_SERVICE`, `INTERMITTENT_SERVICE`, `SLOW_CONNECTION`, `ROUTER_ISSUE`, `BILLING_QUESTION`, `OTHER` | — |
| `startDate` | string (ISO-8601) | No | Fecha válida | — |
| `endDate` | string (ISO-8601) | No | Fecha ≥ startDate | — |
| `lineNumber` | string | No | 10 dígitos numéricos | — |
| `sortBy` | string | No | `createdAt`, `priority`, `status` | `createdAt` |
| `sortOrder` | string | No | `asc`, `desc` | `desc` |

**Response (200 OK):**

```json
{
  "data": [
    {
      "ticketId": "UUID",
      "lineNumber": "string",
      "email": "string",
      "type": "enum",
      "description": "string | null",
      "status": "enum",
      "priority": "enum",
      "createdAt": "ISO-8601",
      "processedAt": "ISO-8601"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 10,
    "totalItems": 50,
    "totalPages": 5
  }
}
```

#### EP-RQ-02: GET /api/tickets/:id

**Path Parameters:**

| Parámetro | Tipo | Validación |
|-----------|------|------------|
| `id` | string | Formato UUIDv4 |

**Responses:**

| Código | Condición | Body |
|--------|-----------|------|
| 200 | Ticket encontrado | Objeto ticket completo |
| 400 | ID con formato inválido | `{ "error": "Formato de ID inválido" }` |
| 404 | Ticket no existe | `{ "error": "Ticket no encontrado" }` |

#### EP-RQ-03: GET /api/tickets/metrics

**Query Parameters:** Mismos filtros que `GET /api/tickets` (sin paginación ni ordenamiento).

**Response (200 OK):**

```json
{
  "totalTickets": 150,
  "byStatus": { "RECEIVED": 80, "IN_PROGRESS": 70 },
  "byPriority": { "HIGH": 30, "MEDIUM": 50, "LOW": 40, "PENDING": 30 },
  "byType": {
    "NO_SERVICE": 25,
    "INTERMITTENT_SERVICE": 30,
    "SLOW_CONNECTION": 20,
    "ROUTER_ISSUE": 25,
    "BILLING_QUESTION": 30,
    "OTHER": 20
  }
}
```

#### EP-RQ-04: PATCH /api/tickets/:ticketId/status

**Path Parameters:**

| Parámetro | Tipo | Validación |
|-----------|------|------------|
| `ticketId` | string | Formato UUIDv4 |

**Request Body:**

```json
{ "status": "RECEIVED" | "IN_PROGRESS" }
```

**Responses:**

| Código | Condición | Body |
|--------|-----------|------|
| 200 | Actualización exitosa | Ticket completo actualizado |
| 400 | ticketId inválido, estado inválido, body malformado | `{ "error": "mensaje descriptivo" }` |
| 404 | Ticket no existe | `{ "error": "Ticket no encontrado" }` |
| 500 | Error de BD | `{ "error": "Error interno del servidor" }` |

### 5.2 Producer Service (Endpoints Existentes)

| ID | Método | Ruta | Descripción |
|----|--------|------|-------------|
| EP-PR-01 | `POST` | `/complaints` | Creación de queja (encola evento a RabbitMQ) |

#### EP-PR-01: POST /complaints

**Request Body:**

```json
{
  "lineNumber": "string (10 dígitos)",
  "email": "string (email válido)",
  "incidentType": "NO_SERVICE | INTERMITTENT_SERVICE | SLOW_CONNECTION | ROUTER_ISSUE | BILLING_QUESTION | OTHER",
  "description": "string | null (obligatorio si incidentType === OTHER)"
}
```

**Responses:**

| Código | Condición | Body |
|--------|-----------|------|
| 202 | Queja aceptada para procesamiento | `{ "ticketId": "UUID", "status": "RECEIVED", "message": "Accepted for processing", "createdAt": "ISO-8601" }` |
| 400 | Validación fallida | `{ "error": "mensaje de validación" }` |
| 500 | Error interno | `{ "error": "Internal server error" }` |
| 503 | RabbitMQ no disponible | `{ "error": "Service unavailable" }` |

---

## 6. Diseño de Pruebas de Integración — Endpoints Nuevos (Reports-Query)

### 6.1 GET /api/tickets — Listado Paginado con Filtros

#### IT-RQ-001: Listado por defecto sin filtros retorna 200 con paginación

- **ID:** IT-RQ-001
- **Endpoint:** `GET /api/tickets`
- **HU:** HU-01
- **Tipo:** Integración de componentes
- **Precondiciones:** Base de datos con ≥ 15 tickets de prueba

```gherkin
Given el endpoint GET /api/tickets está disponible
  And existen 15 tickets en la base de datos
When se envía GET /api/tickets sin query params
Then el código HTTP es 200
  And el body contiene "data" como array con ≤ 10 elementos (pageSize default)
  And el body contiene "pagination" con { page: 1, pageSize: 10, totalItems: 15, totalPages: 2 }
  And cada ticket en "data" contiene los campos: ticketId, lineNumber, email, type, description, status, priority, createdAt, processedAt
  And los tickets están ordenados por createdAt DESC (default)
```

| Partición | Valor | Resultado Esperado |
|-----------|-------|--------------------|
| Sin parámetros | — | 200, page 1, pageSize 10 |
| Con page=2 | `?page=2` | 200, segunda página |
| pageSize personalizado | `?pageSize=5` | 200, 5 elementos |
| pageSize máximo | `?pageSize=20` | 200, hasta 20 elementos |
| pageSize > máximo | `?pageSize=50` | 200, limitado a 20 |
| page inexistente | `?page=999` | 200, data vacío |

---

#### IT-RQ-002: Filtro por estado retorna solo tickets del estado indicado

- **ID:** IT-RQ-002
- **Endpoint:** `GET /api/tickets?status=RECEIVED`
- **HU:** HU-02
- **Precondiciones:** Base de datos con tickets en ambos estados

```gherkin
Given existen tickets en estados RECEIVED e IN_PROGRESS
When se envía GET /api/tickets?status=RECEIVED
Then el código HTTP es 200
  And todos los tickets en "data" tienen status = "RECEIVED"
  And "pagination.totalItems" refleja solo los tickets RECEIVED
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| Estado válido existente | `status=RECEIVED` | 200, solo RECEIVED |
| Estado válido existente | `status=IN_PROGRESS` | 200, solo IN_PROGRESS |
| Estado inválido | `status=CLOSED` | 400, error de validación |
| Estado vacío | `status=` | 200, sin filtro (todos) |

---

#### IT-RQ-003: Filtro por prioridad retorna tickets de la prioridad indicada

- **ID:** IT-RQ-003
- **Endpoint:** `GET /api/tickets?priority=HIGH`
- **HU:** HU-03
- **Precondiciones:** Base de datos con tickets de diferentes prioridades

```gherkin
Given existen tickets con prioridades HIGH, MEDIUM, LOW y PENDING
When se envía GET /api/tickets?priority=HIGH
Then el código HTTP es 200
  And todos los tickets en "data" tienen priority = "HIGH"
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| Prioridad válida | `priority=HIGH` | 200, solo HIGH |
| Prioridad válida | `priority=MEDIUM` | 200, solo MEDIUM |
| Prioridad inválida | `priority=CRITICAL` | 400, error |
| Sin tickets | `priority=LOW` (sin datos) | 200, data vacío |

---

#### IT-RQ-004: Filtro por tipo de incidente

- **ID:** IT-RQ-004
- **Endpoint:** `GET /api/tickets?incidentType=NO_SERVICE`
- **HU:** HU-04
- **Precondiciones:** Tickets con variedad de tipos de incidente

```gherkin
Given existen tickets con diferentes tipos de incidente
When se envía GET /api/tickets?incidentType=NO_SERVICE
Then el código HTTP es 200
  And todos los tickets en "data" tienen type = "NO_SERVICE"
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| Tipo válido | `incidentType=NO_SERVICE` | 200, filtrado |
| Tipo válido | `incidentType=OTHER` | 200, filtrado |
| Tipo inválido | `incidentType=HACKING` | 400, error |

---

#### IT-RQ-005: Filtro por rango de fechas

- **ID:** IT-RQ-005
- **Endpoint:** `GET /api/tickets?startDate=...&endDate=...`
- **HU:** HU-05
- **Precondiciones:** Tickets con fechas variadas

```gherkin
Given existen tickets creados en diferentes fechas
When se envía GET /api/tickets?startDate=2026-02-01T00:00:00Z&endDate=2026-02-15T23:59:59Z
Then el código HTTP es 200
  And todos los tickets en "data" tienen createdAt dentro del rango
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| Rango válido con datos | startDate < endDate, hay datos | 200, filtrado |
| Rango válido sin datos | startDate < endDate, sin datos | 200, data vacío |
| Rango invertido | startDate > endDate | 400, error de validación |
| Solo startDate | Solo startDate | 200, desde fecha hasta ahora |
| Solo endDate | Solo endDate | 200, desde inicio hasta fecha |
| Fecha inválida | `startDate=no-es-fecha` | 400, error |

---

#### IT-RQ-006: Filtro por número de línea

- **ID:** IT-RQ-006
- **Endpoint:** `GET /api/tickets?lineNumber=3001234567`
- **HU:** HU-07
- **Precondiciones:** Tickets con diferentes números de línea

```gherkin
Given existen tickets con lineNumber "3001234567" y otros números
When se envía GET /api/tickets?lineNumber=3001234567
Then el código HTTP es 200
  And todos los tickets en "data" tienen lineNumber = "3001234567"
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| Número válido con resultados | `lineNumber=3001234567` | 200, filtrado |
| Número válido sin resultados | `lineNumber=9999999999` | 200, data vacío |
| Formato inválido (letras) | `lineNumber=abc` | 400, error |
| Formato corto | `lineNumber=123` | 400, error |

---

#### IT-RQ-007: Ordenamiento por diferentes campos

- **ID:** IT-RQ-007
- **Endpoint:** `GET /api/tickets?sortBy=priority&sortOrder=asc`
- **HU:** HU-08
- **Precondiciones:** Tickets con datos variados

```gherkin
Given existen tickets con diferentes prioridades y fechas
When se envía GET /api/tickets?sortBy=priority&sortOrder=asc
Then el código HTTP es 200
  And los tickets están ordenados por prioridad ascendente
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| Orden por fecha ASC | `sortBy=createdAt&sortOrder=asc` | 200, ordenado |
| Orden por fecha DESC | `sortBy=createdAt&sortOrder=desc` | 200, ordenado |
| Orden por prioridad | `sortBy=priority&sortOrder=asc` | 200, ordenado |
| Orden por estado | `sortBy=status&sortOrder=desc` | 200, ordenado |
| Campo inválido | `sortBy=email` | 400, error |
| sortOrder inválido | `sortOrder=random` | 400, error |

---

#### IT-RQ-008: Combinación de múltiples filtros

- **ID:** IT-RQ-008
- **Endpoint:** `GET /api/tickets?status=RECEIVED&priority=HIGH&sortBy=createdAt&sortOrder=desc`
- **HU:** HU-01-HU-08
- **Precondiciones:** Base de datos diversificada

```gherkin
Given existen tickets con variedad de estados, prioridades y tipos
When se envía GET /api/tickets?status=RECEIVED&priority=HIGH&incidentType=NO_SERVICE&page=1&pageSize=5
Then el código HTTP es 200
  And todos los tickets cumplen TODAS las condiciones de filtro
  And la paginación refleja el subconjunto filtrado
```

| Combinación | Resultado Esperado |
|-------------|--------------------|
| status + priority | 200, cumple ambos filtros |
| status + incidentType + dates | 200, cumple todos |
| Todos los filtros + paginación + orden | 200, filtrado/paginado/ordenado |
| Filtros sin resultados | 200, data vacío, totalItems: 0 |

---

#### IT-RQ-009: Query params inválidos retornan 400

- **ID:** IT-RQ-009
- **Endpoint:** `GET /api/tickets` con params inválidos
- **HU:** Transversal

```gherkin
Given el endpoint GET /api/tickets está disponible
When se envía GET /api/tickets?page=-1
Then el código HTTP es 400
  And el body contiene un mensaje de error descriptivo
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| page negativo | `page=-1` | 400 |
| page = 0 | `page=0` | 400 |
| page no numérico | `page=abc` | 400 |
| pageSize = 0 | `pageSize=0` | 400 |
| pageSize negativo | `pageSize=-5` | 400 |

---

### 6.2 GET /api/tickets/:id — Búsqueda por ID

#### IT-RQ-010: Búsqueda por ID existente retorna 200 con ticket completo

- **ID:** IT-RQ-010
- **Endpoint:** `GET /api/tickets/:id`
- **HU:** HU-06

```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000"
When se envía GET /api/tickets/550e8400-e29b-41d4-a716-446655440000
Then el código HTTP es 200
  And el body contiene el ticket con todos los campos
  And ticketId === "550e8400-e29b-41d4-a716-446655440000"
```

---

#### IT-RQ-011: Búsqueda por ID inexistente retorna 404

- **ID:** IT-RQ-011
- **Endpoint:** `GET /api/tickets/:id`
- **HU:** HU-06

```gherkin
Given no existe un ticket con ID "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
When se envía GET /api/tickets/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
Then el código HTTP es 404
  And el body contiene { "error": "Ticket no encontrado" }
```

---

#### IT-RQ-012: Búsqueda por ID con formato inválido retorna 400

- **ID:** IT-RQ-012
- **Endpoint:** `GET /api/tickets/:id`
- **HU:** HU-06

```gherkin
Given el endpoint GET /api/tickets/:id está disponible
When se envía GET /api/tickets/invalid-id
Then el código HTTP es 400
  And el body contiene { "error": "Formato de ID inválido" }
```

| Partición | Valor | Resultado |
|-----------|-------|-----------|
| UUID válido existente | UUID real | 200 |
| UUID válido inexistente | UUID válido no en BD | 404 |
| Formato inválido | `"abc123"` | 400 |
| String vacío (ruta raíz) | `GET /api/tickets/` | Redirige al listado (HU-01) |
| UUID sin guiones | `"550e8400e29b41d4a716446655440000"` | 400 |

---

### 6.3 GET /api/tickets/metrics — Métricas Agregadas

#### IT-RQ-013: Métricas globales sin filtros retorna distribuciones completas

- **ID:** IT-RQ-013
- **Endpoint:** `GET /api/tickets/metrics`
- **HU:** HU-09

```gherkin
Given existen tickets en la base de datos con variedad de estados, prioridades y tipos
When se envía GET /api/tickets/metrics
Then el código HTTP es 200
  And el body contiene "totalTickets" con el total real
  And el body contiene "byStatus" con conteos por cada estado
  And el body contiene "byPriority" con conteos por cada prioridad
  And el body contiene "byType" con conteos por cada tipo de incidente
  And la suma de valores en "byStatus" === totalTickets
  And la suma de valores en "byPriority" === totalTickets
  And la suma de valores en "byType" === totalTickets
```

---

#### IT-RQ-014: Métricas con filtros respetan los filtros activos

- **ID:** IT-RQ-014
- **Endpoint:** `GET /api/tickets/metrics?status=RECEIVED`
- **HU:** HU-09

```gherkin
Given existen tickets RECEIVED e IN_PROGRESS
When se envía GET /api/tickets/metrics?status=RECEIVED
Then el código HTTP es 200
  And "totalTickets" refleja solo los tickets RECEIVED
  And "byStatus" contiene solo la clave "RECEIVED"
  And las distribuciones son consistentes con el filtro aplicado
```

---

#### IT-RQ-015: Métricas con base de datos vacía

- **ID:** IT-RQ-015
- **Endpoint:** `GET /api/tickets/metrics`
- **HU:** HU-09

```gherkin
Given la base de datos está vacía (0 tickets)
When se envía GET /api/tickets/metrics
Then el código HTTP es 200
  And totalTickets === 0
  And byStatus está vacío o con todos los valores en 0
  And byPriority está vacío o con todos los valores en 0
  And byType está vacío o con todos los valores en 0
```

---

### 6.4 PATCH /api/tickets/:ticketId/status — Cambio de Estado

#### IT-RQ-016: Cambio de estado exitoso retorna 200 con ticket actualizado

- **ID:** IT-RQ-016
- **Endpoint:** `PATCH /api/tickets/:ticketId/status`
- **HU:** UH-013

```gherkin
Given existe un ticket con ID "550e8400-e29b-41d4-a716-446655440000" en estado "RECEIVED"
When se envía PATCH /api/tickets/550e8400-e29b-41d4-a716-446655440000/status
  And Content-Type es "application/json"
  And body es { "status": "IN_PROGRESS" }
Then el código HTTP es 200
  And el body contiene el ticket completo
  And status === "IN_PROGRESS"
  And processedAt es una fecha ISO-8601 reciente
```

---

#### IT-RQ-017: Cambio de estado con body inválido retorna 400

- **ID:** IT-RQ-017
- **Endpoint:** `PATCH /api/tickets/:ticketId/status`
- **HU:** UH-013

```gherkin
Given el endpoint PATCH está disponible
When se envía PATCH con body vacío {}
Then el código HTTP es 400
  And el body contiene un mensaje de error descriptivo
```

| Partición | Body | Resultado |
|-----------|------|-----------|
| Body vacío | `{}` | 400 |
| Campo incorrecto | `{ "status": 123 }` | 400 |
| Estado inválido | `{ "status": "CLOSED" }` | 400 |
| Null | `{ "status": null }` | 400 |
| JSON malformado | `"not json"` | 400 |

---

#### IT-RQ-018: Cambio de estado de ticket inexistente retorna 404

- **ID:** IT-RQ-018
- **Endpoint:** `PATCH /api/tickets/:ticketId/status`
- **HU:** UH-013

```gherkin
Given no existe ticket con ID "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
When se envía PATCH /api/tickets/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee/status
  And body es { "status": "IN_PROGRESS" }
Then el código HTTP es 404
  And el body contiene { "error": "Ticket no encontrado" }
```

---

#### IT-RQ-019: Cambio de estado con ticketId formato inválido retorna 400

- **ID:** IT-RQ-019
- **Endpoint:** `PATCH /api/tickets/:ticketId/status`
- **HU:** UH-013

```gherkin
Given el endpoint PATCH está disponible
When se envía PATCH /api/tickets/invalid-uuid/status
  And body es { "status": "IN_PROGRESS" }
Then el código HTTP es 400
  And el body contiene error de formato de ID
```

---

#### IT-RQ-020: Cambio idempotente (mismo estado) retorna 200

- **ID:** IT-RQ-020
- **Endpoint:** `PATCH /api/tickets/:ticketId/status`
- **HU:** UH-013

```gherkin
Given existe un ticket en estado "IN_PROGRESS"
When se envía PATCH con body { "status": "IN_PROGRESS" }
Then el código HTTP es 200
  And status sigue siendo "IN_PROGRESS"
  And processedAt se actualiza con nueva timestamp
```

---

#### IT-RQ-021: Error de base de datos retorna 500

- **ID:** IT-RQ-021
- **Endpoint:** `PATCH /api/tickets/:ticketId/status`
- **HU:** UH-013

```gherkin
Given la base de datos está experimentando problemas de conexión (simulado con mock)
When se envía PATCH con datos válidos
Then el código HTTP es 500
  And el body contiene { "error": "Error interno del servidor" }
  And no se expone información sensible de la BD
```

---

### 6.5 Rutas No Existentes y Métodos No Soportados

#### IT-RQ-022: Ruta inexistente retorna 404

- **ID:** IT-RQ-022
- **Endpoint:** `GET /api/noexiste`

```gherkin
Given el servidor Reports-Query está operativo
When se envía GET /api/noexiste
Then el código HTTP es 404
```

---

#### IT-RQ-023: Método HTTP no soportado retorna 405 o 404

- **ID:** IT-RQ-023
- **Endpoint:** `DELETE /api/tickets/some-id`

```gherkin
Given el endpoint solo soporta GET y PATCH
When se envía DELETE /api/tickets/550e8400-e29b-41d4-a716-446655440000
Then el código HTTP es 404 o 405
```

---

## 7. Diseño de Pruebas de Integración — Endpoints Antiguos (Producer)

### 7.1 POST /complaints — Creación de Queja

#### IT-PR-001: Request válida completa retorna 202

- **ID:** IT-PR-001
- **Endpoint:** `POST /complaints`
- **Servicio:** Producer

```gherkin
Given el Producer está operativo
  And RabbitMQ está disponible (mock)
When se envía POST /complaints con body válido:
  | campo        | valor                    |
  | lineNumber   | "3001234567"             |
  | email        | "user@example.com"       |
  | incidentType | "NO_SERVICE"             |
  | description  | "Sin servicio desde ayer" |
Then el código HTTP es 202
  And el body contiene "ticketId" (UUIDv4)
  And el body contiene "status" = "RECEIVED"
  And el body contiene "message" = "Accepted for processing"
  And el body contiene "createdAt" (ISO-8601)
```

---

#### IT-PR-002: Request sin campo obligatorio retorna 400

- **ID:** IT-PR-002
- **Endpoint:** `POST /complaints`

```gherkin
Given el Producer está operativo
When se envía POST /complaints sin el campo "lineNumber"
Then el código HTTP es 400
  And el body contiene mensaje de campo requerido
```

| Partición | Campo Faltante | Resultado |
|-----------|----------------|-----------|
| Sin lineNumber | `lineNumber` | 400 |
| Sin email | `email` | 400 |
| Sin incidentType | `incidentType` | 400 |
| Sin description con type OTHER | `description` (type=OTHER) | 400 |
| Sin description con type != OTHER | `description` (type=NO_SERVICE) | 202 (válido) |

---

#### IT-PR-003: Email con formato inválido retorna 400

- **ID:** IT-PR-003
- **Endpoint:** `POST /complaints`

```gherkin
Given el Producer está operativo
When se envía POST /complaints con email = "no-es-email"
Then el código HTTP es 400
  And el body contiene error de formato de email
```

| Partición | Valor de email | Resultado |
|-----------|----------------|-----------|
| Email válido | `"user@example.com"` | 202 |
| Sin @ | `"userexample.com"` | 400 |
| Sin dominio | `"user@"` | 400 |
| Cadena vacía | `""` | 400 |
| Con espacios | `"user @mail.com"` | 400 |

---

#### IT-PR-004: incidentType inválido retorna 400

- **ID:** IT-PR-004
- **Endpoint:** `POST /complaints`

```gherkin
Given el Producer está operativo
When se envía POST /complaints con incidentType = "HACKING"
Then el código HTTP es 400
  And el body contiene error de tipo de incidente inválido
```

---

#### IT-PR-005: RabbitMQ no disponible retorna 503

- **ID:** IT-PR-005
- **Endpoint:** `POST /complaints`

```gherkin
Given el Producer está operativo
  And RabbitMQ NO está disponible (mock simula error de conexión)
When se envía POST /complaints con body válido
Then el código HTTP es 503 o 500
  And el body contiene mensaje de servicio no disponible
```

---

#### IT-PR-006: Body vacío retorna 400

- **ID:** IT-PR-006
- **Endpoint:** `POST /complaints`

```gherkin
Given el Producer está operativo
When se envía POST /complaints con body vacío {}
Then el código HTTP es 400
```

---

#### IT-PR-007: Ruta inexistente del Producer retorna 404

- **ID:** IT-PR-007
- **Endpoint:** `GET /api/nonexistent`

```gherkin
Given el Producer está operativo
When se envía GET /api/nonexistent
Then el código HTTP es 404
```

---

## 8. Matriz de Trazabilidad: Historias ↔ Pruebas de Integración

### 8.1 Reports-Query Service

| Historia de Usuario | Requerimiento | Tests de Integración | Cobertura |
|---------------------|---------------|----------------------|-----------|
| **HU-01** Listado paginado | RF-01 | IT-RQ-001 | ✅ |
| **HU-02** Filtro por estado | RF-02 | IT-RQ-002 | ✅ |
| **HU-03** Filtro por prioridad | RF-03 | IT-RQ-003 | ✅ |
| **HU-04** Filtro por tipo de incidente | RF-04 | IT-RQ-004 | ✅ |
| **HU-05** Filtro por rango de fechas | RF-05 | IT-RQ-005 | ✅ |
| **HU-06** Búsqueda por ID | RF-06 | IT-RQ-010, IT-RQ-011, IT-RQ-012 | ✅ |
| **HU-07** Búsqueda por número de línea | RF-07 | IT-RQ-006 | ✅ |
| **HU-08** Ordenamiento | RF-08 | IT-RQ-007 | ✅ |
| **HU-09** Métricas agregadas | RF-09 | IT-RQ-013, IT-RQ-014, IT-RQ-015 | ✅ |
| **UH-013** Cambio de estado | FUNC-013-001 a FUNC-013-005 | IT-RQ-016 a IT-RQ-021 | ✅ |
| Transversal | Validaciones | IT-RQ-008, IT-RQ-009, IT-RQ-022, IT-RQ-023 | ✅ |

### 8.2 Producer Service

| Historia / Funcionalidad | Tests de Integración | Cobertura |
|--------------------------|----------------------|-----------|
| Creación de quejas (flujo feliz) | IT-PR-001 | ✅ |
| Validación de campos obligatorios | IT-PR-002 | ✅ |
| Validación de formato de email | IT-PR-003 | ✅ |
| Validación de incidentType | IT-PR-004 | ✅ |
| Tolerancia a fallos de RabbitMQ | IT-PR-005 | ✅ |
| Body inválido / vacío | IT-PR-006 | ✅ |
| Rutas inexistentes | IT-PR-007 | ✅ |

---

## 9. Gestión de Riesgos

### 9.1 Riesgos de Proyecto

Riesgos que afectan la **planificación, recursos y ejecución** del proyecto.

| ID | Riesgo | Probabilidad | Impacto | Prioridad | Estrategia de Mitigación | Plan de Contingencia |
|----|--------|:------------:|:-------:|:---------:|--------------------------|----------------------|
| RP-01 | **Retraso en desarrollo de endpoints** — Los endpoints nuevos del Reports-Query no están listos a tiempo para ejecutar pruebas de integración | Media | Alta | 🔴 Alta | Priorizar desarrollo de endpoints según dependencias de pruebas; usar mocks temporales para desbloquear QA | Ejecutar pruebas con stubs y re-ejecutar con endpoints reales cuando estén disponibles |
| RP-02 | **Falta de datos de prueba representativos** — La base de datos de test no contiene suficiente variedad para cubrir todos los escenarios | Media | Media | 🟡 Media | Crear seed de datos con cobertura de todos los tipos, estados, prioridades y rangos de fechas | Generar datos dinámicamente en el setup de cada test |
| RP-03 | **Dependencia del entorno Docker** — Problemas de configuración de Docker Compose impiden ejecutar pruebas de integración de sistema | Baja | Alta | 🟡 Media | Documentar setup paso a paso; mantener `docker-compose.yml` actualizado | Pruebas de componentes con mocks como fallback; escalar a DevOps |
| RP-04 | **Rotación de equipo o falta de conocimiento** — Miembros del equipo no familiarizados con la estrategia de testing o herramientas | Baja | Media | 🟢 Baja | Documentación clara (este documento); sesiones de onboarding sobre TDD y Vitest + supertest | Pair programming con miembros experimentados |
| RP-05 | **Conflictos de merge en monorepo** — Cambios concurrentes en diferentes servicios causan conflictos que retrasan integración | Media | Media | 🟡 Media | Feature branches cortas; integración continua frecuente; linting y formateo estandarizado | Resolución de conflictos inmediata; no dejar PRs stale más de 24h |
| RP-06 | **Calendario agresivo** — No hay suficiente tiempo para ejecutar todos los escenarios de prueba diseñados | Media | Alta | 🔴 Alta | Priorizar tests por riesgo (HIGH primero); automatizar la mayor cantidad posible | Ejecutar solo el subset crítico; registrar tests no ejecutados como deuda |

### 9.2 Riesgos de Producto

Riesgos que afectan la **calidad, funcionalidad y confiabilidad** del producto entregado.

| ID | Riesgo | Probabilidad | Impacto | Prioridad | Estrategia de Mitigación | Plan de Contingencia |
|----|--------|:------------:|:-------:|:---------:|--------------------------|----------------------|
| RPD-01 | **Inconsistencia entre datos de lectura y escritura** — El Reports-Query lee datos stale mientras el Consumer escribe datos nuevos | Media | Alta | 🔴 Alta | Definir estrategia de consistencia eventual; tests de integración que validen actualización de datos | Implementar mecanismo de refresh forzado; documentar latencia esperada |
| RPD-02 | **Validaciones insuficientes en endpoints** — Los endpoints aceptan datos malformados que causan errores downstream | Media | Alta | 🔴 Alta | Tests de integración exhaustivos con particiones de equivalencia y valores límite; Chain of Responsibility para errores | Hotfix inmediato; agregar validación en cada capa (defense in depth) |
| RPD-03 | **Degradación de rendimiento con filtros complejos** — Combinaciones de múltiples filtros + paginación + orden causan queries lentas | Media | Alta | 🔴 Alta | Índices en PostgreSQL para campos de filtro; test de rendimiento con 500-1000 tickets; query plan analysis | Limitar combinaciones de filtros; caching de queries frecuentes |
| RPD-04 | **Bug conocido: Consumer rechaza description=null** — El Consumer trata como inválido cualquier mensaje con `description: null` para tipos distintos de OTHER | Alta | Media | 🟡 Media | Documentado en [CALIDAD.md](./CALIDAD.md); test de regresión específico | Fix en la validación del Consumer; test que verifique procesamiento correcto |
| RPD-05 | **Acoplamiento entre servicios por BD compartida** — Consumer y Reports-Query comparten la misma BD PostgreSQL; cambios en el esquema afectan a ambos | Media | Alta | 🔴 Alta | Contrato de esquema versionado; migraciones coordinadas; tests de contrato | Rollback de migración; versionado de esquema con backward compatibility |
| RPD-06 | **Pérdida de datos en Producer (modo asíncrono)** — En el modo actual sin Outbox, si RabbitMQ cae después del 202, el evento se pierde | Media | Alta | 🔴 Alta | Implementar Outbox pattern o persistencia local; test de integración IT-PR-005 para validar manejo de RabbitMQ down | Modo síncrono como fallback; logging de eventos no publicados para re-envío manual |
| RPD-07 | **Cambios de contrato no propagados al frontend** — Se modifica la estructura de response de un endpoint sin actualizar el frontend | Media | Alta | 🟡 Media | Tests de contrato; versionado de DTOs; documentación actualizada de esquemas | Rollback del cambio de API; parche en frontend para nuevo esquema |
| RPD-08 | **Comportamiento inesperado en edge cases de paginación** — pageSize > total de datos, page = última página + 1, valores extremos | Media | Baja | 🟢 Baja | Tests IT-RQ-001 cubren particiones de paginación; tests de valores límite | Fix y re-test |
| RPD-09 | **Error en métricas agregadas por filtros incorrectos** — Las métricas no reflejan los filtros aplicados o muestran datos inconsistentes | Baja | Alta | 🟡 Media | Test IT-RQ-014 valida métricas filtradas; comparar sumas con totalTickets | Calcular métricas en frontend como fallback temporal |
| RPD-10 | **Idempotencia del PATCH no garantizada** — Múltiples PATCH consecutivos causan efectos no deseados o errores | Baja | Media | 🟢 Baja | Test IT-RQ-020 valida idempotencia; diseño del endpoint sin side-effects más allá de la actualización | Agregar guard en Service |

### 9.3 Matriz de Riesgos (Probabilidad × Impacto)

```
         │  Bajo     │  Medio    │  Alto
─────────┼───────────┼───────────┼──────────
  Alta   │           │  RPD-04   │  RPD-06
─────────┼───────────┼───────────┼──────────
  Media  │  RPD-08   │  RP-05    │  RP-01, RP-06
         │           │  RPD-07   │  RPD-01, RPD-02
         │           │           │  RPD-03, RPD-05
─────────┼───────────┼───────────┼──────────
  Baja   │  RPD-10   │  RP-04    │  RPD-09
         │           │           │  RP-03
```

### 9.4 Top 5 Riesgos Críticos y Acciones

| # | Riesgo ID | Descripción Corta | Acción Inmediata |
|---|-----------|-------------------|------------------|
| 1 | RPD-06 | Pérdida de eventos sin Outbox | Evaluar implementación de Outbox pattern; validar IT-PR-005 |
| 2 | RPD-01 | Inconsistencia lectura/escritura | Definir SLA de consistencia eventual; tests de refresh |
| 3 | RPD-02 | Validaciones insuficientes | Completar todos los tests IT-RQ-009 y particiones de 400 |
| 4 | RPD-05 | Acoplamiento por BD compartida | Definir contrato de esquema; migraciones coordinadas |
| 5 | RP-06 | Calendario agresivo | Priorizar tests por riesgo; automatizar subset crítico primero |

---

## 10. Herramientas y Entorno de Pruebas

### 10.1 Stack de Testing

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| **Vitest** | Latest | Framework de tests (runner, assertions, mocks) |
| **supertest** | Latest | Requests HTTP contra Express sin servidor real |
| **vi.mock / vi.fn** | (Vitest built-in) | Mocking de dependencias (BD, RabbitMQ) |
| **Docker Compose** | Latest | Orquestación para pruebas de integración de sistema |
| **PostgreSQL** | 15-alpine | Base de datos (real en E2E, mock en integración de componentes) |
| **RabbitMQ** | 3-management-alpine | Broker de mensajería (real en E2E, mock en integración) |
| **TypeScript** | Modo estricto | Lenguaje de desarrollo y tests |
| **Node.js** | Latest LTS | Runtime |

### 10.2 Configuración de Entorno

**Integración de componentes (sin infraestructura)**:
```bash
# Desde el directorio del servicio (ej: backend/reports-query)
npm run test          # Corre todos los tests (unitarios + integración)
npm run test:integration  # Solo pruebas de integración (si configurado)
```

**Integración de sistema (con Docker)**:
```bash
# Levantar infraestructura completa
docker-compose up -d

# Ejecutar pruebas
npm run test:integration:system

# Tear down
docker-compose down -v
```

### 10.3 Estructura de Archivos de Test

```
backend/
├── producer/
│   └── tests/
│       ├── unit/              # Tests unitarios
│       └── integration/       # Tests de integración (IT-PR-*)
│           └── complaints.api.test.ts
├── reports-query/
│   └── tests/
│       ├── unit/              # Tests unitarios (TC-013-*, TC-028-034)
│       └── integration/       # Tests de integración (IT-RQ-*)
│           ├── tickets-list.api.test.ts
│           ├── tickets-by-id.api.test.ts
│           ├── tickets-metrics.api.test.ts
│           └── tickets-status.api.test.ts
```

---

## 11. Calendario de Pruebas

### 11.1 Fases y Cronograma

| Fase | Actividad | Fecha Inicio | Fecha Fin | Estado | Responsable |
|------|-----------|:------------:|:---------:|:------:|-------------|
| **F1** | Diseño de pruebas de integración (este documento) | 27/02/2026 | 27/02/2026 | ✅ Completado | QA / Desarrollador |
| **F2** | Implementación de tests IT-RQ-001 a IT-RQ-009 (listado, filtros, paginación) | 28/02/2026 | 03/03/2026 | ⬜ Pendiente | Desarrollador |
| **F3** | Implementación de tests IT-RQ-010 a IT-RQ-015 (búsqueda por ID, métricas) | 03/03/2026 | 05/03/2026 | ⬜ Pendiente | Desarrollador |
| **F4** | Implementación de tests IT-RQ-016 a IT-RQ-023 (PATCH status, rutas) | 05/03/2026 | 07/03/2026 | ⬜ Pendiente | Desarrollador |
| **F5** | Implementación de tests IT-PR-001 a IT-PR-007 (Producer) | 07/03/2026 | 09/03/2026 | ⬜ Pendiente | Desarrollador |
| **F6** | Ejecución manual y registro de resultados | 09/03/2026 | 10/03/2026 | ⬜ Pendiente | QA / Desarrollador |
| **F7** | Revisión de resultados, fix de defectos, re-test | 10/03/2026 | 12/03/2026 | ⬜ Pendiente | Equipo |

### 11.2 Prioridad de Ejecución

Los tests se ejecutarán en orden de prioridad de riesgo:

| Prioridad | Tests | Justificación |
|:---------:|-------|---------------|
| 🔴 **1 — Crítica** | IT-RQ-016, IT-RQ-017, IT-RQ-018, IT-PR-005 | PATCH status (nuevo) + tolerancia a fallas de RabbitMQ |
| 🔴 **2 — Alta** | IT-RQ-001, IT-RQ-010, IT-RQ-011, IT-RQ-013, IT-PR-001 | Flujos felices de endpoints principales |
| 🟡 **3 — Media** | IT-RQ-002 a IT-RQ-009, IT-RQ-012, IT-RQ-014 | Filtros, ordenamiento, validaciones |
| 🟢 **4 — Baja** | IT-RQ-015, IT-RQ-019 a IT-RQ-023, IT-PR-002 a IT-PR-007 | Edge cases, rutas inválidas, métricas vacías |

---

## 12. Criterios de Entrada y Salida

### 12.1 Criterios de Entrada (para iniciar pruebas de integración)

| # | Criterio | Verificación |
|---|----------|:------------:|
| 1 | Los endpoints bajo prueba están implementados y compilando en modo estricto | ⬜ |
| 2 | Las pruebas unitarias de la lógica de negocio están en estado GREEN | ⬜ |
| 3 | El entorno de pruebas (Docker Compose o mocks) está configurado | ⬜ |
| 4 | Los datos de seed/test están preparados con cobertura de todos los tipos | ⬜ |
| 5 | Las dependencias (`supertest`, `vitest`) están instaladas | ⬜ |
| 6 | Este plan de pruebas está revisado y aprobado | ⬜ |

### 12.2 Criterios de Salida (para considerar pruebas de integración completadas)

| # | Criterio | Meta | Actual |
|---|----------|------|--------|
| 1 | Todos los tests de prioridad Crítica y Alta pasan | 100% | — |
| 2 | Cobertura de endpoints con al menos 1 test de integración | 100% | — |
| 3 | Todos los códigos HTTP documentados están probados | ≥ 90% | — |
| 4 | Defectos bloqueantes resueltos | 0 abiertos | — |
| 5 | Registro de ejecución manual completado (sección 13) | 100% | — |
| 6 | Sin regresiones en pruebas unitarias existentes | 0 fallos | — |

---

## 13. Registro de Ejecución Manual

### 13.1 Instrucciones

Cada test de integración debe ejecutarse y registrar el resultado en la tabla correspondiente:
- **Pasó (✅)**: El test produce el resultado esperado.
- **Falló (❌)**: El test produce un resultado diferente al esperado.
- **Bloqueado (🚫)**: No se pudo ejecutar por dependencias o entorno.
- **No ejecutado (⬜)**: Pendiente de ejecución.

### 13.2 Reports-Query Service — Ejecución Manual

| ID Test | Descripción | Resultado Esperado | Resultado Obtenido | Estado | Fecha | Ejecutor | Observaciones |
|---------|-------------|--------------------|--------------------|:------:|:-----:|----------|---------------|
| IT-RQ-001 | Listado por defecto sin filtros | 200, paginación default | — | ⬜ | — | — | — |
| IT-RQ-002 | Filtro por estado | 200, solo estado filtrado | — | ⬜ | — | — | — |
| IT-RQ-003 | Filtro por prioridad | 200, solo prioridad filtrada | — | ⬜ | — | — | — |
| IT-RQ-004 | Filtro por tipo de incidente | 200, solo tipo filtrado | — | ⬜ | — | — | — |
| IT-RQ-005 | Filtro por rango de fechas | 200, dentro del rango | — | ⬜ | — | — | — |
| IT-RQ-006 | Filtro por número de línea | 200, solo lineNumber | — | ⬜ | — | — | — |
| IT-RQ-007 | Ordenamiento por campos | 200, ordenado correctamente | — | ⬜ | — | — | — |
| IT-RQ-008 | Combinación de múltiples filtros | 200, cumple todos los filtros | — | ⬜ | — | — | — |
| IT-RQ-009 | Query params inválidos | 400, error descriptivo | — | ⬜ | — | — | — |
| IT-RQ-010 | Búsqueda por ID existente | 200, ticket completo | — | ⬜ | — | — | — |
| IT-RQ-011 | Búsqueda por ID inexistente | 404, "Ticket no encontrado" | — | ⬜ | — | — | — |
| IT-RQ-012 | Búsqueda por ID formato inválido | 400, error de formato | — | ⬜ | — | — | — |
| IT-RQ-013 | Métricas globales sin filtros | 200, distribuciones completas | — | ⬜ | — | — | — |
| IT-RQ-014 | Métricas con filtros | 200, métricas filtradas | — | ⬜ | — | — | — |
| IT-RQ-015 | Métricas con BD vacía | 200, totales en 0 | — | ⬜ | — | — | — |
| IT-RQ-016 | Cambio de estado exitoso | 200, ticket actualizado | — | ⬜ | — | — | — |
| IT-RQ-017 | Cambio de estado body inválido | 400, error descriptivo | — | ⬜ | — | — | — |
| IT-RQ-018 | Cambio de estado ticket inexistente | 404, "Ticket no encontrado" | — | ⬜ | — | — | — |
| IT-RQ-019 | Cambio de estado ticketId inválido | 400, error de formato | — | ⬜ | — | — | — |
| IT-RQ-020 | Cambio idempotente | 200, mismo estado | — | ⬜ | — | — | — |
| IT-RQ-021 | Error de base de datos | 500, error genérico | — | ⬜ | — | — | — |
| IT-RQ-022 | Ruta inexistente | 404 | — | ⬜ | — | — | — |
| IT-RQ-023 | Método HTTP no soportado | 404 o 405 | — | ⬜ | — | — | — |

### 13.3 Producer Service — Ejecución Manual

| ID Test | Descripción | Resultado Esperado | Resultado Obtenido | Estado | Fecha | Ejecutor | Observaciones |
|---------|-------------|--------------------|--------------------|:------:|:-----:|----------|---------------|
| IT-PR-001 | Request válida completa | 202, ticketId generado | — | ⬜ | — | — | — |
| IT-PR-002 | Sin campo obligatorio | 400, campo requerido | — | ⬜ | — | — | — |
| IT-PR-003 | Email formato inválido | 400, error de formato | — | ⬜ | — | — | — |
| IT-PR-004 | incidentType inválido | 400, tipo inválido | — | ⬜ | — | — | — |
| IT-PR-005 | RabbitMQ no disponible | 503 o 500 | — | ⬜ | — | — | — |
| IT-PR-006 | Body vacío | 400 | — | ⬜ | — | — | — |
| IT-PR-007 | Ruta inexistente | 404 | — | ⬜ | — | — | — |

### 13.4 Resumen de Ejecución

| Servicio | Total | Pasó ✅ | Falló ❌ | Bloqueado 🚫 | No Ejecutado ⬜ | % Ejecutado |
|----------|:-----:|:------:|:------:|:-----------:|:--------------:|:-----------:|
| Reports-Query | 23 | 0 | 0 | 0 | 23 | 0% |
| Producer | 7 | 0 | 0 | 0 | 7 | 0% |
| **Total** | **30** | **0** | **0** | **0** | **30** | **0%** |

---

## 14. Principios INVEST Aplicados a las Historias de Prueba

Cada historia de usuario asociada a pruebas de integración se evalúa contra los principios INVEST para asegurar calidad en la definición.

### 14.1 Evaluación INVEST por Historia

#### HU-01: Listado de tickets con paginación

| Principio | Evaluación | Justificación |
|-----------|:----------:|---------------|
| **I**ndependiente | ✅ | Puede probarse sin depender de filtros, ordenamiento u otras HU |
| **N**egociable | ✅ | El tamaño de página default y límite máximo son negociables (actualmente 10 y 20) |
| **V**aliosa | ✅ | Sin paginación, el dashboard no escala a 500-1000 tickets |
| **E**stimable | ✅ | Complejidad conocida: query SQL + serialización + response schema |
| **S**mall (Pequeña) | ✅ | Alcance limitado: un endpoint, un comportamiento |
| **T**esteable | ✅ | Tests IT-RQ-001 definen claramente entradas (query params) y salidas (response schema, pagination object) |

#### HU-06: Búsqueda por ID de ticket

| Principio | Evaluación | Justificación |
|-----------|:----------:|---------------|
| **I**ndependiente | ✅ | No depende de filtros ni paginación; consulta directa por ID |
| **N**egociable | ✅ | El formato del mensaje de error y el comportamiento con ID vacío son negociables |
| **V**aliosa | ✅ | Acceso directo a un caso específico es crítico para operadores |
| **E**stimable | ✅ | Único query por primary key; validación de UUID |
| **S**mall (Pequeña) | ✅ | Un endpoint, tres escenarios (200, 400, 404) |
| **T**esteable | ✅ | Tests IT-RQ-010/011/012 con particiones claras de UUID válido/inválido/inexistente |

#### HU-09: Métricas agregadas

| Principio | Evaluación | Justificación |
|-----------|:----------:|---------------|
| **I**ndependiente | ✅ | Puede probarse independientemente de la UI de gráficas (HU-10) |
| **N**egociable | ✅ | Las métricas específicas (por estado, prioridad, tipo) son negociables |
| **V**aliosa | ✅ | Visión global del sistema para supervisores |
| **E**stimable | ✅ | Queries de agregación COUNT/GROUP BY bien conocidos |
| **S**mall (Pequeña) | ✅ | Un endpoint con tres variantes de respuesta (global, filtrado, vacío) |
| **T**esteable | ✅ | Tests IT-RQ-013/014/015 validan sumas coherentes y respuesta ante filtros |

#### UH-013: Cambio de Estado de Tickets

| Principio | Evaluación | Justificación |
|-----------|:----------:|---------------|
| **I**ndependiente | ⚠️ Parcial | Depende de HU-01 (lista de tickets) para la interfaz, pero el endpoint es independiente |
| **N**egociable | ✅ | Los estados válidos, la bidireccionalidad y la idempotencia son decisiones de diseño negociables |
| **V**aliosa | ✅ | Permite gestionar el ciclo de vida de quejas sin sistemas externos |
| **E**stimable | ✅ | Endpoint PATCH conocido; validaciones y actualización de BD |
| **S**mall (Pequeña) | ✅ | Un endpoint con alcance acotado (solo cambia status y processedAt) |
| **T**esteable | ✅ | Tests IT-RQ-016 a IT-RQ-021 cubren todos los escenarios con particiones de equivalencia |

### 14.2 Resumen INVEST

| Historia | I | N | V | E | S | T | Score |
|----------|:-:|:-:|:-:|:-:|:-:|:-:|:-----:|
| HU-01 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 6/6 |
| HU-02 a HU-05 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 6/6 |
| HU-06 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 6/6 |
| HU-07 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 6/6 |
| HU-08 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 6/6 |
| HU-09 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 6/6 |
| UH-013 | ⚠️ | ✅ | ✅ | ✅ | ✅ | ✅ | 5.5/6 |

---

## 15. Apéndice: Valores de Referencia del Dominio

### Tipos de Incidente Válidos

| Valor | Descripción |
|-------|-------------|
| `NO_SERVICE` | Sin servicio |
| `INTERMITTENT_SERVICE` | Servicio intermitente |
| `SLOW_CONNECTION` | Conexión lenta |
| `ROUTER_ISSUE` | Problema con router |
| `BILLING_QUESTION` | Pregunta de facturación |
| `OTHER` | Otro (requiere descripción obligatoria) |

### Prioridades Válidas

| Valor | Descripción |
|-------|-------------|
| `HIGH` | Alta prioridad |
| `MEDIUM` | Prioridad media |
| `LOW` | Prioridad baja |
| `PENDING` | Pendiente de asignación |

### Estados Válidos de Ticket

| Valor | Descripción |
|-------|-------------|
| `RECEIVED` | Recibido |
| `IN_PROGRESS` | En progreso |

### Formato de Datos

| Campo | Formato | Ejemplo |
|-------|---------|---------|
| `ticketId` | UUIDv4 | `550e8400-e29b-41d4-a716-446655440000` |
| `lineNumber` | 10 dígitos numéricos | `3001234567` |
| `email` | Formato email estándar | `user@example.com` |
| `createdAt` | ISO-8601 | `2026-02-18T10:00:00Z` |
| `processedAt` | ISO-8601 | `2026-02-18T10:05:30Z` |

---

## Historial del Documento

| Versión | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 2.0 | 27/02/2026 | Equipo QA | Creación del documento — Plan de pruebas de integración API REST |

---

*Documento generado el 27 de febrero de 2026*
