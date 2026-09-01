# El transporte

Un socket local, y **nada que configurar**.

## La ruta se deriva

| Sistema | Dónde |
|---|---|
| Unix — Linux, Mac | `~/.lspd/daemon.sock` |
| Windows | `\\.\pipe\lspd` |

No hay flag, ni variable de entorno, ni archivo de configuración. Quien quiera hablarle al daemon calcula la ruta con la misma regla y llega.

**Que no haya configuración es el criterio con que se eligió el transporte**, no una consecuencia. Un socket local es lo único que se puede derivar de nada: existe en un lugar fijo del sistema de archivos, y ese lugar es el mismo para el que escucha y para el que llama.

## Por qué no TCP en loopback

Sería **un solo camino de código** en vez de dos, y ahí se complica:

| | |
|---|---|
| puerto fijo | colisiona con cualquier otra cosa, y con otro `lspd` de otro usuario en la misma máquina |
| puerto dinámico | hay que publicarlo en algún lado, y ese lado es un archivo — el mismo problema con un salto más |
| puerto configurable | **ya es configuración**, que es lo que se estaba evitando |

Y deja algo escuchando en la máquina. En una laptop corporativa eso es una conversación con alguien, y este daemon no está pidiendo esa conversación: lo que quiere es hablar con procesos del mismo usuario.

## Dos transportes, un protocolo

Named pipes en Windows y sockets Unix en el resto son **dos implementaciones del transporte y una sola del protocolo**. Lo que va por adentro —JSON-RPC 2.0 con framing por líneas, ver [`protocol.md`](protocol.md)— es idéntico, y ninguna de las dos puntas sabe por cuál de los dos está hablando.

La frontera está donde tiene que estar: en obtener un par de streams de bytes. Todo lo de arriba es genérico sobre `AsyncRead + AsyncWrite`.

## Por qué se hizo cross-SO ahora, y no antes

Mientras el daemon fue de lattice, no compilar en Windows era una limitación de lattice. Con **bilinker dependiendo de él**, esa limitación se heredaría al subsistema que corre en el repo de otro equipo — y ese equipo puede estar en Windows.

Es la misma clase de razón por la que el daemon salió de lattice: lo que era aceptable para un consumidor deja de serlo cuando aparece el segundo.
