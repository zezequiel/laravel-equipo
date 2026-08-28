---
name: laravel-equipo
description: Cómo trabaja el equipo sobre proyectos Laravel — reconocimiento del repo antes de tocar nada, buenas prácticas durante la implementación y checklist de cierre. Usala apenas trabajes sobre cualquier proyecto Laravel, antes de crear o modificar controladores, servicios, modelos, migraciones, rutas, vistas Blade, comandos artisan, config o tests, y siempre que la tarea involucre la base de datos, envío de emails, jobs, dependencias nuevas o ejecutar comandos artisan. Aplicala también cuando el pedido parezca chico ("agregá un campo", "arreglá esta vista", "sumá un endpoint"), porque define qué sintaxis de PHP se puede usar en este proyecto, qué no hay que ejecutar y cómo se entrega el cambio.
---

# Cómo trabajamos en Laravel

Trabajás como un integrante más del equipo. Tenemos varios proyectos Laravel, de distintas épocas y con
distinto grado de prolijidad, así que lo que se mantiene constante no es el código sino el método:
**reconocer el terreno, respetar lo que ya está, y entregar el cambio explicado.**

Tres ideas guían todo lo que sigue:

1. **El proyecto manda sobre tus preferencias.** Un patrón peor pero consistente vale más que uno mejor
   aplicado en un solo archivo.
2. **Lo que no se puede deshacer se pregunta antes.** Migraciones, datos, emails, dependencias.
3. **Un cambio no está terminado hasta que se explica.** Quien lo revisa tiene que poder probarlo sin
   reconstruir tu razonamiento.

---

## Antes de empezar: reconocimiento

No lleva más de un minuto y evita la mayoría de los errores caros.

```bash
head -40 composer.json          # versión de Laravel y rango de PHP soportado
ls app/                         # qué capas existen (Services, Repositories, Actions, Library...)
ls config/                      # config propia del proyecto
ls tests/ && cat phpunit.xml    # si hay tests, de qué tipo y cómo corren
cat package.json                # Vite o laravel-mix, y qué front-end usa
git log --oneline -15           # cómo se escriben los commits acá
```

### Lo que tenés que sacar en limpio

**Versión de PHP soportada.** La que manda es la del `require` de `composer.json`, **no la de tu terminal**.
Si el proyecto declara `"php": "^7.3|^8.0"`, tu PHP 8.4 local compila `match`, `enum` y `?->` sin chistar y
el servidor de producción explota en el deploy. Ante la duda, escribí en el estilo del archivo que estás
tocando. La tabla de qué se puede usar en cada versión está en `references/versiones.md`.

**Versión de Laravel.** Cambia dónde viven las cosas. El caso que más confunde: desde Laravel 11 no existen
`app/Http/Kernel.php` ni `app/Console/Kernel.php` — el middleware se registra en `bootstrap/app.php` y los
comandos se descubren solos. Si vas a registrar middleware, un comando o un scheduler, confirmá primero
contra qué esqueleto estás trabajando. También está en `references/versiones.md`.

**Arquitectura existente.** Si el proyecto tiene `app/Services`, la lógica nueva va ahí. Si usa Actions,
usá Actions. Si no tiene ninguna capa intermedia y todo vive en controladores, no inventes una arquitectura
nueva en una tarea de dos horas: seguí el patrón vecino y, si te parece que hace falta refactorizar,
proponelo por separado.

**Convenciones del código.** Antes de nombrar algo nuevo, buscá cómo se llama lo equivalente que ya existe
(`grep` sobre `app/`). Idioma de clases y columnas, estilo de las relaciones Eloquent, cómo se validan los
formularios: todo eso se copia del vecino, no se decide de cero.

### Verificá el entorno antes de ejecutar nada

En local y en test, el `.env` suele apuntar a servicios reales o a copias que comparten configuración con
producción. Antes de correr un comando, un job o un endpoint que pueda tener efectos hacia afuera:

```bash
grep -E "^(APP_ENV|MAIL_MAILER|QUEUE_CONNECTION|DB_CONNECTION|DB_HOST|DB_DATABASE)=" .env
```

**El envío de emails tiene que estar desactivado.** `MAIL_MAILER=log` o `array` en local y en test. Si ves
`smtp`, `ses`, `mailgun` o similar, **no ejecutes nada que pueda disparar un mail y avisale a la persona**:
un reenvío masivo de prueba llega a clientes reales y no se puede deshacer. Lo mismo vale para SMS,
notificaciones push, webhooks salientes e integraciones con sistemas de terceros.

Revisá también a qué base apunta `DB_HOST`/`DB_DATABASE`. Si no podés confirmar que es local, tratala como
producción. El detalle está en `references/entorno-local.md`.

---

## Durante la implementación

### Respetá la arquitectura existente

El cambio tiene que parecer escrito por el equipo, no injertado. Ubicación de archivos, namespaces, forma de
inyectar dependencias, manejo de errores: copiá el patrón que ya está.

Buenas prácticas que aplican en cualquier proyecto Laravel, mientras no choquen con lo que ya existe:

- **Controladores delgados**: reciben, delegan y responden. Si un método pasa de unas 40 líneas o abre una
  conexión externa, eso es un Service, una Action o un Job.
- **Validación en Form Requests**, no dispersa en el controlador.
- **Nada hardcodeado**: umbrales, horarios, casillas y rutas de archivos van a `config/*.php` leyendo `env()`
  con un default, y el código lee `config('...')`. **Nunca llames a `env()` fuera de `config/`**: con la
  config cacheada devuelve `null` en producción.
- **Toda variable nueva se agrega a `.env.example` en el mismo commit.** Es el único lugar donde el resto del
  equipo se entera de que hay algo para configurar antes de deployar.
- **Errores con contexto útil en el log** y un mensaje claro para el usuario. Nunca loguees contraseñas,
  tokens, el contenido de `.env` ni datos personales completos: alcanza con identificadores.

Hay más detalle y ejemplos en `references/buenas-practicas.md`.

### No introduzcas dependencias sin autorización

Ningún `composer require` ni `npm install` nuevo por iniciativa propia. Una dependencia se arrastra al deploy,
al mantenimiento, a las auditorías de seguridad y a la versión de PHP que el proyecto puede usar más adelante.

Si te parece que hace falta un paquete: decilo, explicá qué resuelve, qué alternativa hay sin él, y esperá
la respuesta. Antes de proponerlo, fijate si el proyecto ya trae algo que sirva — muchas veces el paquete ya
está instalado y sin usar.

### No modifiques migraciones que ya se ejecutaron

Una migración mergeada ya corrió en la máquina de otro o en un ambiente. Editarla no cambia esas bases: las
deja desincronizadas, y la próxima persona pasa la tarde buscando por qué su esquema no coincide.

Para cambiar algo, **agregá una migración nueva**. Solo se edita una migración que todavía no salió de tu
rama y que sabés que nadie más corrió.

### Evitá cambios ajenos a la solicitud

Vas a cruzarte con código muerto, imports sin usar, formato inconsistente y cosas mal ubicadas. Salvo que
arreglarlas sea inevitable para resolver la tarea, déjalas y mencionalas al final.

Un diff chico se revisa; uno de 400 líneas con "mejoras varias" se aprueba sin leer, y ahí es donde se cuelan
los errores. **No reformatees archivos enteros**: el ruido de formato tapa el cambio real. Si arreglar algo
de paso fue inevitable, decilo explícitamente en el resumen.

### Protegé las operaciones destructivas

Estos comandos no se ejecutan por iniciativa propia, ni "para probar", ni aunque el entorno parezca local:

```
php artisan migrate:fresh
php artisan migrate:refresh
php artisan migrate:rollback
php artisan db:wipe
php artisan db:seed            # si los seeders escriben sobre tablas con datos reales
php artisan cache:clear        # en un ambiente compartido, se lo lleva puesto para todos
```

Migrar hacia adelante en una base local es normal; lo destructivo no. Lo mismo vale para `UPDATE` y `DELETE`
masivos escritos a mano y para borrar archivos de `storage/`: se proponen con el SQL a la vista, no se
ejecutan.

Si algo tiene que correr contra producción, se lo pedís a la persona.

### Usá transacciones cuando toques varias tablas relacionadas

Si una operación escribe en más de una tabla y un fallo a la mitad deja los datos inconsistentes, envolvela:

```php
DB::transaction(function () use ($datos) {
    $pedido = Pedido::create($datos['pedido']);
    $pedido->items()->createMany($datos['items']);
    $stock->descontar($datos['items']);
});
```

Dos cuidados que importan: **no metas llamadas HTTP ni envíos de mail adentro de la transacción** (si hace
rollback, el mail ya salió y la fila no existe — usá eventos o `DB::afterCommit()`), y ojo con los motores
que hacen commit implícito en cambios de esquema. El detalle está en `references/base-de-datos.md`.

### Prevení N+1 y cargas masivas innecesarias

El N+1 no se ve en desarrollo con 20 registros y voltea la producción con 200.000.

```php
// Mal: una consulta por cada pedido
foreach (Pedido::all() as $pedido) {
    echo $pedido->cliente->nombre;
}

// Bien: dos consultas, y sin traer todo a memoria
foreach (Pedido::with('cliente')->cursor() as $pedido) {
    echo $pedido->cliente->nombre;
}
```

Reglas prácticas:

- `with()` para las relaciones que vas a usar; `withCount()` cuando solo necesitás contar.
- **Nunca `Model::all()`** sobre una tabla que crece. Usá `chunkById()` para procesar y `cursor()` para
  recorrer sin cargar todo en memoria.
- Paginá los listados; no traigas la tabla entera para mostrar 20 filas.
- Para agregaciones pesadas, el Query Builder (`DB::table`) evita hidratar modelos que no vas a usar.
- Seleccioná solo las columnas necesarias cuando la tabla tiene campos grandes.

---

## Antes de finalizar

Esta parte no es opcional: es la diferencia entre entregar un cambio y entregar trabajo para otro.

### 1. Ejecutá las pruebas disponibles

```bash
php artisan test          # o ./vendor/bin/pest, o ./vendor/bin/phpunit
```

Si tocaste lógica de negocio, sumá o actualizá el test correspondiente. Si no hay tests en el proyecto,
decilo y explicá cómo verificaste el cambio a mano. **Si los tests fallan, mostrá la salida real**: si el
fallo ya existía antes de tu cambio, decilo; si lo causaste vos, arreglalo antes de entregar. No reportes
"todo verde" sin haber corrido nada.

### 2. Revisá formato y sintaxis

```bash
php -l archivo.php                        # sintaxis de cada archivo tocado
./vendor/bin/pint --dirty                 # si el proyecto usa Pint
git diff                                  # leé tu propio diff antes de entregarlo
```

Leer el diff propio es el paso que más errores atrapa: código de depuración olvidado (`dd()`, `dump()`,
`var_dump()`, `console.log`), credenciales pegadas, archivos que no querías tocar, bloques comentados.

### 3. Cerrá con este resumen

Terminá siempre con esta estructura. No es burocracia: es lo que permite revisar y probar sin preguntarte nada.

```markdown
**Qué cambié**
Una o dos frases sobre el problema y la solución.

**Archivos modificados**
- ruta/al/archivo.php — qué se cambió y por qué
- ruta/a/otro.blade.php — ídem

**Cómo probarlo**
Comandos concretos y el recorrido en la app.
Si hay pasos previos (migración, variable nueva en .env, npm run dev), van acá.

**Riesgos y supuestos**
- Lo que asumí y no pude verificar
- Qué se rompe si el supuesto es falso
- Impacto en datos existentes, performance o compatibilidad

**Pendiente**
- Lo que quedó fuera de alcance y por qué
- Lo que encontré mal en el camino y no toqué
```

Si algo no aplica (no hay riesgos, no quedó nada pendiente), decilo en una línea en vez de borrar la sección:
que esté vacía a propósito es información, que falte no.

Sé honesto en "Riesgos y supuestos". Si no pudiste probar algo, si el cambio depende de una variable que
alguien tiene que cargar, si hay una parte que quedó a medias — decilo. Un problema anunciado se maneja; uno
descubierto en producción, no.

---

## Referencias

- `references/versiones.md` — qué sintaxis de PHP podés usar según el rango de `composer.json`, y qué cambia
  de estructura entre Laravel 8, 9, 10, 11 y 12. Leelo apenas confirmes las versiones del proyecto.
- `references/entorno-local.md` — cómo verificar que emails, colas, notificaciones e integraciones externas
  estén desactivados antes de ejecutar algo. Leelo antes de correr cualquier comando o job.
- `references/base-de-datos.md` — migraciones, transacciones, N+1, procesamiento por lotes y operaciones
  destructivas, con ejemplos.
- `references/buenas-practicas.md` — controladores, validación, config, logs, nombres y criterio para tocar
  código heredado.
