---
title: Verificar la versión de formato contra el hash del esquema
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T02:23:33Z
parent: r
---

# Verificar la versión de formato contra el hash del esquema

`sha256(esquema generado) == <hash registrado para la versión N>`. Cambiar los tipos sin subir la versión falla.

Es lo que convierte `.bilink/version` de una promesa en una propiedad del artefacto. El caso que motiva: [ADR-0005](../../subsystems/bilinker/.stratum/impl/docs/adr/0005-frontera-entre-proyectos.md) agrega endpoints **sin migración** porque son aditivos, y un parser viejo leería `abstract` como un path de capa sin fallar — exactamente el cambio que uno se olvida de bumpear.

## Cómo quedó

`SCHEMA_HASHES` mapea versión a hash y el test compara contra la versión del crate. El esquema lleva su propia versión adentro, así que el hash certifica las dos cosas: qué tipos describe y bajo qué nombre se publicó.

Se probó cambiando el formato sin tocar la versión, y falla con el mensaje que dice qué hacer.

## El guard no cubría su propio caso

Con el esquema del endpoint escrito como `{"type": "string"}` a secas, **agregar un tipo de endpoint no movía el hash**. Se verificó agregando `repo <alias>`: el test pasaba.

O sea que el guard fallaba justo en el caso que [ADR-0006](../../subsystems/bilinker/.stratum/impl/docs/adr/0006-formato-como-crate-versionado.md) pone como su motivo. La causa: un esquema que describe de menos no puede servir de guarda, y `"es un string"` borra las variantes.

Se arregló haciendo que **el parser y el esquema salgan de la misma tabla**. `ENDPOINT_PREFIXES` lleva los prefijos reconocidos y su constructor; `from_str` la recorre y `JsonSchema` la publica. Agregar un tipo obliga a tocarla, eso cambia el esquema, y el guard lo detecta — verificado, ahora falla.

De ahí sale una regla que quedó escrita en la spec: **lo que discrimina al parsear tiene que ser visible en el esquema.** Si el parser distingue por algo que el esquema no menciona, ese algo puede cambiar sin que nada se entere.

## Lo que el esquema todavía no es

Describe **el modelo, no el archivo**. Los `.bilink` de hoy son texto plano leído por un parser a mano, y un esquema JSON no puede describir eso. Cuando los archivos pasen a YAML con serde, el esquema pasa a describir el archivo literal sin reescribirse, porque los tipos son los mismos.

Hasta entonces el guard sirve para lo suyo —que ningún cambio de formato pase sin versión— y lo que falta es que un tercero pueda validar un archivo ajeno. Está dicho en `concepts/format-version.md` § "Qué describe el esquema, hoy" para que nadie lo suponga.

## Publicación

```sh
cargo run -q -p bilink-format --bin schema > bilink-format-<version>.json
```

Que la release lo suba como artefacto es trabajo de CI, y no hay CI todavía.
