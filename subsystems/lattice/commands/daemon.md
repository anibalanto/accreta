# Especificación: comando `lattice daemon`

## Propósito

El ciclo de vida de [`lspd`](../../lspd/overview.md), con el nombre que los usuarios de lattice ya tenían.

**No es un comando propio: es el mismo, reexportado.** `lattice daemon start` hace exactamente lo que `lspd start`, y existe porque el daemon salió de lattice y quitarle a la gente el comando que venía usando sería cobrarle a ella una reorganización que no pidió.

```
lattice daemon start [--workspace <path>]
lattice daemon stop
lattice daemon status
```

Los tres delegan. Qué hacen, qué imprimen y con qué código salen está en [`lspd`](../../lspd/commands/lspd.md), y no se repite acá: dos specs del mismo comando divergen el día que alguien toca una.

## Auto-start, que sí es de lattice

El proveedor `lsp` intenta conectarse antes de cada consulta. Si no hay nadie, **arranca el daemon** y espera hasta 5s.

| Resultado | Estado del proveedor |
|---|---|
| El daemon ya estaba corriendo | `Available` |
| Se arrancó recién y respondió | `Degraded` — el language server está indexando |
| No se pudo arrancar | `Unavailable` |

La distinción importa: el daemon contesta al `ping` apenas arranca, pero rust-analyzer y sus pares tardan bastante más en indexar. Durante esa ventana `callers` devuelve vacío, y reportar `Available` haría pasar **"todavía no sé"** por **"no hay llamadas"**.

**Es una política de lattice y no del daemon**, y por eso vive de este lado. `lspd` no decide cuándo arrancar; bilinker, el otro consumidor, tomó la decisión contraria — no lo arranca nunca y degrada a *no verificado*. Las dos son legítimas y ninguna es del daemon.

Ningún comando de lattice **requiere** el daemon: su ausencia siempre degrada, nunca aborta.

## Exit codes

| Código | Condición |
|---|---|
| 0 | Éxito |
| 1 | Daemon ya corriendo (`start`) / no corriendo (`stop`, `status`) |
| 2 | Language server no instalado para un lenguaje requerido |
