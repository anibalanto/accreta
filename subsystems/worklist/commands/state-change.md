# Comando: `worklist state change`

Cambia el estado de un ítem. **Escribe una propuesta, no un hecho.**

```
worklist state change <id> <estado>
```

```bash
worklist state change ACC-347 done
worklist state change ACC-347 dropped
```

`worklist start` y `worklist done` son las dos transiciones frecuentes escritas corto, y no otra cosa: `start <id>` es `state change <id> in-progress`, y `done <id>` es `state change <id> done`.

## Es del cliente, y por eso no pregunta

> **El cliente no habla con el proveedor.** `state change` escribe el `status` en el archivo y termina ahí; quién decide si esa transición es legal es el proveedor, y quien le pregunta es el servidor cuando el push llega.

No es una limitación que haya que rodear — es el mismo reparto que todo lo demás: el cliente propone escribiendo, el servidor dispone hablando. Ver [`distribution.md`](../concepts/distribution.md) y [`states.md`](../concepts/states.md).

De ahí sale lo único que puede sorprender:

```
$ worklist state change ACC-347 done
ACC-347  in-progress -> done   (propuesto)

$ git push
remote: rechazado ACC-347: "In Progress" -> "Done" no es una transición de este workflow
remote:   disponibles: "Ready for Review"
```

**El comando salió con `0` y la transición no ocurrió.** Es correcto y hay que decirlo así: lo que el comando promete es haber escrito la propuesta, no haberla consumado.

## Comportamiento

1. Resuelve el ítem por id.
2. Verifica que `<estado>` esté en el vocabulario de [`.metadata/states.yaml`](../concepts/states.md). Si no está, falla **sin escribir**.
3. Escribe `status` y `updated_at` en el frontmatter.
4. Imprime la transición propuesta.

**No commitea**, por lo mismo que [`new`](new.md): el cambio de estado entra al repo con el resto del trabajo, y un ítem que pasa a `in-progress` viaja en el mismo commit que lo que se empezó a hacer.

**Y no valida la transición**, sólo el estado destino. Que `open -> done` sea legal es del workflow; que `done` exista es del vocabulario, y eso sí se puede contestar sin salir.

## Lo que pasa del otro lado, y es nuevo

Hasta ahora el `status` **no viajaba**: era el único campo del frontmatter que el servidor no subía, precisamente porque no era comparable con el del proveedor. Con el mapeo puesto, viaja — y viaja **como una transición**, no como una escritura de campo.

| | |
|---|---|
| el título y el cuerpo | se **escriben**: el valor nuevo pisa al viejo |
| el `status` | se **transiciona**: se le pide al proveedor que lo mueva, y puede negarse |

Es la diferencia que obliga a que exista este comando en vez de dejar que el `status` viaje con el resto: **una escritura no se rechaza por regla, una transición sí.**

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | la propuesta quedó escrita |
| `1` | id no encontrado, o estado que el vocabulario no declara |
| `2` | error de escritura |

**Un rechazo del proveedor no aparece acá**: llega en el push, que es donde ocurre. Ver [`states.md`](../concepts/states.md) § "Y hay dos formas de rechazo, que no se pueden confundir".
