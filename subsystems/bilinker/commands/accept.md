# Comando: `bilinker accept`

Registra el estado actual de un endpoint como aprobado, escribiendo el bloque `accepted` del bilink.

Es el único comando que escribe una decisión. Ver [la aceptación](../concepts/accept.md) para el modelo.

## Uso

```
bilinker accept <uuid>.<N>
bilinker accept <uuid>.<N> --place
bilinker accept <uuid>.<N> --content
bilinker accept .
bilinker accept <path>
```

| Argumento | Descripción |
|-----------|-------------|
| `<uuid>.<N>` | Endpoint a aceptar: UUID del bilink + índice (0 o 1). |
| `--place` | Aprueba sólo la ubicación: escribe `accepted.link` y deja `accepted.hash` como estaba. |
| `--content` | Aprueba sólo el contenido: escribe `accepted.hash` y `accepted.hash_ast`. |
| `.` o `<path>` | Acepta en bulk todo lo que necesita atención en la capa actual (o bajo el path dado). |

Sin flags, aprueba las dos dimensiones.

## Comportamiento

1. Resolver el bilink y, para endpoints estructurales, el capture que su `link` referencia.
2. Si el capture no resuelve, **fallar**: no se puede aprobar contenido que no se pudo localizar.
3. Si el fragmento no está commiteado, **fallar** (ver más abajo).
4. Calcular el hash del fragmento actual y su `hash_ast` si hay gramática.
5. Escribir `accepted` en el endpoint: `link` con el id del capture vigente, `hash` y `hash_ast`.
6. Calcular el `commit` del contenido y escribirlo en [la cache](../concepts/cache.md).
7. Cerrar el acto con **un commit sobre [`refs/bilink/<branch>`](../concepts/ref.md)**, absorbiendo en él el commit del proyecto contra el que se aceptó si no lo estaba ya.

El bilink cambia, y eso dispara `CHAIN_DIRTY` en el nodo adyacente en el próximo `check`. Es la única forma de mover una cadena: `check` no propaga nada porque no escribe ninguna decisión.

## Exige el fragmento commiteado

`accept` falla sobre un archivo sucio:

```
$ bilinker accept 7f3d8e9a.0
error: crates/bilinker/src/check.rs tiene cambios sin commitear.
       Aceptar fija un contenido, y ese contenido tiene que existir en la historia.
```

No es una recomendación. `commit` es el commit en que el fragmento quedó con el contenido aprobado, y ese commit **no existe** si el fragmento no está commiteado. Sin él no hay `git show <commit>:<file>`, y sin eso `check` no puede recuperar el texto aceptado — que es lo que distingue `EXPANDED` y `REANCHORED` de un `ALTERED` genérico.

## Un commit por invocación, no por aceptación

**La granularidad sigue al acto, no al objeto.** `accept <uuid>.0` da un commit; `accept .` da un commit, con el mensaje enumerando los endpoints — porque es una persona mirando y decidiendo **una vez**, y partirlo en N commits firmados no agrega verdad: sugiere N decisiones donde hubo una.

Ese commit es el registro de la decisión, y el artefacto que se puede firmar. Ver [la ref](../concepts/ref.md#autoría-atestación-y-autorización): la superficie de revisión es la de bilinker, no la de la forja.

**Absorber no es un paso de `accept`, es una precondición de escribir sobre la ref.** Cuando el proyecto no se movió desde la última absorción el commit tiene un solo padre; cuando sí, el mismo commit lo absorbe. No hay un `sync` implícito adentro: hay una condición que se verifica.

Y en un repo que todavía no cortó a la ref no hay commit: los bilinks viven en la rama del proyecto, y commitearlos es de quien trabaja. Ver [la ref](../concepts/ref.md#antes-del-corte-no-hay-ref-y-el-commit-no-ocurre).

## `commit` es del contenido, no de quien acepta

`commit` es el commit en que el fragmento quedó con el contenido aprobado, y **no** el HEAD del repo al momento de aceptar.

Se calcula con `git log -L <start>,<end>:<file>` —nativo y offline— o con un walk acotado hacia atrás resolviendo la query hasta que el hash cambie. Se calcula acá, una vez, porque acá está todo el contexto a mano; y se escribe en la cache, porque es derivable de `(accepted.link, accepted.hash)`.

Con el HEAD, el mismo acto daba distinto según quién y cuándo lo hiciera, y el valor no describía nada del fragmento. Con el commit del contenido, **aceptar es determinista**: dos personas que aprueban el mismo fragmento en el mismo estado escriben lo mismo.

Y la ventana de arqueología queda bien sin ajustes: `commit..HEAD` excluye `commit`, así que es exactamente "lo que pasó después de que el contenido aprobado quedó establecido".

## Ejemplos

```bash
# Aprobar ubicación y contenido
bilinker accept 7f3d8e9a-1b2c-4d5e-8f6a-7b8c9d0e1f2a.1

# El archivo se renombró y el código no cambió: aprobar sólo la mudanza
bilinker accept 7f3d8e9a.1 --place

# Aceptar todo lo que necesita atención en la capa actual
bilinker accept .
```

## Salida

```
accepted: 7f3d8e9a.1
  link:   capture 67ba7217…
  hash:   479922a1…
  commit: d4e5f6a7…   (el contenido quedó así en este commit)

note: el nodo adyacente detectará CHAIN_DIRTY en el próximo check
```

## Cuándo usarlo

- Tras `check`, sobre `PENDING`, `ALTERED`, `RESTYLED` o `CHAIN_DIRTY`, cuando el cambio es coherente con la intención del bilink.
- Tras `apply`, **siempre**: `apply` repunta la ubicación y la deja en `RELOCATED`. Ningún fix cierra solo.

## Lo que no es

`accept .` existe para el caso en que ya se revisó todo, no para el caso en que no se revisó nada. Cada estado no-OK es un puntero al fragmento que hay que mirar, y el inventario de trabajo de un cambio *es* esa lista: vaciarla para dejar el árbol verde tira justamente lo que hacía falta.

## Exit codes

- `0`: aceptación registrada.
- `1`: UUID no encontrado, endpoint inválido, capture sin resolver, o fragmento sin commitear.
