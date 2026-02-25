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

## 3. Arquitectura Actual

Actualmente todo el sistema está integrado en un único repositorio con la siguiente estructura general:

```
/
├── frontend/                # Aplicación React
├── backend/
│   ├── producer/            # API REST para recibir quejas
│   ├── consumer/            # Worker que procesa quejas desde RabbitMQ
│   └── reports-query/       # Worker que genera reportes desde DB
├── docker-compose.yml       # Orquestación local
└── documentación/           # Documentación del proyecto
```

Esto genera un problema de independencia entre servicios, acoplamiento en el desarrollo y despliegue, y dificulta la escalabilidad.
Al ser un monorepo, los despliegues CI no pueden ser totalmente independientes, lo que puede probocar un aumento en el tiempo de despliegue y una mayor probabilidad de errores al afectar a servicios no relacionados.
Para mejorar esta arquitectura y eliminar el sistema monolítico, se propone separar cada servicio en su propio repositorio, con su propio ciclo de vida, pipeline de CI/CD y despliegue independiente. Esto permitirá una mayor flexibilidad, escalabilidad y mantenibilidad a largo plazo.
Estos repositorios son organizados en la organización `AITraining-SofkaProyects-Team2` de GitHub, con los siguientes nombres:
- Frontend: `Semana-03-frontend`
- Producer: `Semana-3-microservicio-Producer`
- Consumer: `Semana-3-microservicio-Consumer`
- Reports-Query: `Semana-3-microservicio-Reports-query`

---
## 4. Endpoints de servicios

### Producer
#### POST api/v1/tikets
url => api/v1/tikets
verbo http => POST
códigos de estado:
- 202: ticket recibido y en proceso
- 500: error interno
- 503: servicio no disponible (RabbitMQ down)

## Producer

### `POST /complaints`

url = `/complaints`
http = POST
códigos de estado:
- ~~`201 Created`~~ => `202 Accepted`: ticket recibido y en proceso
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

- **Conclusion:** 
    Se cambió el comportamiento de `POST /complaints`: antes el endpoint aguardaba la confirmación de publicación en RabbitMQ y devolvía `201 Created` con el ticket completo. Ahora el endpoint devuelve `202 Accepted` y un cuerpo reducido ({ ticketId, status: 'RECEIVED', message, createdAt }) inmediatamente; la publicación a RabbitMQ se hace en modo asíncrono.Errores no controlados siguen mapeándose a `500 Internal Server Error`.

    Si NO hay garantía (no hay outbox/local-persist, y no puede conectarse a RabbitMQ), lo correcto es devolver 503 Service Unavailable o 5xx equivalente: no puedes aceptar la petición para procesamiento si no puedes siquiera encolarla ni garantizar reintento.
    Alternativas operativas (elige según requisitos de fiabilidad):
      1. Síncrono + confirmación broker: intentar publicar y, si OK, devolver 201/202; si broker down, devolver 503. (simple, fuerte garantía)
      2. Asíncrono con Outbox/cola local durable: persistir el evento localmente atomically, devolver 202 inmediatamente; un worker publica a RabbitMQ en segundo plano (recomendado si quieres disponibilidad del API con durable guarantee).
      3. Asíncrono sin persistencia local: devolver 202 al iniciar el envío y confiar en logs/metricas/reintentos en memoria — riesgo de pérdida de mensajes si Rabbit cae (poco recomendable para producción).
