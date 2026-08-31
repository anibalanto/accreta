# Especificación: comando `stratum links`

## Propósito

Verifica que los links entre documentos del ecosistema **lleguen a algún lado**.

Un link roto en markdown no falla: simplemente no lleva a ningún lado. Y eso ya pasó a escala — **31 links roto** convivieron con el proyecto sin que nadie se enterara, 27 de ellos por la misma causa: contar `../` a mano desde un archivo que está a dos niveles de la raíz.

> **No es un descuido, es la forma de la referencia.** Un `../../` no dice nada sobre el destino: dice dónde está parado el que escribe.

## Firma

```
stratum links check [<path>] [--anchors]
```

| Argumento | Descripción |
|---|---|
| `<path>` | Qué recorrer. Default: la raíz de la capa actual, recursivo. |
| `--anchors` | Verifica también que el `#heading` exista. |

## Por qué es de stratum

Los paths son suyos. Recorrer markdown no lo es —eso es de un proveedor de lattice— pero **lo que se verifica es una resolución de path**, y ahí es donde vive el conocimiento: qué es una capa, dónde termina, cómo se resuelve un token.

Que el comando exista acá es lo que permite que mañana resuelva un [metalink](../concepts/paths.md) sin que nadie más aprenda a hacerlo.

## No duplica a `lattice graph --via doclink`

Es la objeción obvia: el proveedor `doc` de lattice ya emite `doclink` con `broken`, así que *"qué links están muertos"* ya tiene respuesta. **Son dos preguntas distintas, y se nota en qué hacen con la respuesta:**

| | Qué pregunta | Qué hace con un link muerto |
|---|---|---|
| `lattice graph --via doclink` | qué hay en el grafo | lo **describe** — *"un link muerto en un documento es información, no un error de lattice"* |
| `stratum links check` | ¿esto está sano? | **falla** |

Un verificador de CI necesita fallar, y lattice tiene decidido no hacerlo. Además el grafo no mira anchors: su nodo destino es el **archivo completo**, así que `#un-commit-hace-una-cosa` le es invisible — y los anchors fueron 4 de los 31.

Y hay una diferencia de costo que importa para CI: esto no necesita el grafo indexado, sólo los archivos.

## Qué no cuenta como link

**Las mismas exclusiones que el proveedor `doc`**, y por el mismo motivo — [`provider.md`](../../lattice/concepts/provider.md) § "Qué no cuenta como link":

| Construcción | Por qué no |
|---|---|
| links dentro de bloques de código | un ejemplo no es una referencia |
| imágenes `![alt](x.png)` | un embed no es una referencia a otro documento |

**Que las dos herramientas compartan exclusiones no es prolijidad, es correctitud**: si difirieran, un link sería roto para una y sano para la otra, y la contradicción no tendría dónde resolverse.

La regla se validó sola. Al escribir esta spec, el verificador corrió sobre el árbol y sus **cinco** falsos positivos restantes eran todos ejemplos citados en prosa — cuatro de ellos, los ejemplos que `provider.md` usa para *explicar esta misma regla*. La regla, que se había escrito preventivamente sobre cero casos observados, se probó a sí misma el día que hubo con qué.

## Anchors

`#un-commit-hace-una-cosa` es la otra mitad de los links que nadie chequea, y verificarlo cuesta lo mismo que verificar el archivo: los headings del destino ya están leídos.

El slug se calcula como lo hace un renderer de markdown: minúsculas, se sacan los signos, los espacios pasan a guiones. **Los backticks se sacan sin dejar guion** — de ahí salieron 3 de los 4 anchors roto, que escribían `#bilinkhead--de-dónde` con dos guiones donde `` `.bilink/head` — de dónde `` produce uno.

## Salida

```
$ stratum links check --anchors
577 links internos · 31 roto

.stratum/worklist/g.task.md
    [no existe] ../subsystems/bilinker/commands/init.md
    [no existe] ../subsystems/bilinker/concepts/cache.md
subsystems/bilinker/concepts/cache.md
    [anchor]    ref.md#bilinkhead--de-dónde-salió-el-árbol
```

## Código de salida

| Código | Condición |
|---|---|
| 0 | Ningún link roto. |
| 1 | Al menos uno. |

## Invariantes

- **No escribe nada.** Es un verificador.
- Las exclusiones son las mismas que las del proveedor `doc` de lattice.
- Un link a un anchor se verifica sólo con `--anchors`; sin la flag se verifica el archivo y nada más.

## Lo que no entra

**El metalink** —`stratum:*/subsystems/bilinker/concepts/cache.md`, el destino escrito con tokens Stratum— es lo que arregla la causa, y este comando sólo detecta el síntoma. Está en el ítem `1i` del worklist, con el argumento de por qué **no puede salir sin que el proveedor `doc` de lattice aprenda el esquema en el mismo cambio**: sin eso, cada metalink se clasificaría como `external`, que no se espera que resuelva, y el grafo se vería sano.
