# Patrón reutilizable de categorías (CP-BACK-01)

Patrón canónico a aplicar para `auth_usuario` y `cat_producto`.
Estructura obligatoria: `.agents/guides/estructura_backend.md`.

## Flujo

```text
Request → Router → Controller → Validator → Service → Repository → PDO → MySQL
```

## Capas y reglas

| Capa | Archivo | Regla |
|---|---|---|
| Core | `app/Core/{Router,Request,Response,Database,Session}.php` | infraestructura; sin SQL de negocio |
| Controller | `app/Controllers/Api/*.php` | recibe request, llama validator/service, responde; **sin PDO ni SQL** |
| Validator | `app/Validators/*.php` | valida entrada; lanza `RuntimeException(code 400/404)` |
| Service | `app/Services/*.php` | reglas de negocio; lanza `RuntimeException(404)`; **sin PDO** |
| Repository | `app/Repositories/*.php` | todo el SQL/PDO con prepared statements |
| Routes | `routes/api.php` | `[method, pattern, [Controller::class, action]]` o closure |

## Respuesta estándar

- Éxito: `{"success": true, "data": ...}` — 200 (lectura), 201 (creación).
- Error: `{"success": false, "error": "mensaje"}` — 400/404/500.
- 500 siempre `Error de base de datos`, sin stack trace (log en `php_errors.log`).

## Ejecución de acciones (reutilizar, no copiar)

`Response::execute(callable $action, int $status): never` en `app/Core/Response.php`
traduce `RuntimeException` → 400/404 y `Throwable` → 500. El controller solo:

```php
public function store(Request $request): never
{
    Response::execute(function () use ($request): array {
        $data = $this->validator->validateBody($request->body());
        return $this->service->create($data);
    }, 201);
}
```

## Errores de negocio

Sin clase de excepción propia: `throw new RuntimeException($msg, 404)`.
Orden de validación: id → body → existencia → relaciones.

## Inactivación lógica

`DELETE` = `UPDATE ... estado='inactivo'`; nunca `DELETE FROM`. Doble DELETE → 200.

## Checklist de validación arquitectónica (antes de aprobar checkpoint)

- `public/index.php` pequeño y sin SQL;
- Controllers sin PDO; Services sin SQL; Repositories con todo el SQL;
- rutas en `routes/`;
- `php -l` limpio en ficheros tocados;
- suite de regresión de categorías en verde.
