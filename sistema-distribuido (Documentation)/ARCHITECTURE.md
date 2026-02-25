# Reporte de Arquitectura de Software
**Proyecto:** Taller Sistema Distribuido - Gestión de Quejas ISP  
**Fecha:** 24 de febrero de 2026  
**Analizado por:** SoftwareArchitect Agent  

---

## 1. Resumen Ejecutivo
El proyecto implementa una arquitectura orientada a eventos bien pensada: frontend (React) → Producer (Express) → RabbitMQ → Consumer (worker). La separación de responsabilidades y el uso de patrones (Singleton para conexiones, Facade para mensajería, Strategy para priorización) está presente en el diseño. Sin embargo hay problemas operativos y de consistencia (Dockerfiles y versiones de Node, healthchecks, inicio ordenado) y algunas mejoras en organización (compartir tipos/contratos, CI de imágenes) que harán el sistema más robusto y fácil de desplegar en producción. No es necesario un cambio arquitectónico radical; propongo mejoras incrementales y saneamiento de infra.

---

## 2. Stack Tecnológico Identificado
- Lenguaje: TypeScript (strict)  
- Frontend: React + Vite  
- Backend: Node.js + Express (Producer), Node.js worker (Consumer)  
- Mensajería: RabbitMQ (rabbitmq:3-management-alpine)  
- Persistencia: Postgres (postgres:15-alpine)  
- Testing: Vitest  
- Orquestación local: Docker Compose

---

## 3. Arquitectura Actual

### 3.1 Diagrama de estructura actual
```
Frontend (React/Vite)  --->  Producer (Express)  --->  RabbitMQ  --->  Consumer (Worker)
        :80                     :3000                     :5672         health:3001
                             Postgres (DB) <--------------/
```

### 3.2 Patrón arquitectónico identificado
Intento de arquitectura de microservicios orientada a eventos (Producer -> Broker -> Consumer). Se aplican patrones: Singleton (RabbitMQConnectionManager), Facade (MessagingFacade), Strategy (prioridad), Chain of Responsibility (error handlers). La intención es respetar SRP y DIP.

### 3.3 Análisis de capas
- Producer: `backend/producer/src/app.ts` — controlador/routers, middleware, conexión a RabbitMQ via Singleton, handlers externos — buena separación del servidor y lifecycle. ([backend/producer/src/app.ts](backend/producer/src/app.ts#L1-L120))  
- Consumer: `backend/consumer/src/index.ts` — inicialización, separa repository y mensaje; `MessageHandler`/estrategias en carpetas dedicadas — diseño consistente. ([backend/consumer/src/index.ts](backend/consumer/src/index.ts#L1-L120))  
- Frontend: proyecto Vite con `src/` y hooks; pruebas de estrés incluidas.

---

## 4. Problemas Identificados

### P-01: Inconsistencia de imágenes y versiones de Node
- **Severidad:** Media  
- **Ubicación:** `backend/producer/Dockerfile`, `backend/consumer/Dockerfile`, `frontend/Dockerfile`  
- **Descripción:** Producer usa `node:20-alpine`, Consumer usa `node:18-alpine`, frontend builder usa `node:20-alpine` → inconsistencias en runtimes y toolchains.  
- **Impacto:** Diferencias en comportamiento de runtime/compilación, problemas sutiles en producción, dificultad al reproducir localmente.

### P-02: HEALTHCHECK y uso de variables en forma exec (no expanden env)
- **Severidad:** Media  
- **Ubicación:** `backend/producer/Dockerfile`  
- **Descripción:** HEALTHCHECK usa `CMD wget ... http://localhost:${PORT:-3000}/health` en forma exec; en exec form no se expanden variables de shell.  
- **Impacto:** Healthcheck puede fallar o no apuntar al puerto correcto; orquestadores recibirán señales incorrectas.

### P-03: Uso de `ENTRYPOINT` en Consumer para script de wait/seed sin control fino
- **Severidad:** Media  
- **Ubicación:** `backend/consumer/Dockerfile`, `backend/consumer` scripts  
- **Descripción:** Entrypoint ejecuta `dist/scripts/wait-and-seed.js` (bloqueante) antes de arrancar servidor; falta separación entre wait, seed y proceso principal.  
- **Impacto:** Dificulta retries, debugging y uso en orquestadores; mayor coupling en startup.

### P-04: Dependencias de Docker Compose y restart policy / depends_on
- **Severidad:** Media  
- **Ubicación:** `docker-compose.yml`  
- **Descripción:** `depends_on` usa condición `service_healthy` y `restart: no` para servicios críticos.  
- **Impacto:** En entornos reales conviene `restart: unless-stopped`/`on-failure`; `depends_on` no garantiza readiness (solo health), y hay falta de backoff/retries en arranque.

### P-05: Contratos compartidos no versionados/centralizados
- **Severidad:** Baja / Media  
- **Ubicación:** `frontend` vs `backend/*`  
- **Descripción:** Tipos/contratos (payloads) deben compartirse entre frontend y backend; actualmente parecen duplicados o no centralizados.  
- **Impacto:** Riesgo de desincronización de contratos y tests e2e rotos si se modifica payload.

### P-06: Falta de CI para construir y escanear imágenes y tests
- **Severidad:** Media  
- **Ubicación:** raíz del repo (no se encontró configuración CI)  
- **Descripción:** No hay pipeline visible para build/test/scan/adquisición de imágenes.  
- **Impacto:** Riesgo de romper contratos en integración; despliegues manuales y error-prone.

---

## 5. Recomendaciones

### R-01: Homogeneizar imágenes y versiones de Node
- **Prioridad:** Alta  
- **Esfuerzo estimado:** Bajo  
- **Problema que resuelve:** P-01  
- **Descripción:** Elegir una versión LTS única (p. ej. `node:20-alpine`) y usarla en todos los `Dockerfile` (producer, consumer, frontend builder). Pinnear versiones e idealmente usar digests en producción.  
- **Ejemplo:**  
```dockerfile
FROM node:20-alpine AS builder
```

### R-02: Corregir HEALTHCHECK para usar shell form o comando que expanda variables
- **Prioridad:** Alta  
- **Esfuerzo estimado:** Bajo  
- **Problema que resuelve:** P-02  
- **Descripción:** Reemplazar exec HEALTHCHECK por shell `sh -c` o usar curl/wget sin variables, o bien fijar `ENV PORT=3000` antes del HEALTHCHECK.  
- **Ejemplo:**  
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["sh", "-c", "wget -qO- http://localhost:${PORT:-3000}/health || exit 1"]
```

### R-03: Separar responsabilidades en startup (wait / seed / server)
- **Prioridad:** Media  
- **Esfuerzo estimado:** Bajo-Medio  
- **Problema que resuelve:** P-03  
- **Descripción:** Evitar usar un único `ENTRYPOINT` que haga wait+seed+start. Preferir:
  - Un `healthcheck` robusto.
  - Un script de init opcional (`init-db`) ejecutable desde compose o job de migración.
  - `CMD` para iniciar el servicio principal.  
- **Nota:** Mejor ejecutar migraciones/seed como job separado en compose o pipeline.

### R-04: Mejorar políticas de reinicio y readiness en compose; usar readiness probe o wait-for-it
- **Prioridad:** Media  
- **Esfuerzo estimado:** Bajo  
- **Problema que resuelve:** P-04  
- **Descripción:**  
  - Cambiar `restart: no` por `restart: unless-stopped` o `on-failure`.  
  - Añadir scripts de readiness o sidecar que esperen por DB/RabbitMQ antes de iniciar el servicio.  
  - Considerar Kubernetes para producción donde probes son nativas.

### R-05: Centralizar tipos/contratos y versionarlos
- **Prioridad:** Alta  
- **Esfuerzo estimado:** Medio  
- **Problema que resuelve:** P-05  
- **Descripción:** Mover tipos compartidos (payloads de quejas) a un paquete compartido (`packages/contracts`) o publicar en un paquete privado. Consumir desde `frontend`, `producer`, `consumer` para asegurar compatibilidad.  
- **Ejemplo:** crear `packages/contracts/src/index.ts` y usar `npm workspaces` o referencias locales.

### R-06: Añadir pipeline CI que valide builds, tests y escanee imágenes
- **Prioridad:** Alta  
- **Esfuerzo estimado:** Medio  
- **Problema que resuelve:** P-06  
- **Descripción:** CI debe:
  - Ejecutar `npm ci`, `npm run build`, `npm test:coverage` en cada servicio.  
  - Construir y etiquetar imágenes; correr linting y security scanning.  
  - Publicar artefactos (imágenes) al registry.

### R-07: Mejoras en los Dockerfiles (optimización y seguridad)
- **Prioridad:** Media  
- **Esfuerzo estimado:** Bajo  
- **Problema que resuelve:** P-01/P-02/P-03  
- **Descripción:**  
  - Usar `npm ci` en builder para reproducibilidad.  
  - Copiar sólo `package*.json` antes de instalar para cachear dependencias.  
  - En runtime, usar `npm ci --omit=dev`.  
  - Crear usuario no-root (producer ya lo hace; replicar en consumer).  
  - Minimizar capas y limpiar cache.

---

## 6. Arquitectura Propuesta

### 6.1 ¿Es necesario cambiar la arquitectura?
No. La arquitectura orientada a eventos es adecuada para el dominio (picos, desacoplamiento). Recomiendo mejoras operativas y de gobernanza (tipos compartidos, CI, consistencia de imágenes) antes que re-arquitecturar.

### 6.2 Diagrama de arquitectura propuesta
```
[Frontend]  ->  [Producer API]  ->  [RabbitMQ Broker]  ->  [Consumer(s) scaled]
      \                                              /
       \-> [E2E tests & CI pipelines]  -> Registry -> Deploy
Shared contracts package (monorepo or npm) referenced by all services
```

### 6.3 Plan de migración (incremental)
1. Homogeneizar Node version en Dockerfiles, ajustar healthchecks (R-01,R-02).  
2. Centralizar contratos (R-05): crear `packages/contracts` y actualizar imports.  
3. Añadir CI que valide build/test/image for each service (R-06).  
4. Ajustar compose restart policies y readiness scripts (R-04).  
5. Refinar Dockerfiles (R-07) y añadir scanning.  
6. Opcional: migrar orquestación a Kubernetes si se requiere escala/observabilidad.

---

## 7. Conclusión y Próximos Pasos
- Prioridad inmediata:
  - Homogeneizar versiones de Node en los Dockerfiles (R-01).  
  - Corregir HEALTHCHECK a shell form (R-02).  
  - Crear paquete/shared `contracts` para tipos y actualizar imports (R-05).  
- Mediano plazo:
  - Añadir CI que compile y ejecute tests por servicio (R-06).  
  - Mejorar restart policies y separar scripts de seed/init (R-03,R-04).  

¿Querés que aplique los cambios y genere los parches (Dockerfiles y `docker-compose.yml`) y cree `documentación/ARCHITECTURE_REPORT.md` en el repo? Indica si querés que cree PRs o simplemente te entregue los patches.