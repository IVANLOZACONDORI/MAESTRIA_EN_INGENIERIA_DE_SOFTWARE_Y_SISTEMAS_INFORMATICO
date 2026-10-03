# Landing design review — CP-LAND-12

Fecha: 2026-09-30
Skill: `impeccable` (registro: brand). Referencias leídas: init, brand, layout, typeset, clarify, adapt.
Pasadas acotadas: layout / typeset / clarify / adapt. Base: PRODUCT.md (creado en este checkpoint), `docs/decisiones_visuales.md`, `docs/brief_landing.md`.
Método: render Chrome headless (1400 desktop, 390 mobile, full-page) + DOM dump + revisión de tokens CSS. Sin cambios de código (auditoría, según workflow CP-LAND-12).

## Layout

| ID | Hallazgo | Sev | Dueño |
|----|----------|-----|-------|
| L1 | Grid de categorías (`auto-fill minmax(240px)`) = 5 columnas con 7 items → fila 2 con solo 2 tarjetas y 3 celdas vacías (hueco ~60% de la fila). | MEDIUM | CP-LAND-15 |
| L2 | Grid de productos = 6 columnas con 10 items por categoría → cola 6+4, 2 celdas vacías (menor que L1). 5 columnas daría 10 = 5×2 exacto. | LOW | CP-LAND-15 |
| L3 | Squint test OK: H1 domina → H2 de sección → precios verdes como pop de color. Ritmo hero (aire) → catálogo (densidad) consistente con registro editorial. | OK | — |
| L4 | Anti-slop verificado: sin hero centrado, sin 3 tarjetas idénticas en hero, sin hero-métrica decorativa (los stats 7/70/Bs/0 son datos reales de la demo), sin tarjetas anidadas. | OK | — |

## Typeset

| ID | Hallazgo | Sev | Dueño |
|----|----------|-----|-------|
| T1 | Outfit (familia única, pesos 400/600/700), tokens `rem`, `clamp()` fluido en h1/h2 con ratio ≈1.4, letter-spacing negativo en display, body 1rem (16px). Cumple todo el checklist del pass. | OK | — |
| T2 | `--text-small: 0.875rem` solo en captions/creditos/autor (no body). | OK | — |
| T3 | Jerarquía de precios (`--text-price` 1.125rem, verde, bold) coherente en 70 cards. | OK | — |

## Clarify

| ID | Hallazgo | Sev | Dueño |
|----|----------|-----|-------|
| C1 | "demostrativo/a" ×4 en intros adyacentes: sub-hero, sub-categorías, sub-productos, disclaimer+footer. Redundancia; conservar disclaimer (hero) + footer, recortar los subs de sección. | MEDIUM | CP-LAND-15 |
| C2 | H1 "Productos de calidad a precios en Bs" → "con precios en Bs" suena más natural en es. | LOW | CP-LAND-15 |
| C3 | Stat "0 / cuenta o registro requerido": con el número 0 el label debería ser plural ("cuentas o registros requeridos") o label "sin cuenta ni registro". | LOW | CP-LAND-15 |
| C4 | CTAs específicos ("Ver ofertas" / "Consultar disponibilidad"), sin "Click here/OK", sin jerga, terminología consistente (ofertas/disponibilidad), tono claro cercano-comercial. Testimonios marcados ficticios; "Contacto disponible próximamente" honesto (WhatsApp=null). | OK | — |

## Adapt

| ID | Hallazgo | Sev | Dueño |
|----|----------|-----|-------|
| A1 | Regresión CP-10: overflow=0 y smallTargets=0 en 360/768/1280; hover gated en `(hover: hover) and (pointer: fine)` (CP-11); `prefers-reduced-motion` global + override. | OK | — |
| A2 | `<meta viewport>` sin `viewport-fit=cover` ni `env(safe-area-inset-bottom)` en footer → pie recortado en celulares con notch. | LOW | CP-LAND-15 |
| A3 | 78 imgs, 0 `srcset/sizes` (solo `src` + lazy). Riesgo de peso en móvil. | INFO | CP-LAND-15 (optimize) |
| A4 | Mobile-first correcto: hero apila copy→foto, CTAs full-width, grids colapsan a 1–2 col, sin dependencia de hover para ninguna función. | OK | — |

## Veredicto

- BLOCKER: 0 · HIGH: 0 · MEDIUM: 2 (L1, C1) · LOW: 5 · INFO: 1
- Nada bloquea los siguientes checkpoints. L1/C1/A2 programados en CP-LAND-15 (impeccable optimize/harden); A3 nace ahí.
- CP-LAND-12: PASSED (auditoría completada, entregable `.agents/reviews/landing-design-review.md`).
