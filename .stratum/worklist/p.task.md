---
title: Ejecutar el corte de formato (`004`) en los 4 repos
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T05:06:19Z
parent: 9
---

# Ejecutar el corte de formato (`004`) en los 4 repos

El formato nuevo vivo en las ramas, con git normal. Nada de la ref todavía.

## El corte es un comando, no una secuencia a mano

`bilinker migrate --cut`. Verifica, regenera, mueve y registra —en ese orden— sobre las capas que se le den. Hacerlo a mano habría sido reproducir seis veces una secuencia que se puede equivocar una vez.

**Regenera siempre antes de mover.** La carpeta transitoria es un derivado, y regenerar es lo que recupera un `accept` hecho con el binario viejo entre la generación y el corte. La regla operativa del ADR —regenerar justo antes de cortar— está incorporada al comando, para que no dependa de que alguien se acuerde.

**Se planifican todas las capas antes de mover ninguna.** Si una no verifica, no se corta nada: un corte a medias deja el repo con dos formatos y ningún binario que entienda los dos.

**El ledger va al final.** Estaba escribiéndose al generar, que es lo que ADR-0003 prohíbe: el repo quedaría marcado como migrado mientras sigue corriendo el formato viejo. Se partió `accreta_migrate::run` en `generate` —sin ledger— y `record`, que es la mitad del corte.

## Y tiene vuelta

`bilinker migrate --rollback` restaura `.bilink/` desde el backup y quita la entrada del ledger. Un paso irreversible sin camino de vuelta obliga a deshacerlo a mano, que es justo lo que la migración evita en todos los demás pasos.

Se usó de verdad: el primer corte destapó un defecto, se deshizo, se arregló y se rehízo. Sin eso habría habido que revertir cuatro repos a mano.

## El defecto que el corte destapó

**118 de 118 bilinks en `CHAIN_DIRTY`** después del primer corte.

Un endpoint `path` aprueba **dos** valores de su vecino: su contenido y su ubicación. En el formato 1 esa segunda copia no existía —sólo se copiaba el hash— así que la migración la dejaba ausente, y cada endpoint `path` nacía desalineado contra su propio vecino.

La migración ahora resuelve el vecino y acuña su id de capture con la misma función que acuña el propio. Sigue sin consultar git ni resolver queries: es leer dos archivos más.

Los tests unitarios no lo habían atrapado porque ninguno tenía una cadena de dos capas. Ahora hay dos que sí.

## Resultado

| | |
|---|---|
| Capas migradas | 6, en 4 repos |
| Bilinks | 286 |
| Captures | 250 (36 colapsados por dedup) |
| `resolved_at` eliminados | 286 |
| Aceptaciones perdidas | **0** |

**El inventario de trabajo sobrevivió exacto**: 70 endpoints no-OK en la capa raíz y 4 en lattice, los mismos que antes del corte. Es lo que la Decisión 6 del ADR pide y lo que hace que la task `3` siga teniendo su lista.

Las capas de stratum quedaron en `all clean`, con las cadenas cruzando bien: cada endpoint `path` con los dos valores de su vecino.

## Lo que queda del formato 1

`.bilink-formato-1/` en cada capa, y `.git/info/exclude` con `.bilink-migrate-*` y `.bilink-formato-1`. Borrarlo es una decisión aparte, que se toma con el resultado a la vista — no en el mismo paso que lo reemplaza.

El crate `bilink-format-v1` se queda para siempre: el conjunto de formatos es de sólo-agregar, y es lo que permite que alguien parado en el formato 1 llegue al 2.
