# Landing animation review — CP-LAND-13

Fecha: 2026-09-30
Skills: `review-animations` + `improve-animations` (ambos read-only). Referencia: `STANDARDS.md` (valores exactos citados).
Método: recon completo de la superficie motion (barrido `transition`/`animation`/`@keyframes`/`transform`/`prefers-reduced-motion`/`hover:hover` en CSS/JS + vistas PHP), verificación manual de cada hallazgo en su `file:line`. **Sin cambios de código** (auditoría read-only, workflow CP-LAND-13).

## Superficie motion (recon)

- **Stack**: CSS nativo + 18 líneas de JS (`landing.js`). Cero librerías de motion, **cero `@keyframes`**, cero `animation:` (solo los overrides de reduced-motion).
- **Tokens**: `--dur: 0.3s` (`variables.css:37`), `--ease: cubic-bezier(0.16, 1, 0.3, 1)` (`variables.css:36`) — curva fuerte ease-out, conforme a STANDARDS (no usa easings built-in débiles).
- **Movimientos existentes** (todos state indication/feedback, justificados — estándar 1 ✓):
  - Sombra del header al hacer scroll (`landing.css:11` + `landing.js:9`, rAF-throttled y `{passive: true}` ✓).
  - Press del botón `scale(0.98)` (`components.css:20`).
  - Lift de tarjetas `translateY(-4px)` + sombra (`landing.css:154-161`, `230-236`).
  - `scroll-behavior: smooth` en anclas (`base.css:6`).
- **A11y**: reduced-motion global (`base.css:51-58`) + override de movimiento (`landing.css:412-415`); hover gating `(hover: hover) and (pointer: fine)` (`landing.css:156`, `232`).
- **Frecuencia**: hover de tarjetas de catálogo = decenas por sesión → tabla STANDARDS: "tens of times/day → remove or drastically reduce".
- **Personalidad**: registro editorial sobrio (decisiones_visuales) → motion mínimo = cohesión correcta.

## Tabla de hallazgos

| # | Antes | Después | Por qué | Sev | Ubicación | Dueño |
|---|-------|---------|---------|-----|-----------|-------|
| M1 | `transition: … var(--dur)` = 300ms en hover de tarjetas | `transition-duration: 180ms` en `.cat-card__link` y `.prod-card` (mantener `--dur` global para el resto) | "UI animations stay under 300ms" (STANDARDS/Duration): 0.3s está en el límite, no bajo él; hover de catálogo es acción de decenas/día → reducir drásticamente | MEDIUM | `variables.css:37` + `landing.css:154`, `230` | CP-LAND-15 |
| M2 | `transition: transform var(--dur)` (300ms) + `scale(0.98)` en `:active` | `transition: transform 160ms var(--ease), background-color 200ms var(--ease)` y `transform: scale(0.97)` | Button press feedback: 100–160ms, `scale(0.97)`, ease-out (STANDARDS/Physicality). 300ms simétrico en press hace el botón feel sluggish; el press es la fase del sistema → snap | MEDIUM | `components.css:18-20` | CP-LAND-15 |
| M3 | `transition-duration: 0.01ms !important` global (`base.css:56`) bajo reduce = cero | Conservar zeroing de `animation-duration` y `scroll-behavior`; quitar el blanket de `transition-duration` y matar solo el movimiento con reglas dirigidas (ya existentes `landing.css:412-415` + añadir `.btn:active { transform: none; }` en ese bloque) | "Reduced motion means fewer and gentler animations, not zero — keep transitions that aid comprehension, remove movement" (STANDARDS/Accessibility). Además el comentario `landing.css:411` promete la sombra "conservada como feedback" pero a 0.01ms es instantánea | MEDIUM | `base.css:51-58`, `landing.css:410-415` | CP-LAND-15 (solape CP-LAND-14) |
| L1 | `transition: box-shadow` en header sticky con `backdrop-filter: blur(8px)` (`landing.css:9,11`) | Quitar `box-shadow` de la transición del header (sombra instantánea al pasar `scrollY>8`) o mover la sombra a un pseudo-elemento con `opacity` | `box-shadow` no es GPU-composited (paint); corre sobre elemento sticky con backdrop-filter mientras la página hace scroll. Riesgo bajo (toggle único por umbral, no por frame) ⇒ LOW, no bloquea | LOW | `landing.css:9-13` | CP-LAND-15 |

## Chequeos OK verificados

- Sin `transition: all`, sin `ease-in`, sin `scale(0)`, sin `transform-origin` centralizado en triggers (no hay popovers/dropdowns/tooltips), sin keyframes en triggers rápidos (no hay keyframes).
- Movimiento de tarjetas gateado en `(hover: hover) and (pointer: fine)` (`landing.css:156`, `232`) ✓ (estándar 8).
- Interruptibilidad ✓: todo con CSS transitions (retargetables), cero keyframes que reinicien desde cero.
- Smooth-scroll desactivado bajo reduce (`base.css:52`) ✓.
- Scroll handler rAF-throttled + `passive: true` (`landing.js:12-17`) ✓ — no recalc storm.
- `scale(0.98)` dentro del rango aceptable 0.95–0.98 (estándar 5) ✓.
- Easing: curva fuerte custom `cubic-bezier(0.16,1,0.3,1)` ✓ (estándar 3); el lift es entrada/salida con carácter ease-out ✓.
- Sin animación en acciones de teclado/100+por-día ✓ (estándar 2).

## Oportunidades perdidas (aditivas, no correctivas)

1. **Cero motion de entrada**: ni hero ni grids tienen fade/stagger (el move canónico sería `opacity 0→1` + `translateY(8px)`, 300ms ease-out, stagger 30–80ms). `decisiones_visuales` fija "Regla del Plano por Defecto" y registro editorial sobrio ⇒ la quietud es probablemente **deliberada** (no se re-litiga); se presenta como decisión del humano, no como defecto.
2. **Press en tarjetas**: `.cat-card__link`/`.prod-card` no tienen estado `:active`; en touch (hover gateado) la tarjeta no da feedback visual al tocar. Fix candidato: `transform: scale(0.99)` 120ms en `:active` (el press no requiere gating de hover).

## Veredicto

1. **Feel-breaking regressions** — ninguno: sin ease-in, sin scale(0), sin transition:all, sin motion injustificado.
2. **Missed simplifications** — M1 (duración en el límite); quietud de entrada (deliberada, ver oportunidades perdidas).
3. **Performance** — L1 (box-shadow no-GPU en header sticky); resto en `transform` ✓.
4. **Interruptibility & timing** — M2 (press simétrico 300ms debería ser asimétrico/snappy).
5. **Origin, physicality & cohesion** — ✓; personalidad editorial = motion mínimo coherente.
6. **Accessibility** — M3 (reduced-motion a cero en vez de gentler); gating y smooth-scroll ✓.

**Decisión: APPROVE.** BLOCKER=0, HIGH=0 → no hay BLOCKER/HIGH pendientes de resolver (regla "Resolver BLOCKER/HIGH" cumplida: 0). MEDIUM/LOW con dueño CP-LAND-15.

**Incertidumbre de feel**: el "feel" no se juzga solo desde código. Antes de dar por bueno visualmente: DevTools Animations panel / slow-motion 2–5× sobre hover de tarjeta y press de botón, y dispositivo real para el header sticky + `backdrop-filter` durante scroll.
