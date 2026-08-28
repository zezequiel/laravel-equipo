# Buenas prácticas al escribir código

La regla que está por encima de todas las de este archivo: **si el proyecto ya hace algo de otra forma y de
manera consistente, seguí el proyecto.** Estas son las prácticas que aplicamos cuando el código nuevo no
tiene un patrón previo que copiar.

Contenido:
- [Dónde va cada cosa](#dónde-va-cada-cosa)
- [Controladores](#controladores)
- [Validación](#validación)
- [Configuración](#configuración)
- [Logs y manejo de errores](#logs-y-manejo-de-errores)
- [Nombres](#nombres)
- [Formato](#formato)
- [Tocar código heredado](#tocar-código-heredado)
- [Tests](#tests)

---

## Dónde va cada cosa

| Capa | Qué va | Qué NO va |
|---|---|---|
| `routes/` | Declaración de rutas, con `->name()` | Lógica, consultas, closures largos |
| `app/Http/Controllers` | Recibir el request, delegar, devolver respuesta | Reglas de negocio, queries complejas, integraciones |
| `app/Http/Requests` | Validación y mensajes de error | Lógica de negocio |
| `app/Services` (o Actions) | Lógica de negocio y orquestación | HTML, acceso a `request()` |
| `app/Models` | Relaciones, casts, scopes, accessors | Procesos largos, envío de mails |
| `app/Jobs` | Trabajo pesado o diferido | Lógica que solo se usa desde ahí y podría ser un Service |
| `app/Console/Commands` | Tareas programadas y de mantenimiento | Lógica duplicada de un Service |
| `config/*.php` | Umbrales, horarios, cuentas, rutas de archivos | Valores literales dispersos por el código |

Si el proyecto no tiene capa de servicios y todo vive en controladores, no la introduzcas en una tarea chica.
Seguí el patrón vecino y proponé el refactor por separado, con su propio alcance.

## Controladores

Delgados: reciben, delegan y responden. Si un método pasa de unas 40 líneas, abre una conexión externa o
mezcla tres responsabilidades, eso es un Service.

```php
public function store(GuardarPedidoRequest $request, PedidoService $service)
{
    try {
        $pedido = $service->crear($request->validated(), $request->user());
    } catch (StockInsuficienteException $e) {
        return back()->withInput()->with('error', 'No hay stock suficiente para completar el pedido.');
    }

    return redirect()
        ->route('pedidos.show', $pedido)
        ->with('success', 'El pedido se creó correctamente.');
}
```

Lo que vale la pena copiar de ahí:

- Las dependencias entran por la firma del método (o por el constructor); no se instancian a mano.
- El mensaje al usuario ya está escrito para el usuario, en el idioma y el tono del resto de la app.
- Cuando la operación falla, el mensaje **dice qué pasó y si hay que reintentar**. "Error inesperado" obliga
  a alguien a ir al log.

Si vas a agregar una ruta, ponele siempre `->name()`: además de ser lo que usan `route()` y los redirects,
en muchos proyectos el sistema de permisos autoriza por nombre de ruta, y una ruta sin nombre se cuela sin
control de acceso.

## Validación

En Form Requests, con reglas en array y mensajes propios cuando el default no se entiende:

```php
class GuardarPedidoRequest extends FormRequest
{
    public function rules()
    {
        return [
            'cliente_id' => ['required', 'exists:clientes,id'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.cantidad' => ['required', 'integer', 'min:1'],
        ];
    }

    public function messages()
    {
        return [
            'items.required' => 'Tenés que agregar al menos un ítem al pedido.',
        ];
    }
}
```

Usá `$request->validated()` en vez de `$request->all()` para pasar datos a la capa de negocio: así lo que
llega al modelo es solo lo que validaste.

Validar en el servidor no es opcional aunque el formulario ya valide en el front.

## Configuración

Nada de valores mágicos desperdigados. Umbrales, horarios, casillas, rutas de archivos y flags van a
`config/*.php` leyendo `env()` con un default sensato:

```php
// config/pedidos.php
return [
    'maximo_items' => (int) env('PEDIDOS_MAXIMO_ITEMS', 50),
    'notificaciones_habilitadas' => (bool) env('PEDIDOS_NOTIFICACIONES', true),
];
```

Tres reglas que van juntas:

1. **`env()` solo dentro de `config/`.** Con la config cacheada (`config:cache`, habitual en producción)
   devuelve `null` en cualquier otro lado. Es un bug que no aparece en desarrollo.
2. **Toda variable nueva se agrega a `.env.example` en el mismo commit.** Es el único lugar donde el resto
   del equipo se entera de que hay algo para configurar antes de deployar.
3. **Los efectos hacia afuera van detrás de un flag de config** (envío de mails, llamadas a APIs de terceros,
   procesos programados), para que un ambiente pueda tenerlos apagados sin comentar código.

## Logs y manejo de errores

Un `catch` que no deja rastro convierte un incidente en un misterio. Logueá con contexto suficiente para
reconstruir qué pasó, y devolvé un mensaje claro:

```php
Log::error('No se pudo sincronizar el pedido con el sistema externo.', [
    'pedido_id' => $pedido->id,
    'exception_type' => get_class($exception),
]);
```

- Si el proyecto tiene canales por dominio en `config/logging.php`, usá el que corresponda
  (`Log::channel('pedidos')->error(...)`).
- Elegí el nivel con criterio: `error` para lo que requiere intervención, `warning` para lo raro pero
  manejado, `info` para hitos de un proceso.
- **Nunca loguees** contraseñas, tokens, el contenido de `.env` ni datos personales completos. Con
  identificadores alcanza para diagnosticar.
- No atrapes excepciones para silenciarlas. Si no vas a hacer nada con el error, dejá que suba.

## Nombres

Buscá primero cómo se llama lo equivalente que ya existe (`grep` sobre `app/`) antes de inventar un término
nuevo para el mismo concepto. Tener `Pedido`, `Order` y `Compra` conviviendo es lo que hace que después nadie
encuentre nada.

En proyectos con dominio en español, lo habitual es: **clases y columnas de negocio en español**
(`Pedido`, `pedidos`, `fecha_baja`), **infraestructura en inglés** (`PedidoService`, `GuardarPedidoRequest`),
**comentarios en español** y **mensajes al usuario en el tono del resto de la app**. Si un archivo ya mezcla
idiomas, seguí el criterio del archivo: la consistencia local vale más que la global.

## Formato

- 4 espacios, LF, UTF-8, sin espacios al final de línea (lo habitual del `.editorconfig` de Laravel).
- Si el proyecto tiene Pint: `./vendor/bin/pint --dirty` antes de entregar. Si tiene StyleCI, respetá el
  preset configurado.
- **No reformatees archivos enteros** ni limpies imports sin usar en un archivo que estás tocando por otro
  motivo: el ruido tapa el cambio real en el diff. Si querés hacer esa limpieza, que sea un cambio propio.

## Tocar código heredado

Muchos de estos proyectos tienen años: controladores enormes, lógica de negocio en lugares raros, rutas
comentadas, código muerto.

- **Cambio quirúrgico.** Modificá lo mínimo necesario. Un diff chico se revisa; uno grande con "mejoras
  varias" se aprueba sin leer, y ahí es donde se cuelan los errores.
- **Extraé al salir, no al entrar.** Si tenés que modificar lógica que está mal ubicada, movela como parte
  del cambio y decilo en el resumen. No hagas refactors preventivos de código que no ibas a tocar.
- **No borres código comentado ajeno.** Muchas veces es un proceso apagado a propósito, no basura.
- Lo que encuentres mal y no toques, mencionalo en la sección "Pendiente" del resumen final.

## Tests

Si el proyecto tiene tests, tu cambio en lógica de negocio viene con test. Si no tiene, decilo y explicá cómo
verificaste el cambio a mano.

- `tests/Unit` para lógica pura de una clase; `tests/Feature` para el recorrido HTTP completo.
- Nombres que describan la regla, no el método: `test_no_permite_crear_un_pedido_sin_items`.
- **Dejá entrar las dependencias por parámetro** para poder testear: un service que llama a `Carbon::now()`
  adentro no se puede testear con fechas fijas; uno que recibe `?Carbon $ahora = null`, sí.
- Mockeá lo externo: `Mail::fake()`, `Http::fake()`, `Notification::fake()`, `Queue::fake()`. El entorno de
  tests protege de la base y del correo, pero no de una llamada HTTP saliente.
