---
title: `capture` ancla al primer nodo del tipo cuando no hay campo `name`
status: done
created_at: 2026-08-29T23:53:30Z
updated_at: 2026-08-30T00:34:35Z
parent: n
---

# `capture` ancla al primer nodo del tipo cuando no hay campo `name`

Al capturar una posición cuyo anchor estable no tiene campo `name`, `bilinker capture` genera una query sin predicado —`(impl_item) @target`— que matchea **el primer nodo de ese tipo en el archivo**, no el seleccionado. No falla: devuelve un capture que apunta a otra cosa.

Casos verificados en Rust:

| Selección | Query generada | Resuelve a |
|---|---|---|
| `link.rs:148` `impl FromStr for LinkEndpoint` | `(impl_item) @target` | `impl fmt::Display for EndpointState` (línea 31) |
| `model.rs:80` `impl NodeId` (lattice) | `(impl_item) @target` | `impl fmt::Display for Guarantee` (línea 25) |
| `provider.rs:94` `impl Provider for BilinkProvider` | `(impl_item) @target` | `impl Availability` (línea 24) |
| `lib.rs:1` | `(visibility_modifier) @target` | el primer `pub` del archivo |
| `link.rs:85` (comentario de doc) | `(line_comment) @target` | el primer comentario del archivo |

Bajar a una función nombrada dentro del `impl` tampoco alcanza cuando el nombre se repite: `(impl_item body: (declaration_list (function_item name: @n0 "from_str") @target))` tiene cuatro candidatos en `link.rs`.

## Por qué importa acá

Un capture mal anclado es peor que uno roto: **reporta OK sobre una correspondencia que no existe.** Durante la task `o` hubo que detectarlo a ojo con `bilinker get` en tres de cuarenta y ocho cadenas nuevas. Con el volumen de la épica eso no escala, y el error sobrevive en silencio.

`impl_item` está listado en `grammar.rs::stable_anchor_kinds` para Rust, así que el walk-up lo elige — pero `grammar.rs::name_field` no le da nombre, y la query sale sin discriminante.

## Qué mirar

`capture.rs::query_from_path` y `real_name_predicate`, contra `grammar.rs::name_field` / `name_node_type`. Para `impl_item` el discriminante natural es el tipo y el trait (`type:` y `trait:`), no un campo `name`.

Como red de seguridad: si la query generada matchea más de un nodo, o el nodo que matchea no contiene la posición seleccionada, `capture` debería negarse en vez de escribir.

## Cuándo está hecha

Capturar cualquiera de las cinco posiciones de la tabla produce un capture que `bilinker get` muestra en esa posición, o un error explícito.

## Resuelta

La tabla que faltaba ya existía y no se usaba: `grammar.rs::name_field` sabe que el discriminante de un `impl_item` de Rust es `type:` y no `name:`, pero `real_name_predicate` pedía `child_by_field_name("name")` a mano. Ahora consulta la tabla, con `name` de default.

Y `type:` solo no alcanza: `impl Foo` y `impl Bar for Foo` comparten tipo. `rust_impl_predicate` emite los dos campos, `trait:` y `type:`.

```
$ bilinker capture --dry-run crates/bilinker/src/link.rs 148:1 148:1
query:  (impl_item
  trait: (type_identifier) @n0 (#eq? @n0 "FromStr")
  type: (type_identifier) @n1 (#eq? @n1 "LinkEndpoint")) @target
```

## La red, que es lo que cierra el agujero de verdad

`verify_query_identifies` resuelve la query recién armada contra el mismo archivo y exige **un** match, en los bytes del nodo seleccionado. Si no, `capture` falla sin escribir. Cubre los casos donde no hay discriminante posible y también los que aparezcan mañana, sin tener que preverlos:

```
$ bilinker capture --dry-run crates/bilinker/src/lib.rs 1:1 1:1
Error: la query generada matchea 17 nodos. El ancla `visibility_modifier` no tiene
con qué distinguirse en crates/bilinker/src/lib.rs:
(visibility_modifier) @target

Seleccionar un nodo con nombre propio adentro —una función, un método— da un ancla
única sin inventar un criterio.
```

## Lo que sigue sin poder capturarse, a propósito

Un `impl` inherente que convive con un `impl` de trait sobre el mismo tipo —`impl NodeId` junto a `impl fmt::Display for NodeId`— no se distingue: una query tree-sitter no puede exigir la **ausencia** del campo `trait:`. Falla con el mensaje de arriba, y el camino es anclar en una función de adentro. Fallar es la respuesta correcta: la alternativa sería elegir uno de los dos y llamarlo referencia.

## Specs tocadas

`commands/capture.md`: § "Flujo interno" gana el caso de `impl_item` y un paso 6 de verificación, y § "Propiedades garantizadas" gana **Unicidad de la referencia**, que era la garantía que faltaba. Esa sección no tenía bilink; ahora apunta a `verify_query_identifies`.

## Tests

`capture_disambiguates_a_rust_impl_block` y `capture_refuses_an_anchor_it_cannot_identify`, en `crates/bilinker-cli/tests/integration.rs`. Los dos fallan contra el código viejo.

## El conjunto que ya existía

Auditados los 118 captures de las seis capas buscando queries sin predicado: aparecieron dos. `commands/capture.md` § "Lenguajes soportados" apuntaba a `grammar.rs:1` —`use anyhow::{bail, Result};`— y estaba **aceptada en OK**; se repuntó a `language_for_file`. La otra, `lattice/editors/vscode.md` § "Activación" sobre `(export_statement) @target`, acertaba por casualidad —`activate` es el primer export del archivo— y se repuntó a `function_declaration name: "activate"`. Hoy no queda ninguna query sin predicado en el proyecto.
