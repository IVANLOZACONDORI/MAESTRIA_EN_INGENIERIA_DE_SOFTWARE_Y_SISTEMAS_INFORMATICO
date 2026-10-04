# Supuestos y restricciones

## Sistema

**SIGICI-RI – Sistema Inteligente de Gestión de Incidentes de Ciberseguridad y Respuesta Institucional**

## Restricciones del caso de estudio

- **RES-01.** En esta etapa no se seleccionará definitivamente un lenguaje de programación, framework, arquitectura, motor de base de datos, proveedor de nube ni plataforma de despliegue.

- **RES-02.** Los requerimientos deberán expresar necesidades del sistema y no imponer decisiones tecnológicas prematuras.

- **RES-03.** El sistema deberá considerar el tratamiento de información potencialmente sensible relacionada con incidentes de ciberseguridad.

- **RES-04.** El acceso a información, evidencias, funciones administrativas y registros de auditoría deberá limitarse a usuarios autorizados según su rol.

- **RES-05.** Las acciones relevantes realizadas sobre los incidentes deberán mantener trazabilidad suficiente para identificar usuario, acción y momento de ejecución.

- **RES-06.** No deberán almacenarse contraseñas, tokens, claves privadas, secretos ni credenciales sensibles en archivos versionados del proyecto.

- **RES-07.** No deberán realizarse operaciones destructivas sobre datos o sistemas reales sin autorización expresa.

- **RES-08.** La validación mediante agentes tendrá carácter de apoyo. Las decisiones finales sobre los requisitos corresponderán al estudiante.

- **RES-09.** Cualquier tecnología propuesta en fases posteriores deberá justificarse utilizando los RF, RNF, restricciones y decisiones arquitectónicas aprobadas.

- **RES-10.** Los valores definitivos relacionados con disponibilidad, recuperación, capacidad, concurrencia, crecimiento y tiempos de respuesta deberán ser validados antes de la implementación.

## Supuestos iniciales

- **SUP-01.** Los usuarios del sistema pertenecerán a una institución y dispondrán de distintos niveles de autorización.

- **SUP-02.** Un incidente podrá contener información descriptiva, evidencias, responsables, estados, comentarios y registros de seguimiento.

- **SUP-03.** La institución definirá posteriormente su catálogo definitivo de niveles de severidad y políticas de escalamiento.

- **SUP-04.** La institución deberá definir los periodos de conservación de incidentes, evidencias y registros de auditoría.

- **SUP-05.** Los requisitos iniciales podrán ser modificados después de analizar y justificar las observaciones emitidas por el agente especializado.

## Restricciones de la actividad académica

- No implementar el sistema en esta fase.
- No desarrollar código.
- No seleccionar una base de datos por preferencia.
- No seleccionar microservicios por defecto.
- No elegir infraestructura definitiva.
- No introducir credenciales en el repositorio.
- No aceptar automáticamente las observaciones del agente.
