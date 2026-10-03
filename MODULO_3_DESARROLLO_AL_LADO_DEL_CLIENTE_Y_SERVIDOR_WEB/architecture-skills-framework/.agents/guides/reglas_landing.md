# Reglas obligatorias — Landing

## Propósito
Landing pública de conversión para campañas/anuncios.

## Tecnología permitida
- PHP 8.x puro
- MySQL 8.x / InnoDB
- PDO
- HTML5
- CSS3
- JavaScript ES6+
- jQuery
- Bootstrap/Tailwind

## Prohibido sin decisión explícita
- Laravel/Symfony
- React/Vue/Angular
- Node/Vite/Webpack
- GSAP u otra librería de motion

## Mercado 
- español;
- precios en Bs;
- datos demostrativos plausibles para Bolivia;
- no presentar precios como vigentes reales;
- no insinuar afiliación con marcas reales.

## Catálogo
- 7 categorías exactas;
- 10 productos exactos por categoría;
- 70 productos total.

## Runtime de BD
La landing es READ-ONLY.
En runtime solo `SELECT` parametrizado mediante Repository.
`INSERT/UPDATE/DELETE` solo en scripts de instalación/seed.

## Conversión
- CTA principal above-the-fold;
- CTA secundario;
- CTA final;
- valores configurables;
- no inventar teléfono/WhatsApp real.

## Imágenes
- no IA;
- no Google Images/Pinterest;
- origen/licencia verificable;
- fallback;
- lazy loading salvo recurso crítico del hero.

## Accesibilidad
- semántica;
- navegación teclado;
- focus visible;
- `alt`;
- targets táctiles;
- `prefers-reduced-motion`;
- no depender solo de color.

## Performance
- no cargar 70 imágenes pesadas de golpe;
- reservar espacio/aspect-ratio;
- JS mínimo;
- imágenes optimizadas.

## Post-create obligatorio
No se aprueba al terminar de codificar.
Debe pasar auditoría:
1. visual;
2. motion;
3. accesibilidad/UI;
4. responsive;
5. performance/hardening;
6. estructura automática;
7. aprobación humana.
