# Especificación: comando `bilinker verify-ref`

Verifica que una `refs/bilink/*` tenga la forma que [la ref](../concepts/ref.md) promete. Es la misma verificación en dos lugares que no se parecen: **del lado del servidor**, donde puede rechazar un push, y **del lado del que recibe una ref ajena**, donde puede avisar antes de calcular drift contra un árbol fabricado.

> **No mira si los bilinks están en `OK`.** Una ref con drift es normal — es el estado que el método existe para reportar— y exigir `OK` haría imposible [`track`](track.md). Lo que se verifica es la **forma**, nunca el contenido de una decisión.

## Firma

```
bilinker verify-ref [<rango>] [--signers <archivo>] [--stdin]
```

| Argumento | Descripción |
|---|---|
| `<rango>` | `<viejo>..<nuevo>`, o un nombre de ref —y entonces son sus commits propios, del corte para acá. Sin argumento, la ref de la rama actual. |
| `--signers` | El archivo de firmantes autorizados. Sin él, la firma no se verifica y **se dice**. |
| `--stdin` | Lee `<viejo> <nuevo> <ref>` por línea: el protocolo de un `pre-receive`. |

## Qué verifica, y con qué

**Nada de esto necesita tree-sitter ni resolver una query.** Son comparaciones de tree oids, parseo de YAML, y hashes — que es lo que permite correrlo en un servidor que no adoptó bilinker.

### Del rango

| | Por qué |
|---|---|
| la ref no se borra | sin esto, *"sólo avanza"* se esquiva borrándola y empujándola de nuevo |
| el tip viejo es ancestro del nuevo | la ref es append-only; un no-fast-forward es una reescritura |

**Los dos los verifica este comando y nadie más.** `receive.denyNonFastForwards` y `receive.denyDeletes` son las dos opciones que uno pondría y no sirven acá: git las chequea sólo sobre `refs/heads/`, así que sobre `refs/bilink/*` un delete y un force-push pasan. Ver [la ref](../concepts/ref.md#la-config-del-servidor-no-alcanza-y-no-es-una-omisión-no-aplica).

### De cada commit

| | Con qué |
|---|---|
| el mensaje parsea contra [la gramática](../concepts/ref.md#el-mensaje-es-el-comando) | el vocabulario cerrado |
| cae en uno de [los tres tipos](../concepts/ref.md#un-commit-hace-una-cosa), y no en dos | los padres, y cuál de los dos árboles se movió |
| [disyunción](../concepts/ref.md#las-dos-verificaciones-previas): lo absorbido no trae `.bilink/` | `ls-tree` del segundo padre |
| [fidelidad](../concepts/ref.md#la-invariante-de-fidelidad): el árbol de código es el del absorbido | `diff-tree` |
| cada archivo que toca valida contra el formato | el esquema, con campos desconocidos rechazados |
| el nombre de un capture **es** `sha256(file \0 query \0)[..32]` | `Capture::id()` |
| un capture sólo se **agrega**, nunca se modifica ni se borra | el diff del commit |
| el formato declarado no es más nuevo que el conocido | `.bilink/version` |
| **a `agree` sólo se agrega el autor del commit** | el diff de los dos `accepted` |
| está firmado por una clave de la allowlist | `git verify-commit` |

## `agree` sólo se agrega a sí mismo

Es la fila que convierte [`agree`](../concepts/accept.md#quiénes-aprobaron) de atribución en atestación, y la que hace que no haga falta ningún mapeo de nombres a claves.

**El problema:** `agree: [ana]` es texto, y cualquiera puede escribirlo sin que Ana se entere.

**La cadena que lo cierra**, en dos pasos que ya existen:

1. La **firma** ata el commit a una clave de la allowlist, y con ella al autor que el commit declara.
2. El **hook** exige que los nombres que ese commit agregó a algún `agree` sean exactamente el autor del commit.

Con las dos, `- ana` sólo puede haberlo escrito un commit firmado cuya autora es Ana. **Nadie aprueba en nombre de otro**, y no hace falta traducir un nombre a una clave: los dos extremos ya se comparan contra el mismo campo, el autor.

Sacar un nombre está permitido y no necesita ser el propio: es lo que hace [`adopt`](adopt.md) al traer valores distintos, y lo que hace un `accept` cuando los valores cambian y la lista se vacía. **Lo que se protege es agregar**, que es lo único que afirma algo sobre otra persona.

## La gramática no se aplica hacia atrás, y tampoco se puede volver

Un commit **sin** `Bilinker-Version` es [anterior a la gramática](../concepts/ref.md#la-gramática-no-es-retroactiva), y su forma no se verifica: la ref es append-only, así que exigírsela rechazaría el primer push de cualquier repo que haya cortado antes de que la gramática existiera.

Pero eso deja una puerta, y se cierra con una regla de orden:

> **Una vez que un commit de la ref lleva el trailer, ninguno de sus descendientes puede no llevarlo.**

Es barato —un recorrido del rango, oldest-first— y deja el prefijo viejo pasar exactamente una vez, que es lo que se quiere. Un commit sin trailer empujado encima de uno que lo tiene no es historia vieja: es alguien esquivando la verificación.

De la misma regla sale que la **firma** tampoco se le exige al prefijo. Un repo que cortó antes no tiene por qué haber firmado, y pedírselo retroactivamente es lo mismo que pedirle la gramática.

## Del lado del servidor

### El `pre-receive` es la única capa

No hay una capa de config debajo. Las dos opciones que uno pondría —`receive.denyNonFastForwards` y `receive.denyDeletes`— **no aplican fuera de `refs/heads/`**, que es exactamente donde esta ref vive. Ponerlas no hace daño y no hace nada; lo que protege es el hook.

```sh
#!/bin/sh
exec bilinker verify-ref --stdin --signers /etc/bilinker/allowed-signers
```

El hook recibe `<viejo> <nuevo> <ref>` por línea y sale distinto de cero para rechazar el push entero. **Las refs que no son `refs/bilink/*` se ignoran**: este hook no opina sobre las ramas del proyecto.

Un servidor que no quiera instalar el binario puede implementar las mismas filas desde [el esquema publicado](../concepts/format-version.md): ninguna necesita nada que el esquema no describa. Lo que se ofrece acá es la implementación ya hecha, no la única posible.

### El replay en CI

Que `accepted.hash` sea de verdad el hash del fragmento **sí** necesita resolver la query, así que no es de acá. Está en [`proposals/verificar-ref-ajena.md`](../proposals/verificar-ref-ajena.md) § "El replay en CI".

## Los firmantes

El archivo es [el formato de `allowed_signers` de ssh](https://man.openbsd.org/ssh-keygen#ALLOWED_SIGNERS), que es el que git ya consume por `gpg.ssh.allowedSignersFile`:

```
ana@example.com ssh-ed25519 AAAAC3Nza...
pablo@example.com ssh-ed25519 AAAAC3Nza...
```

**No se inventa un formato**, y por dos razones: git ya sabe leerlo, y una allowlist propia sería una tercera lista de personas en un proyecto que ya tiene dos.

**Vive en el servidor, no en la ref.** Una allowlist versionada la edita quien pushea, que es exactamente quien no debería poder ampliarla.

**Y bilinker firma lo que escribe, si el repo está configurado para firmar.** No con una opción propia: con `commit.gpgsign`, la misma con la que se firma cualquier otro commit. Hizo falta decirlo porque `git commit-tree` —con el que se arma todo commit de la ref— **no la mira**, a diferencia de `git commit`: sin pasarle `-S` los commits salen sin firmar, y la allowlist se quedaría sin nada que verificar.

Sin `--signers`, la verificación de firma no corre y la salida lo dice: *"sin allowlist: la firma no se verificó"*. Es la diferencia entre "verifiqué y está bien" y "no verifiqué", y confundirlas sería el peor resultado posible.

## Salida

```
$ bilinker verify-ref refs/bilink/main
refs/bilink/main  47 commit(s)

  ✓  38  con la gramática, firmados
  ·   9  anteriores a la gramática — forma no verificada

ok
```

Y cuando algo no da:

```
$ bilinker verify-ref refs/bilink/main --signers ./allowed-signers
refs/bilink/main  47 commit(s)

  ✗  3f8b41c  absorbe y decide a la vez: el diff de .bilink/ de una absorción es vacío
  ✗  9c1f0ab  agrega `- ana` a 7f3d8e9a.0 y el autor es pablo
  ✗  b1e3f55  sin firma de la allowlist

3 de 47 rechazados
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todo lo verificable verifica. |
| 1 | Algún commit no cumple. |
| 2 | El rango no se puede leer — la ref no existe, el commit no está. |

## Invariantes

- `verify-ref` **no escribe nada**, ni siquiera cache. Es lo único que un hook puede correr sin efectos.
- No resuelve ninguna query ni carga ninguna gramática.
- Un commit anterior a la gramática no se rechaza por su forma, y ninguno de sus descendientes puede serlo.
- Sin allowlist, la firma no se verifica **y se dice**: nunca se reporta como verificada.
