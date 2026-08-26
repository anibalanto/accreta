# Especificación: lenguaje de paths Stratum

## Concepto

Un path Stratum es una expresión que describe un path en el filesystem, opcionalmente usando navegación por layers. Es un superconjunto de los paths relativos tradicionales: cualquier path relativo es un path Stratum válido — la biblioteca lo retorna sin modificación.

## Gramática

```
stratum-path := traditional-path
              | down-path
              | up-path
              | root-path

traditional-path := path relativo del filesystem (no comienza con '>' ni '<')
down-path        := ('>' layer-name)+ ('/' fs-path)?
up-path          := '<'+ ('/' fs-path)?
root-path        := '*' root-tail?
root-tail        := ('/' fs-segment) (('>' layer-name)+ ('/' fs-path)?)?
                  | ('>' layer-name)+ ('/' fs-path)?

layer-name := cualquier carácter excepto '>' y '/'
fs-path    := path tradicional del filesystem
fs-segment := componente de path sin '>' ni '<'
```

## Resolución

### Path tradicional (identidad)

Un path sin ningún token de navegación es retornado sin cambios.

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

### Prefijo + Down: `fs-path>name/...`

Un path puede empezar con un tramo de filesystem y seguir con navegación `>`. El tramo previo se toma tal cual y los segmentos `>name` se expanden desde ahí.

```
subsystems/bilinker>impl              →  subsystems/bilinker/.stratum/impl
subsystems/lattice>impl/src/main.rs   →  subsystems/lattice/.stratum/impl/src/main.rs
```

Es la forma que usan los tips de una cadena cuando se los nombra desde fuera de la capa: `bilinker chain new --tip 'subsystems/lattice>impl/editors/vscode/src/extension.ts:15:1'`.

Sin esto habría que estar parado en la capa para nombrarla, o escribir `.stratum/` a mano — que es justo lo que el lenguaje de paths evita.

### Up: `<</fs-path`

Cada `<` equivale a `../..` (un nivel Stratum = dos directorios). El `fs-path` opcional se concatena al final.

```
<                      →  ../..
<<                     →  ../../../..
</src/main.rs          →  ../../src/main.rs
<</docs/api.md         →  ../../../../docs/api.md
```

### Root: `*`

`*` resuelve al directorio raíz del proyecto, definido como el ancestro más lejano que contiene `.git` (outermost). Solo se usa para referenciar la capa de nivel más alto del proyecto. La resolución camina hacia arriba desde la layer actual hasta encontrarlo; si no existe, es un error.

Después de `*` pueden seguir un fs-path y/o tokens `>` de navegación hacia abajo.

```
*                                          →  <raíz del proyecto>
*/subsystems/stratum                       →  <raíz>/subsystems/stratum
*>impl                                     →  <raíz>/.stratum/impl
*/subsystems/stratum>impl/crates/cli/src   →  <raíz>/subsystems/stratum/.stratum/impl/crates/cli/src
```

### Up: semántica de root

Cada `<` sube una capa stratum (`../../`) y luego encuentra el root verdadero de esa capa buscando el ancestro más cercano con `.git` o `.bilink`. Esto garantiza que la navegación hacia arriba siempre aterriza en la raíz de la capa, no en un subdirectorio intermedio.

## Invariantes

1. Un path sin tokens de navegación (`>`, `<`, `*`) es válido y se retorna tal cual. Un `>` en cualquier posición inicia navegación hacia abajo, no solo al principio.
2. `<` (Up) y `>` (Down) no se mezclan. `*` (Root) puede ir seguido de tokens `>`.
3. El `fs-path` no contiene `>` ni `<`.
4. Cada `layer-name` es no vacío.
5. `*` solo puede aparecer al inicio de la expresión.

## Uso en herramientas

El path Stratum es el lenguaje de expresión de paths del ecosistema. Lo usan:

- **`stratum` CLI** — como argumento de navegación interactiva.
- **bilinker** — en endpoints layer de archivos `.bilink`.
- Cualquier herramienta del ecosistema que necesite referenciar paths dentro
  de un árbol de layers Stratum.
