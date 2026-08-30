---
title: Crear un bilink entre subsistemas exige escribir los archivos a mano
status: open
created_at: 2026-08-29T23:53:30Z
updated_at: 2026-08-30T00:22:05Z
---

# Crear un bilink entre subsistemas exige escribir los archivos a mano

El CLI no puede crear las cadenas que este proyecto necesita. Dos huecos, los dos en el camino más común.

## `chain new` no alcanza la capa de un subsistema

```
$ bilinker chain new --tip 'subsystems/bilinker/concepts/capture.md:29:1' \
                     --tip 'subsystems/bilinker>impl/crates/bilinker/src/capture.rs:523:1'
Error: unexpected token in layer navigation: Simple("subsystems/bilinker")
```

`main.rs::layer_tokens_to_fs_path` acepta `Down` y `Up` y rechaza `Simple` y `TopRoot`. O sea: se puede bajar a `.stratum/<name>` desde la capa actual, pero no atravesar un directorio común primero — que es exactamente la forma de accreta, donde la capa raíz contiene las specs de cinco subsistemas y cada uno tiene su impl abajo.

Las 63 cadenas que existían antes de la task `o` tienen `link.1: subsystems/bilinker>impl`, así que el formato lo soporta desde siempre; lo que no lo soporta es el comando que las crea.

## `capture` no puede capturar un archivo completo

```
$ bilinker capture crates/bilinker/src/lib.rs
Error: uso: bilinker capture <file> <start> <end>
```

El formato lo contempla —`query` ausente = el archivo completo, [`concepts/capture.md`](../../../subsystems/bilinker/concepts/capture.md) § "Formato"— y `capture.rs::capture_file_whole` está implementado, pero solo lo llama `chain new` cuando el tip viene sin posición. Por el hueco anterior, ese camino tampoco está disponible acá.

Los captures de archivo completo que hay en disco son la forma más usada del lado de las specs: `bilink.md`, `reference.md`, `chain.md`, `status.md`, `watch.md` y varios más.

## El costo

La task `o` creó 48 cadenas con un script que llama a `capture` dos veces y escribe los dos `.bilink` con `printf`. Escribir a mano el campo que define a qué apunta un vínculo es justo lo que [`commands/recapture.md`](../../../subsystems/bilinker/commands/recapture.md) § "Propósito" da como razón de existir de `recapture`.

## Cuándo está hecha

`bilinker chain new` crea la cadena del ejemplo de arriba, con y sin posición en cada tip, sin escribir un archivo a mano.

## Por qué acá y no en el sprint 1

Salió de la US `n` por prioridad, no por dependencia: el arreglo está en `main.rs::layer_tokens_to_fs_path`, que parsea paths Stratum y no toca el formato. Pero no corrompe nada —el workaround es un script de diez líneas— y `chain new` va a reescribirse igual para emitir el formato nuevo. Arreglarlo acá es hacerlo una vez.
