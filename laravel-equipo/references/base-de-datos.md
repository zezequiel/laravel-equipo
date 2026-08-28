# Base de datos

Contenido:
- [Migraciones](#migraciones)
- [Operaciones destructivas](#operaciones-destructivas)
- [Transacciones](#transacciones)
- [Consultas N+1](#consultas-n1)
- [Procesar muchos registros](#procesar-muchos-registros)
- [Cómo detectar problemas antes de entregar](#cómo-detectar-problemas-antes-de-entregar)

---

## Migraciones

**Una migración que ya se ejecutó no se modifica.** Cuando se mergea, alguien más ya la corrió en su máquina
o en un ambiente: editarla no cambia esas bases, las deja desincronizadas. La próxima persona pierde la tarde
buscando por qué su esquema no coincide con el código.

Para cambiar algo, agregá una migración nueva:

```php
// 2026_08_26_120000_add_telefono_to_clientes_table.php
public function up()
{
    Schema::table('clientes', function (Blueprint $table) {
        $table->string('telefono', 30)->nullable()->after('email');
    });
}

public function down()
{
    Schema::table('clientes', function (Blueprint $table) {
        $table->dropColumn('telefono');
    });
}
```

Solo se edita una migración que todavía no salió de tu rama y que sabés que nadie corrió.

Otras cosas que importan:

- **Escribí el `down()` para que deje la base como estaba.** Aunque casi nunca se corra, es lo que te salva
  en un deploy fallido.
- **Columnas nuevas en tablas con datos: `nullable()` o con `default()`.** Un `NOT NULL` sin default falla
  en una tabla que ya tiene filas.
- **Cuidado con las migraciones que tardan.** Agregar un índice o cambiar el tipo de una columna en una tabla
  de millones de filas puede bloquearla durante el deploy. Si la tabla es grande, avisalo como riesgo.
- **Seguí la convención de nombres del proyecto** para tablas y claves foráneas. En bases heredadas es común
  que las FK no sigan el estándar de Laravel (`id_cliente` en vez de `cliente_id`); en ese caso, las
  relaciones Eloquent tienen que declarar las columnas explícitamente, o Laravel va a buscar una que no existe.
- **Migración y modelo van juntos.** Si agregás una columna que se escribe desde la app, sumala a `$fillable`;
  si es fecha, booleano o JSON, sumala a `$casts`; si es sensible, a `$hidden`.

## Operaciones destructivas

No se ejecutan por iniciativa propia, ni "para probar", ni aunque el entorno parezca local:

```
php artisan migrate:fresh      # borra todas las tablas y vuelve a migrar
php artisan migrate:refresh    # rollback completo + migrate
php artisan migrate:rollback   # revierte el último lote
php artisan db:wipe            # elimina todas las tablas
php artisan db:seed            # si los seeders escriben sobre tablas con datos reales
```

Migrar hacia adelante en una base local es normal. Lo destructivo es lo que está vedado por defecto, porque
desde el código no hay forma de distinguir con certeza si la conexión apunta a una copia o a la base que usa
el negocio.

La misma regla para los `UPDATE` y `DELETE` masivos escritos a mano: se proponen con el SQL a la vista para
que la persona lo revise y lo ejecute, no se ejecutan.

Si necesitás corregir datos, escribí el `SELECT` equivalente primero y mostrá cuántas filas afecta.

## Transacciones

Si una operación escribe en más de una tabla y quedar a medias deja los datos inconsistentes, envolvela:

```php
DB::transaction(function () use ($datos) {
    $pedido = Pedido::create($datos['pedido']);
    $pedido->items()->createMany($datos['items']);
    $this->stock->descontar($datos['items']);
});
```

Si algo lanza una excepción, se revierte todo. La versión manual (`DB::beginTransaction()` /
`commit()` / `rollBack()`) sirve cuando necesitás control fino, pero exige un `try/catch` que haga rollback
en **todos** los caminos de salida; el closure lo resuelve solo.

### Qué NO va adentro de una transacción

**Efectos que no se pueden revertir**: envío de mails, llamadas HTTP a terceros, escritura de archivos,
despacho de jobs. Si la transacción hace rollback, el mail ya salió y la fila no existe.

```php
// Mal
DB::transaction(function () use ($pedido) {
    $pedido->save();
    Mail::to($pedido->cliente)->send(new PedidoConfirmado($pedido));
});

// Bien: el mail sale solo si la transacción se confirmó
DB::transaction(function () use ($pedido) {
    $pedido->save();
});
Mail::to($pedido->cliente)->send(new PedidoConfirmado($pedido));

// O, si el envío está en un listener o un job:
DB::afterCommit(function () use ($pedido) { ... });
```

Los jobs tienen `public $afterCommit = true;` para lo mismo: evita que un worker tome el job antes de que la
fila exista.

### Otros cuidados

- **No pongas cambios de esquema (`Schema::`) dentro de una transacción**: en MySQL hacen commit implícito y
  la transacción deja de protegerte.
- **Transacciones cortas.** Cuanto más tiempo abierta, más filas bloqueadas y más chances de deadlock. No
  metas un proceso largo adentro.
- **Ojo con las transacciones anidadas**: Laravel usa savepoints, así que una transacción interna que falla
  no necesariamente revierte la externa como esperás.

## Consultas N+1

Es el problema de performance más común y el que peor se detecta: con 20 registros de prueba no se nota, con
200.000 en producción tira el servidor.

```php
// Mal: 1 consulta + N consultas (una por cada pedido)
$pedidos = Pedido::all();
foreach ($pedidos as $pedido) {
    echo $pedido->cliente->nombre;
}

// Bien: 2 consultas en total
$pedidos = Pedido::with('cliente')->get();
```

- `with('relacion')` para lo que vas a usar; `with('a.b')` para relaciones anidadas.
- `withCount('items')` cuando solo necesitás la cantidad, en vez de traer los items.
- `load('relacion')` si el modelo ya está cargado y después descubrís que necesitás la relación.
- **En Blade también cuenta**: un `@foreach` que accede a `$item->relacion->campo` genera una consulta por
  fila. Si la vista usa una relación, cargala en el controlador.

En desarrollo podés forzar que el N+1 sea un error en vez de un problema silencioso, agregando
`Model::preventLazyLoading(! app()->isProduction());` en el `AppServiceProvider` (Laravel 9+). Si el proyecto
ya lo tiene activado, respetalo: un `with()` faltante va a tirar excepción.

## Procesar muchos registros

`Model::all()` sobre una tabla que crece trae todo a memoria y termina en un fatal error por memoria agotada.

```php
// Recorrer sin cargar todo en memoria (una fila por vez)
foreach (Pedido::where('estado', 'pendiente')->cursor() as $pedido) {
    // ...
}

// Procesar por lotes, seguro incluso si el bucle modifica las filas
Pedido::where('estado', 'pendiente')->chunkById(500, function ($pedidos) {
    foreach ($pedidos as $pedido) {
        // ...
    }
});
```

- `chunkById()` en vez de `chunk()` cuando el bucle modifica las filas que está recorriendo: con `chunk()` y
  offsets, al cambiar el criterio se saltean registros.
- **Paginá los listados**. Nunca traigas la tabla entera para mostrar 20 filas.
- **Traé solo las columnas que necesitás** (`select('id', 'nombre')`) cuando la tabla tiene campos grandes.
- **Para agregaciones y reportes, el Query Builder** (`DB::table(...)->selectRaw(...)`) evita hidratar
  modelos que no vas a usar. Es la diferencia entre segundos y minutos en tablas grandes.
- Si el proceso es largo, que sea un comando artisan o un job, no un request HTTP con el timeout subido.

## Cómo detectar problemas antes de entregar

- Contá las consultas de la pantalla o el endpoint que tocaste. Con Telescope o Debugbar instalados es
  inmediato; si no, `DB::enableQueryLog()` y `DB::getQueryLog()` en una prueba puntual alcanzan.
- Probá con volumen: si en desarrollo hay 20 filas y en producción hay 200.000, calculá qué pasa a esa escala
  y decilo como riesgo si no podés probarlo.
- Ante una query nueva y pesada, mirá si las columnas del `where` y del `join` tienen índice.
