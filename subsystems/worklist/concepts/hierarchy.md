# Jerarquía

La jerarquía de worklist es flexible. Cualquier tipo puede estar en la raíz del árbol y los niveles intermedios son opcionales.

**La relación padre-hijo se declara con el campo `parent`**, no con la ubicación del archivo. Todos los ítems viven juntos en `worklist/`; lo que los ordena es el campo. Ver [ítem](item.md) § "Jerarquía" para el porqué.

## Niveles permitidos

| Item | Puede tener de padre | Puede tener de hijo |
|------|----------------------|---------------------|
| Epic | nada | user-stories, tasks |
| User Story | nada, epic | tasks |
| Task | nada, epic, user-story | nada |

El sprint no entra en esta tabla: no es un tipo de ítem, así que no tiene padre ni hijo — **referencia** user stories y tasks desde `.metadata/product.yaml`, del servidor. Ver [§ Sprints](#sprints) más abajo.

## Estructura de ejemplo

```
insecure/all — el panorama, y en una ventana lo mismo con menos archivos
  1.epic.md                        ← sin parent: raíz del árbol
  2.user-story.md                  ← parent: 1
  3.task.md                        ← parent: 2
  4.task.md                        ← parent: 2
  5.task.md                        ← parent: 1   · task directa bajo la épica
  6.user-story.md                  ← sin parent: story suelta
  7.task.md                        ← parent: 6
  8.task.md                        ← sin parent: task suelta
```

El árbol no se ve en el `ls`: se deriva leyendo los `parent`. Es el precio de que la dirección de un ítem sea componible y estable, y se paga una vez —lo rinde un comando que lo dibuje— mientras que buscar un ítem por id se paga en cada uso.

## Referencia por ID

Los IDs son cortos y directos — se usan completos:

```bash
$ worklist show 1
1  [open]  Implementar hash.N en Rust  .epic

$ worklist show 3
3  [open]  Actualizar struct BiLinkFile  .task
```

Si por alguna razón el ID es ambiguo (no debería ocurrir, pero si el worklist es compartido entre proyectos), el CLI reporta los candidatos:

```bash
$ worklist show 1
Ambiguous ID '1' — matches:
  1   Implementar hash.N en Rust       .epic
  10  Actualizar parser de BiLinkFile  .user-story
Use the full ID.
```

## IDs secuenciales y jerarquía

Los IDs no codifican la jerarquía — un ítem hijo puede tener un ID posterior o anterior al de su padre. La estructura es el campo `parent`, no el número.

## Sprints

Un sprint no contiene: referencia. Y **queda fuera del árbol de descomposición**, porque no participa de él: puede llevarse ítems de épicas distintas, y tiene su propio contador.

### La membresía es un campo, y el campo no vive en un ítem

**Un sprint no es un archivo del árbol.** Su composición —qué ítems, `key`, `status`, `titulo`— vive en [`.metadata/product.yaml`](composition.md), en el panorama. Su prosa —por qué esos ítems, en qué orden, qué quedó afuera, cómo cerró— no tiene dueño nuevo: se descartó con el `.sprint.md` que la llevaba, y los 22 que existían se borraron sin migrarla.

```yaml
sprints:
  - id: 21
    titulo: "El worklist se muda"
    status: in-progress
    key: 6525
    items: [ACC-315, ACC-316, …]
```

`items` lleva los ids que el sprint referencia directamente, y ni uno más: la [regla del ancestro](#la-regla-del-ancestro) dice que un ítem entra con su subárbol entero, así que **nunca nombra una task cuya user story ya está en la lista**. Eso no cambió al mudarse.

`key` es el único campo que **no** escribe una persona: lo pone el servidor cuando el sprint existe del otro lado, y su ausencia significa que todavía no. Ver [`sync.md`](sync.md#la-correspondencia-con-el-sprint-del-proveedor-se-guarda-no-se-busca).

**Y la prosa se descartó, no se mudó.** Guardarla habría pedido un lugar nuevo —de una persona, no del servidor— que todavía no existe; hasta que exista, lo que un `.sprint.md` decía sobre por qué y en qué orden no se recupera. Ver [`composition.md`](composition.md#el-sprintmd-era-cuatro-cosas-mezcladas).

> **Un ítem está en un sprint porque una lista lo nombra**, y ya no porque su archivo esté en una rama. Ver [§ la membresía deja de ser una propiedad de la rama](composition.md#y-la-membresía-deja-de-ser-una-propiedad-de-la-rama).

### Los directorios reservados llevan punto adelante

> **Todo directorio dentro de `worklist/` empieza con `.`.**

Los ítems son archivos sueltos en la raíz, así que un directorio nunca es un ítem — es un espacio de nombres para otra cosa: `.metadata/`, `.bilink/`.

**Lo que garantiza que no choquen es el alfabeto de un id, y no una convención de lectura.** `.` ya está excluido de `[A-Za-z0-9_-]+` —ver [`item.md`](item.md) § "El alfabeto de un id"— así que ningún id puede nombrarse como uno de estos directorios ni empezar como ellos. La clase de colisión es vacía, no vigilada.

**No siempre fue así.** Mientras `_sprints/` existió, `_` — que sí es un carácter de id legal — necesitaba una segunda garantía: el `/` del stem, que el parseo del nombre ya descarta. `.metadata/removes/ACC-268.task.md` sigue dependiendo de esa misma garantía —su stem lleva `/` y por eso no se lee como un ítem de la raíz—, pero ya no hace falta el prefijo `_` para nombrar un espacio de nombres nuevo: alcanza con el punto.

```
1.epic.md                     ← épica 1
n.user-story.md               ← parent: 1
8.task.md                     ← parent: n
o.task.md                     ← parent: n
.metadata/
  product.yaml                ← composición: sprints, backlog — ver composition.md
```

### La regla del ancestro

> **Lo que entra a un sprint es un subárbol entero.**

Una user story entra con **todas** sus tasks, o no entra. Sus tasks no se enumeran: van con ella. Y una task se nombra sola sólo cuando no cuelga de ninguna user story — porque es de raíz, o porque cuelga directamente de una épica.

La cadena de ancestros se lee siguiendo `parent` hasta que se acaba.

**No alcanza con decir que un ítem entra sólo si ninguno de sus ancestros entra.** Eso deja pasar una task suelta cuando su user story no entra a *ningún* sprint, y por esa puerta la user story queda partida igual — con la única diferencia de que nadie la nombró. La unidad es el subárbol, no la ausencia de conflicto.

Dos consecuencias, y las dos son deliberadas:

**Una user story no puede atravesar sprints.** Si entra, entra entera. Con lo cual una que no cabe en una iteración deja de ser algo que se parte en el planning y pasa a ser una user story mal dimensionada — el problema se vuelve visible en vez de esconderse en un compromiso parcial.

**Y si una task se quiere sola pero cuelga de una user story**, la pregunta no es cómo sacarla al sprint sino si está bien colgada de esa user story. La salida es **reacomodar la descomposición** —la task pasa a colgar de otra user story, o de ninguna— y nunca planificarla dejando a su user story atrás. La regla lleva la discusión a la descomposición, que es donde va.

### Al cerrar, lo que no se hizo sale del `items`

> **Un sprint cerrado dice lo que se hizo. Lo que quedó vuelve al backlog.**

El ítem sin terminar se saca de `items`.

**Dejarlo adentro lo haría desaparecer.** El backlog se calcula sobre *"ningún sprint lo nombra"* —no *"ningún sprint abierto"*—, así que un ítem en un sprint cerrado no está en el backlog **ni** en un sprint en curso: no aparece en ninguna de las dos preguntas que este formato sabe contestar, y se pierde de vista sin que nada lo reporte.

**Lo que se querría conservar dejándolo —el registro de lo que se había comprometido— hoy sí se pierde.** Mientras existió `.sprint.md`, el cuerpo anotaba qué quedó afuera y por qué; `.metadata/product.yaml` no tiene un campo de prosa, y nada lo reemplazó. No es una decisión: es un hueco que abrió `drop-sprint-files` y que sigue sin dueño.

### El backlog no es un archivo

Un ítem que no está referenciado por ningún sprint está en el backlog **por definición**, y lo lista un comando. Mantenerlo como archivo obligaría a editar dos lugares para mover algo, y los dos podrían divergir.

El cálculo va sobre el subárbol, que es lo que un sprint referencia: **un ítem está en el backlog si el tope de su rama no lo nombra ningún sprint.** Las tasks de una user story planificada no se cuentan aparte —están donde está su user story— y una user story que ningún sprint nombra está en el backlog con todas sus tasks, sin importar cuántas de ellas alguien haya querido adelantar.

#### Y el que lo calcula es el servidor

*"Ningún sprint lo nombra"* es una afirmación sobre **todos** los sprints y sobre todos los ítems, así que sólo la puede hacer quien tiene el tronco entero — y desde que [el panorama vive de un solo lado](sync.md#el-panorama-vive-en-un-solo-lado-y-la-ventana-en-los-dos), ése es el servidor.

> **Un backlog calculado sobre una ventana no es un backlog más chico: es uno equivocado.**

Diría que está sin planificar todo lo que ese recorte no nombra, que es casi todo el corpus. Es la misma razón por la que [`bootstrap`](../commands/bootstrap.md) y [`push-states`](../commands/push-states.md) corren sobre el panorama y no sobre lo que uno tenga cortado: **una pregunta que se contesta recorriendo todo no se puede contestar con una parte, ni siquiera mal.**

### Épicas

Una épica no entra a un sprint. No lo prohíbe el formato, pero no tiene sentido: si cabe en una iteración, no era una épica.
