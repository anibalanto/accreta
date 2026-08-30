# Propuesta: el corrimiento sale del diff, no de un escaneo

`DISPLACED` se detecta hoy deslizando una ventana del largo del fragmento por todo el nodo, hasheando en cada posición. Git ya sabe qué bytes se tocaron: preguntárselo es más barato y **más exacto**.

> Esta propuesta está escrita y no implementada. `check.md` § "Intersección hunk / fragmento" ya describe el modelo de solapamiento; lo que falta es que alguien lo use.

## Lo que git ya sabe y no se le pregunta

`git diff <commit> -- <file>` devuelve hunks con sus rangos viejo y nuevo. De ahí sale, sin abrir nada más:

| Relación hunk / fragmento | Qué implica |
|---|---|
| todos los hunks **antes** del fragmento | el contenido no se tocó; se corrió por la suma de sus deltas |
| algún hunk **se superpone** | el contenido pudo cambiar — hay que mirar |
| todos **después** | ni se tocó ni se corrió |

La primera fila es `DISPLACED`, y el corrimiento no es una estimación: es la suma de los deltas. Git no está sugiriendo que esos bytes no cambiaron, está afirmándolo.

## Predecir y confirmar

El diff da la posición nueva; **un solo hash** la confirma. Lo heurístico sería creerle sin verificar, y no hace falta.

```
1. git diff <accepted.commit> -- <file>   → hunks
2. sumar los deltas de los hunks anteriores al fragmento
3. hashear el fragmento en la posición predicha
4. ¿coincide con accepted.hash? → DISPLACED, con la ubicación nueva ya calculada
```

El paso 4 es lo que lo vuelve seguro: si no coincide, el diff no alcanzó y se cae al camino de siempre.

## Qué cuesta hoy y qué costaría

| | Costo |
|---|---|
| hoy | O(largo del nodo × largo del fragmento) en SHA-256. Un nodo de 49 KB con un fragmento de 495 bytes son ~24 MB hasheados. |
| con hunks | un `git diff` por **archivo** —compartido entre todos sus endpoints— más aritmética, más un SHA-256. |

## Resuelve además un callejón sin salida

Cuando el texto aceptado quedó **fuera** del nodo, `find_in_node` no puede verlo —sólo mira adentro— y `apply` avisa que no puede calcular el fix. Es el caso de la sección que se mudó a otra parte del archivo.

El diff sí sabe adónde fue, y con eso `apply` puede proponer una **query** nueva en vez de un offset — que es la forma correcta de repuntar algo que cambió de contenedor.

## Dónde no alcanza

No reemplaza el escaneo, lo adelanta. Tres casos caen al camino de siempre:

- **El fragmento cae adentro de un hunk.** Un movimiento de bloque es borrado + agregado, y los dos se superponen: el diff no distingue "se movió igual" de "cambió".
- **Sin baseline.** Necesita `accepted.commit`, que desde la task `17` siempre se puede derivar.
- **Granularidad.** Los hunks son de líneas y el rango de bytes; convertir es barato pero exige el archivo leído, cosa que ya pasa.

## Lo que abre para lattice

Los deltas de hunk **son** el corrimiento, y salen del mismo diff que ya se pidió. Con eso se puede publicar algo que hoy nadie sabe:

- *este fragmento se corrió 400 bytes* — ruido
- *doce fragmentos de este archivo se movieron juntos* — se reestructuró
- *este nodo cambió de padre* — cambió de qué es parte

Bilinker publica el movimiento; **lattice decide qué significa**. "Se movió `vote`, revisá quién la llama" cruza una arista `derived` con una `accepted`, que es lo que [`lattice/concepts/edge.md`](../../lattice/concepts/edge.md) § "Contención" define como su trabajo. Bilinker no tiene call graph y no debe tenerlo.

## Relación con sacar el `offset`

Si el `offset` desaparece —task `19`—, `DISPLACED` desaparece con él y esta propuesta pierde su motivo original. **No pierde el otro**: el corrimiento sigue siendo el dato que lattice quiere, y el diff sigue siendo la forma barata de obtenerlo.

O sea que esto sobrevive a la simplificación, con otro consumidor.
