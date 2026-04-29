# pi-amazon-presupuesto

Skill para Pi (pi.dev) que genera presupuestos en PDF con precios de Amazon.es.

## Qué hace

1. El usuario lista los productos que necesita
2. El skill busca en Amazon.es la opción de **mejor precio** y la de **mejor valoración**
3. Genera un **PDF** con el presupuesto comparativo

## Instalación

### Opción A — Desde git (recomendado)
```bash
pi install git:github.com/tuusuario/pi-amazon-presupuesto
```

### Opción B — Manual
Copia la carpeta `skills/amazon-presupuesto/` a `~/.pi/agent/skills/`

## Uso

Simplemente pide a Pi lo que necesitas:

```
busca en Amazon una impresora láser y una webcam, hazme un presupuesto
```

El skill se activa automáticamente. También puedes forzarlo con:
```
/skill:amazon-presupuesto impresora láser, webcam
```

## Requisitos

- Python 3 con `reportlab` (`pip install reportlab`)
- Acceso a búsqueda web (el modelo debe tener herramienta de búsqueda)

## Nota

Los precios son orientativos y se obtienen vía búsqueda web. Amazon.es bloquea
el scraping directo, por lo que los datos se extraen de los resultados de búsqueda.
