# Bilinker

Bilinker es la herramienta de referencias estructurales persistentes y bidireccionales del ecosistema Accreta. Conecta fragmentos de texto entre capas del sistema — spec, decisiones técnicas, código — de forma que los links sobreviven reformateos, renombres y movimientos, y se detecta automáticamente cuando el contenido cambia.

## Problema que resuelve

Las referencias por número de línea son frágiles: cualquier inserción o reformateo las invalida sin aviso. En proyectos con múltiples capas (spec → ADR → implementación), el drift entre lo que dice la spec y lo que hace el código se acumula silenciosamente.

Bilinker resuelve esto con dos mecanismos:
- **Query AST via tree-sitter**: la referencia identifica el fragmento por estructura
  sintáctica, no por posición. Sobrevive reformateos e inserciones.
- **Hash SHA-256 en `accepted`**: detecta cuando el contenido del fragmento
  cambia y requiere aceptación explícita del nuevo estado.

## Posición en el ecosistema

```
bilinker     ← detecta drift entre capas linkedeadas
lattice      ← agrega las conexiones del proyecto en un grafo único
impact       ← analiza el alcance, abre hilos de discusión
worklist     ← registra el trabajo concreto pendiente
accreta      ← gobierna la resolución del cambio
```

Bilinker produce y verifica un solo tipo de conexión: la referencia aceptada entre fragmentos. No recorre el grafo completo del proyecto ni consulta language servers — expone sus aristas vía `bilinker graph --format json` y lattice las compone con las de los demás proveedores.

## Principios de diseño

- **Estabilidad estructural**: la referencia sobrevive reformateos, cambios de
  indentación y movimientos de bloques completos.
- **Universal por gramática**: funciona para cualquier lenguaje con gramática
  tree-sitter — Java, Rust, Python, TypeScript, Markdown, YAML, SQL, etc.
- **Bidireccional por diseño**: dado un punto en un archivo, bilinker lista todos
  los bilinks que lo referencian, sin índice inverso ni escaneo del proyecto.
- **Responsabilidad acotada**: git y tree-sitter, nada más. El análisis de alcance
  y el recorrido del grafo agregado viven fuera.
- **Auto-reparable con confirmación**: 4 estados tienen auto-fix determinístico
  (MOVED, REANCHORED). Ningún fix se aplica sin `bilinker apply`.
- **Git como fuente de verdad**: check usa git para rastrear el origen de cada
  cambio. Los auto-fixes se aplican como commits git.
- **Aceptación explícita**: el hash de un fragmento solo se acepta con
  `bilinker accept`. No hay actualizaciones silenciosas de estado.
