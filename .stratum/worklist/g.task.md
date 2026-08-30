---
title: Especificar e implementar `init`, `sync`, `track` y `adopt`
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T18:37:58Z
parent: f
---

# Especificar e implementar `init`, `sync`, `track` y `adopt`

Los cuatro comandos no tienen spec: `commands/init.md`, `sync.md`, `track.md` y `adopt.md` se crean acá, y recién después se implementan. Más `log` y `diff` sobre la ref propia, `.bilink/version` escrita por `init`, y `.bilink/head` con su guarda, y la materialización automática al cambiar de rama.

> El cambio de una línea en `task.rs` que esta task llevaba —`format!("{task_id}.task")` a `.task.md`— ya no aplica: la task `u` lo resolvió de otra forma, buscando el ítem por prefijo en vez de fijar la extensión, y el archivo se llama `issue.rs` desde la task `z`.

## Lo que quedó hecho

Los cuatro comandos, con spec y escenarios: [`init`](../subsystems/bilinker/commands/init.md), [`sync`](../subsystems/bilinker/commands/sync.md), [`track`](../subsystems/bilinker/commands/track.md) y [`adopt`](../subsystems/bilinker/commands/adopt.md). Más `log`, `diff` y `status --porcelain` sobre el índice y la ref propios —la superficie de revisión, que pasa a ser parte del producto porque la forja no muestra la ref—, `.bilink/head` con su guarda, y el preludio que corre antes de todo comando y materializa el `.bilink/` de la rama actual sin ceremonia.

`accept` y `apply` cierran su acto con un commit sobre la ref, absorbiendo en el mismo commit el commit del proyecto contra el que se calcularon.

## Lo que se descubrió escribiendo el código

**`.bilink/version` no la escribe `init`.** Está versionada, así que viaja en el árbol de la ref y la materialización la trae con todo lo demás. Que `init` la calculara la volvería lo único del directorio que no sale del commit, y por lo tanto lo único que puede discrepar de los archivos que describe.

**La cache no se estaba invalidando al cambiar de rama**, y lo que eso cuesta no es un reporte equivocado: `accept` le cree a la cache, así que una cache de otra rama contestando `OK` deja el `accepted` viejo en su lugar sin error y sin línea en ningún reporte. El commit que la cache anota sale de `head`, que es un hecho sobre el árbol. Ver [`concepts/cache.md`](../subsystems/bilinker/concepts/cache.md) § "Sabe a qué rama corresponde".

**El interruptor del commit sobre la ref es la ref misma.** En un repo que todavía no cortó, `accept` y `apply` no commitean nada y los bilinks se ven con git como siempre. No hace falta un flag ni una versión de formato — y es lo que permite que el binario nuevo corra sobre los repos que todavía no cortaron, que es exactamente lo que la transición necesita.

**Exigir `init` va por el ledger, no por el filesystem.** El ledger está commiteado, así que un clon fresco de un repo que cortó lo sabe antes de tener una sola `refs/bilink/*` local — que es justo el caso en el que hay que exigirlo. Mirar si hay refs daría la respuesta contraria ahí.

**Y el preludio no puede exigir git.** La raíz cae al cwd cuando no hay marcador, que es lo que permite usar bilinker en un proyecto nuevo sin ningún paso de inicialización.

## Lo que queda para el corte

`init-is-required-and-explicit` sólo se puede ejercer de verdad después del `005`: hoy el gate del ledger lo deja pasar, correctamente, porque ningún repo cortó todavía.
