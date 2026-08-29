---
title: Ejecutar el corte de formato (`004`) en los 4 repos
status: open
created_at: 2026-08-29T00:00:00Z
updated_at: 2026-08-29T00:00:00Z
---

# Ejecutar el corte de formato (`004`) en los 4 repos

El primero de los dos cortes. Deja vivo el formato nuevo **sin tocar dónde viven los bilinks**: siguen en las ramas, con `git status` y `git diff` funcionando normal.

```
1. Generar (o regenerar) .bilink-migrate-003…/ en todas las capas   [sin git]
2. Reemplazar .bilink/ por esa carpeta en el árbol
3. UN commit en la rama del proyecto con el .bilink/ nuevo
4. Ledger: 002, 003, 004
```

Cuatro veces: raíz `accreta` y los tres `.stratum/impl`. `--recursive` desde la raíz no alcanza para los otros tres — están gitignoreados por sus padres y son repos independientes.

Escribe también `.bilink/version` con la versión de formato, que es lo que impide que un binario viejo lea estos archivos y los vacíe en silencio.

## La verificación, que son tres cosas y no una

No alcanza con que la migración corra. Antes de cortar:

1. **Todo parsea** bajo los tipos de la versión destino — los 158 bilinks y 129 captures de los cuatro repos, sin campo desconocido y sin aridad distinta de dos.
2. **No se perdió nada** — y lo verifica **la migración**, no `check`: es el único componente que depende de los dos crates de formato, porque el binario linkea un solo parser y no lee el formato viejo. La afirmación es acotada por definición — `resolved_at` se descarta y `commit.N`/`state.N` se mudan a la cache—, así que lo que se asegura es que todo endpoint aceptado antes siga aceptado con los valores equivalentes. Parsear bien no alcanza: una migración que omite un campo produce un archivo válido.
3. **La serialización es estable** — `parse → serialize → parse` byte-idéntico. Es lo que garantiza que ningún id de capture se mueva, porque es `H(file, query, offset)`.

Escenarios: `migrated-corpus-parses`, `migration-loses-nothing`, `serialization-is-stable`.

**Y no es un `cargo test`.** El corpus vive en cuatro repos, tres gitignoreados por sus padres; la suite de bilinker corre en el de impl y no los alcanza. Es un comando que toma un path y corre sobre el árbol real, como paso del corte.

**Es la puerta de la US siguiente.** Hasta que acá no pasen esas tres, más la suite y las aceptaciones, no se toca la ref.
