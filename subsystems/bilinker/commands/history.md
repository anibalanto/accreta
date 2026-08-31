# Especificación: comando `bilinker history`

## Propósito

Contesta *"qué le pasó a este bilink"*: quién aceptó qué, cuándo, contra qué código, y por qué ubicaciones fue pasando.

Los demás comandos —[`get`](get.md), [`status`](status.md), [`chain`](chain.md)— miran el presente. Éste mira [la ref](../concepts/ref.md), que es donde vive el registro de decisiones.

> **No persiste nada nuevo: arma una vista.** Todos los datos ya están en la ref, y esta task los junta.

## Firma

```
bilinker history <uuid>[.<N>] [--format json]
```

| Argumento | Descripción |
|---|---|
| `<uuid>` | Todos los actos sobre ese bilink. Acepta un prefijo. |
| `<uuid>.<N>` | Filtra a un endpoint. |
| `--format json` | Para un consumidor que no es una persona, que es el principal. |

## De dónde sale cada dato

Un solo query da la historia:

```
git log --first-parent refs/bilink/<rama> -- .bilink/<uuid>.yaml
```

Cada commit ahí es un acto sobre ese bilink. Lo demás sale del DAG y del diff:

| | De dónde |
|---|---|
| commit de la ref, autor, fecha | el commit |
| tipo — absorción · decisión · sincronización · corte | [la taxonomía](../concepts/ref.md#un-commit-hace-una-cosa): los padres y qué árbol se movió |
| el comando canónico | [el mensaje](../concepts/ref.md#el-mensaje-es-el-comando) |
| el commit del proyecto contra el que se calculó | la absorción más cercana, su 2º padre |
| qué cambió: el endpoint, y el antes/después de `link`, `hash`, `hash_ast`, `agree` | el diff del YAML |
| para un cambio de `link`: los dos captures con su `{file, query}` | los blobs de **ese** commit |

**Los datos son de git, no del mensaje.** El comando canónico le da a cada acto su nombre sin heurísticas, pero todo lo demás es derivable sin él — y por eso la vista sirve sobre la historia que ya existe.

## La historia de un capture es la secuencia de `link`

Un capture es **inmutable**: su id es el hash de su ubicación, así que no tiene historia propia. La que hay es la de los `link` que fueron apuntando a uno y después a otro, y cada cambio de esa secuencia es un [`apply`](apply.md).

Con `link` y `accepted.link` los dos presentes se leen las **dos dimensiones**: cuándo se propuso una ubicación, y cuándo se aprobó.

### Y un capture que `prune` borró se sigue leyendo

Es una propiedad que la ref regala y que no estaba escrita: todo commit que tenía ese capture **lo sigue teniendo**, así que se lee del árbol de ese commit aunque ya no esté en el del tip.

Sin la ref, [`capture prune`](capture.md) sería destructivo para la arqueología: borraría la única copia de una ubicación que alguien aprobó. Con ella, sólo saca del presente lo que nadie referencia.

## Degrada por acto, nunca por corrida

Dos degradaciones, y las dos dicen lo que no saben en vez de inventarlo:

**Un acto anterior a [la gramática](../concepts/ref.md#el-mensaje-es-el-comando)** —sin `Bilinker-Version`— no tiene comando canónico que leer. Se reporta con todo lo que **sí** es derivable de git: autor, fecha, tipo por los padres, y el diff del YAML. El comando queda `desconocido`, y **nunca se adivina del texto libre**: un mensaje viejo que empieza con `accept` no es un `accept <uuid>.<N>`, y tratarlo como si lo fuera sería fabricar precisión.

**Un repo que todavía no cortó a la ref** no tiene registro de decisiones: los bilinks viven en la rama del proyecto y su historia es la de esa rama. Se muestra, diciendo que es eso — *"sin ref: la historia sale de la rama"*— porque callar la diferencia haría parecer completa una vista que no lo es.

## Muestra el acuerdo sin inventar nada

N commits firmados sobre el mismo valor **son** N personas de acuerdo, y eso se lee del log sin ningún campo. [`agree`](../concepts/accept.md#quiénes-aprobaron) lo dice además en el artefacto, que es lo que le permite existir a un endoso que no cambia ningún valor; `history` muestra los dos: quién lo escribió, y qué decía la lista en cada momento.

## Salida

```
$ bilinker history 7f3d8e9a
7f3d8e9a-…  docs/spec.md ↔ src/Service.java

  9c1f0ab  Ana    2026-08-31  decisión       accept --place 7f3d8e9a.0
           contra e91f0c4
           .0  link       3ca90f81… → 7d21b0ae…
               agree      —         → ana

  4e77d20  Ana    2026-08-31  decisión       apply 7f3d8e9a.0 7d21b0ae…
           .0  link       3ca90f81… → 7d21b0ae…
               3ca90f81  docs/spec.md         (section (atx_heading …
               7d21b0ae  docs/renombrada.md   (section (atx_heading …

  77a0c94  Luis   2026-08-30  decisión       accept 7f3d8e9a.0
           contra c4e1770
           .0  hash       —         → c00e0760…
               agree      —         → luis

  0af3c12  Luis   2026-08-29  corte          (anterior a la gramática)
           .0  el bilink aparece
```

Con `--format json`, un array de actos con los mismos campos y sin abreviar.

## Códigos de salida

| Código | Condición |
|---|---|
| 0 | Se listó la historia, aunque esté vacía. |
| 1 | El uuid no existe, o es ambiguo. |

## Invariantes

- `history` **no escribe nada**: es una vista.
- Ningún dato se adivina del texto libre de un mensaje.
- Un acto sin comando canónico se reporta igual, con lo que es derivable de git.
- Un capture ya borrado del árbol se lee del commit que lo tenía.

## Lo que no entra

**La vista completa.** Ensamblar decisiones + comentarios + contexto del grafo es de [impact](../../impact/), que es el único que consume bilinker y lattice. Acá está la primitiva, que es de bilinker porque es su formato y su ref.

Los comentarios no existen todavía y viven en [`proposals/discutir-una-aceptacion.md`](../proposals/discutir-una-aceptacion.md); esta spec no los espera.
