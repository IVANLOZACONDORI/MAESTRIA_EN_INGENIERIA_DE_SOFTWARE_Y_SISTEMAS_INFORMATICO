# Alternativas Arquitectónicas

Fecha: 2026-09-17
Fuente: 02_decision_scope.md

---

## Alternativa A — Monolito Transaccional

```
┌─────────────────────────────────────────────┐
│              API Gateway / Router            │
├─────────────────────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐  │
│  │Auth │ │Inv. │ │Ventas│ │Pagos│ │Comp.│  │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘  │
│     └───────┴───────┴───────┴───────┘      │
│              Transaction Manager             │
├─────────────────────────────────────────────┤
│         PostgreSQL (ACID completo)           │
└─────────────────────────────────────────────┘
```

| Dimensión | Valoración | Justificación |
|-----------|------------|---------------|
| **Complejidad** | Baja | Todo en un proceso. Un deploy, una DB, una transacción por operación |
| **Escalabilidad** | Media | Escala vertical (más CPU/RAM). Réplicas de lectura para reportes. Límite natural en ~30k transacciones/hora con optimización |
| **Consistencia** | Alta | ACID nativo. Una transacción SQL cubre inventario + venta + pago |
| **Costo operativo** | Bajo | Un contenedor, una DB, monitoreo básico. Sin orquestación |
| **Mantenibilidad** | Media | Fácil de entender, pero acoplamiento entre módulos puede crecer |

**Idempotencia**: Constraint UNIQUE + idempotency key en tabla.
**Concurrencia**: SELECT FOR UPDATE en filas de stock.
**Pagos**: Adapter síncrono dentro del monolito.

**Riesgo principal**: Un bug en cualquier módulo puede afectar todo el sistema.

---

## Alternativa B — Modular Monolith (Boundary by Module)

```
┌──────────────────────────────────────────────────────┐
│                   API Gateway                         │
├──────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Module:  │  │ Module:  │  │ Module:  │  ...      │
│  │ Inventario│ │ Ventas  │  │ Pagos   │           │
│  │ [own DB  │  │ [own DB  │  │ [own DB  │           │
│  │  schema] │  │  schema] │  │  schema] │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       └──────────────┴──────────────┘                │
│            Inter-module communication:                │
│            Events (in-process) + DB triggers          │
├──────────────────────────────────────────────────────┤
│         PostgreSQL (schemas separados por módulo)     │
└──────────────────────────────────────────────────────┘
```

| Dimensión | Valoración | Justificación |
|-----------|------------|---------------|
| **Complejidad** | Media | Módulos con límites claros. Una DB pero schemas aislados. Más estructura que A |
| **Escalabilidad** | Media-Alta | Puede escalar módulos independientemente con réplicas. Sharding por schema si es necesario |
| **Consistencia** | Alta | ACID dentro de cada módulo. Eventos transaccionales entre módulos (outbox pattern) |
| **Costo operativo** | Bajo-Medio | Un contenedor, una DB con múltiples schemas. Outbox polling o CDC para eventos |
| **Mantenibilidad** | Alta | Límites claros. Un módulo puede refactorizarse sin afectar otros. Despliegue único pero código organizado |

**Idempotencia**: Idempotency key por módulo + constraint.
**Concurrencia**: Reserva temporal (stock_available, stock_reserved) + lock en confirmación.
**Pagos**: Módulo separado con cola de salida (outbox) + webhook de confirmación.

**Riesgo principal**: Despliegue monolítico — un cambio en un módulo redeploya todo.

---

## Alternativa C — Event-Sourced (Event Store + Projections)

```
┌──────────────────────────────────────────────────────┐
│                   API Gateway                         │
├──────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Command  │  │ Command  │  │ Command  │           │
│  │ Handler  │  │ Handler  │  │ Handler  │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       └──────────────┴──────────────┘                │
│              Event Store (append-only)                │
├──────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │Projection│  │Projection│  │Projection│           │
│  │ (stock)  │  │ (ventas) │  │ (reportes)│          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       └──────────────┴──────────────┘                │
│         PostgreSQL + Event Store Table               │
└──────────────────────────────────────────────────────┘
```

| Dimensión | Valoración | Justificación |
|-----------|------------|---------------|
| **Complejidad** | Alta | Event sourcing + projections + manejo de eventos. Equipo debe entender el patrón |
| **Escalabilidad** | Alta | Projections escalan independientemente. Read/write separation natural |
| **Consistencia** | Alta eventual | Eventual consistency en proyecciones. Idempotencia inherente por orden de eventos |
| **Costo operativo** | Medio-Alto | Más componentes, más infraestructura. Event store crece sin límite |
| **Mantenibilidad** | Media | Audit trail gratuito. Replay de eventos. Pero debugging más complejo |

**Idempotencia**: inherente (eventos únicos por sequence number).
**Concurrencia**: Append-only + proyecciones materializadas.
**Pagos**: Evento de confirmación de pago async.

**Riesgo principal**: Curva de aprendizaje. Complejidad operativa. Over-engineering si el dominio no lo justifica.

---

## Alternativa D — Monolito + Cola Externa (Pagos Async)

```
┌──────────────────────────────────────────────────────┐
│                   API Gateway                         │
├──────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Auth    │  │ Inventario│ │  Ventas  │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       └──────────────┴──────────────┘                │
│              PostgreSQL (ACID)                        │
├──────────────────────────────────────────────────────┤
│     Cola de Mensajes (Redis/RabbitMQ/Bull)           │
│  ┌──────────────────────────────────────────┐        │
│  │  Pagos pendientes → Proveedor → Webhook  │        │
│  └──────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────┘
```

| Dimensión | Valoración | Justificación |
|-----------|------------|---------------|
| **Complejidad** | Media | Monolito simple + una cola para pagos. Dos infraestructuras但 conceptuales simples |
| **Escalabilidad** | Media-Alta | Monolito escala vertical. Cola absorbe pico de pagos. Puede separar worker de pagos |
| **Consistencia** | Alta | ACID para dominio. Cola garantiza delivery al menos una vez. Reconciliación para consistencia final |
| **Costo operativo** | Medio | Monolito + Redis/RabbitMQ. Más que A pero manejable |
| **Mantenibilidad** | Alta | Pagos desacoplados sin complejidad de event sourcing. Dominio simple y directo |

**Idempotencia**: Idempotency key + dedup en consumer de pagos.
**Concurrencia**: Reserva temporal + lock en confirmación (como B).
**Pagos**: Cola Redis/Bull + worker separado. Webhook de confirmación actualiza DB.

**Riesgo principal**: Cola como punto único de fallo. Necesita monitoring y dead-letter queue.

---

## Alternativa E — Monolito Vertical con Redis como Cache de Stock

```
┌──────────────────────────────────────────────────────┐
│                   API Gateway                         │
├──────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Auth    │  │ Inventario│ │  Ventas  │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       └──────────────┴──────────────┘                │
│              PostgreSQL (fuente de verdad)            │
├──────────────────────────────────────────────────────┤
│         Redis (stock cache + reservation locks)       │
│  ┌──────────────────────────────────────────┐        │
│  │  stock:sucursal:producto → count         │        │
│  │  reserve:order_id → TTL (expirable)      │        │
│  └──────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────┘
```

| Dimensión | Valoración | Justificación |
|-----------|------------|---------------|
| **Complejidad** | Baja-Media | Dos infraestructuras pero roles claros: Redis = stock rápido, PG = persistencia |
| **Escalabilidad** | Alta | Redis absorbe lecturas de stock (100k+ ops/seg). PG maneja transacciones pesadas |
| **Consistencia** | Media-Alta | Stock en Redis puede tener drift con PG. Reconciliación periódica necesaria |
| **Costo operativo** | Medio | PostgreSQL + Redis. Redis es ligero pero requiere memoria |
| **Mantenibilidad** | Media | Dos sistemas que mantener. Pero Redis es simple y bien documentado |

**Idempotencia**: Constraint DB + idempotency key.
**Concurrencia**: Redis原子操作 (DECR/INCR) para stock + reservas expirables + sync a PG.
**Pagos**: Adapter síncrono (como A).

**Riesgo principal**: Consistencia entre Redis y PostgreSQL. Necesita sync strategy confiable.

---

## Alternativa F — Monolito con Event-Driven Interno (In-Process)

```
┌──────────────────────────────────────────────────────┐
│                   API Gateway                         │
├──────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────┐    │
│  │           Event Bus In-Process                │    │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐        │    │
│  │  │Auth │→ │Inv. │→ │Ventas│→ │Pagos│        │    │
│  │  └─────┘  └─────┘  └─────┘  └─────┘        │    │
│  └──────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────┤
│         PostgreSQL (con CDC/Outbox para eventos)      │
└──────────────────────────────────────────────────────┘
```

| Dimensión | Valoración | Justificación |
|-----------|------------|---------------|
| **Complejidad** | Media | Un proceso, pero con event bus interno. Más estructura que A sin tanta infra como C |
| **Escalabilidad** | Media | Como A pero con mejor separación. Workers externos pueden consumir eventos vía CDC |
| **Consistencia** | Alta | ACID + eventos transaccionales (outbox). Consistencia fuerte dentro del proceso |
| **Costo operativo** | Bajo-Medio | Un contenedor + PG. Eventos in-process sin infra adicional |
| **Mantenibilidad** | Alta | Eventos desacoplan módulos sin infra externa. Fácil de debuggear |

**Idempotencia**: Outbox pattern + consumer idempotency.
**Concurrencia**: Reserva temporal (como B) + outbox para eventos cross-module.
**Pagos**: Evento async in-process → worker consume → llama proveedor → confirma vía evento.

**Riesgo principal**: Eventos in-process no sobreviven crash del proceso. Outbox en DB mitiga esto.

---

## Matriz Comparativa

| Criterio (peso) | A: Monolito TX | B: Modular | C: Event-Sourced | D: Monolito + Cola | E: Redis Cache | F: Event-Driven |
|-----------------|:--------------:|:----------:|:-----------------:|:-------------------:|:--------------:|:---------------:|
| **Complejidad** (20%) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Escalabilidad** (25%) | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Consistencia** (25%) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Costo operativo** (15%) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Mantenibilidad** (15%) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Puntuación ponderada** | **4.20** | **4.40** | **3.30** | **4.05** | **3.85** | **3.95** |

---

## Análisis por Driver Arquitectónico

| Driver | A | B | C | D | E | F |
|--------|---|---|---|---|---|---|
| D1. Concurrencia inventario | FOR UPDATE | Reserva temporal | Event store | Reserva + cola | Redis atomic | Outbox + reserva |
| D2. Transaccionalidad | ACID simple | ACID por schema | Eventual | ACID + cola | ACID + Redis sync | ACID + outbox |
| D3. Idempotencia | Constraint DB | Key + constraint | Inherente | Key + dedup | Constraint DB | Outbox + key |
| D4. Escalabilidad 10x | Vertical + réplicas | Horizontal por schema | Projections | Vertical + worker | Redis cache hit | CDC workers |
| D5. Portabilidad | Docker simple | Docker simple | Docker multi-svc | Docker + Redis | Docker + Redis | Docker simple |
| D6. Seguridad | Monolítico | Por módulo | Por projection | Monolito + cola | Monolito + Redis | Monolito + events |

---

## Recomendación Preliminar

**Alternativa B (Modular Monolith)** como punto de partida con elementos de D:

1. **Monolito con schemas separados** — simplicidad operativa, límites claros
2. **Reserva temporal** para stock (D-01 crítico)
3. **Cola simple para pagos** si se justifica async (D-04)
4. **Evitar C (Event Sourced)** a menos que audit trail sea crítico — over-engineering para el 80% de los casos
5. **Evitar E (Redis cache)** inicialmente — introduce consistencia dual sin necesidad demostrada

> La pregunta #1 (¿reserva web bloquea stock para POS?) determina si B es suficiente o se necesita E.

---

## Siguiente Fase

`database-specialist` evalúa D-09 (motor DB), D-01 (concurrencia), D-02 (transaccionalidad) y D-03 (idempotencia) con las alternativas definidas.
