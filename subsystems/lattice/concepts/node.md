# Concepto: nodo

Un nodo es un fragmento direccionable del proyecto. Todos los proveedores emiten el mismo tipo de nodo — un fragmento de bilink, una función vista por el LSP y un documento markdown son el mismo tipo de cosa, vista por fuentes distintas.

## Forma canónica

```
<layer-root>::<path>#<start>~<end>    fragmento (rango de bytes)
<layer-root>::<path>                   archivo completo
issue:<id>                             ítem de worklist
<uri>                                  recurso externo
```

`<layer-root>` es la ruta de la capa relativa a la raíz del proyecto (`.` para la capa raíz, `.stratum/impl`, …). `<path>` es relativo a la raíz de esa capa. Los offsets son bytes absolutos dentro del archivo, con la misma semántica que el `range` de un capture de bilinker.

La forma canónica la produce el proveedor. Un proveedor que emite una arista hacia otra capa entrega el nodo **ya resuelto**: bilinker resuelve `../<layer>/.bilink/<uuid>.bilink` antes de emitir, porque la topología de cadena es conocimiento de su formato.

## Identidad

Dos nodos son el mismo si su forma canónica coincide exactamente.

No se intenta unificar nodos con rangos parecidos. Un proveedor que dice `crates/bilinker/src/check.rs#5100~7300` y otro que dice `crates/bilinker/src/check.rs#5140~7300` producen **dos nodos**, y la relación entre ellos se expresa por contención, no por identidad. Intentar fusionarlos requeriría un criterio de tolerancia arbitrario que fallaría de formas silenciosas.

## Contención

```
contiene(a, b)  ⟺  a.layer == b.layer  ∧  a.path == b.path  ∧  a.start ≤ b.start  ∧  b.end ≤ a.end
```

Lattice la calcula a partir de los rangos que recibe; ningún proveedor la emite.

Es la operación central del subsistema, porque es la que permite cruzar de una garantía a otra. El LSP dice "la función `check_structural` en `check.rs:210` llama a esto"; bilinker dice "tengo un endpoint aceptado que cubre los bytes 5100~7300 de `check.rs`". Sin contención son dos grafos disjuntos; con contención, un cambio en una función se conecta con la spec que la gobierna.

```
cubriendo(<layer>::<path>#<pos>)  →  nodos cuyo rango contiene esa posición,
                                      del más específico al más general
```

El orden importa: si dos bilinks cubren la misma posición, el consumidor casi siempre quiere el más ajustado.

### Contención por línea vs por byte

La contención se define sobre bytes. Un consumidor que parte de una posición del LSP —que trabaja en líneas y columnas— debe convertir a byte antes de consultar.

La implementación actual de `impact.rs:177-183` hace lo contrario: convierte el rango a una comparación contra el byte inicial de la línea. Eso da falsos positivos cuando dos fragmentos comparten línea, y falsos negativos con rangos que empiezan a mitad de línea. La conversión correcta es posición → byte, no rango → línea.

## Anclaje: el desajuste posición ↔ rango

Los dos tipos de fuente localizan las cosas de manera incompatible:

| Fuente | Localiza por |
|---|---|
| bilinker | rango de bytes (el `range` del capture), derivado de una query tree-sitter |
| LSP (`callHierarchy`) | línea + columna **del identificador** |

Para preguntarle al LSP por los callers de un nodo que vino de un bilink, hay que encontrar la posición exacta del identificador dentro del rango.

El dato ya existe. Un endpoint estructural no guarda solo un rango: guarda la **query tree-sitter** con la que `capture` lo localizó, y esa query incluye un predicado sobre el nombre del anchor:

```
link.1: src/check.rs :: (function_item
  name: (identifier) @n0 (#eq? @n0 "check_structural")) @target
```

La captura `@n0` **es** el nodo del identificador. Resolver la query da su posición sin ningún parsing adicional, y tree-sitter la entrega como `Point { row, column }` — el mismo formato que consume el LSP, sin conversión de bytes a líneas.

```
1. captura del predicado de nombre en la query almacenada
2. symbol_at(file, line, col) del daemon LSP
```

El paso 2 cubre los endpoints cuya query no tiene predicado de nombre — nodos sin campo `name`, o fragmentos capturados sobre un archivo completo sin query.

Un nodo cuyo anclaje no se resuelve por ninguna de las dos vías no genera aristas `derived`; se reporta y el traversal sigue.

> La implementación actual no usa ninguna de las dos: `fn_col_from_source` (`impact.rs:285`) escanea la línea salteando una lista hardcodeada de keywords de cuatro lenguajes. Funciona para declaraciones simples y falla con atributos, anotaciones o genéricos antes del nombre. Se elimina en la migración.

## Nodos sin rango

Tres casos no tienen rango de bytes:

| Nodo | Contención |
|---|---|
| Archivo completo | Contiene a todos los fragmentos de ese archivo. |
| `issue:<id>` | No participa de contención. |
| URI externo | No participa de contención. |

## Invariantes

1. Todo nodo tiene forma canónica; no hay nodos anónimos ni identificados por objeto.
2. La identidad es igualdad exacta de forma canónica.
3. La contención la calcula lattice, nunca un proveedor.
4. La contención se define sobre bytes, no sobre líneas.
5. `cubriendo(pos)` devuelve los nodos ordenados de más específico a más general.
6. Un nodo sin anclaje resoluble no produce aristas `derived`, pero sigue en el grafo.
