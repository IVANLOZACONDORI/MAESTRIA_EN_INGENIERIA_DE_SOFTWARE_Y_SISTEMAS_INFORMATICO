# Análisis de Requisitos — Tienda Omnicanal de Alimentos

Fecha: 2026-09-17

---

## Resumen Ejecutivo

Sistema transaccional omnicanal (físico + web) para venta de alimentos. Dominio complejo con inventario compartido entre canales, compras a proveedores, transferencias entre sucursales, pagos y devoluciones. Requiere integridad de datos bajo concurrencia y escalabilidad horizontal por crecimiento de sucursales.

---

## Requisitos Funcionales Críticos

| ID | Requisito | Impacto | Notas |
|----|-----------|---------|-------|
| RF-041 | Entradas, salidas, reservas, ajustes y transferencias de inventario | **ALTO** | Core del dominio. Requiere concurrencia controlada |
| RF-062 | Validar/reservar stock antes de confirmar pedido web | **ALTO** | Reserva temporal con expiración (RN-03) |
| RF-064 | Reflejar ventas web e inventario sin sobreventa frente al canal físico | **ALTO** | Requiere coordinación transaccional entre canales |
| RF-050/051 | Ventas presenciales y descuento de inventario | **ALTO** | Punto de venta con disponibilidad inmediata |
| RF-042 | Evitar doble descuento ante reintentos | **ALTO** | Idempotencia obligatoria (RN-07) |
| RF-063 | Pagos con interfaz desacoplada del proveedor | **MEDIO** | Patrón anti-correlación requerido |
| RF-070 | Devoluciones parciales o totales | **MEDIO** | Revierte inventario y afecta pagos |
| RF-031 | Recepción parcial o total de mercadería | **MEDIO** | Actualización condicional de inventario |
| RF-040 | Inventario por producto y sucursal | **ALTO** | Modelo de datos central |
| RF-010 | Sucursales y cajas/POS | **MEDIO** | Entidad escalable (RNF-010) |

---

## Requisitos No Funcionales Críticos

| ID | Requisito | Impacto | Notas |
|----|-----------|---------|-------|
| RNF-001 | 25.000 transacciones de negocio/hora | **ALTO** | Capacidad inicial. Define escalabilidad |
| RNF-005 | Integridad bajo concurrencia entre canal físico y web | **ALTO** | Requiere mecanismo de concurrencia explícito |
| RNF-020 | Evitar sobreventa por condiciones de carrera | **ALTO** | Locking o control de concurrencia a nivel de fila |
| RNF-021 | Fronteras transaccionales para dinero e inventario | **ALTO** | Transacciones ACID o compensatorias |
| RNF-022 | Evitar duplicados ante reintentos | **ALTO** | Idempotencia en capa de servicios |
| RNF-030-035 | Seguridad completa (auth, authz, cifrado, secretos) | **ALTO** | No negociable |
| RNF-040-042 | Backup, RPO/RTO, tolerancia a fallos externos | **MEDIO** | Definir antes de producción |
| RNF-050-052 | Observabilidad (logs, métricas, correlación) | **MEDIO** | Operabilidad del sistema |
| RNF-060-062 | Linux, contenerización, separación de ambientes | **ALTO** | Restricción de despliegue |

---

## Inconsistencias Detectadas

| # | Descripción | Severidad |
|---|-------------|-----------|
| 1 | RNF-001 define 25.000 transacciones/hora pero RNF-003 aclara que "una transacción de negocio no equivale a una sentencia SQL". Falta definición operativa de qué constituye una "transacción de negocio" para dimensionar correctamente. | **MEDIA** |
| 2 | RF-062 (reserva de stock) y RF-064 (sin sobreventa entre canales) requieren mecanismo de reserva temporal, pero no se define si las reservas web bloquean stock disponible para POS o solo lo "apuntan". | **ALTA** |
| 3 | RN-03 indica que "reservas web expiran según parámetro configurable" pero no se define qué pasa con el stock cuando expira (¿vuelve a disponible automáticamente? ¿notifica al cliente?). | **MEDIA** |
| 4 | RF-042 (evitar doble descuento) y RN-07 (idempotencia) se mencionan pero no se define el mecanismo de idempotencia esperado (token, deduplicación, etc.). | **BAJA** |

---

## Riesgos Identificados

| # | Riesgo | Probabilidad | Impacto | Mitigación |
|---|--------|--------------|---------|------------|
| 1 | **Sobreventa entre canales**: La coordinación en tiempo real entre POS y web es el riesgo técnico más alto. Un retraso de milisegundos puede causar sobreventa. | Alta | Crítico | Mecanismo de reserva con locks optimistas o pesimistas + expiración |
| 2 | **Escalabilidad de inventario**: 25.000 transacciones/hora con inventario por sucursal puede generar cuellos de botella en tablas de stock. | Media | Alto | Evaluar particionamiento, caché de stock, o estrategia de concurrencia |
| 3 | **Complejidad de devoluciones**: Devoluciones parciales con pagos parciales y reversión de inventario es un flujo complejo que puede causar inconsistencias. | Media | Alto | Flujo state-machine con auditoría completa |
| 4 | **Integración de pagos**: Múltiples proveedores de pago con comportamiento impredecible (timeouts, reintentos). | Media | Medio | Capa de anti-correlación + cola de reconciliación |
| 5 | **Definición de RPO/RTO**: Sin definir antes de producción, el sistema puede diseñarse sin tolerancia a fallos adecuada. | Media | Alto | Definir en fase de diseño antes de seleccionar infraestructura |

---

## Preguntas Abiertas

| # | Pregunta | Requiere respuesta de | Impacto en diseño |
|---|----------|----------------------|-------------------|
| 1 | ¿Qué constituye exactamente una "transacción de negocio" para el cálculo de 25.000/hora? ¿Una venta, un descuento de stock, o una operación compuesta? | Producto/Dominio | Define dimensionamiento de capacidad |
| 2 | ¿Las reservas web deben bloquear stock físicamente para POS o solo mantener una reserva lógica que puede ser sobreescrita por el POS en caso de conflicto? | Producto/Dominio | Define mecanismo de concurrencia |
| 3 | ¿Cuántos productos concurrentes pueden modificarse en una sola transferencia entre sucursales? | Producto/Dominio | Define granularidad de locks |
| 4 | ¿Qué proveedores de pago se integrarán inicialmente? ¿Se necesita soporte para múltiples simultáneos? | Producto/Dominio | Complejidad de capa de pagos |
| 5 | ¿El sistema debe soportar offline POS (venta sin conexión) con sincronización posterior? | Producto/Dominio | Arquitectura de conectividad |
| 6 | ¿Cuántos usuarios administrativos concurrentes se esperan? | Producto/Dominio | Dimensionamiento de auth |
| 7 | ¿Los reportes operativos (RF-080) deben ser en tiempo real o con tolerancia a latencia? | Producto/Dominio | Estrategia de réplica/consulta |

---

## Drivers Arquitectónicos

Estos son los factores que más influirán en la selección arquitectónica:

| # | Driver | Fuente | Justificación |
|---|--------|--------|---------------|
| D1 | **Integridad de inventario bajo concurrencia** | RF-064, RNF-005, RNF-020 | El problema central: evitar sobreventa entre canales simultáneos requiere un mecanismo de concurrencia robusto a nivel de datos |
| D2 | **Transaccionalidad financiera** | RF-050, RF-063, RNF-021 | Pagos y ventas requieren atomicidad. Las operaciones de dinero no pueden perderse ni duplicarse |
| D3 | **Idempotencia en operaciones críticas** | RF-042, RN-07, RNF-022 | Reintentos son inevitables; el sistema debe ser tolerante a duplicación |
| D4 | **Escalabilidad por sucursales** | RNF-010, RNF-011 | El sistema crece horizontalmente: más sucursales, más cajas, más volumen |
| D5 | **Portabilidad y contenerización** | RNF-060, RNF-061, RNF-062 | Debe ejecutarse en Linux, ser contenedorizable y separar ambientes |
| D6 | **Seguridad sin excepciones** | RNF-030 a RNF-035 | Autenticación, autorización, cifrado, secretos — todo obligatorio |
| D7 | **Observabilidad operativa** | RNF-050 a RNF-052 | Logs estructurados, métricas y trazabilidad para operar en producción |
| D8 | **Configurabilidad sin redeploy** | RF-100, RN-03 | Parámetros operativos (expiración de reservas, etc.) deben ser modificables sin cambiar código |

---

## Requisitos que Requieren Validación Humana

| # | Requisito | Pregunta pendiente |
|---|-----------|-------------------|
| 1 | RF-062/RF-064 | ¿La reserva de stock web bloquea o solo "apunta"? Definir política de conflictos POS vs Web |
| 2 | RNF-001 | ¿Qué operación exacta se mide como "transacción de negocio"? Necesario para dimensionar |
| 3 | RN-03 | ¿Comportamiento al expirar reserva web? ¿Stock vuelve automáticamente? |
| 4 | RF-063 | ¿Proveedor(es) de pago iniciales? ¿Multi-proveedor desde el inicio? |
| 5 | RNF-041 | ¿RPO/RTO objetivo? Define si se necesita replicación, failover, o ambos |

---

## Clasificación de Requisitos por Naturaleza

### Transacciones de Negocio (dominio)
- RF-050, RF-051, RF-060, RF-061, RF-062, RF-064, RF-070

### Operaciones Técnicas Internas (infraestructura)
- RNF-001, RNF-002, RNF-004, RNF-010, RNF-011, RNF-040, RNF-041, RNF-042, RNF-060, RNF-061, RNF-062

### Operaciones de Soporte (cross-cutting)
- RF-080, RF-090, RF-100, RNF-050, RNF-051, RNF-052

---

## Siguiente Fase

Este análisis será consumido por `solution-leader` para definir el alcance de decisiones arquitectónicas antes de pasar a `architect`.
