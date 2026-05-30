# Especificación: lenguaje de paths Stratum

## Concepto

Un path Stratum es una expresión que describe un path en el filesystem, opcionalmente usando navegación por layers. Es un superconjunto de los paths relativos tradicionales: cualquier path relativo es un path Stratum válido — la biblioteca lo retorna sin modificación.

## Gramática

```
stratum-path := traditional-path
              | down-path
              | up-path

traditional-path := path relativo del filesystem (no comienza con '>' ni '<')
down-path        := ('>' layer-name)+ ('/' fs-path)?
up-path          := '<'+ ('/' fs-path)?

layer-name := cualquier carácter excepto '>' y '/'
fs-path    := path tradicional del filesystem
```

## Resolución

### Path tradicional (identidad)

Un path que no comienza con `>` ni `<` es retornado sin cambios.

```
.stratum/tech-decisions      →  .stratum/tech-decisions
../..                        →  ../..
src/main.rs                  →  src/main.rs
```

### Down: `>name1>name2/.../fs-path`

Cada segmento `>name` se expande a `.stratum/<name>`. Los segmentos se encadenan. El `fs-path` opcional se concatena al final.

```
>tech-decisions                    →  .stratum/tech-decisions
>tech-decisions>impl               →  .stratum/tech-decisions/.stratum/impl
>tech-decisions>impl/src/main.rs   →  .stratum/tech-decisions/.stratum/impl/src/main.rs
>tasks/pending.md                  →  .stratum/tasks/pending.md
```

### Up: `<</fs-path`

Cada `<` equivale a `../..` (un nivel Stratum = dos directorios). El `fs-path` opcional se concatena al final.

```
<                      →  ../..
<<                     →  ../../../..
</src/main.rs          →  ../../src/main.rs
<</docs/api.md         →  ../../../../docs/api.md
```

## Invariantes

1. Un path que no comienza con `>` ni `<` es válido y se retorna tal cual.
2. Los `<` y los `>` no se mezclan en una misma expresión (fase futura).
3. El `fs-path` no contiene `>` ni `<`.
4. Cada `layer-name` es no vacío.

## Uso en herramientas

El path Stratum es el lenguaje de expresión de paths del ecosistema. Lo usan:

- **`stratum` CLI** — como argumento de navegación interactiva.
- **bilinker** — en endpoints layer de archivos `.bilink`.
- Cualquier herramienta del ecosistema que necesite referenciar paths dentro
  de un árbol de layers Stratum.
