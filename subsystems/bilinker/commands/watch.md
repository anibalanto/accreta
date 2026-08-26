# Especificación: comando `bilinker watch`

## Propósito

Monitorea el filesystem en tiempo real y reporta cuando archivos vinculados por bilinks son modificados. Permite detectar drift en el momento en que ocurre, sin esperar a correr `bilinker check`.

## Firma

```
bilinker watch
```

Sin argumentos. Opera sobre la capa actual (directorio de trabajo).

## Comportamiento

1. Inicia un watcher sobre el directorio raíz de la capa.
2. Cuando un archivo modificado está referenciado por al menos un bilink estructural, emite una línea por cada cadena afectada.
3. Continúa hasta recibir Ctrl-C.

Los archivos dentro de `.bilink/` se ignoran.

## Salida

```
$ bilinker watch
watching /home/user/proyecto/subsystems/stratum  (Ctrl-C to stop)

ALTERED  crates/stratum-cli/src/main.rs  chain 8e2e749a-fbb9-44aa-9b7f-a57972498371
ALTERED  crates/stratum/src/path.rs      chain 9cfe0db7-65d3-4b26-bb08-ba661dcb071d
```

Cada línea de alerta contiene:
- `ALTERED` — estado esperado al correr `bilinker check`
- Path relativo del archivo modificado
- UUID completo de la cadena afectada

## Notas

- `watch` no actualiza los archivos `.bilink` ni cambia estados — solo notifica.
- Para confirmar el drift y actualizar el estado, correr `bilinker check` y luego `bilinker accept`.
- El watcher usa eventos del SO (inotify en Linux, FSEvents en macOS).

## Códigos de salida

| Código | Condición |
|--------|-----------|
| 0 | Terminado por Ctrl-C. |
| 1 | Error al iniciar el watcher. |
