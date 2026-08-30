# Especificación: comando `bilinker recapture`

## Propósito

Repunta un endpoint estructural a otro fragmento: acuña el [capture](../concepts/capture.md) nuevo, valida, y reescribe su `link`.

Existe porque hay estados que no tienen auto-fix y tampoco se resuelven aceptando. Una sección de spec renombrada o un test reescrito dejan el capture en `UNANCHORED`: el fragmento ya no está donde estaba, y `apply` no puede adivinar dónde quedó. Sin este comando la única salida es editar el `link` a mano — un reemplazo de texto sobre el campo que define a qué apunta un vínculo, sin validar que el capture exista, que esté en la misma capa, ni que el endpoint sea estructural.

## Firma

```
bilinker recapture <uuid>.<N> <file> [<pos>] [<end>]
```

| Argumento | Descripción |
|---|---|
| `<uuid>.<N>` | Endpoint a repuntar. Acepta prefijo del UUID. |
| `file` | Archivo del fragmento nuevo, relativo a la raíz de la capa. |
| `pos` | Posición `línea:col` (1-based). Omitir para capturar el archivo completo. |
| `end` | Fin de la selección `línea:col`. Default: igual que `pos`. |

## Comportamiento

1. Resolver el bilink y verificar que el endpoint sea estructural. Un endpoint `path` o `issue` no tiene capture que repuntar.
2. Crear el capture del fragmento nuevo — reusa uno existente si la referencia `(file, query, offset)` es idéntica, igual que `bilinker capture`.
3. Escribir el `link` apuntando al capture nuevo.
4. **Limpiar `state.N`**: el estado anterior describía el capture viejo, y dejarlo mentiría hasta el próximo `check`.
5. Reportar si el capture anterior quedó sin referentes.

```
$ bilinker recapture 430a5d51.0 subsystems/bilinker/scenarios/check.yaml 69:9

5fdff600-8690-4a75-85f6-e91072489d67
link.0 → capture 5fdff600
  antes: fbd0ca92  (quedó sin referentes)

revisar con `bilinker get 430a5d51.0` y aceptar con `bilinker accept 430a5d51.0`
```

El UUID del capture va a stdout para poder usarlo en pipes; el resto a stderr.

## No acepta

`recapture` corrige **a dónde apunta** el endpoint y nada más. Que el contenido del fragmento nuevo sea el correcto lo decide un humano con `accept`.

Es la misma separación que entre [`apply`](apply.md) y [`accept`](accept.md), y por el mismo motivo: reparar una ubicación es mecánico y verificable; afirmar que un contenido sigue cumpliendo lo que el vínculo promete, no.

## No borra el capture anterior

Puede tener otros referentes — la deduplicación hace que un capture sea compartido con frecuencia. Se informa que quedó huérfano y se limpia con `bilinker capture prune`.

Borrar por si acaso dejaría bilinks apuntando a la nada; dejar un archivo huérfano no rompe nada.

## Cuándo usarlo

- El capture está en `UNANCHORED`: la query no matchea y la similitud no alcanzó para `REANCHORED`.
- El fragmento se movió a otro archivo, y no fue un rename que git pueda detectar — el caso de código que migra a otro repo.
- El vínculo sigue teniendo sentido pero conviene apuntarlo a un fragmento distinto.

Para lo que sí tiene auto-fix —MOVED, DISPLACED, EXPANDED, REANCHORED— corresponde [`apply`](apply.md), que recalcula el fix en vez de pedir la posición.

## Código de salida

| Código | Condición |
|---|---|
| 0 | Endpoint repuntado. |
| 1 | UUID no encontrado, endpoint no estructural, el `link` ya apuntaba a ese capture, o el fragmento nuevo no se pudo capturar. |
