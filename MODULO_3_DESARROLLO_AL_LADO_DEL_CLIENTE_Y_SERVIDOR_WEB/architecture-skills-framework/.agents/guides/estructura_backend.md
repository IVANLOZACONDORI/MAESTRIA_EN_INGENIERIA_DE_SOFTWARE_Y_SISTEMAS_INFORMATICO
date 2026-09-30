# Estructura obligatoria del Backend

## Arquitectura

PHP puro, MVC, sin framework.

La implementación debe respetar:

```text
proyecto/06_codigo/
├── public/
│   └── index.php
│
├── app/
│   ├── Core/
│   │   ├── Router.php
│   │   ├── Request.php
│   │   ├── Response.php
│   │   ├── Database.php
│   │   └── Session.php
│   │
│   ├── Controllers/
│   │   ├── Api/
│   │   │   ├── AuthController.php
│   │   │   ├── CategoriaController.php
│   │   │   └── ProductoController.php
│   │   └── Web/
│   │       ├── AuthController.php
│   │       ├── DashboardController.php
│   │       ├── CategoriaController.php
│   │       └── ProductoController.php
│   │
│   ├── Models/
│   │   ├── Usuario.php
│   │   ├── Categoria.php
│   │   └── Producto.php
│   │
│   ├── Repositories/
│   │   ├── UsuarioRepository.php
│   │   ├── CategoriaRepository.php
│   │   └── ProductoRepository.php
│   │
│   ├── Services/
│   │   ├── AuthService.php
│   │   ├── CategoriaService.php
│   │   └── ProductoService.php
│   │
│   ├── Validators/
│   │   ├── AuthValidator.php
│   │   ├── CategoriaValidator.php
│   │   └── ProductoValidator.php
│   │
│   └── Middleware/
│       ├── AuthMiddleware.php
│       └── CsrfMiddleware.php
│
├── routes/
│   ├── api.php
│   └── web.php
│
└── config/
    ├── app.php
    ├── database.php
    └── session.php
```

## Responsabilidades

### `public/index.php`

Debe ser solamente el Front Controller.

Puede:

- cargar bootstrap;
- cargar configuración;
- registrar rutas;
- ejecutar el Router.

No puede contener:

- SQL;
- consultas PDO;
- CRUD;
- lógica de autenticación;
- validaciones de negocio;
- lógica específica de categorías o productos.

### `Router.php`

Responsable de:

- método HTTP;
- URI;
- parámetros;
- resolución de controlador.

### Controllers

Responsables de:

- recibir request;
- invocar validator/service;
- devolver response.

No contienen SQL.

### Validators

Responsables de validar entrada.

### Services

Responsables de reglas de negocio y casos de uso.

### Repositories

Responsables de:

- consultas SQL;
- PDO;
- persistencia.

### Database

Responsable de crear/proporcionar la conexión PDO.

## Regla obligatoria

```text
Controller != SQL
Service != PDO
View != Database
Repository = SQL/PDO
```

## Validación arquitectónica

Antes de aprobar un checkpoint del backend, comprobar:

- `public/index.php` sigue siendo pequeño;
- no contiene SQL;
- Controllers no contienen PDO;
- Services no hacen consultas SQL;
- Repositories concentran la persistencia;
- rutas están separadas;
- el código del checkpoint existe físicamente;
- las pruebas pasan.

Si se incumple:

```text
TECHNICAL_STATUS: FAILED
CAN_CONTINUE: false
```