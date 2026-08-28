# laravel-equipo

Skill para Claude Code que define una forma segura y consistente de trabajar en proyectos Laravel existentes, especialmente en equipos que mantienen aplicaciones con distintas versiones de PHP, Laravel y arquitecturas.

La skill le indica a Claude que primero reconozca el proyecto, respete sus convenciones, limite el alcance de los cambios y verifique el resultado antes de finalizar.

## ¿Qué hace?

Antes de modificar un proyecto, Claude revisa:

* La versión de PHP declarada en `composer.json`.
* La versión y estructura de Laravel.
* La arquitectura existente del proyecto.
* Las convenciones utilizadas por el equipo.
* La configuración local, la base de datos y los servicios externos.
* Los tests y herramientas disponibles.

Durante la implementación aplica criterios para:

* Respetar la arquitectura existente.
* Mantener controladores delgados.
* Utilizar Form Requests para validación.
* Evitar valores y configuraciones hardcodeadas.
* No instalar dependencias sin autorización.
* No modificar migraciones que ya fueron ejecutadas.
* Prevenir consultas N+1 y cargas masivas.
* Utilizar transacciones cuando se modifican varias tablas relacionadas.
* Evitar comandos destructivos y efectos externos accidentales.
* Mantener los cambios acotados a la solicitud.

Al finalizar, entrega un resumen con:

* Qué se modificó.
* Archivos afectados.
* Cómo probar el cambio.
* Riesgos y supuestos.
* Trabajo pendiente.

## Principios principales

1. El proyecto manda sobre las preferencias personales.
2. Lo que no se puede deshacer se consulta antes.
3. Un cambio no está terminado hasta que puede explicarse y probarse.

## Estructura

```text
laravel-equipo/
├── SKILL.md
└── references/
    ├── base-de-datos.md
    ├── buenas-practicas.md
    ├── entorno-local.md
    └── versiones.md
```

### Referencias incluidas

* `versiones.md`: compatibilidad de sintaxis PHP y diferencias entre versiones de Laravel.
* `entorno-local.md`: controles antes de ejecutar comandos, jobs o integraciones.
* `base-de-datos.md`: migraciones, transacciones, consultas N+1 y operaciones masivas.
* `buenas-practicas.md`: arquitectura, validación, configuración, logs, nombres y tests.

## Instalación

### Mediante SkillsMP

```bash
npx skills add https://github.com/zezequiel/laravel-equipo --skill laravel-equipo
```

### Instalación manual en Claude Code

Copiá la carpeta interna `laravel-equipo` en el directorio personal de skills:

```text
~/.claude/skills/laravel-equipo/
```

En Windows, `~` normalmente corresponde a:

```text
C:\Users\TU_USUARIO
```

Por lo tanto, la ubicación sería:

```text
C:\Users\TU_USUARIO\.claude\skills\laravel-equipo\
```

La estructura final debe contener:

```text
~/.claude/skills/laravel-equipo/SKILL.md
```

Al instalarla como skill personal, estará disponible en todos tus proyectos.

También puede instalarse solamente dentro de un proyecto:

```text
proyecto-laravel/.claude/skills/laravel-equipo/SKILL.md
```

## Uso

Claude puede activar la skill automáticamente cuando detecta una tarea relacionada con Laravel. También puede invocarse manualmente:

```text
/laravel-equipo
```

Ejemplos de solicitudes:

```text
Analizá este proyecto Laravel antes de realizar modificaciones.
```

```text
Usá laravel-equipo para agregar un campo teléfono a los clientes.
```

```text
Revisá este endpoint y verificá posibles consultas N+1.
```

```text
Implementá este cambio respetando las convenciones existentes del proyecto.
```

## Alcance

Esta es una skill general para proyectos Laravel. No incluye reglas de negocio ni configuraciones específicas de una aplicación determinada.

Las instrucciones particulares de cada sistema deberían mantenerse en skills separadas.

## Contribuciones

Las sugerencias, correcciones y mejoras son bienvenidas mediante issues o pull requests.

## Autor

Creado por [Ezequiel Quiroga](https://github.com/zezequiel).

## Licencia

Este proyecto se distribuye bajo la [licencia MIT](LICENSE).

