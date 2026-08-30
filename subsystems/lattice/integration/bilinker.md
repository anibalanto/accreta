# Integración con bilinker

Bilinker es un proveedor de lattice. Produce y verifica un tipo de conexión; lattice la compone con las de otras fuentes.

## Reparto

| | bilinker | lattice |
|---|---|---|
| Crear una referencia | `capture`, `chain new` | — |
| Verificar drift | `check` | — |
| Aceptar / reparar | `accept`, `apply` | — |
| Resolver una cadena entre capas | sí — es su formato | no |
| Recorrer el grafo del proyecto | no | sí |
| Consultar el call graph | no | sí, vía proveedor LSP |

La línea divisoria: bilinker sabe qué significa un `.bilink` y cómo resolverlo; lattice sabe cómo componer aristas heterogéneas. Ninguno de los dos hace el trabajo del otro.

## El contrato

```
bilinker graph <selector> --recursive --format json
```

Bilinker emite sus aristas con los nodos ya en forma canónica y la topología de cadena resuelta. Ver [`bilinker graph`](../../bilinker/commands/graph.md) § "Formato `json`".

```json
{
  "from":      ".stratum/impl::crates/bilinker/src/check.rs#5100~7300",
  "to":        ".::commands/check.md#1240~2180",
  "kind":      "bilink",
  "guarantee": "accepted",
  "provider":  "bilinker",
  "directed":  false,
  "ref":       "fac79bf8-1b2c-4d5e-8f6a-7b8c9d0e1f2a",
  "state":     ["OK", "ALTERED"],
  "commit":    ["ca76a590", "ca76a590"]
}
```

`kinds()` del proveedor: `bilink` e `issue`, los dos con garantía `accepted`. `governs` está definido pero bilinker no lo emite todavía: ver la sección de más abajo.

### Qué resuelve bilinker antes de emitir

- **La cadena entera.** Una cadena de N nodos produce **una** arista, entre sus dos tips estructurales. Los mids son mecanismo interno del formato — si lattice los viera, el grafo se llenaría de nodos `.bilink` que no son contenido del proyecto.
- **Los paths Stratum.** `link.N: .stratum/impl` se resuelve a la raíz de capa concreta antes de emitir.
- **El rango vigente.** El `range` que el último `check` dejó en la cache del capture.

Nada de eso es conocimiento que lattice pueda tener sin duplicar el formato de bilinker.

## `state` y `commit`

Dos campos que ningún otro proveedor emite y que existen porque impact los necesita:

- `state` — la tupla `(state.0, state.1)`. Permite filtrar por no-OK sin abrir archivos.
- `commit` — el commit en que el contenido aceptado de cada tip quedó establecido. Es el baseline de `git log <commit>..HEAD`.

Sin ellos en la arista, un consumidor tendría que reabrir los `.bilink` para completarlos — exactamente la duplicación que lattice elimina.

Lattice no los interpreta: los transporta. La semántica de un `ALTERED` es de bilinker.

## Frescura

Lattice refleja lo que los `.bilink` dicen en el momento de la consulta. No corre `check` ni lo dispara.

Un `state` desactualizado es un `check` que no se corrió, no un error de lattice. Un consumidor que necesite estados frescos corre `bilinker check` antes de consultar.

Esto mantiene la propiedad de que lattice es solo lectura y no tiene efectos sobre las fuentes.

## El endpoint bilink y las aristas `governs`

> **Pendiente del proveedor.** Todo lo de esta sección depende del endpoint de tipo bilink, que bilinker tiene especificado y no implementado — ver [`proposals/bilink-endpoint.md`](../../bilinker/proposals/bilink-endpoint.md). El diseño de lattice queda escrito acá porque no cambia: cuando el endpoint vuelva, estas aristas se emiten sin tocar nada de este lado.

Un bilink con `kind: governs` apunta en su `link.1` a otro archivo `.bilink`, no a un fragmento. Su nodo canónico es el archivo:

```
.::.bilink/fac79bf8-1b2c-4d5e-8f6a-7b8c9d0e1f2a.bilink
```

Ese nodo comparte `ref` con la arista `bilink` de esa misma cadena, y por ahí se hace el join: *"el documento D gobierna el vínculo `fac79bf8`"*.

Es una relación sobre una arista, no sobre un nodo — la relación ternaria que describe [impact-element.md](../../impact/concepts/impact-element.md). Lattice no modela hiperaristas: expone el nodo del archivo `.bilink` y deja que el consumidor una por `ref`. Es la representación mínima que preserva la semántica sin inventar un tipo nuevo.

## Lo que se movió de bilinker a lattice

| Antes | Ahora |
|---|---|
| `bilinker impact` | `lattice graph <sel> --up --via bilink,call` |
| `bilinker get --diff --recursive` | `lattice graph <sel> --down --via call` + `bilinker get --diff` |
| `bilinker daemon` | `lattice daemon` |
| `bilinker graph --format dot\|html` | renderers de `lattice graph` |

`bilinker graph` sobrevive en su forma de consulta local y como proveedor (`--format json`).
