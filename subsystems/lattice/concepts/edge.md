# Concepto: arista

Una arista es una conexión entre dos nodos del grafo que declara siempre **de dónde viene** y **qué garantiza**. Es la unidad fundamental de lattice: todo el resto del modelo existe para que las aristas puedan convivir sin perder esas dos propiedades.

## Nodo

Un nodo es un fragmento direccionable. Su identidad es su forma canónica:

```
<layer-root>::<path>#<start>~<end>    fragmento (rango de bytes en un archivo de una capa)
<layer-root>::<path>                   archivo completo
task:<id>                              ítem de worklist
<uri>                                  recurso externo (http, https)
```

La forma canónica la produce el proveedor, no lattice. Un proveedor que emite una arista hacia otra capa debe entregar el nodo ya resuelto: bilinker resuelve `../<layer>/.bilink/<uuid>.bilink` porque la topología de cadena es conocimiento de su formato, y entrega el fragmento del otro extremo en forma canónica.

## Anatomía

| Campo | Descripción |
|---|---|
| `from` / `to` | Nodos en forma canónica. |
| `kind` | Tipo de conector. Ver "Tipos de arista". |
| `guarantee` | `accepted` · `derived` · `asserted`. Ver "Garantía". |
| `provider` | Quién emitió la arista. |
| `directed` | Si el orden `from → to` tiene significado semántico. |
| `ref` | Identificador de la arista en su fuente (UUID del bilink, símbolo LSP, path + anchor). |
| `state` | Estado que reporta el proveedor. Solo para `accepted`; ausente en el resto. |
| `commit` | Commit en que se aceptó cada extremo. Solo para `accepted`. |

`commit` existe porque es el baseline de todo diff: un consumidor que quiera saber *qué* cambió desde el último estado aceptado corre `git log <commit>..HEAD`. Sin él en la arista, tendría que reabrir el archivo del proveedor para recuperarlo.

## Garantía

El eje que distingue a las aristas no es su forma sino cuánto se puede afirmar a partir de ellas.

| Garantía | Significado | Qué afirma |
|---|---|---|
| `accepted` | Declarada por un humano y verificada por su dueño (hash + commit). | "Esto está vinculado, y alguien aceptó explícitamente el estado en que estaba." |
| `derived` | Calculada por una herramienta a partir del contenido actual. | "Esto probablemente está relacionado, según lo que una herramienta externa pudo inferir." |
| `asserted` | Escrita en el contenido, sin ninguna verificación. | "Alguien escribió que esto se relaciona." |

La distinción no es cosmética. Una arista `derived` proviene de un language server, que falla sistemáticamente con dispatch dinámico, trait objects, macros, callbacks e inyección de dependencias: su ausencia no prueba nada. Una arista `asserted` puede apuntar a un archivo que ya no existe. Una arista `accepted` es la única sobre la que se puede afirmar que hubo drift, porque es la única con un estado anterior aceptado contra el cual comparar.

Un consumidor puede filtrar por garantía, pero nunca recibe una arista sin ella.

## Tipos de arista

| `kind` | Proveedor | `guarantee` | Dirigida | `ref` |
|---|---|---|---|---|
| `bilink` | bilinker | `accepted` | no | `<uuid>.<N>` |
| `governs` | bilinker | `accepted` | no | `<uuid>` |
| `task` | bilinker · worklist | `accepted` | no | `<uuid>` + id de tarea |
| `call` | proveedor LSP | `derived` | sí (caller → callee) | símbolo LSP |
| `doclink` | proveedor markdown | `asserted` | sí | path + anchor |
| `external` | proveedor markdown | `asserted` | sí | URI |

Un proveedor puede emitir varios `kind`, pero la garantía de cada `kind` es fija: no existe un `call` aceptado ni un `bilink` derivado.

## Contención

Dos proveedores nunca van a nombrar el mismo fragmento con el mismo rango. Bilinker dice que el fragmento son los bytes `245~389`; el LSP dice que la función `vote` empieza en la línea 11. Si el grafo exigiera identidad exacta de nodos, se fragmentaría en casi-duplicados y ninguna consulta cruzaría de un proveedor a otro.

Por eso lattice resuelve la **contención** como relación estructural de primera clase:

```
contiene(a, b)  ⟺  a.layer == b.layer  ∧  a.path == b.path  ∧  a.range ⊇ b.range
```

No la emite ningún proveedor — lattice la calcula a partir de los rangos de los nodos que recibe. Habilita la consulta que es el núcleo de cualquier análisis de alcance:

```
cubriendo(<layer>::<path>#<pos>)  →  nodos cuyo rango contiene esa posición
```

Que es exactamente la pregunta *"¿hay un bilink que cubra esta función?"* — el paso que permite saltar de una arista `derived` a una `accepted`, y con eso pasar de "esto podría estar afectado" a "esto está roto".

## Contrato de proveedor

Un proveedor debe:

1. **Enumerar** sus aristas en un scope dado (una capa, o el proyecto completo).
2. **Declarar** los `kind` que emite y la garantía fija de cada uno.
3. **Entregar** los nodos en forma canónica, ya resueltos a través de capas.
4. **Reportar disponibilidad** antes de ser consultado.

El punto 4 es el que hace honesta la degradación: si el proveedor LSP no está disponible, el grafo pierde todas sus aristas `call` y lattice lo declara en el resultado. Un análisis sobre un grafo degradado es válido, pero el consumidor tiene que poder saber que lo es.

## Deduplicación

Dos proveedores pueden emitir la misma conexión. La regla es deduplicar por `(from, to, kind)`, conservar la garantía más fuerte (`accepted` > `derived` > `asserted`) y registrar todos los proveedores que la emitieron.

Un corolario útil: si una conexión que un proveedor deriva automáticamente se declara además a mano, la declaración manual es un duplicado permanente que hay que mantener sincronizado. Declarar a mano solo se justifica para aristas que **ningún proveedor puede derivar** — llamadas entre lenguajes, entre repos, o hacia un binario externo.

## Frescura

Lattice no persiste el grafo. Cada consulta lo compone desde los proveedores:

- Las `accepted` las persiste su dueño; lattice lee también el `state` que ese dueño reporta.
- Las `derived` se calculan en el momento y no se escriben nunca.
- Las `asserted` se validan al leerlas (¿el destino existe?) y no se escriben nunca.

Persistir aristas derivadas ya se intentó en el ecosistema — los `subgraph.N` y los sciplinks de bilinker — y se revirtió: un índice derivado bajo control de versiones se desincroniza del código y reporta drift que no existe.

## Invariantes

1. Toda arista tiene `provider` y `guarantee`. No existe una arista anónima.
2. La garantía de un `kind` es fija y la declara el proveedor, no la arista.
3. Solo las aristas `accepted` tienen `state`.
4. Los nodos de una arista están siempre en forma canónica y ya resueltos entre capas.
5. La contención la calcula lattice, nunca un proveedor.
6. Lattice no escribe en ninguna fuente ni persiste el grafo.
7. Un proveedor no disponible reduce el grafo y se reporta; nunca produce un resultado silenciosamente incompleto.
8. La deduplicación conserva la garantía más fuerte y todos los proveedores de origen.
