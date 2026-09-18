# Alcance de Decisiones Arquitectónicas

Fecha: 2026-09-17
Fuente: 01_requirements_analysis.md

---

## Decisiones que Deben Tomarse

### D-01. Estrategia de Concurrencia para Inventario

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D1 (Integridad bajo concurrencia), RF-064, RNF-020 |
| **Alcance** | Seleccionar mecanismo para evitar sobreventa entre POS y web |
| **Opciones a comparar** | 1) Pessimistic locking (SELECT FOR UPDATE) — 2) Optimistic locking (version column) — 3) Reserva temporal con stock separado (reserved vs available) — 4) híbrido (reserva + lock en confirmación) |
| **Obligatoria** | Sí. Es el driver #1 del sistema |
| **Especialista requerido** | `database-specialist` |
| **Bloquea** | D-03 (transaccionalidad), D-05 (escalabilidad) |

### D-02. Modelo de Datos Transaccional

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D2 (Transaccionalidad financiera), RNF-021 |
| **Alcance** | Definir fronteras de transacciones para ventas, pagos, inventario y devoluciones |
| **Opciones a comparar** | 1) Transacciones ACID monolíticas — 2) Saga con compensación — 3) Event sourcing con proyección — 4) Híbrido (ACID para dominio, saga para pagos externos) |
| **Obligatoria** | Sí. Dinero e inventario requieren atomicidad |
| **Especialista requerido** | `database-specialist` |
| **Depende de** | D-01 (si se usa reserva temporal, la saga es más natural) |

### D-03. Estrategia de Idempotencia

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D3 (Idempotencia), RF-042, RN-07, RNF-022 |
| **Alcance** | Mecanismo para evitar duplicados ante reintentos en operaciones críticas |
| **Opciones a comparar** | 1) Idempotency key por operación — 2) Deduplicación por constraint DB — 3) Lock distributed + TTL — 4) Event sourcing (inherente) |
| **Obligatoria** | Sí. RN-07 lo exige |
| **Especialista requerido** | `database-specialist` |
| **Bloquea** | Nada directamente, pero afecta diseño de APIs |

### D-04. Arquitectura del Sistema de Pagos

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D2 (Transaccionalidad), RF-063 |
| **Alcance** | Capa de integración con proveedores de pago desacoplada del dominio |
| **Opciones a comparar** | 1) Adapter pattern síncrono — 2) Cola de mensajes asíncrona — 3) Híbrido (síncrono para POS, asíncrono para web) — 4) Gateway centralizado con retry |
| **Obligatoria** | Sí. RF-063 exige desacoplamiento |
| **Especialista requerido** | `architect` |
| **Pendiente** | Proveedor(es) de pago iniciales (pregunta #4 abierta) |

### D-05. Estrategia de Escalabilidad

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D4 (Escalabilidad por sucursales), RNF-010, RNF-011 |
| **Alcance** | Cómo crece el sistema sin rediseñar el dominio |
| **Opciones a comparar** | 1) Monolito modular con sharding por sucursal — 2) Microservicios por dominio — 3) Monolito + réplicas de lectura — 4) Modular monolith con despliegue independiente |
| **Obligatoria** | Sí. RNF-011 exige evaluar 10x |
| **Especialista requerido** | `architect` |
| **Nota** | CA-09 prohíbe microservicios sin justificación |

### D-06. Despliegue y Contenerización

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D5 (Portabilidad), RNF-060, RNF-061, RNF-062 |
| **Alcance** | Cómo se empaqueta y despliega el sistema |
| **Opciones a comparar** | 1) Docker simple + docker-compose — 2) Docker + orquestador (K3s/k3d) — 3) Docker + CI/CD pipeline — 4) Nix/Flox para entornos reproducibles |
| **Obligatoria** | Sí. RNF-061 lo exige |
| **Especialista requerido** | `devops` |

### D-07. Estrategia de Seguridad

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D6 (Seguridad), RNF-030 a RNF-035 |
| **Alcance** | Autenticación, autorización, cifrado, gestión de secretos |
| **Opciones a comparar** | 1) JWT + RBAC interno — 2) OAuth2/OIDC con provider externo — 3) Session-based + CSRF — 4) Híbrido (JWT para API, sesiones para POS) |
| **Obligatoria** | Sí. No negociable |
| **Especialista requerido** | `security-reviewer` |

### D-08. Estrategia de Observabilidad

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D7 (Observabilidad), RNF-050 a RNF-052 |
| **Alcance** | Logs, métricas, trazabilidad distribuida |
| **Opciones a comparar** | 1) OpenTelemetry + backend (Grafana/Signoz) — 2) ELK stack — 3) Solución managed (Datadog/similar) — 4) Structured logs + métricas básicas |
| **Obligatoria** | Sí. RNF-050 a RNF-052 |
| **Especialista requerido** | `devops` |

### D-09. Estrategia de Base de Datos

| Aspecto | Detalle |
|---------|---------|
| **Driver** | D1 (Concurrencia), D2 (Transaccionalidad), D5 (Escalabilidad) |
| **Alcance** | Motor de persistencia que soporte los drivers anteriores |
| **Opciones a comparar** | 1) PostgreSQL — 2) MySQL — 3) Híbrido (transaccional + caché) |
| **Obligatoria** | Sí. Todo lo demás depende de esto |
| **Especialista requerido** | `database-specialist` |
| **Nota** | CA-04: justificar sin sesgo previo |

---

## Especialistas Necesarios

| Orden | Especialista | Para | Dependencias |
|-------|--------------|------|--------------|
| 1 | `database-specialist` | D-01 (concurrencia), D-02 (transaccionalidad), D-03 (idempotencia), D-09 (motor DB) | Ninguna |
| 2 | `architect` | D-04 (pagos), D-05 (escalabilidad) | D-09 (requiere saber el motor DB) |
| 3 | `security-reviewer` | D-07 (seguridad) | Independiente |
| 4 | `devops` | D-06 (despliegue), D-08 (observabilidad) | D-09 (requiere saber el motor DB) |

---

## Restricciones Críticas

| # | Restricción | Fuente | Impacto |
|---|-------------|--------|---------|
| RC-01 | **Linux obligatorio** | RNF-060 | Elimina opciones Windows-only |
| RC-02 | **Contenerización obligatoria** | RNF-061 | Tecnología debe soportar Docker |
| RC-03 | **Sin secretos en código** | RNF-034 | Requiere vault o env vars |
| RC-04 | **Sin Kubernetes sin justificación** | CA-09 | No orquestación compleja a menos que se justifique |
| RC-05 | **Tecnologías con mantenimiento activo** | Restricción general | No usar tecnologías abandonadas |
| RC-06 | **RPO/RTO por definir** | RNF-041 | Afecta selección de infraestructura DB |
| RC-07 | **25.000 transacciones/hora** | RNF-001 | Define mínimo de capacidad |
| RC-08 | **10x evaluable** | RNF-011 | Arquitectura debe soportar escalado |

---

## Preguntas Pendientes (Bloqueantes para D-01 y D-02)

| # | Pregunta | Afecta | Estado |
|---|----------|--------|--------|
| 1 | ¿Las reservas web bloquean stock para POS o solo lo reservan lógicamente? | D-01 (estrategia de concurrencia) | **PENDIENTE** |
| 2 | ¿Qué es exactamente una "transacción de negocio" para dimensionar? | D-05 (escalabilidad) | **PENDIENTE** |
| 3 | ¿Comportamiento al expirar reserva web? | D-01 (diseño de tabla de reservas) | **PENDIENTE** |
| 4 | ¿Proveedor(es) de pago iniciales? | D-04 (arquitectura de pagos) | **PENDIENTE** |
| 5 | ¿RPO/RTO objetivo? | D-06 (despliegue), D-09 (DB replication) | **PENDIENTE** |

> **Nota**: Las preguntas 1 y 3 son **críticas** para diseñar D-01 (concurrencia). Sin respuesta, no se puede definir el mecanismo correcto.

---

## Orden de Resolución

```
1.database-specialist  →  D-09, D-01, D-02, D-03
         ↓
2.architect            →  D-04, D-05
         ↓
3.security-reviewer    →  D-07
         ↓
4.devops               →  D-06, D-08
         ↓
5.solution-leader      →  Consolidación final + ADR
```

---

## Criterios de Aceptación Cubiertos

| CA | Estado |
|----|--------|
| CA-01. Cada decisión mapeada a RF/RNF | ✅ Cada D-XX tiene driver y fuente |
| CA-02. Comparar al menos 2 alternativas | ✅ Cada decisión lista 2-4 opciones |
| CA-03. Evitar sobreventa | ✅ D-01 dedicado a esto |
| CA-04. Base de datos sin sesgo | ✅ D-09 compara opciones |
| CA-05. 25.000 transacciones/hora | ✅ RC-07 + D-05 |
| CA-06. Evaluar 10x | ✅ D-05 incluye escenario |
| CA-07. Idempotencia | ✅ D-03 dedicado |
| CA-08. Seguridad, observabilidad, recuperación | ✅ D-07, D-08, RC-06 |
| CA-09. No Kubernetes sin justificación | ✅ RC-04 |
| CA-10. Sin hallazgos bloqueantes | ⏳ Pendiente preguntas #1 y #3 |
