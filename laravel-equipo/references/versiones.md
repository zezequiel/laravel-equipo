# Versiones: qué podés usar en este proyecto

El objetivo de este archivo es que no escribas código que compila en tu máquina y falla en el servidor, y que
no busques archivos que en esta versión de Laravel no existen.

Contenido:
- [Cómo averiguar las versiones reales](#cómo-averiguar-las-versiones-reales)
- [Sintaxis de PHP por versión](#sintaxis-de-php-por-versión)
- [Estructura por versión de Laravel](#estructura-por-versión-de-laravel)
- [Front-end: mix o vite](#front-end-mix-o-vite)

---

## Cómo averiguar las versiones reales

```bash
grep -A5 '"require"' composer.json          # el rango declarado: eso es lo que manda
php artisan --version                       # versión de Laravel instalada
composer show laravel/framework | head -3   # versión exacta del framework
php -v                                      # tu PHP local (NO es el del servidor)
```

**El rango de `composer.json` es el contrato.** Si dice `"php": "^7.3|^8.0"`, el código tiene que correr en
7.3, aunque tu terminal tenga 8.4 y aunque el servidor de hoy tenga 8.0. Ese rango está ahí porque alguien
decidió mantener compatibilidad, y romperlo se descubre recién en el deploy.

Si el rango te resulta innecesariamente viejo, decilo, pero no lo subas por tu cuenta: cambiar el mínimo de
PHP es una decisión de infraestructura, no de una tarea.

Ante la duda sobre un caso puntual, mirá cómo está escrito el archivo que estás tocando. El código existente
es la mejor prueba de qué acepta el proyecto.

## Sintaxis de PHP por versión

Cada fila indica desde qué versión está disponible. Si el mínimo del proyecto es menor, no lo uses.

| Disponible desde | Característica |
|---|---|
| 7.4 | Arrow functions (`fn () => ...`), propiedades tipadas (`private int $x`), null coalescing assignment (`??=`), spread en arrays |
| 8.0 | Constructor property promotion, `match`, argumentos nombrados, nullsafe (`?->`), union types, atributos, `str_contains()` / `str_starts_with()`, `throw` como expresión |
| 8.1 | `enum`, `readonly` en propiedades, `never` como tipo de retorno, first-class callable (`$obj->metodo(...)`), `array_is_list()` |
| 8.2 | Clases `readonly`, tipos DNF, constantes en traits |
| 8.3 | Constantes de clase tipadas, `json_validate()`, `#[Override]` |
| 8.4 | Property hooks, visibilidad asimétrica, `new` sin paréntesis extra |

### Equivalencias para proyectos con soporte 7.3 / 7.4

| No disponible | Escribilo así |
|---|---|
| Constructor promotion | Propiedad `protected` declarada + asignación en el constructor |
| `match (...)` | `switch` o un array de mapeo |
| `enum` | Constantes de clase (`const ESTADO_ACTIVO = 1;`) |
| `?->` | `if ($obj) { ... }` o `optional($obj)->campo` |
| Propiedades tipadas | Propiedad sin tipo + docblock `@var` |
| Argumentos nombrados | Argumentos posicionales |
| `str_contains($a, $b)` | `Str::contains($a, $b)` (helper de Laravel, funciona en todas) |

Los helpers `Str` y `Arr` de Laravel son un buen atajo general: te dan la funcionalidad moderna sin depender
de la versión de PHP.

## Estructura por versión de Laravel

Esto es lo que más confunde, porque el esqueleto cambió fuerte en la 11.

| | Laravel 8 | Laravel 9 | Laravel 10 | Laravel 11 / 12 |
|---|---|---|---|---|
| PHP mínimo | 7.3 | 8.0.2 | 8.1 | 8.2 |
| Middleware se registra en | `app/Http/Kernel.php` | `app/Http/Kernel.php` | `app/Http/Kernel.php` | `bootstrap/app.php` |
| Comandos y scheduler | `app/Console/Kernel.php` | `app/Console/Kernel.php` | `app/Console/Kernel.php` | Auto-descubiertos; scheduler en `routes/console.php` |
| Excepciones | `app/Exceptions/Handler.php` | ídem | ídem | `bootstrap/app.php` |
| Casts del modelo | Propiedad `$casts` | ídem | ídem | Método `casts()` (la propiedad sigue andando) |
| Providers | `config/app.php` | ídem | ídem | `bootstrap/providers.php` |
| Front-end por defecto | laravel-mix | mix (Vite desde 9.19) | Vite | Vite |

**Antes de registrar un middleware, un comando o una tarea programada, confirmá cuál de estos dos mundos es
el proyecto.** Un `ls bootstrap/` alcanza: si hay `app.php` con configuración de middleware y no existe
`app/Http/Kernel.php`, estás en 11+.

Otras diferencias que aparecen seguido:

- **Rutas con string de acción** (`'App\Http\Controllers\FooController@bar'`) funcionan en 8 pero no en las
  versiones nuevas sin el namespace configurado. En proyectos nuevos usá la sintaxis de array:
  `[FooController::class, 'bar']`. En proyectos viejos, seguí el estilo del archivo.
- **`app/Models`** existe desde Laravel 8. En proyectos migrados desde 7, los modelos pueden estar en `app/`.
- **Pint** viene desde Laravel 9. En proyectos anteriores el formato lo maneja StyleCI o no hay herramienta.

## Front-end: mix o vite

Miralo en `package.json`, no lo asumas:

- Si hay `webpack.mix.js` y `laravel-mix` en devDependencies → **laravel-mix**: `npm run dev`, `npm run watch`,
  `npm run prod`. En Blade, los assets se referencian con `mix('js/app.js')`.
- Si hay `vite.config.js` y `vite` → **Vite**: `npm run dev`, `npm run build`. En Blade, `@vite(['resources/js/app.js'])`.

Mezclarlos rompe la compilación. Y si el proyecto usa Tailwind, mirá la versión: la sintaxis de utilidades y
la configuración cambian bastante entre Tailwind 2, 3 y 4.
