# Especificación: comando `bilinker apply`

## Propósito

Corrige **dónde apunta** un endpoint cuando el fragmento se movió. Calcula el fix en el momento re-resolviendo contra git y el AST actuales — nunca reutiliza un cálculo previo.

`apply` acuña el [capture](../concepts/capture.md) de la ubicación nueva y repunta el `link` del endpoint. **No escribe `accepted`**, así que el endpoint no queda `OK`: queda en `RELOCATED` hasta que alguien apruebe la ubicación nueva.

> **`apply` propone, `accept` dispone.**

Ésa es la diferencia con el comportamiento anterior, donde MOVED volvía a `OK` solo. Mover un vínculo a otro fragmento es una decisión —el fragmento nuevo puede no ser el que la spec describe— y una decisión sin aprobar es trabajo pendiente, no trabajo hecho.

Requiere git como dependencia dura.

## Firma

```
bilinker apply [--dry-run] [--filter <estado>] [-y]
```

| Flag | Descripción |
|---|---|
| `--dry-run` | Muestra los fixes que se aplicarían sin escribir nada. |
| `--filter <estado>` | Aplica sólo fixes de un estado específico (e.g. `--filter MOVED`). |
| `-y` | Omite la confirmación interactiva. |

## Flujo

1. Escanear los bilinks de la capa actual.
2. Para cada endpoint en un estado con fix, **re-resolverlo** con el mismo algoritmo que usa `check` (ver [`check`](check.md) § "Algoritmo de detección por tipo de endpoint"). Eso produce un estado re-derivado y la ubicación actual del fragmento.
3. Validar el estado re-derivado contra el cacheado (ver "Validación de frescura"). Si difieren, descartar ese endpoint.
4. Calcular la ubicación nueva. Si coincide con la que el `link` ya tiene, es un no-op y se omite.
5. Mostrar el resumen y pedir confirmación (o `-y`).
6. Para cada fix: acuñar el capture de la ubicación nueva —si no existía— y repuntar el `link`.
7. Si el commit del proyecto contra el que se calcularon los fixes no está absorbido, **absorberlo en un commit propio** sobre [`refs/bilink/<branch>`](../concepts/ref.md) — un merge que sólo trae código.
8. Cerrar **cada fix** con un commit sobre la ref de un solo padre: [un commit sobre la ref hace una cosa](../concepts/ref.md#un-commit-hace-una-cosa), y absorber y repuntar son dos. En un repo que todavía no cortó a la ref, commitear los archivos escritos en la rama del proyecto.

**Un commit por `link` repuntado, no por invocación.** Un `apply -y` que corrige tres endpoints absorbe una vez —el paso 7— y escribe tres commits encadenados sobre ese merge. Es la misma [granularidad por objeto](../concepts/ref.md#la-correspondencia-con-el-proyecto-es-el-segundo-padre) que `accept`, y por el mismo motivo: repuntar un vínculo a otro fragmento es una decisión, se firma sola y se audita sola.

Y lo fuerza [el mensaje](../concepts/ref.md#el-mensaje-es-el-comando): `apply <uuid>.<N> <capture-nuevo>` nombra **un** endpoint, y un mensaje que nombrara tres no sería reproducible contra el árbol de un solo padre.

**`apply` y `accept` son dos commits porque son dos actos**, con dos autores posibles: `apply` describe —repunta los `link` y deja los endpoints en `RELOCATED`— y `accept` bendice. El commit de `apply` no toca ningún `accepted`.

**Mensaje de commit**, uno por fix, con [la gramática de la ref](../concepts/ref.md#el-mensaje-es-el-comando):

```
apply 7f3d8e9a-….1 3ca90f81…: MOVED → specs/domain/voting.yaml

Invocation: bilinker apply -y
Bilinker-Version: 0.1.0
```

El segundo argumento es el capture **nuevo**: es lo que el `link` pasa a nombrar, y lo que un replay compara. `Invocation:` guarda lo que la persona tipeó —`apply -y`, que produjo tres commits— y es dato de auditoría, no de verificación.

## Cálculo del fix por estado

| Estado | Cómo se encuentra la ubicación nueva |
|---|---|
| **MOVED** | El índice de renames de git: `git diff -M --name-status`. Se verifica que la query resuelva en el destino. |
| **REANCHORED** | La query relajada matchea un nodo con nombre distinto, por encima del umbral y con margen sobre el segundo. |

Los dos producen lo mismo: **un `(file, query)` nuevo**, y con él un capture nuevo. Ver [`check`](check.md) para los criterios de detección.

**Ningún estado de aceptación tiene fix.** `apply` corrige dónde está el fragmento, y eso lo dice el capture; que el contenido coincida con lo aprobado es una decisión, y las decisiones las escribe `accept`.

## No hay fork, porque no hay mutación

Un capture es inmutable y su id es el hash de su ubicación, así que corregir una ubicación **siempre** produce un capture distinto. `apply` lo acuña y repunta un solo `link`: el del endpoint que está corrigiendo. Los demás referentes no se enteran, sin que haya que decidir nada.

Eso reemplaza el copy-on-write del formato anterior, donde `apply` mutaba el capture en el lugar salvo que estuviera compartido, y una tabla decidía por tipo de fix si forkear:

| Antes | Ahora |
|---|---|
| MOVED y EXPANDED mutaban en el lugar | todos acuñan |
| REANCHORED forkeaba si había más de un referente | no hay caso especial |
| había que contar referentes antes de aplicar | no hace falta |

El capture viejo queda: sigue vivo mientras algún `accepted.link` lo nombre —es la ubicación que alguien aprobó— y lo limpia [`capture prune`](capture.md) cuando ya no lo alcanza nadie.

## El estado se re-deriva; la cache no decide

`apply` **no lee el estado cacheado**. El que escribió el último `check` describe el árbol de ese momento, y el archivo pudo cambiar después: aplicar un fix derivado de esa foto es corregir contra algo que ya no está.

Así que re-resuelve el capture contra el árbol actual y decide con eso. De la cache sólo sale `commit`, que es un dato de git y no una conclusión sobre el estado del árbol.

Es más simple que la validación que había antes —re-derivar y comparar contra lo cacheado— y no puede quedar desincronizada, porque no hay dos valores que comparar.

## Cuando el fix no se puede calcular

Un capture en `MOVED` o `REANCHORED` no siempre produce una ubicación nueva: git puede no reportar el rename, el anchor puede no localizarse, la query puede no tener predicado de nombre que reescribir.

**Cuando no puede, lo dice y sigue.** Abortar dejaría sin revisar a todos los demás endpoints por culpa de uno:

```
warn: 09b88fe0… endpoint.0: MOVED: git no reporta un rename de 'docs/spec.md'.
      Si el archivo nuevo no está trackeado, `git add` y volver a correr.
```

## Qué escribe

| Archivo | Qué |
|---|---|
| `capture/<id>.yaml` | El capture de la ubicación nueva, si no existía. |
| el bilink | `link` del endpoint corregido, y nada más. |
| `cache/state` | El estado re-derivado tras el fix: `RELOCATED`. |

**`accepted` no se toca nunca.** Es la invariante que separa proponer de aprobar, y con el bloque aparte es casi imposible de violar por accidente.

## Salida

```
$ bilinker apply

Pending fixes (3):
  MOVED      7f3d8e9a…  endpoint.1  → specs/domain/voting.yaml
  REANCHORED 3a4b5c6d…  endpoint.0  → anchor check_endpoint  (similitud 83%)

Apply? [y/N] y

Repuntados 3 endpoint(s). Los 3 quedan en RELOCATED.
  Revisar con `bilinker get <uuid>.<N>` y aprobar con `bilinker accept --place`.
commit:  refs/bilink/… @ 9f7020e  (absorbe 24ae0f6)
commit:  refs/bilink/… @ a4b5c6d
commit:  refs/bilink/… @ 3e1f8b2
commit:  refs/bilink/… @ c70d914
```

Ningún fix cierra solo. Por eso el resumen dice qué falta antes de listar los commits: el trabajo no terminó cuando `apply` termina, y los tres endpoints quedaron esperando un `accept --place`.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Todos los fixes aplicados. |
| 1 | Error al calcular o aplicar algún fix, o algún fix descartado por cache desactualizada. |
| 2 | No hay endpoints con fix disponible. |

## Invariantes

- `apply` nunca escribe `accepted`. Corrige dónde está el fragmento, nunca qué se aprobó.
- `apply` nunca modifica un capture existente: acuña.
- El único efecto de `apply` sobre un bilink es repuntar un `link`.
- Tras `apply`, el endpoint queda en `RELOCATED`. Ningún fix lo devuelve a `OK`.
- `apply` nunca aplica un fix derivado de la cache: cada uno se recalcula re-resolviendo contra el árbol y el índice git actuales.
- Si el estado re-derivado no coincide con el cacheado, `apply` descarta el fix y avisa.
- Si el fix calculado no se puede verificar —el hash no coincide en el path nuevo, el anchor no aparece— `apply` lo rechaza y avisa.
- `apply` es idempotente: un fix ya aplicado se detecta como no-op.
- `apply` escribe un commit de decisión por `link` repuntado, todos hijos de la misma absorción.
- `ALTERED`, `UNRESOLVED` y `CHAIN_DIRTY` no tienen fix y `apply` no los toca.
