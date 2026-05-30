# Integración con bilinker

Bilinker usa paths Stratum en los endpoints layer de los archivos `.bilink`.

## Endpoints layer

Cuando un bilink conecta fragmentos en capas distintas, el endpoint layer usa un path Stratum para referenciar la capa adyacente:

```
# tip en spec layer
link.1: .stratum/impl      ← path Stratum

link.0: ../..                        ← path tradicional (sube a spec)
link.1: ../impl                      ← path tradicional (baja a impl)
```

La resolución de un endpoint layer toma como base la raíz de la layer actual y agrega `.bilink/<uuid>.bilink`:

```
link.1: .stratum/impl
→ .stratum/impl/.bilink/<uuid>.bilink
```

## Navegación entre nodos de una cadena

Para recorrer una cadena, bilinker usa los endpoints layer como paths de navegación entre repositorios. La estructura `.stratum/` de Stratum es lo que hace predecible esa navegación.
