# Análisis Técnico de Base de Datos

Fecha: 2026-09-17
Especialista: `database-specialist`
Fuente: `02_decision_scope.md`, `03_architecture_options.md`

---

## D-09. Motor de Base de Datos

**Drivers**: D1 (concurrencia), D2 (transaccionalidad), D5 (escalabilidad)
**RF/RNF**: RF-064, RNF-020, RNF-021, RNF-001, RNF-011

### Opciones

| Opción | Resumen |
|--------|---------|
| **PostgreSQL** | RDBMS open-source, extensible, soporta `SELECT FOR UPDATE`, `SKIP LOCKED`, partitioning, JSON/JSONB, pub/sub |
| **MySQL (InnoDB)** | RDBMS open-source, amplio ecosistema, transacciones ACID, row-level locking |
| **Híbrido (PG + Redis)** | PostgreSQL como fuente de verdad + Redis para caché de stock y locks distribuidos |

### Evaluación

#### PostgreSQL

| Criterio | Evaluación |
|----------|------------|
| **Concurrencia (D-01)** | Excelente. `SELECT FOR UPDATE`, `SKIP LOCKED`, advisory locks. Soporta patrón de reserva temporal nativamente |
| **Transaccionalidad (D-02)** | ACID completo. MVCC (Multi-Version Concurrency Control) permite lecturas concurrentes sin bloqueo. Transacciones SAVEPOINT para compensaciones parciales |
| **Escalabilidad (D-05)** | Read replicas nativas (streaming replication). Partitioning por rango/hash para tablas grandes. Hasta ~50k transacciones/seg con hardware adecuado |
| **Oportunidades de negocio** | JSONB para metadatos flexibles (attributos de producto, respuestas de pago). Pub/sub nativo para eventos entre schemas. Extensiones: `pgcrypto`, `uuid-ossp`, `pg_partman` |
| **Operación en contenedor** | Soporte oficial. Imágenes base optimizadas. pgBouncer para connection pooling. Soporta PVC para persistencia |

#### MySQL (InnoDB)

| Criterio | Evaluación |
|----------|------------|
| **Concurrencia (D-01)** | Aceptable. Row-level locking en InnoDB. Falta `SKIP LOCKED` nativo (disponible desde 8.0.1 como `FOR UPDATE SKIP LOCKED`, pero menos maduro que PG) |
| **Transaccionalidad (D-02)** | ACID completo. Menos flexibilidad en SAVEPOINT y CTEs transaccionales comparado con PG |
| **Escalabilidad (D-05)** | Replicación nativa. Group Replication para HA. Partitioning disponible pero menos flexible |
| **Oportunidades de negocio** | Sin JSONB nativo (tiene JSON tipo, pero no indexable como PG). Menos extensiones disponibles. Ecosistema más orientado a webapps simples |
| **Operación en contenedor** | Excelente. Imágenes oficiales optimizadas |

#### Híbrido (PG + Redis)

| Criterio | Evaluación |
|----------|------------|
| **Concurrencia (D-01)** | Redis ofrece atomicidad O(1) para DECR/INCR de stock. TTL para reservas expirables. Pero introduce consistencia dual |
| **Transaccionalidad (D-02)** | PG mantiene ACID para dominio. Redis es eventualmente consistente para stock cache. Reconciliación necesaria |
| **Escalabilidad (D-05)** | Redis absorbe lecturas de stock (100k+ ops/seg). PG maneja transacciones pesadas. Excelente para lecturas intensivas |
| **Oportunidades de negocio** | Stock hot-path en Redis (consulta rápida). Sesiones, rate limiting, colas de trabajo |
| **Operación en contenedor** | Dos servicios a orquestar. Más complejidad operativa. Redis necesita persistencia (AOF/RDB) |

### Riesgos

| Opción | Riesgo principal | Severidad |
|--------|-----------------|-----------|
| PostgreSQL | Curva de aprendizaje si el equipo no lo conoce. Replicación streaming requiere configuración cuidadosa | Baja |
| MySQL | `SKIP LOCKED` menos probado en producción para este patrón. Falta pub/sub nativo para eventos cross-schema | Media |
| Híbrido | Consistencia dual entre PG y Redis. Si Redis pierde datos, stock puede desincronizarse. Más puntos de fallo | Alta |

### Recomendación Preliminar

**PostgreSQL** como motor principal.

- `SELECT FOR UPDATE SKIP LOCKED` resuelve concurrencia de stock de forma nativa
- MVCC evita bloqueos innecesarios en lecturas (reportes, búsquedas)
- JSONB cubre metadatos flexibles sin schema rígido
- Pub/sub nativo facilita eventos entre schemas (outbox pattern)
- Hasta 25k transacciones/hora es ~7 transacciones/seg — PostgreSQL lo maneja con un solo nodo

**Redis se justifica solo si**: preguntas pendientes #1 y #3 indican que el stock hot-path necesita latencia sub-milisegundo en consultas de disponibilidad. Inicialmente, PG solo es suficiente.

> **Ponytail**: No añadir Redis hasta que haya métricas que lo justifiquen. PostgreSQL solo cubre los requisitos actuales. Agregar Redis duplica la superficie operativa sin necesidad demostrada.

---

## Análisis Profundo de Estrategia de Persistencia

### Dimensión 1: Concurrencia

**Escenario**: 25K transacciones/hora (~7 tx/seg), 10x eventual (~70 tx/seg). Múltiples POS + web concurrentes.

| Motor | Mecanismo | Ventajas | Desventajas | Adecuación |
|-------|-----------|----------|-------------|------------|
| **PostgreSQL** | MVCC + `SELECT FOR UPDATE SKIP LOCKED` | Lecturas no bloquean escrituras. `SKIP LOCKED` permite que POS no espere por web. Advisory locks para operaciones administrative | Configuración de `max_locks_per_transaction` para evitar locks excesivos | **Excelente** — diseño nativo para este patrón |
| **MySQL (InnoDB)** | Row-level locking + `FOR UPDATE SKIP LOCKED` (8.0.1+) | Similar a PG en teoría | `SKIP LOCKED` menos maduro. Menos opciones de tuning para concurrencia mixta | **Aceptable** — funciona pero con menos herramientas |
| **Híbrido (PG+Redis)** | Redis para hot-path (DECR/INCR atómico) | Latencia sub-ms para consultas de stock | Consistency dual. Si Redis se cae, stock puede estar desincronizado. Necesita reconciliación periódica | **Overkill** — Redis añade complejidad sin necesidad demostrada |

**Análisis**: Para 7 tx/seg, PostgreSQL maneja cómodamente con `SKIP LOCKED`. A 70 tx/seg, la contención crece pero sigue siendo manejable con índices adecuados y connection pooling. Redis solo se justificaría si hay picos de 1000+ consultas de stock por segundo, que es poco probable en food retail.

### Dimensión 2: Consistencia

**Escenario**: Stock debe ser consistente entre POS y web. No se permite oversell (RN-02).

| Motor | Consistencia | Garantías | Trade-offs |
|-------|--------------|-----------|------------|
| **PostgreSQL** | ACID completa | Serialización de transacciones. MVCC garantiza lecturas consistentes. `SERIALIZABLE` level disponible | Lecturas pueden necesitar retry en `SERIALIZABLE` |
| **MySQL (InnoDB)** | ACID completa | Similar a PG. `REPEATABLE READ` como default | Menos flexibilidad en niveles de aislamiento |
| **Híbrido** | Eventual entre PG y Redis | PG es fuente de verdad. Redis es caché con TTL | Puede haber desfase temporal entre PG y Redis. Requiere reconciliación |

**Análisis**: Para food retail, la consistencia fuerte es crítica (no oversell). PostgreSQL ofrece `SERIALIZABLE` pero `READ COMMITTED` con `SELECT FOR UPDATE` es suficiente y más performante. MySQL es comparable pero con menos opciones de tuning.

### Dimensión 3: Recuperación (Backup/Restore)

**Escenario**: RPO/RTO no definidos (pregunta #5 pendiente). Estimación: RPO < 1 hora, RTO < 4 horas.

| Motor | Backup | Restore | Ventajas | Desventajas |
|-------|--------|---------|----------|-------------|
| **PostgreSQL** | `pg_dump` (lógico), `pg_basebackup` (físico), WAL archiving | Point-in-time recovery (PITR) con WAL. Streaming replication para HA | Backup consistente. PITR preciso. Soporte para replication slots | Backup lógico puede ser lento en DBs grandes |
| **MySQL** | `mysqldump` (lógico), `mysqlpump`, `xtrabackup` (físico) | Binary log para PITR. Group Replication para HA | `xtrabackup` es rápido para restores. MySQL Enterprise Backup disponible | `mysqldump` puede ser lento. Community edition limita algunas features |
| **Híbrido** | Backup de PG + Redis (RDB/AOF) | Restore de PG + Redis. Reconciliación post-restore | Redis tiene persistencia propia | Dos sistemas a respaldar. Redis puede perder datos si no está configurado correctamente |

**Análisis**: PostgreSQL tiene mejor soporte para PITR y streaming replication. Para RPO < 1 hora, WAL archiving con `archive_timeout = 3600` es suficiente. MySQL es comparable pero con menos flexibilidad en configuración.

### Dimensión 4: Volumen y Rendimiento

**Escenario**: 25K tx/hora, ~100K productos, ~50 tiendas, ~100 usuarios concurrentes. Crecimiento esperado 3x en 3 años.

| Motor | Lecturas | Escrituras | Volumen datos | Índices |
|-------|----------|------------|---------------|---------|
| **PostgreSQL** | Excelente (MVCC) | Excelente | Tablas grandes manejables con partitioning | B-tree, Hash, GiST, SP-GiST, GIN, BRIN |
| **MySQL** | Buena | Buena | Similar a PG | B-tree, Hash, Full-text, Spatial |
| **Híbrido** | Redis: 100K+ ops/seg | PG para escrituras pesadas | Redis: caché limitada por RAM | Redis: solo keys. PG: completos |

**Análisis**: Para el volumen estimado, PostgreSQL es sobrado. 100K productos con índices adecuados da consultas en milisegundos. Partitioning por `store_id` o `date` mejora rendimiento de reportes. MySQL es comparable pero con menos opciones de indexación avanzada.

### Dimensión 5: Auditoría y Compliance

**Escenario**: RF-042 requiere trazabilidad de cambios. RN-08 prohíbe eliminación física de transacciones confirmadas.

| Motor | Auditoría | Triggers | Row Versioning | Extensions |
|-------|-----------|----------|----------------|------------|
| **PostgreSQL** | `pgAudit` para logging detallado. `SELECT FOR UPDATE` con timestamps | Triggers robustos con PL/pgSQL | `xmin/xmax` system columns para versionado | `pgAudit`, `pgcrypto` |
| **MySQL** | Binary log para cambios. Enterprise Audit Log | Triggers simples | No hay system columns equivalentes | MySQL Enterprise Audit |
| **Híbrido** | PG para auditoría completa. Redis sin auditoría nativa | Solo en PG | Solo en PG | Solo en PG |

**Análisis**: PostgreSQL con `pgAudit` ofrece auditoría granular (quién, qué, cuándo, desde dónde). Para compliance de food retail, esto es más que suficiente. MySQL Enterprise Audit es comparable pero requiere licencia.

### Dimensión 6: Crecimiento y Evolución

**Escenario**: Sistema debe crecer de 50 a 200 tiendas en 3 años. Posible expansión a otros países.

| Motor | Escalabilidad | Extensibilidad | Migraciones | Community |
|-------|---------------|----------------|-------------|-----------|
| **PostgreSQL** | Read replicas, partitioning, Citus para sharding | Extensiones (PostGIS, pg_trgm, etc.) | Flyway/Liquibase nativo | Muy activa |
| **MySQL** | Replicación, sharding manual | Menos extensiones | Similar a PG | Muy activa |
| **Híbrido** | PG escala. Redis escala vertical | Solo en PG | Solo en PG | Depende de PG |

**Análisis**: PostgreSQL escala mejor con extensiones como Citus para sharding si se necesita. Para 200 tiendas, un cluster con read replicas es suficiente. MySQL escala pero con más trabajo manual.

---

## D-01. Estrategia de Concurrencia para Inventario

**Drivers**: D1 (integridad bajo concurrencia), RF-064, RNF-020
**Preguntas pendientes**: #1 (¿reserva web bloquea stock para POS?), #3 (¿comportamiento al expirar reserva?)

### Opciones

| Opción | Descripción |
|--------|-------------|
| **Pessimistic locking** | `SELECT FOR UPDATE` — bloquea filas de stock durante la operación |
| **Optimistic locking** | Columna `version` — detecta conflictos al escribir, requiere retry |
| **Reserva temporal** | Stock separado: `available` vs `reserved`. Reserva con TTL |
| **Híbrido** | Reserva temporal para disponibilidad + pessimistic lock al confirmar |

### Evaluación

#### Pessimistic Locking (SELECT FOR UPDATE)

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `BEGIN; SELECT stock FROM products WHERE id = ? FOR UPDATE; UPDATE stock = stock - qty; COMMIT;` |
| **Ventaja** | Garantiza exclusividad. Sin conflictos, sin retry. Simple de implementar |
| **Desventaja** | Bajo alta concurrencia, contención de filas. Los POS y web compiten por las mismas filas. Puede causar deadlocks si hay múltiples productos por orden |
| **Escalabilidad** | Limitada. Cada transacción de venta mantiene un lock hasta COMMIT. Con 25k tx/hora (~7 tx/seg) es manejable, pero a 10x (70 tx/seg) la contención crece |
| **Adecuación** | Adecuada para POS donde la transacción es corta. Problemática para web con sesiones largas |
| **Deadlock** | Riesgo real si dos órdenes compiten por los mismos productos en orden inverso. PostgreSQL detecta deadlocks pero requiere retry |

#### Optimistic Locking (Version Column)

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `UPDATE products SET stock = stock - 1, version = version + 1 WHERE id = ? AND version = ?` |
| **Ventaja** | Sin locks. Alta concurrencia en lecturas. Ideal para read-heavy workloads |
| **Desventaja** | Requiere retry en conflictos. El usuario web puede recibir error si hay conflicto. No evita oversell — lo detecta y rechaza |
| **Escalabilidad** | Excelente para lecturas. Escrituras con alta contención generan muchos retries |
| **Adecuación** | No adecuada para food retail donde el cliente espera confirmación inmediata. Reintentos son confusos para el usuario final |
| **Oportunidad** | Puede complementar reserva temporal para operaciones de lectura |

#### Reserva Temporal (Available + Reserved)

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `stock_available`, `stock_reserved`. Reserva: decrementa available, incrementa reserved con TTL. Confirmación: mueve reserved → vendido. Expiración: retorna reserved → available |
| **Ventaja** | El cliente ve stock real disponible. Reserva con TTL maneja carritos abandonados. Natural para omnichannel — POS y web compiten por el mismo pool |
| **Desventaja** | Más complejo. Requiere job de limpieza para expirar reservas. Dos operaciones en vez de una |
| **Escalabilidad** | Excelente. Las lecturas de `stock_available` son no bloqueantes. La reserva es una escritura ligera |
| **Adecuación** | Perfecta para food retail donde el cliente puede agregar/quitar items. Reserva natural para carrito web |
| **Deadlock** | Mínimo. La reserva es una operación atómica (decrementar available, incrementar reserved) |

#### Híbrido (Reserva + Lock en Confirmación)

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | Reserva temporal para disponibilidad. `SELECT FOR UPDATE` solo al momento de confirmar la venta |
| **Ventaja** | Lo mejor de ambos mundos: disponibilidad sin lock + confirmación con garantía |
| **Desventaja** | Dos mecanismos a mantener. Más complejo que cada uno individual |
| **Escalabilidad** | Excelente. El lock es breve (solo durante la confirmación) |
| **Adecuación** | La más adecuada para omnichannel. Web reserva con TTL, POS confirma con lock |
| **Complejidad** | Media. Requiere diseño cuidadoso de la reserva y la confirmación |

### Riesgos

| Opción | Riesgo | Mitigación |
|--------|--------|------------|
| Pessimistic | Deadlocks entre órdenes multi-producto | `SKIP LOCKED` + retry con backoff. Timeout corto |
| Optimistic | Conflictos frecuentes en horas pico | No recomendada como único mecanismo para este dominio |
| Reserva | Reservas expiradas generan stock "fantasma" si el job falla | Job de limpieza robusto + monitoreo de reservas stale |
| Híbrido | Complejidad de dos mecanismos | Documentación clara, tests de integración para edge cases |

### Recomendación Preliminar

**Reserva temporal como mecanismo principal**, con análisis pendiente de preguntas #1 y #3.

- Reserva temporal natural para food retail (carrito web → confirmación)
- POSIX `FOR UPDATE` solo al confirmar si se necesita garantía adicional
- El patrón `available / reserved` es estándar en e-commerce y se conoce bien

**Preguntas que afectan la recomendación**:
- Si la reserva web bloquea stock para POS → se necesita `SKIP LOCKED` para que POS no espere
- Si la reserva web NO bloquea stock para POS → reserva temporal simple es suficiente

---

## D-02. Modelo Transaccional

**Drivers**: D2 (transaccionalidad financiera), RNF-021
**Depende de**: D-01 (reserva temporal favorece Saga)

### Opciones

| Opción | Descripción |
|--------|-------------|
| **ACID monolítico** | Una transacción SQL cubre inventario + venta + pago |
| **Saga con compensación** | Cada paso es una transacción ACID. Si falla, se ejecuta compensación |
| **Event Sourcing** | Append-only de eventos. Consistencia eventual. Idempotencia inherente |
| **Híbrido (ACID dominio + Saga pagos)** | ACID para inventario/venta. Saga para integración con proveedores de pago |

### Evaluación

#### ACID Monolítico

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `BEGIN; INSERT venta; UPDATE stock; INSERT pago; COMMIT;` |
| **Ventaja** | Simplicidad. Una transacción, un rollback. Garantía fuerte de consistencia |
| **Desventaja** | El pago externo (proveedor) no puede estar dentro de la transacción. Timeout de transacción si el pago tarda |
| **Adecuación** | Adecuada si el pago es síncrono y rápido (<500ms). Problemática si el proveedor de pago es lento o falla |
| **RNF-021** | Cumple ACID completamente. Dinero e inventario son atómicos |
| **Limitación** | No maneja pagos externos con latencia variable. Transacción abierta mucho tiempo = contención |

#### Saga con Compensación

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | Paso 1: Reservar stock (ACID). Paso 2: Procesar pago (ACID). Paso 3: Confirmar venta. Si falla Paso 2 → compensar Paso 1 |
| **Ventaja** | Cada paso es corto y atómico. Maneja pagos externos con latencia. Escalable |
| **Desventaja** | Consistencia eventual entre pasos. Compensaciones pueden fallar. Más complejo de debuggear |
| **Adecuación** | Natural para pagos externos. Reserva temporal + saga es un patrón probado |
| **RNF-021** | Consistencia eventual, no fuerte. Requiere reconciliación |
| **Oportunidad** | La reserva temporal de D-01 ya crea un "paso" natural para el saga |

#### Event Sourcing

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | Append-only de eventos: `StockReserved`, `PaymentProcessed`, `OrderConfirmed`. Proyecciones materializadas |
| **Ventaja** | Idempotencia inherente. Audit trail gratuito. Replay para debugging |
| **Desventaja** | Alta complejidad operativa. Consistencia eventual. Curva de aprendizaje. Over-engineering para food retail |
| **Adecuación** | No justificada. El dominio de food retail no requiere audit trail completo ni replay de eventos |
| **RNF-021** | Consistencia eventual. Requiere manejo cuidadoso de proyecciones |
| **Riesgo** | Over-engineering. Complejidad innecesaria para el 80% de los casos |

#### Híbrido (ACID Dominio + Saga Pagos)

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | ACID para inventario + venta (una transacción SQL). Saga para pago externo (reservar → pagar → confirmar o compensar) |
| **Ventaja** | Lo mejor de ambos: consistencia fuerte para dominio + flexibilidad para pagos externos |
| **Desventaja** | Dos patrones a mantener. Más complejo que cada uno individual |
| **Adecuación** | La más adecuada. Inventario y venta son dominio interno (ACID). Pago es externo (saga) |
| **RNF-021** | ACID para dinero e inventario dentro del dominio. Saga solo para integración externa |
| **Oportunidad** | Outbox pattern para eventos entre módulos (como en Alternativa B) |

### Riesgos

| Opción | Riesgo | Mitigación |
|--------|--------|------------|
| ACID monolítico | Transacción larga si el pago tarda | Timeout corto en transacción. Pago síncrono con timeout agresivo |
| Saga | Compensación falla (stock ya reservado por otro) | Reintentos con backoff. Monitoreo de estados inconsistentes |
| Event Sourcing | Over-engineering. Complejidad innecesaria | No recomendada para este dominio |
| Híbrido | Complejidad de dos patrones | Documentación clara. Tests para edge cases de compensación |

### Recomendación Preliminar

**Híbrido (ACID dominio + Saga pagos)**.

- ACID para inventario + venta: una transacción SQL cubre `INSERT venta`, `UPDATE stock_reserved → vendido`
- Saga para pago externo: reservar stock → llamar proveedor → confirmar o compensar
- Outbox pattern para eventos entre módulos (si se adopta Alternativa B o D)
- La reserva temporal de D-01 ya crea el "paso 1" natural del saga

**Si el pago es síncrono y rápido** (<500ms), ACID monolítico es suficiente. La decisión depende del proveedor de pago (pregunta #4 pendiente).

---

## D-03. Estrategia de Idempotencia

**Drivers**: D3 (idempotencia), RF-042, RN-07, RNF-022

### Opciones

| Opción | Descripción |
|--------|-------------|
| **Idempotency key** | Cliente envía UUID único. DB almacena resultado. Reintentos retornan el mismo resultado |
| **Constraint DB** | `UNIQUE` constraint en campos naturales (número de orden, referencia de pago) |
| **Lock distribuido + TTL** | Redis/DB lock con TTL. Un solo proceso puede ejecutar la operación |
| **Event sourcing** | Inherente — eventos únicos por sequence number. No aplica si no se usa event sourcing |

### Evaluación

#### Idempotency Key

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `INSERT INTO idempotency_keys (key, result, created_at) VALUES (?, ?, ?) ON CONFLICT (key) DO NOTHING` |
| **Ventaja** | Universal. Funciona para cualquier operación. Cliente controla la key |
| **Desventaja** | Requiere tabla de idempotency keys. Cleanup periódico. Más movimiento en DB |
| **Adecuación** | Excelente para APIs externas (web → backend). El cliente genera el UUID |
| **RNF-022** | Cumple. Reintentos con la misma key retornan el mismo resultado |
| **Oportunidad** | Se puede combinar con constraint DB para doble protección |

#### Constraint DB

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `UNIQUE (order_number)`, `UNIQUE (payment_reference)` — la DB rechaza duplicados |
| **Ventaja** | Simplicidad. Un constraint. Sin tabla adicional. La DB lo hace todo |
| **Desventaja** | Solo funciona si hay un campo natural único. Error de constraint es poco amigable para el cliente |
| **Adecuación** | Adecuada como complemento. No suficiente sola para operaciones complejas |
| **RNF-022** | Cumple parcialmente — previene duplicados pero no retorna el resultado anterior |
| **Oportunidad** | Complemento ideal para idempotency key |

#### Lock Distribuido + TTL

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | `SET lock:order:123 NX EX 30` en Redis. Si el lock existe, el reintento se rechaza |
| **Ventaja** | Previene ejecución simultánea. TTL evita locks permanentes |
| **Desventaja** | Requiere Redis. No cubre reintentos después de que el lock expira. Más infraestructura |
| **Adecuación** | Adecuada para operaciones concurrentes, no para idempotencia pura |
| **RNF-022** | No cumple directamente — previene concurrencia, no duplicación |
| **Oportunidad** | Útil para prevención de race conditions, no como estrategia de idempotencia principal |

#### Event Sourcing

| Aspecto | Evaluación |
|---------|------------|
| **Mecanismo** | Cada evento tiene sequence number único. El mismo comando se ignora si ya fue procesado |
| **Ventaja** | Idempotencia inherente. Sin esfuerzo adicional |
| **Desventaja** | Solo funciona con event sourcing. No aplica si se usa ACID monolítico o saga |
| **Adecuación** | No recomendada — el dominio no justifica event sourcing |
| **RNF-022** | Cumple inherentemente, pero con la complejidad de event sourcing |

### Riesgos

| Opción | Riesgo | Mitigación |
|--------|--------|------------|
| Idempotency key | Tabla crece sin límite | Job de cleanup periódico (retener 30 días). Indexación en `key` |
| Constraint DB | Error poco amigable | Mapear error de constraint a respuesta idempotente |
| Lock distribuido | Lock expira y segundo proceso ejecuta | No recomendada como estrategia principal de idempotencia |
| Event Sourcing | Over-engineering | No recomendada para este dominio |

### Recomendación Preliminar

**Idempotency key + constraint DB** (doble protección).

- Idempotency key para APIs externas (web → backend): cliente envía UUID, backend almacena resultado
- Constraint DB para campos naturales (número de orden, referencia de pago): previene duplicados a nivel de DB
- Tabla `idempotency_keys` con TTL de 30 días y cleanup periódico
- Para POS: si el POS genera la orden localmente, el idempotency key viene del POS

**No se necesita lock distribuido** para idempotencia. El lock es útil para concurrencia (D-01), no para prevención de duplicados.

---

## Tabla Resumen Comparativa

### D-09 — Motor de Base de Datos

| Criterio | PostgreSQL | MySQL | Híbrido (PG+Redis) |
|----------|:----------:|:-----:|:-------------------:|
| SELECT FOR UPDATE SKIP LOCKED | ✅ Maduro | ⚠️ Desde 8.0.1 | ✅ vía Redis |
| MVCC (lecturas sin lock) | ✅ Nativo | ⚠️ Limitado | ✅ vía Redis |
| JSONB indexable | ✅ Nativo | ❌ No | ✅ Nativo |
| Pub/sub entre schemas | ✅ Nativo | ❌ No | ✅ Nativo |
| Complejidad operativa | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Recomendación** | **✅ Elegido** | ❌ No | ⏳ Evaluar después |

### D-01 — Estrategia de Concurrencia

| Criterio | Pessimistic | Optimistic | Reserva Temporal | Híbrido |
|----------|:-----------:|:----------:|:----------------:|:-------:|
| Evita oversell | ✅ | ⚠️ Detecta | ✅ | ✅ |
| Escalabilidad | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Simplicidad | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Deadlock risk | ⚠️ Alto | ✅ Nulo | ✅ Bajo | ✅ Bajo |
| Adecuación food retail | ⚠️ | ❌ | ✅ | **✅ Elegido** |

### D-02 — Modelo Transaccional

| Criterio | ACID | Saga | Event Sourcing | Híbrido |
|----------|:----:|:----:|:--------------:|:-------:|
| Consistencia fuerte | ✅ | ⚠️ Eventual | ⚠️ Eventual | ✅ (dominio) |
| Pagos externos | ⚠️ Lento | ✅ Natural | ✅ Natural | ✅ Natural |
| Simplicidad | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐⭐ |
| Over-engineering | ✅ No | ✅ No | ❌ Sí | ✅ No |
| **Recomendación** | ⏳ Si pago síncrono | ⏳ Si pago async | ❌ | **✅ Elegido** |

### D-03 — Estrategia de Idempotencia

| Criterio | Idempotency Key | Constraint DB | Lock Distribuido | Event Sourcing |
|----------|:---------------:|:-------------:|:-----------------:|:--------------:|
| Previene duplicados | ✅ | ✅ | ⚠️ Parcial | ✅ |
| Retorna resultado anterior | ✅ | ❌ | ❌ | ✅ |
| Simplicidad | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| Sin infra adicional | ✅ | ✅ | ❌ Redis | ❌ |
| **Recomendación** | **✅ Elegido** | **✅ Elegido** | ❌ | ❌ |

---

## Preguntas que Afectan las Recomendaciones

| # | Pregunta | Afecta | Impacto |
|---|----------|--------|---------|
| 1 | ¿Reserva web bloquea stock para POS? | D-01 | Si SÍ: necesita `SKIP LOCKED` para que POS no espere. Si NO: reserva temporal simple |
| 3 | ¿Comportamiento al expirar reserva? | D-01 | Determina diseño de tabla de reservas y job de limpieza |
| 4 | ¿Proveedor de pago iniciales? | D-02 | Si síncrono rápido: ACID monolítico. Si async: saga necesario |
| 5 | ¿RPO/RTO objetivo? | D-09 | Afecta si se necesita streaming replication o no |

---

## Recomendación Preliminary Consolidada

| Decisión | Recomendación | Condición |
|----------|--------------|-----------|
| **D-09** | PostgreSQL | Sin Redis inicialmente |
| **D-01** | Reserva temporal + lock en confirmación | Depende de preguntas #1 y #3 |
| **D-02** | Híbrido (ACID dominio + Saga pagos) | Si pago es async. ACID simple si síncrono |
| **D-03** | Idempotency key + constraint DB | Combinación de ambos mecanismos |

> **Nota**: Estas son recomendaciones preliminares del database-specialist. El solution-leader consolidará con las demás áreas (architect, security, devops) para la decisión final.