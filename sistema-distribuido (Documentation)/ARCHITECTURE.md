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
