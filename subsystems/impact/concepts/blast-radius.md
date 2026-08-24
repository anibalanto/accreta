# Blast Radius

El blast radius de un cambio es el conjunto de cadenas de bilinks afectadas directa o indirectamente por una modificación en un archivo o endpoint.

## Definición

Dado un archivo `F` modificado:

1. **Afectación directa**: todas las cadenas que tienen un endpoint estructural
   apuntando a `F`. El nodo con ese endpoint pasa a estado `ALTERED` o `DELETED`.

2. **Propagación reactiva**: cada nodo alterado cambia el contenido de su archivo
   `.bilink`. Esto hace que el hash almacenado en el nodo adyacente ya no coincida, propagando `CHAIN_DIRTY` hacia los extremos.

El blast radius incluye ambos niveles: los nodos directamente afectados y todos los nodos que detectarán `CHAIN_DIRTY` en el próximo `check`.

## Cálculo

```
blast_radius(F) = {
  nodos con endpoint estructural → F,          (afectación directa)
  ∪ nodos adyacentes en las mismas cadenas     (propagación)
}
```

El cálculo es una consulta al grafo:

```
lattice graph <F> --both --guarantee accepted --format json
```

Impact no recorre cadenas ni lee `.bilink`: lattice devuelve las aristas alcanzadas con sus nodos, `state` y `commit`, ya resueltas entre capas.

### Blast radius por llamadas

Con el proveedor LSP disponible, el alcance se extiende un nivel que antes no existía:

```
lattice graph <F> --up --via bilink,governs,call --format json
```

Esto alcanza también las specs que gobiernan funciones que *llaman* al código modificado, no solo las que lo referencian directamente. Esas aristas llegan con `guarantee: derived` y el reporte debe distinguirlas de las `accepted` — ver [integración con lattice](../integration/lattice.md) § "Garantía".

## Uso

El blast radius aparece en el Impact Report como la lista de cadenas afectadas. Es la respuesta a la pregunta: *¿qué más puede estar roto?*

No implica que todo lo del blast radius esté efectivamente roto — solo que necesita ser revisado. Un cambio compatible (e.g., renombrar un método pero actualizar todas las referencias) puede tener blast radius grande con todos los estados finales en `OK` tras el check.

## Relación con la propagación reactiva de bilinker

El mecanismo de propagación es el descripto en [el formato bilink](../../bilinker/concepts/bilink.md): `state.N` es parte del archivo `.bilink`, por lo que cambiar el estado de un nodo cambia el hash que el nodo adyacente tiene almacenado, desencadenando `CHAIN_DIRTY` en el próximo check.

impact no replica ese mecanismo ni lo consulta directamente — lo recibe en el campo `state` de cada arista. El blast radius que calcula impact es una proyección de lo que bilinker detectaría si se corriese `check` en todas las capas afectadas.

Lattice tampoco dispara `check`: refleja lo que los `.bilink` dicen al momento de la consulta. Un blast radius calculado sobre estados viejos es un `check` que no se corrió.
