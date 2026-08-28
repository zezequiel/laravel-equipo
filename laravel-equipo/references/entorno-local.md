# Entorno local y de test: qué verificar antes de ejecutar

En local, el `.env` suele apuntar a servicios reales o a copias que comparten configuración con producción.
La regla de fondo: **antes de ejecutar algo que escriba o que salga hacia afuera, verificá contra qué está
apuntando el proyecto.** Si no podés confirmarlo, tratalo como producción y preguntá.

Contenido:
- [Chequeo rápido](#chequeo-rápido)
- [Emails: lo más importante](#emails-lo-más-importante)
- [Colas y jobs](#colas-y-jobs)
- [Base de datos](#base-de-datos)
- [Integraciones externas](#integraciones-externas)
- [El entorno de tests](#el-entorno-de-tests)
- [.env: qué no hacer](#env-qué-no-hacer)

---

## Chequeo rápido

```bash
grep -E "^(APP_ENV|APP_DEBUG|MAIL_MAILER|QUEUE_CONNECTION|DB_CONNECTION|DB_HOST|DB_DATABASE)=" .env
```

Lo que querés ver en una máquina de desarrollo:

```
APP_ENV=local
MAIL_MAILER=log          # o array, o smtp apuntando a Mailhog/Mailpit
QUEUE_CONNECTION=sync    # o database, pero nunca la cola de producción
DB_HOST=127.0.0.1        # o localhost
```

Cuidado con la config cacheada: si existe `bootstrap/cache/config.php`, los valores efectivos pueden no ser
los del `.env`. Se limpia con `php artisan config:clear` (en tu máquina; en un ambiente compartido, no).

## Emails: lo más importante

Un envío de prueba llega a clientes reales y no se puede deshacer. No hay "deshacer" ni disculpa técnica.

**Antes de ejecutar cualquier cosa que pueda mandar un mail** — un comando artisan, un job, un endpoint, un
seeder, un test que no mockea — verificá `MAIL_MAILER`:

| Valor | Qué hace | ¿Seguro? |
|---|---|---|
| `log` | Escribe el mail en `storage/logs/laravel.log` | Sí |
| `array` | Lo guarda en memoria, no lo envía | Sí, es el de tests |
| `smtp` apuntando a Mailhog / Mailpit (`localhost:1025`) | Lo captura un servidor local | Sí |
| `smtp` / `ses` / `mailgun` / `postmark` con credenciales reales | **Envía de verdad** | No |

Si ves cualquiera de los últimos: **no ejecutes y avisá.** Decilo explícitamente ("el `.env` tiene
`MAIL_MAILER=smtp` con un host real, así que no corrí el comando") en vez de asumir que no pasa nada porque
"es solo una prueba".

Lo mismo aplica a todo lo que sale hacia una persona: SMS, notificaciones push, mensajes a Slack o
WhatsApp, y webhooks salientes hacia sistemas de terceros.

Cuando escribas código nuevo que envíe correo, dejá el envío detrás de una condición de config
(por ejemplo `config('mail.envios_habilitados')`) para que un ambiente pueda tenerlo apagado sin comentar
código.

En tests, `Mail::fake()` y `Notification::fake()` son la forma correcta de verificar que se iba a enviar algo
sin enviarlo.

## Colas y jobs

`QUEUE_CONNECTION=sync` ejecuta los jobs en el momento, dentro del request: es lo más simple para desarrollar
y lo que evita sorpresas. Con `database` o `redis`, un job queda encolado y puede ser tomado por un worker que
alguien dejó corriendo — posiblemente contra otra base.

Antes de despachar un job a mano, mirá qué conexión hay configurada y si hay un worker vivo.

## Base de datos

Mirá `DB_HOST` y `DB_DATABASE`. Un host que no sea `127.0.0.1` o `localhost` es motivo suficiente para
frenar y preguntar.

Aunque la base sea local, puede ser una copia de producción con datos reales de personas. Eso implica:

- No copies datos reales a fixtures, archivos de prueba, mensajes o issues. Usá datos inventados o factories.
- No los pegues en un log para depurar; logueá identificadores.

Las operaciones destructivas están detalladas en `base-de-datos.md`.

## Integraciones externas

Muchos proyectos hablan con APIs de terceros. En local, esas credenciales suelen ser las mismas de
producción porque el proveedor no da un ambiente de pruebas.

Antes de ejecutar código que llame a una API externa, preguntate si esa llamada **escribe** algo del otro
lado. Leer un catálogo es inofensivo; dar de alta, dar de baja o disparar un proceso remoto, no. En ese caso,
verificá con un payload guardado o mockeá el cliente HTTP (`Http::fake()`), en vez de llamar de verdad.

## El entorno de tests

`phpunit.xml` define su propio entorno y normalmente ya es seguro:

```xml
<server name="APP_ENV" value="testing"/>
<server name="DB_CONNECTION" value="sqlite"/>
<server name="DB_DATABASE" value=":memory:"/>
<server name="MAIL_MAILER" value="array"/>
<server name="QUEUE_CONNECTION" value="sync"/>
```

Dos advertencias igual:

- Esa configuración **no protege de llamadas HTTP salientes**. Si el código bajo prueba llama a una API,
  mockeala.
- Si el proyecto tiene un bootstrap o un guard que aborta los tests cuando detecta `APP_ENV=production`, no lo
  saltees ni lo modifiques para que "pase". Está ahí porque el riesgo es real.

## `.env`: qué no hacer

- **No lo edites.** Si tu cambio necesita una variable nueva, agregala a `.env.example` con un valor de
  ejemplo y decíselo a la persona para que la cargue en cada ambiente.
- **No muestres su contenido** ni lo copies a un log, a un mensaje o a un archivo temporal.
- **No crees copias** tipo `.env.save`, `.env.bak`, `.env.old`. Terminan versionadas por accidente y eso
  significa credenciales en el historial de git.
- Antes de commitear, mirá qué archivos entran: `.gitignore` cubre `.env` pero casi nunca sus variantes.
- Si encontrás una credencial versionada, avisá en vez de borrarla por tu cuenta: borrar el archivo no la
  saca del historial, y rotarla es una decisión de la persona.
