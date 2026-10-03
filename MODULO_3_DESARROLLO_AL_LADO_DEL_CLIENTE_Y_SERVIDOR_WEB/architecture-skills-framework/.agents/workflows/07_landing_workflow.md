# Workflow 07 — Landing comercial

Regla: UN checkpoint por interacción. Siempre terminar con `HUMAN_STATUS: PENDING`.

## CP-LAND-01 — Contrato y estructura
Validar guías, decisiones, carpetas y prohibiciones.
Prueba: `php proyecto/07_landing/tools/validate_checkpoint.php CP-LAND-01`
STOP.

## CP-LAND-02 — Esquema
Crear/revisar `landing_categoria` y `landing_producto`.
No tocar tablas `cat_*`.
MySQL/InnoDB + trazabilidad + índices.
STOP.

## CP-LAND-03 — Seed Bolivia
Exactamente 7 categorías × 10 productos.
Precios ficticios en Bs.
Prueba: `php proyecto/07_landing/tools/validate_checkpoint.php CP-LAND-03`
STOP.

## CP-LAND-04 — Imágenes


## CP-LAND-05 — Sistema visual
Skill obligatorio: `design-taste-frontend`.
Definir tokens, jerarquía, spacing y dirección visual.
Guardar `docs/decisiones_visuales.md`.
STOP.

## CP-LAND-06 — Header + Hero + CTA
Skill: `design-taste-frontend`.
CTA visible. Cero login.
STOP.

## CP-LAND-07 — Categorías
Skill: `design-taste-frontend`.
Leer desde Repository/Service. 7 categorías.
STOP.

## CP-LAND-08 — Productos
Skill: `design-taste-frontend`.
70 productos disponibles, pero render/carga eficiente.
Precios en Bs.
STOP.

## CP-LAND-09 — Beneficios + prueba social + CTA final
Skill: `design-taste-frontend`.
No claims falsos. Testimonios ficticios deben identificarse como demostrativos.
STOP.

## CP-LAND-10 — Responsive base
Móvil/tablet/desktop. Sin scroll horizontal. Targets táctiles.
STOP.

## CP-LAND-11 — Motion
Skill obligatorio: `animate` (Emil Kowalski).
Motion con propósito, preferir CSS, reduced-motion obligatorio.
No librerías sin aprobación.
STOP.

## CP-LAND-12 — Auditoría visual post-create
Skill obligatorio: `impeccable`.
Usar pasadas acotadas: layout/typeset/clarify/adapt.
Guardar `.agents/reviews/landing-design-review.md`.
STOP.

## CP-LAND-13 — Auditoría motion
Skills: `review-animations` + `improve-animations` (read-only).
Guardar `.agents/reviews/landing-animation-review.md`.
Resolver BLOCKER/HIGH.
STOP.

## CP-LAND-14 — UI/UX/accesibilidad
Skill: `web-design-guidelines`.
Guardar `.agents/reviews/landing-accessibility-review.md`.
STOP.

## CP-LAND-15 — Performance + hardening
Skill: `impeccable` (optimize/harden).
Guardar `.agents/reviews/landing-performance-review.md`.
STOP.

## CP-LAND-16 — Validación completa
Ejecutar:
- `php proyecto/07_landing/tools/validate_structure.php`
- `php proyecto/07_landing/tools/validate_checkpoint.php CP-LAND-16`
Cualquier error => FAILED + CAN_CONTINUE=false.
STOP.

## CP-LAND-17 — Cierre
Completar `docs/validacion_final.md`.
`TECHNICAL_STATUS: PASSED`
`HUMAN_STATUS: PENDING`
El humano decide la aprobación final.
