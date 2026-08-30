---
title: Extraer el crate de formato
status: done
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-30T02:23:33Z
parent: r
---

# Extraer el crate de formato

Los tipos con `serde` y `schemars`, su serialización y el esquema generado, en un crate propio del que dependa todo lo demás.

Lo que no es mover archivos: **partir `capture.rs`**, que hoy junta el formato del `.capture` con el algoritmo de captura por tree-sitter — anclas, queries, offsets. Sólo la primera mitad va al crate de formato.

Las migraciones van en la misma carpeta, cada una declarando el par de versiones que puentea y dependiendo de las dos.

## Cómo quedó

```
bilink-format     los tipos y su serialización. No resuelve nada.
  └── bilinker    los interpreta: tree-sitter, git, estados
        ├── bilinker-cli
        └── bilinker-lsp
```

`link.rs` y `bilink.rs` se mudaron enteros. `capture.rs` se partió por su propio separador —`// ─── el archivo .capture ───`—: la mitad de abajo, que describe el archivo, es el crate de formato; el walk-up por el AST y la construcción de la query se quedaron, porque dependen de las gramáticas.

**Los dos módulos se re-exportan desde `bilinker`.** `bilinker::link` y `bilinker::bilink` siguen resolviendo, así que ningún consumidor tuvo que cambiar un `use`. La extracción es un refactor puro y se pudo verificar como tal: los 92 tests que había siguen pasando, repartidos entre los dos crates.

## La serialización

Los tipos llevan derives de `serde` y `schemars`. Los tres que en disco son un string —`LinkEndpoint`, `ByteRange`, y los enums de estado— se serializan como tal y no como objetos, que es lo que hace que el modelo y el archivo digan lo mismo:

```
link: capture 67ba7217-…      →  "capture 67ba7217-…"
offset: 3226~5109             →  "3226~5109"
state: CHAIN_DIRTY            →  "CHAIN_DIRTY"
```

Coincide con la forma que [ADR-0003](../../subsystems/bilinker/.stratum/impl/docs/adr/0003-formato-captures-y-aceptacion.md) fija para el YAML, así que cuando la serialización pase a serde el esquema ya describe el archivo literal sin reescribirse.

## Lo que no entró

**Las migraciones no se mudaron.** El enunciado las ponía en la misma carpeta, con cada una declarando el par de versiones que puentea. Eso exige que existan **dos** crates de formato, y hoy hay uno: `bilinker-001-capture-split` migra desde un formato que nunca fue un crate. Mudarla ahora sería mover un archivo sin poder darle la forma que la decisión pide.

Queda para el primer bump real de versión, que es el sprint 3. Ahí la migración `002` nace con las dos dependencias y la `001` se acomoda al lado.

## Un hueco que la extracción destapó

`const_item` no estaba en `stable_anchor_kinds` para Rust, así que **no se podía bilinkear una constante**: la query caía en `(visibility_modifier) @target`. En un crate de formato eso es justo lo que la spec describe —una tabla de prefijos, un registro de versiones—, así que las specs de este sprint no se podían cubrir.

Se agregaron `const_item` y `static_item`, en la spec (`commands/capture.md` § "Lenguajes soportados") y en `grammar.rs`. Lo detectó la verificación que puso la task `w`: antes habría escrito el capture mal anclado y nadie se enteraba.
