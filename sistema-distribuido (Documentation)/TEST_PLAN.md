# 🧪 Plan de Pruebas - Dashboard de Gestión de Reportes (Sistema Distribuido ISP)

**Fecha de creación**: 18 de febrero de 2026
**Basado en**: [FASE_3_HISTORIAS_RIESGOS.md](./FASE_3_HISTORIAS_RIESGOS.md)

---

## Estrategia de Desarrollo e Implementación de Pruebas unitarias
Estrategia de desarrollo e implementación de pruebas siguiendo la metodología TDD, con enfoque en pruebas unitarias.

1. **Desarrollo de Tests Unitarios**: Para cada historia de usuario, se desarrollarán tests unitarios que cubran los casos de uso principales, así como casos de borde y escenarios negativos.
Para crear los tests unitarios se utilizará un **AI Agent** (concretamente el agente **historiador**) especializado en generación de tests unitarios, que se alimentará de la documentación técnica, contratos de API, y ejemplos de datos para generar tests precisos y relevantes.

2. **Fase de implementación de test**: Se implementarán los tests generados por el AI Agent, asegurando que cada test sea independiente, repetible y cubra un caso específico. Para esta tarea se utilizará un **AI Agent** (concretamente el agente **testGriter**) especializado en implementación de tests unitarios, que se encargará de escribir el código de los tests.
Para su utilización se deberá configurar en github copilot en agente y solo escribir la ID del test a implementar (ej: `TC-001`) y el agente se encargará de generar el código del test unitario correspondiente, siguiendo las mejores prácticas de testing y asegurando la correcta cobertura del caso de uso.
Esta etapa es la etapa RED de la metodología TDD, donde se implementan los tests y se verifica que fallen inicialmente.
El **AI Agent** solo realizará el test para cumplir con la etapa RED, sin implementar ninguna lógica de negocio ni realizar refactors en esta etapa.
IMPORTANTE: El **AI Agent** no realizará ningún refactor ni cambio en el código de producción durante esta etapa, su única función es implementar el test unitario que corresponda a la ID del test especificada. Si en algún momento el **AI Agent** intenta realizar un refactor o cambio en el código de producción o realizar la implementación para el test indicado, se deberá detener su ejecución y revisar la configuración para asegurar que solo implemente el test unitario correspondiente.
El **AI Agent** también completará la documentación indicando los tests implementados y su estado (RED) en la tabla de tests del índice.

3. **Pruebas manuales del test desarrollado y commit**: Se deberá probar manualmente que el test implementado falla correctamente al ejecutarlo, para verificar que el test está correctamente implementado y que efectivamente está validando el caso de uso esperado. Esta verificación manual es crucial para asegurar que el test cumple su función de detectar la ausencia de la funcionalidad requerida antes de proceder a la etapa de desarrollo y que el test está correctamente en la etapa RED.
Si el test falla correctamente, se deberá realizar un commit del código con un mensaje descriptivo (por ejemplo: `test: TC-001 creado en estado ROJO`) para llevar un control claro de los cambios y las etapas de desarrollo de cada test.

4. **Fase de desarrollo de la funcionalidad**: Una vez que los tests estén implementados y en estado RED, se procederá a desarrollar la funcionalidad necesaria para que el test pase (etapa GREEN de TDD). En esta etapa se realizarán los cambios necesarios en el código de producción para implementar la funcionalidad requerida por cada historia de usuario, asegurando que los tests unitarios implementados anteriormente pasen correctamente. Para esta tarea se puede utilizar el **AI Agent** especializado para esta etapa (**greenTestCreater**), que se encargará de generar el código de producción necesario para que los tests pasen, siguiendo las mejores prácticas de desarrollo y asegurando la correcta implementación de la funcionalidad requerida.
Como segunda opción se puede utilizar el **promptfile** llamado `implementDevelop.prompt.md` que contiene todas las instrucciones básicas y repetibles para que independientemente del agente que se utilice, realice la tarea para cumplir la etapa GREEN de TDD, enfocándose únicamente en implementar la funcionalidad mínima necesaria para que el test pase, sin agregar funcionalidades adicionales ni realizar optimizaciones en esta etapa.

5. **Pruebas manuales de funcionalidad y commit**: Se deberá probar manualmente que el test implementado pasa correctamente al ejecutarlo, para verificar que la funcionalidad desarrollada cumple con los requisitos del caso de uso esperado y que el test está correctamente en la etapa GREEN. Esta verificación manual es crucial para asegurar que la funcionalidad implementada es correcta y que el test cumple su función de validar la presencia de la funcionalidad requerida antes de proceder a la etapa de refactorización y mejoras.
Si el test pasa correctamente, se deberá realizar un commit del código con un mensaje descriptivo (por ejemplo: `feat: implementación para TC-001 VERDE`) para llevar un control claro de los cambios y las etapas de desarrollo de cada test.

6. **Refactorización y mejoras**: Una vez que los tests estén en estado GREEN, se procederá a realizar refactors y mejoras en el código de producción, asegurando que los tests sigan pasando correctamente después de cada cambio. Para esta tarea se puede utilizar el **AI Agent** especializado para esta etapa (**refactorAgent**), que se encargará de generar el código de producción necesario para realizar refactors y mejoras, siguiendo las mejores prácticas de desarrollo y asegurando la correcta implementación de la funcionalidad requerida.
Como alternativa, se puede utilizar el **promptfile** llamado `simplification.prompt.md` que contiene todas las instrucciones básicas y repetibles para que independientemente del agente que se utilice, se realice la tarea de simplificar tanto el test como la implementación para pasar el test, enfocándose únicamente en realizar los cambios necesarios para mejorar el código sin afectar la funcionalidad ni el resultado de los tests implementados.

7. **Pruebas manuales de refactorización y commit**: Se deberá probar manualmente que el test implementado sigue pasando correctamente al ejecutarlo después de cada refactor o mejora realizada, para verificar que los cambios realizados no han afectado la funcionalidad ni el resultado esperado del test. Esta verificación manual es crucial para asegurar que el código refactorizado sigue cumpliendo con los requisitos del caso de uso esperado y que el test sigue siendo válido para validar la presencia de la funcionalidad requerida.
Si el test sigue pasando correctamente, se deberá realizar un commit del código con un mensaje descriptivo (por ejemplo: `refactor: TC-001 refactorizado`) para llevar un control claro de los cambios y las etapas de desarrollo de cada test.

8. **Etapa opcional de refactorización y commit**
Una vez realizada la simplificación del código, se podrá utilizar el **promptfile** `refactorTest.prompt.md` que contiene las instrucciones para buscar una mejora de la implementación y test desarrollados en las etapas anteriores. Si existe una mejora, entonces la IA sugerirá esa mejora y el desarrollador deberá confirmar si quiere registrar esta mejora como documentación en el directorio `documentación/mejoras/*ID de la historia de usuario*`. Dentro del archivo se explicará toda la posible mejora para el test evaluado.
Se deberá actualizar el estado del test en la tabla de tests del índice a [REFACTOR] para indicar que el test ha sido refactorizado y simplificado, pero sigue siendo funcional y pasa correctamente. Esto permitirá llevar un seguimiento claro del estado de cada test en el proceso de desarrollo y refactorización.
Si existe una mejora sugerida y el desarrollador decide registrarla en la documentación, se deberá realizar un commit del código con un mensaje descriptivo (por ejemplo: `doc: registrado posible mejora de implementación de TC-028`) para llevar un control claro de los cambios y mejoras realizadas en cada test.

---

### 🏗️ Arquitectura del Sistema

#### Servicios Involucrados

El sistema distribuido de gestión de quejas ISP consta de los siguientes servicios:

| Servicio | Descripción | Puerto | Responsabilidad en Tests |
|----------|-------------|:------:|------------------------|
| **Frontend** | Aplicación React + Vite | 80 | Tests de UI, interacción de usuario, visualización de datos, exportación CSV, gráficas |
| **Producer** | API Express de ingreso de quejas | 3000 | Validación de requests, publicación a RabbitMQ (no involucrado en dashboard) |
| **Consumer** | Worker de procesamiento de mensajes | 3001 (health) | Priorización de tickets, persistencia inicial (no involucrado directamente en dashboard) |
| **Query Service** | Microservicio de consultas (nuevo) | 3002 | API REST de consultas (`GET /api/tickets`), filtros, paginación, búsquedas, métricas agregadas |

#### Mapeo de Historias de Usuario a Servicios

| Historia | Descripción | Servicios Involucrados |
|----------|-------------|------------------------|
| **HU-01 a HU-08** | Listado, filtros, búsquedas, ordenamiento | **Query Service** (backend) + **Frontend** (visualización) |
| **HU-09** | Métricas agregadas | **Query Service** (endpoint `/api/tickets/metrics`) + **Frontend** (visualización) |
| **HU-10** | Visualización gráfica | **Frontend** (renderizado) + **Query Service** (datos) |
| **HU-11** | Exportación CSV | **Frontend** (generación de archivo) |
| **HU-12** | Actualización de datos | **Frontend** (refresh manual/auto) + **Query Service** (datos actualizados) |

---

### Valores de referencia del dominio

**Tipos de incidente válidos**: `NO_SERVICE`, `INTERMITTENT_SERVICE`, `SLOW_CONNECTION`, `ROUTER_ISSUE`, `BILLING_QUESTION`, `OTHER`

**Prioridades válidas**: `HIGH`, `MEDIUM`, `LOW`, `PENDING`

**Estados válidos**: `RECEIVED`, `IN_PROGRESS`

**Formato ID de ticket**: UUIDv4 (ej: `550e8400-e29b-41d4-a716-446655440000`)

**Formato fecha**: ISO-8601 (ej: `2026-02-18T10:00:00Z`)

---

# Información del documento

## 📅 Fecha de diseño
18 de febrero de 2026

## Fecha de actualización
23 de febrero de 2026

## 📎 Referencias
- [FASE_3_HISTORIAS_RIESGOS.md](./FASE_3_HISTORIAS_RIESGOS.md) - Historias de usuario y matriz de riesgos
- Dominio: Sistema Distribuido de Gestión de Quejas ISP
