# lspd

`lspd` multiplexa language servers. Mantiene un servidor por lenguaje vivo entre invocaciones, y contesta por un socket las preguntas que necesitan un índice: quién llama a esto, a qué llama esto, qué símbolo hay en esta posición.

No es de nadie. Lo usa [lattice](../lattice/overview.md) para el proveedor `lsp`, y lo va a usar [bilinker](../bilinker/overview.md) para el cierre de firma.

## Por qué existe como capa propia

**Ya vivía afuera en todo salvo el nombre.** Se llamaba `lattice-daemon` y su `Cargo.toml` nunca mencionó al crate `lattice`: dependía de `async-lsp`, `tokio` y `tower`, y lattice le hablaba como le hablaría cualquiera — JSON-RPC 2.0 sobre un socket. Sacarlo no fue un refactor sino reconocer lo que ya era cierto.

**Y el nombre importaba.** `lattice-daemon` hace que todos supongan que es de lattice, y un bilinker que dependiera de algo llamado así se leería como una inversión de capas que no está ocurriendo:

```
      lattice ──┐
                ├──►  lspd    (socket)
     bilinker ──┘
```

Ningún ciclo, ninguna inversión, y **ningún compositor**: cada uno pide lo que necesita.

## Por qué no lleva nombre propio

Un nombre propio se gana cuando la pieza tiene modelo propio. Bilinker tiene el formato, lattice el grafo, stratum los paths. Esto tiene **una tabla de lenguajes y un socket**: `lspd` dice lo que hace y nadie tiene que preguntar.

## Lo que no es

**No tiene modelo del proyecto.** No sabe qué es una capa, ni un bilink, ni un nodo del grafo. Recibe `(archivo, línea, columna)` y devuelve posiciones. Todo lo que signifique algo lo pone quien pregunta.

**No decide cuándo arrancar.** Se lo arranca; quién y con qué política es del consumidor. Lattice lo levanta solo cuando el proveedor `lsp` lo necesita, y eso es una decisión de lattice.

## Qué hay acá

| | |
|---|---|
| [`concepts/transport.md`](concepts/transport.md) | cómo se llega al daemon en cada sistema operativo, y por qué no hay nada que configurar |
| [`concepts/protocol.md`](concepts/protocol.md) | los métodos, sus parámetros y sus respuestas |
| [`concepts/language-servers.md`](concepts/language-servers.md) | la tabla de lenguajes, cuándo se levantan, y qué deja en el disco |
| [`commands/lspd.md`](commands/lspd.md) | `start`, `stop`, `status` |

La implementación vive en [`.stratum/impl`](.stratum/impl), y son dos crates: el daemon y **el cliente**, que es compartido. Un cliente por consumidor sería el mismo protocolo escrito dos veces, y ya pasó una vez — antes de que existiera `daemon_client.rs`, bilinker tenía dos.
