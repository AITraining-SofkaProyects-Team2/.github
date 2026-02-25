# Reporte de Arquitectura de Software
**Proyecto:** Taller Sistema Distribuido - Gestión de Quejas ISP  
**Fecha:** 24 de febrero de 2026  
**Analizado por:** SoftwareArchitect Agent  

---

## 1. Resumen Ejecutivo
El proyecto implementa una arquitectura orientada a eventos con el objetivo de ser eficiente frente a usos intencivos: 
          frontend (React)
      ┌──────────┘└─────────┐
      ↓                     ↓
Producer (Express)    Reports-Query
      ↓                     ↑
  RabbitMQ                  │
      ↓                     │
Consumer (worker)           │
      │                     │
      └────→PostgreSQL←─────┘


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

## Consumer