# Comando: `worklist provider set-status`

Muta el proveedor de prueba directamente, sin pasar por git — simula que alguien cambió un ítem del lado del tracker real. Sin esto, [`check-push`](check-push.md) nunca tiene forma de rechazar nada: en un sistema cerrado, la creencia escrita y el estado en vivo siempre coinciden.

Es una herramienta de prueba, no una integración real. La forma real —Jira, vía `acli`— es otro puerto detrás de la misma interfaz de proveedor; ver [`concepts/sync.md`](../concepts/sync.md#qué-se-compara-y-contra-qué`).

## Firma

```
worklist provider set-status --provider-file <archivo> <clave> <status>
```

## Comportamiento

Escribe `{clave: status}` en el archivo — un JSON plano, `clave -> status`. Si el archivo no existe, lo crea. Si la clave ya estaba, la pisa.

## Salida

```
$ worklist provider set-status --provider-file provider.json ACC-101 done
ACC-101: open -> done
```
