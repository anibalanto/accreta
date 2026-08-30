---
title: La salida de `check` no es fiel a lo que `check` sabe
status: done
created_at: 2026-08-29T23:52:00Z
updated_at: 2026-08-30T00:28:21Z
parent: n
---

# La salida de `check` no es fiel a lo que `check` sabe

Dos defectos distintos, el mismo daño: quien lee la salida de `check` se lleva menos trabajo del que hay. Es el supuesto de la épica en su forma más literal —**el inventario de trabajo de un cambio es la lista de no-OK**— y un inventario que dice "no hay nada" es peor que uno incompleto.

## 1. EXPANDED se escribe y no se imprime

`bilinker check .` guarda `state.0: EXPANDED` en el bilink y **no imprime la línea**.

Reproducción con la cobertura recién creada, que lo vuelve el caso más común de toda la épica:

```
$ printf '\nlínea nueva\n' >> subsystems/bilinker/concepts/capture.md
$ bilinker check .
1e318d3a  (ALTERED, OK)          ← los de siempre, ninguno es capture.md
…
$ grep state.0 .bilink/01e13be3*.bilink
state.0: EXPANDED                ← concepts/capture.md § "Migración…" ↔ migrations.rs::capture_split
$ bilinker apply --dry-run
Pending fixes (1):
  EXPANDED  01e13be3…  link.0  offset → 0~572 ampliado
```

Agrandar una sección de spec es el gesto más frecuente del sprint 3, y es justo el que no se ve. `apply` sí lo ve, así que el estado está bien calculado: el filtro de salida es el que no lo deja pasar.

**Dónde mirar:** `check.rs::check` arma los `CheckResult`; el filtro de qué se imprime está en `main.rs`, handler de `Command::Check`. Si enumera estados en vez de excluir OK, cada estado nuevo nace mudo.

## 2. Un estado no-OK sobrevive a que se revierta la edición

Siguiendo del ejemplo anterior: al deshacer la línea agregada, el fragmento vuelve a ser byte por byte el aceptado —`bilinker get 01e13be3.0 --diff` dice `[sin cambios]`— y `check` **deja `state.0: EXPANDED`**.

La causa es la optimización de [`commands/check.md`](../../subsystems/bilinker/commands/check.md) § "Optimización por diff de git": `check.rs::git_file_changed` pregunta si el archivo cambió desde `commit.N`, y como la edición y su reversión se cancelan, la respuesta es "no" y se conserva el `state.N` cacheado. Pero ese estado se calculó contra el árbol de trabajo, no contra el commit, así que la premisa de la optimización —"si el archivo no cambió desde `commit.N`, el estado cacheado sigue valiendo"— no se sostiene.

Queda pegado hasta que algo más fuerce la re-evaluación. `bilinker accept <uuid>.N` lo limpia, que es exactamente lo que no habría que hacer para arreglar un estado falso.

## Cuándo está hecha

- Un endpoint en cualquier estado no-OK aparece en la salida de `check`, con un test que lo fija para EXPANDED.
- Editar un archivo, correr `check`, revertir la edición y correr `check` de nuevo deja el endpoint en OK.

## Resuelta

El primer defecto era más ancho de lo reportado: `CheckResult::is_clean()` **enumeraba** los estados considerados limpios —OK, MOVED, DISPLACED, REANCHORED, EXPANDED, TODO, RESTYLED— y el CLI usaba ese mismo predicado para decidir qué imprimir. O sea que no era EXPANDED el mudo: eran los **seis**.

Partido en dos predicados, que es la distinción que faltaba:

- `CheckResult::all_ok()` — los dos endpoints en OK. Decide qué se imprime.
- `CheckResult::is_clean()` — el criterio de [`commands/check.md`](../../subsystems/bilinker/commands/check.md) § "Código de salida", donde los estados con auto-fix no hacen fallar a `check`.

El segundo defecto: el fast-path de `check_structural` ahora solo conserva un `state.N` de OK. Cualquier estado no-OK cacheado se recalcula, porque se escribió leyendo el árbol de trabajo y no el commit.

## Specs tocadas

`commands/check.md` § "Salida" —que ahora dice explícitamente que qué se imprime y qué código de salida se devuelve son dos preguntas distintas— y § "Optimización por diff de git". La primera no tenía bilink propio; ahora apunta a `all_ok`.

## Tests

`check_reports_an_expanded_endpoint` y `check_clears_a_stale_state_when_the_edit_is_reverted`, en `crates/bilinker-cli/tests/integration.rs`. Verificados contra el código viejo: los dos fallan.
