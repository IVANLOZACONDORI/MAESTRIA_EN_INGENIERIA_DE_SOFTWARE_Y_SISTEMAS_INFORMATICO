# Estructura obligatoria — 07_landing

```text
proyecto/07_landing/
├── app/
│   ├── Core/
│   │   └── Database.php
│   ├── Repositories/
│   │   ├── LandingCategoriaRepository.php
│   │   └── LandingProductoRepository.php
│   ├── Services/
│   │   └── LandingCatalogoService.php
│   └── Views/
│       ├── layouts/
│       │   ├── header.php
│       │   └── footer.php
│       └── sections/
│           ├── hero.php
│           ├── categorias.php
│           ├── productos.php
│           ├── beneficios.php
│           ├── prueba_social.php
│           ├── llamada_accion.php
│           └── contacto.php
├── config/
│   ├── app.php
│   ├── database.php
│   └── landing.php
├── database/
│   ├── 01_landing_schema.sql
│   └── 02_landing_seed.sql
├── public/
│   ├── index.php
│   ├── .htaccess
│   └── assets/
│       ├── css/
│       │   ├── variables.css
│       │   ├── base.css
│       │   ├── layout.css
│       │   ├── components.css
│       │   ├── landing.css
│       │   └── responsive.css
│       ├── js/
│       │   └── landing.js
│       └── img/
├── docs/
│   ├── brief_landing.md
│   ├── fuentes_imagenes.md
│   ├── decisiones_visuales.md
│   └── validacion_final.md
├── tests/
│   ├── fixtures/catalogo_bolivia.json
│   └── results/
├── tools/
│   ├── validate_structure.php
│   └── validate_checkpoint.php
├── .env.example
└── README.md
```

## Responsabilidades
`public/index.php`: Front Controller/composición únicamente.
`Repositories`: único lugar autorizado para SQL de runtime.
`Service`: orquestación; sin SQL/PDO.
`Views`: presentación; sin SQL/PDO.
`CSS`: siempre en `public/assets/css`.
`JS`: siempre en `public/assets/js`; sin secretos/SQL.

## Regla anti-monolito
FALLA técnica si:
- `public/index.php` supera 140 líneas sin decisión registrada;
- `index.php` contiene SQL o PDO;
- una View contiene SQL/PDO;
- CSS/JS importante está incrustado en `index.php`;
- las secciones se fusionan en un único archivo gigante;
- los 70 productos se codifican inline en `index.php`.

Ante falla:
`TECHNICAL_STATUS: FAILED`
`CAN_CONTINUE: false`
