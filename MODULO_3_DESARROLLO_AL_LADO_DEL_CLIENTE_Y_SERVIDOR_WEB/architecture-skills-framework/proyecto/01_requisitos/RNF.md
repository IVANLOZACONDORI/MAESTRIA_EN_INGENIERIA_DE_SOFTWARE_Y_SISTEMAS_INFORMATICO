# Requisitos no funcionales

## Capacidad y rendimiento
- RNF-001. Soportar inicialmente 25.000 transacciones de negocio por hora.
- RNF-002. La capacidad objetivo debe ser configurable externamente.
- RNF-003. Una transacción de negocio no equivale a una sentencia SQL.
- RNF-004. Evaluar picos con el multiplicador configurado.
- RNF-005. Mantener integridad bajo concurrencia entre canal físico y web.

## Escalabilidad
- RNF-010. Permitir crecimiento de sucursales/cajas sin rediseñar todo el dominio.
- RNF-011. Evaluar un escenario de 10x la carga inicial.

## Integridad
- RNF-020. Evitar sobreventa por condiciones de carrera.
- RNF-021. Definir fronteras transaccionales para dinero e inventario.
- RNF-022. Evitar duplicados ante reintentos.

## Seguridad
- RNF-030. Autenticación para operaciones administrativas.
- RNF-031. Autorización por mínimo privilegio.
- RNF-032. Cifrado en tránsito.
- RNF-033. Contraseñas nunca en texto plano.
- RNF-034. Secretos fuera del repositorio.
- RNF-035. Controles de seguridad para APIs.

## Disponibilidad y recuperación
- RNF-040. Definir estrategia de backup/restauración.
- RNF-041. RPO/RTO deben definirse antes de producción.
- RNF-042. Dependencias externas deben manejar timeout y reintentos controlados.

## Observabilidad
- RNF-050. Logs estructurados.
- RNF-051. Métricas de latencia, errores, disponibilidad y volumen.
- RNF-052. Correlación de operaciones distribuidas cuando corresponda.

## Portabilidad
- RNF-060. Ejecutarse en Linux.
- RNF-061. Poder contenerizarse.
- RNF-062. Separar configuraciones de desarrollo, pruebas y producción.
