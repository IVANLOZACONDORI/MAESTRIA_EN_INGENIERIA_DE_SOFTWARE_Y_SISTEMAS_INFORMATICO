# Estructura obligatoria del Frontend

## Arquitectura

El frontend se implementará con:

- PHP Views;
- HTML5 semántico;
- CSS3 puro;
- JavaScript ES6+;
- Fetch API;
- sin React;
- sin Vue;
- sin Angular;
- sin framework CSS;
- sin proceso de compilación.

## Estructura

```text
proyecto/06_codigo/
├── app/
│   ├── Controllers/
│   │   └── Web/
│   │       ├── AuthController.php
│   │       ├── DashboardController.php
│   │       ├── CategoriaController.php
│   │       └── ProductoController.php
│   │
│   └── Views/
│       ├── layouts/
│       │   ├── main.php
│       │   ├── auth.php
│       │   ├── header.php
│       │   ├── sidebar.php
│       │   └── footer.php
│       │
│       ├── auth/
│       │   └── login.php
│       │
│       ├── dashboard/
│       │   └── index.php
│       │
│       ├── categorias/
│       │   ├── index.php
│       │   └── form.php
│       │
│       └── productos/
│           ├── index.php
│           └── form.php
│
├── routes/
│   └── web.php
│
└── public/
    └── assets/
        ├── css/
        │   ├── variables.css
        │   ├── layout.css
        │   ├── components.css
        │   ├── forms.css
        │   └── responsive.css
        │
        └── js/
            ├── api/
            │   ├── auth-api.js
            │   ├── categorias-api.js
            │   └── productos-api.js
            │
            ├── modules/
            │   ├── login.js
            │   ├── dashboard.js
            │   ├── categorias.js
            │   └── productos.js
            │
            └── app.js
```

## Responsabilidades

### Web Controllers

Responsables de:

- resolver la vista;
- preparar datos mínimos para renderizado;
- verificar acceso a la página cuando corresponda.

No deben contener:

- SQL;
- consultas PDO;
- reglas de negocio complejas.

### Views

Responsables únicamente de presentación.

Pueden contener:

- HTML;
- estructuras PHP simples de presentación;
- escape de salida;
- inclusión de layouts/components.

No deben contener:

- SQL;
- PDO;
- consultas a MySQL;
- reglas de negocio;
- autenticación implementada directamente;
- lógica CRUD de backend.

### Layouts

Evitan duplicar:

- header;
- sidebar;
- footer;
- estructura HTML general;
- carga de CSS/JS.

### `public/assets/css/`

Responsable únicamente de estilos.

No concentrar todo el CSS dentro de las Views.

### `public/assets/js/api/`

Responsable exclusivamente de comunicación HTTP con la API.

Ejemplo:

```text
categorias-api.js
    ↓
fetch('/api/v1/categorias')
```

No debe manipular directamente la interfaz más allá de devolver resultados al módulo que lo llama.

### `public/assets/js/modules/`

Responsable de comportamiento de cada pantalla.

Ejemplo:

```text
categorias.js
    ↓
categorias-api.js
    ↓
REST API
```

Puede:

- escuchar eventos;
- recoger datos de formularios;
- mostrar loading;
- mostrar errores;
- actualizar DOM.

No puede:

- ejecutar SQL;
- contener credenciales;
- decidir autorización;
- implementar reglas críticas de negocio.

## Separación obligatoria

```text
PHP View
→ presentación

Web Controller
→ selección de vista

JavaScript Module
→ interacción de pantalla

API JS
→ comunicación HTTP

Backend
→ reglas de negocio

Repository
→ base de datos
```

## Pantallas permitidas

Solo:

### Login

```text
/auth/login.php
```

Debe incluir:

- email;
- contraseña;
- mensaje de error;
- envío hacia API de login.

No incluir:

- registro;
- recuperación de contraseña;
- administración de usuarios.

### Dashboard

```text
/dashboard/index.php
```

Solo debe mostrar:

- usuario actual;
- cantidad de categorías;
- cantidad de productos;
- enlaces a categorías y productos.

No convertirlo en un módulo de reportes.

### Categorías

```text
/categorias/index.php
/categorias/form.php
```

Funciones:

- listar;
- crear;
- editar;
- inactivar.

### Productos

```text
/productos/index.php
/productos/form.php
```

Funciones:

- listar;
- crear;
- editar;
- inactivar;
- seleccionar categoría.

## Estados obligatorios de interfaz

Cuando una pantalla consuma la API debe contemplar:

```text
loading
empty
success
error
unauthorized
```

## Seguridad

Los valores impresos desde PHP deben escaparse, por ejemplo:

```php
htmlspecialchars($value, ENT_QUOTES, 'UTF-8');
```

La validación JavaScript es solamente de ayuda.

La validación real permanece en backend.

No almacenar:

- contraseñas;
- secretos;
- tokens sensibles;

en JavaScript.

## Regla de archivos

Evitar archivos gigantes.

No crear, por ejemplo:

```text
productos.php
```

con:

- HTML;
- CSS;
- JavaScript;
- fetch;
- validación;
- backend;

todo mezclado.

Cada responsabilidad debe permanecer en su carpeta.

## Validación arquitectónica

Antes de aprobar un checkpoint frontend comprobar:

- la View correspondiente existe;
- el Web Controller correspondiente existe si es necesario;
- CSS está fuera de la View;
- JavaScript está fuera de la View;
- llamadas API están en `js/api/`;
- comportamiento de pantalla está en `js/modules/`;
- no existe SQL en Views;
- no existe PDO en Views;
- no hay credenciales en JS;
- la pantalla funciona;
- el endpoint consumido responde correctamente.

Si se incumple la estructura:

```text
TECHNICAL_STATUS: FAILED
CAN_CONTINUE: false
```

No avanzar hasta corregirla.