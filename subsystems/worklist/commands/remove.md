# Comando: `worklist remove`

Saca un ítem del worklist cuando **no debería haberse creado**.

```
worklist remove <id> [--force] [--remove-bilink]
```

## No es lo mismo que descartarlo

Son dos cosas parecidas y la diferencia importa, porque una de las dos destruye el registro de una decisión:

| | Cuándo | Qué queda |
|---|---|---|
| [`state change <id> dropped`](state-change.md) | **se decidió no hacerlo** | el archivo, con su porqué |
| `remove <id>` | **no tendría que existir** — se creó por error, o duplicado | nada en el árbol |

> **El valor de una decisión revertida es el registro de por qué se descartó**, que es lo que evita volver a proponer lo mismo dentro de un mes. Eso es `dropped`, y no se borra.

Se borra cuando ese porqué **ya está absorbido** en el ítem que lo reemplazó — ahí el archivo no aporta nada que no esté escrito en otro lado, y `remove` es lo correcto.

## Es recuperable, y no por una funcionalidad

> **Deshacer un `remove` es un `revert` de git.** No hay comando de restauración, y no hace falta uno.

El archivo vuelve con su historia entera porque nunca se fue de la historia: lo que el `remove` hace es un commit que lo saca del árbol. Es la misma propiedad que tiene todo lo demás en este sistema, y decirla acá evita que alguien construya una papelera.

## Comportamiento

1. Resuelve el ítem por id.
2. Si tiene hijos, pide confirmación explícita — o `--force`.
3. Saca el archivo del árbol.
4. Si hay un bilink que lo nombra en un endpoint `issue`, **no lo toca** salvo `--remove-bilink`: el bilink puede seguir siendo útil aunque el ítem ya no aplique.

**No commitea**, igual que `new` y `state change`.

## Y del otro lado el issue no se borra: se cierra

> **`remove` propone la transición a `dropped`, no un borrado.**

Es una decisión, y va contra la lectura ingenua —*"lo borré acá, que se borre allá"*—, por dos motivos que se suman:

**El issue puede tener trabajo registrado que no es del worklist.** Comentarios, worklogs, adjuntos, links desde otros issues: nada de eso lo escribió este sistema, y borrar el issue lo destruye sin que nadie lo haya pedido. Un worklist no es dueño del issue: es dueño de su lado.

**Y el proveedor puede negarse igual.** Borrar issues suele estar restringido a un rol de administración, así que un `remove` que prometiera borrar fallaría en la mitad de las instalaciones — y fallaría **después** de haber sacado el archivo de este lado, que es el peor momento.

Cerrarlo como `dropped` es lo que las dos cosas permiten: es la misma transición que ya existe, pasa por el mismo mapeo, y se rechaza por regla como cualquier otra si el workflow no la admite.

**Y el servidor no necesita que se lo digan.** Un archivo que el push **borra** es una transición a `dropped`, y eso se lee del diff igual que todo lo demás: el nombre del archivo borrado da la clave, y que ya no esté en el árbol nuevo da la intención. No hace falta un campo, ni una lápida, ni un commit especial.

## Flags

| Flag | Descripción |
|------|-------------|
| `--force` | saca el ítem sin pedir confirmación aunque tenga hijos |
| `--remove-bilink` | borra también el bilink que lo nombraba |

**Los hijos no se borran en cascada.** Un ítem sin padre es válido —la jerarquía es un campo, no una contención— así que quedan colgando de la raíz, que es un estado que el formato admite. Borrar en cascada convertiría un error de un ítem en la pérdida de varios.

## Salida

```
$ worklist remove ACC-347

removed:  ACC-347  Implementar el método vote
provider: ACC-347  -> dropped   (propuesto)
bilink:   b2c3d4e5 conservado   (--remove-bilink para borrarlo)

hint: se deshace con git revert
```

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | ítem sacado del árbol |
| `1` | id no encontrado, o tiene hijos y no vino `--force` |
| `2` | error de escritura |
