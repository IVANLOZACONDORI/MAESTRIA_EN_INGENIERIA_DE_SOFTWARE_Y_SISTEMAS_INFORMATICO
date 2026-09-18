# Revisión adversarial de arquitectura

## Metodología
Cruce sistemático de RF/RNF contra los 6 análisis previos (01-06). Clasificación de hallazgos por severidad según CA-10.

---

## 1. Hallazgos bloqueantes (CA-10: resolver antes de consolidación)

### B-01. Devoluciones (RF-070) sin estrategia arquitectónica
**Hallazgo:** RF-070 requiere registrar devoluciones parciales o totales. Ningún análisis (01-06) define la estrategia de devoluciones: qué transacciones inversas, cómo afectan inventario, cómo afectan pagos, o cómo se sincronizan entre canales.
**Riesgo:** RF-070 es funcionalidad core. Sin definición, RNF-005 (integridad entre canales) y RNF-020 (evitar sobreventa) quedan parcialmente cubiertos.
**Acción:** Agregar B-07 a 02_decision_scope.md y crear análisis dedicado antes de la consolidación.

### B-02. Resolución de preguntas #1 y #3 bloquea D-01
**Hallazgo:** 02_decision_scope.md lista 5 preguntas abiertas. Las preguntas #1 (¿reserva web bloquea stock POS?) y #3 (¿comportamiento al expirar reserva?) son prerequisitos para D-01 (política de inventario). Sin respuesta, el diseño de tablas en 04_database_analysis.md y la estrategia en 03_architecture_options.md no pueden cerrarse.
**Riesgo:** D-01 alimenta D-02 (transacciones) y RNF-020 (sobreventa). Dependencia circular no resuelta.
**Acción:** Responder con el usuario antes de consolidación final.

### B-03. RNF-042 (timeout/reintentos de dependencias externas) sin análisis
**Hallazgo:** RNF-042 requiere manejo de timeout y reintentos controlados para dependencias externas. Ningún análisis define patrón (circuit breaker, retry con backoff, dead-letter). RF-063 (pagos desacoplados) y RF-042 (idempotencia) dependen de esto.
**Riesgo:** Sin patrón de resiliencia, integración con pasarelas de pago falla sin control.
**Acción:** Definir en 05_security_analysis.md o en análisis nuevo antes de consolidación.

---

## 2. Hallazgos altos (resolver antes de decisión final)

### A-01. Contradicción Redis: 06_devops_analysis.md vs 04_database_analysis.md
**Hallazgo:** 06_devops_analysis.md asume "posible Redis para cola de pagos" en arquitectura. 04_database_analysis.md recomienda explícitamente "PostgreSQL solo" y descarta Redis inicialmente.
**Riesgo:** Configuración de infraestructura inconsistente entre análisis.
**Acción:** Aliniar: confirmar si Redis se incluye o no en fase 1. Si solo para pagos async, justificar.

### A-02. Contradicción alternativa B: 03 vs 04
**Hallazgo:** 03_architecture_options.md recomienda "Alternativa B (Modular Monolith)" como recomendación preliminar. 04_database_analysis.md recomienda PostgreSQL sin anclar a una alternativa arquitectónica específica.
**Riesgo:** El análisis de base de datos no valida la recomendación arquitectónica.
**Acción:** 04 debe confirmar o refutar la viabilidad de B concreta (schema compartido vs por módulo, etc.).

### A-03. RF-080 (reportes operativos) sin estrategia
**Hallazgo:** RF-080 requiere reportes de ventas, compras e inventario. Ningún análisis define: ¿reportes en vivo?, ¿OLAP separado?, ¿vistas materializadas?, ¿exportación?
**Riesgo:** Si los reportes son pesados sobre la misma DB operativa, RNF-001 (25K tx/hora) puede degradarse.
**Acción:** Definir estrategia de lectura (read replica, vistas materializadas, o reporting separado) antes de consolidación.

### A-04. RF-100 (parámetros sin redeploy) sin estrategia
**Hallazgo:** RF-100 requiere administrar parámetros operativos sin modificar código. Ningún análisis define: ¿tabla de configuración?, ¿feature flags?, ¿archivo de configuración externo?
**Riesgo:** Sin estrategia, cada cambio operativo requiere reinstalación (violación directa de RF-100).
**Acción:** Definir en análisis de configuración o agregar a D-06 (infraestructura).

### A-05. RNF-041 (RPO/RTO) con valores no justificados
**Hallazgo:** 04_database_analysis.md asume "RPO < 1 hora, RTO < 4 horas" sin fuente. restricciones.md dice explícitamente "RPO/RTO y presupuesto aún deben validarse".
**Riesgo:** Estrategia de backup basada en estimaciones no validadas por el cliente.
**Acción:** Responder pregunta #5 de 02_decision_scope.md con el usuario. No cerrar D-09 sin validación.

---

## 3. Hallazgos medios (mejorar antes de entrega final)

### M-01. RF-002 (clientes) sin análisis de modelo
**Hallazgo:** RF-002 requiere administrar clientes. Ningún análisis define el modelo de cliente: ¿por sucursal?, ¿compartido entre canales?, ¿datos de contacto vs historial?
**Riesgo:** Modelo de cliente afecta RF-060 (catálogo web), RF-050 (ventas presenciales), RF-070 (devoluciones).

### M-02. RF-020 (productos, categorías, precios, promociones) sin análisis
**Hallazgo:** RF-020 es funcionalidad core. Ningún análisis define: ¿precios por sucursal o globales?, ¿promociones aplicables por canal?, ¿categorías con jerarquía?
**Riesgo:** Afecta directamente D-01 (inventario) y D-02 (transacciones).

### M-03. RF-061 (entrega o retiro en sucursal) sin análisis
**Hallazgo:** RF-061 define dos modos de fulfillment. Ningún análisis define: ¿inventario reservado por canal?, ¿estados de pedido web con fulfillment?, ¿asignación a sucursal?
**Riesgo:** Afecta RF-062 (reserva stock) y RF-064 (reflejar ventas web sin sobreventa).

### M-04. RF-010 (sucursales/POS) sin análisis de conectividad
**Hallazgo:** RF-010 requiere administrar sucursales y cajas/POS. 05_security_analysis.md menciona POS offline como restricción, pero 01_requirements_analysis.md no lo lista como supuesto. 04_database_analysis.md no define patrón de sincronización offline/online.
**Riesgo:** Sin patrón de conectividad, RF-050 (ventas presenciales) y RF-041 (inventario) pueden fallar cuando el POS esté offline.

### M-05. Inconsistencia "POS offline": 05 vs 01
**Hallazgo:** 05_security_analysis.md asume POS puede funcionar offline. 01_requirements_analysis.md pregunta #6 lista "POS funciona offline" como pregunta abierta, no como supuesto confirmado.
**Riesgo:** Toda la capa de seguridad (JWT vs API keys para POS) depende de esta respuesta.

---

## 4. Hallazgos bajos (observar, no bloquean)

### L-01. RNF-002 (configurabilidad externa) mencionada pero no analizada
**Hallazgo:** RNF-002 dice "capacidad configurable externamente". Se menciona en 01 pero no se define: ¿configuración por archivo?, ¿tabla?, ¿variable de entorno?
**Impacto:** Bajo. Puede resolverse en implementación.

### L-02. Escenario base no documentado
**Hallazgo:** 04_database_analysis.md usa "~100K productos, ~50 sucursales, ~100 usuarios concurrentes" como escenario base. Fuente no documentada.
**Impacto:** Bajo si las estimaciones son razonables, pero falta trazabilidad a requisitos del cliente.

### L-03. CA-02 (comparar al menos 2 alternativas) parcialmente satisfecho
**Hallazgo:** 03_comparation.md compara 6 alternativas. CA-02 se cumple. Pero 04, 05, 06 comparan opciones internas sin sempre anclar a las 6 alternativas de 03.
**Impacto:** Bajo. La comparación principal existe.

---

## 5. Cobertura RF/RNF vs análisis

| Requisito | Análisis | Estado |
|-----------|----------|--------|
| RF-001 (usuarios, roles, permisos) | 05_security | Cubierto |
| RF-002 (clientes) | — | **SIN ANÁLISIS** |
| RF-010 (sucursales, POS) | 01, 05 | Parcial (conectividad pendiente) |
| RF-020 (productos, categorías, precios) | — | **SIN ANÁLISIS** |
| RF-030 (proveedores, órdenes compra) | 04 (mencionado) | Parcial |
| RF-031 (recepción mercadería) | 01 (mencionado) | Parcial |
| RF-040 (inventario) | 03, 04 | Cubierto |
| RF-041 (movimientos inventario) | 04 | Cubierto |
| RF-042 (idempotencia) | 05 (RN-07) | Parcial |
| RF-050 (ventas presenciales) | 03, 06 | Cubierto |
| RF-051 (descontar inventario) | 04 (D-01) | Cubierto |
| RF-060 (catálogo, carrito, pedido web) | 03 (mencionado) | Parcial |
| RF-061 (entrega/retiro sucursal) | — | **SIN ANÁLISIS** |
| RF-062 (reserva stock web) | 04 (D-01) | Cubierto |
| RF-063 (pagos desacoplados) | 05 (D-04) | Cubierto |
| RF-064 (ventas web sin sobreventa) | 04 (D-01) | Cubierto |
| RF-070 (devoluciones) | — | **SIN ANÁLISIS** |
| RF-080 (reportes) | — | **SIN ANÁLISIS** |
| RF-090 (auditar) | 05 (RNF-035) | Cubierto |
| RF-100 (parámetros sin redeploy) | — | **SIN ANÁLISIS** |
| RNF-001 (25K tx/hora) | 03, 04, 06 | Cubierto |
| RNF-002 (configurable externa) | — | Mencionado, sin profundidad |
| RNF-003 (txn ≠ SQL) | 04 (D-02) | Cubierto |
| RNF-004 (picos) | 04 | Cubierto |
| RNF-005 (integridad concurrencia) | 04 (D-01) | Cubierto |
| RNF-010 (crecimiento sucursales) | 03 | Cubierto |
| RNF-011 (10x) | 03, 04, 06 | Cubierto |
| RNF-020 (sobreventa) | 04 (D-01) | Cubierto |
| RNF-021 (fronteras transaccionales) | 04 (D-02) | Cubierto |
| RNF-022 (duplicados) | 04 (D-03) | Cubierto |
| RNF-030-035 (seguridad) | 05 | Cubierto |
| RNF-040 (backup) | 04 (Dim.3) | Cubierto |
| RNF-041 (RPO/RTO) | 04 (estimado) | Parcial (sin validar) |
| RNF-042 (timeout/reintentos) | — | **SIN ANÁLISIS** |
| RNF-050-052 (observabilidad) | 06 | Cubierto |
| RNF-060-062 (Linux, contenedor) | 06 | Cubierto |

---

## 6. Resumen ejecutivo

| Severidad | Cantidad | Acción requerida |
|-----------|----------|------------------|
| Bloqueantes | 3 | Resolver antes de consolidación |
| Altos | 5 | Resolver antes de decisión final |
| Medios | 5 | Mejorar antes de entrega final |
| Bajos | 3 | Observar, no bloquean |

**CA-10 (sin hallazgos bloqueantes):** NO CUMPLIDO. Tres hallazgos bloqueantes (B-01, B-02, B-03) impiden avanzar a consolidación.

**Requisitos sin cobertura completa:** RF-002, RF-020, RF-061, RF-070, RF-080, RF-100, RNF-042.

**Acción inmediata:** Resolver preguntas #1, #3, #5 de 02_decision_scope.md; agregar análisis de devoluciones (B-01) y resiliencia externa (B-03).
