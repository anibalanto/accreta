# Comando: `worklist done`

Marca un ítem como completado. Opcionalmente completa el bilink abierto asociado
capturando el fragmento que resuelve el trabajo.

## Uso

```
worklist done <id> [capture <selector>]
```

## Comportamiento

### Sin `capture`

1. Resuelve el ítem por ID.
2. Actualiza `status: done` y `updated_at` en el frontmatter.
3. Si todos los hijos del padre están `done`, sugiere marcar el padre también.

### Con `capture <selector>`

1. Resuelve el ítem por ID.
2. Lee `source_bilink` del frontmatter para localizar el bilink asociado.
3. Ejecuta `bilinker capture` sobre el selector resuelto desde cwd.
4. Reemplaza `link.N: todo` en el bilink con el endpoint capturado.
5. Actualiza `status: done` y `updated_at` en el frontmatter.
6. Si todos los hijos del padre están `done`, sugiere marcar el padre también.

Si el ítem no tiene `source_bilink`, o el bilink no tiene ningún endpoint `todo`,
`capture` falla con código de salida `1`.

## Ejemplos

```bash
# Cerrar sin bilink
worklist done 3

# Cerrar completando el bilink abierto
worklist done 3 capture src/Persona.java:45:1
```

## Salida

```
$ worklist done 3 capture src/Persona.java:45:1

done: 3  Implementar método vote
bilink: a3f9c821  completado
  link.0  specs/voting.yaml :: impl
  link.1  src/Persona.java :: Persona#vote

hint: all children of 1 are done — consider: worklist done 1
```

## Exit codes

| Código | Condición |
|--------|-----------|
| `0` | ítem marcado como done |
| `1` | ID no encontrado, selector inválido, o bilink sin endpoint todo |
| `2` | error de escritura |
