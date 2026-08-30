---
title: Extraer el crate de formato
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
parent: r
---

# Extraer el crate de formato

Los tipos con `serde` y `schemars`, su serialización y el esquema generado, en un crate propio del que dependa todo lo demás.

Lo que no es mover archivos: **partir `capture.rs`**, que hoy junta el formato del `.capture` con el algoritmo de captura por tree-sitter — anclas, queries, offsets. Sólo la primera mitad va al crate de formato.

Las migraciones van en la misma carpeta, cada una declarando el par de versiones que puentea y dependiendo de las dos.
