---
name: amazon-presupuesto
description: Genera presupuestos en PDF buscando productos en Amazon.es. Usa este skill cuando el usuario quiera buscar precios de productos en Amazon, comparar opciones por precio y valoración, o crear un presupuesto con artículos de Amazon. Se activa ante frases como "busca en Amazon", "haz un presupuesto con productos de Amazon", "cuánto cuesta X en Amazon", "encuentra el mejor precio de X", "presupuesto Amazon", o cuando el usuario liste productos y pida buscar precios. Siempre que haya intención de compra o comparación de precios en Amazon.es, usa este skill.
---

# Skill: Presupuesto con precios de Amazon.es

## Objetivo

Dado uno o varios productos que el usuario quiere comprar, busca en Amazon.es las mejores opciones (mejor precio y mejor valoración) y genera un PDF con el presupuesto comparativo.

---

## Flujo de trabajo

### Paso 1 — Entender la lista de productos

Si el usuario no ha especificado los productos, pregúntale qué quiere comprar. Acepta nombres genéricos ("una impresora láser") o referencias concretas ("HP LaserJet M110w").

### Paso 2 — Buscar en Amazon.es

Para **cada producto**, realiza **dos búsquedas web** dirigidas a Amazon.es:

**Búsqueda de mejor precio:**
```
site:amazon.es [nombre del producto] precio más bajo
```

**Búsqueda de mejor valoración:**
```
site:amazon.es [nombre del producto] mejor valorado opiniones
```

Extrae para cada opción encontrada:
- Nombre del producto (título)
- Precio (€)
- Valoración (estrellas, ej: 4,5/5)
- Número de opiniones
- URL del producto en Amazon.es
- ASIN si es visible

> **Nota**: Amazon.es a veces bloquea el scraping directo. Si web_search no devuelve precios claros, usa `web_fetch` con la URL de búsqueda de Amazon:  
> `https://www.amazon.es/s?k=[producto_urlencoded]`  
> y extrae los resultados del HTML.

### Paso 3 — Seleccionar las mejores opciones

Para cada producto, selecciona **2 candidatos**:

| Criterio | Descripción |
|---|---|
| **Mejor precio** | El más barato con valoración ≥ 3,5 estrellas y al menos 10 opiniones |
| **Mejor valorado** | El de mayor puntuación (mínimo 50 opiniones), aunque sea más caro |

Si ambos criterios apuntan al mismo producto, muéstralo una sola vez marcado como "Mejor opción".

### Paso 4 — Generar el PDF

Usa `reportlab` para crear el PDF. Instala si hace falta:

```bash
pip install reportlab --break-system-packages -q
```

#### Estructura del PDF

```
PRESUPUESTO DE PRODUCTOS — Amazon.es
Fecha: DD/MM/AAAA
─────────────────────────────────────────

PRODUCTO 1: [Nombre del producto buscado]

  💰 Mejor precio
     [Nombre del artículo]
     Precio: XX,XX €
     Valoración: ⭐ X,X/5 (NNN opiniones)
     URL: amazon.es/...

  ⭐ Mejor valorado
     [Nombre del artículo]
     Precio: XX,XX €
     Valoración: ⭐ X,X/5 (NNN opiniones)
     URL: amazon.es/...

─────────────────────────────────────────

PRODUCTO 2: ...

─────────────────────────────────────────

RESUMEN
  Opción más económica total:  XXX,XX €
  Opción mejor valorada total: XXX,XX €
```

#### Código base para el PDF

```python
from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, HRFlowable, Table, TableStyle
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import cm
from datetime import datetime

def generar_presupuesto_pdf(productos, output_path):
    """
    productos: lista de dicts con estructura:
    {
        "busqueda": "nombre que pidió el usuario",
        "mejor_precio": {
            "nombre": "...", "precio": 29.99, "estrellas": 4.2,
            "opiniones": 345, "url": "https://amazon.es/..."
        },
        "mejor_valorado": {
            "nombre": "...", "precio": 45.00, "estrellas": 4.8,
            "opiniones": 1200, "url": "https://amazon.es/..."
        }
    }
    """
    doc = SimpleDocTemplate(
        output_path,
        pagesize=A4,
        leftMargin=2*cm, rightMargin=2*cm,
        topMargin=2*cm, bottomMargin=2*cm
    )
    styles = getSampleStyleSheet()
    story = []

    # Título
    titulo_style = ParagraphStyle('titulo', parent=styles['Title'], fontSize=16, spaceAfter=4)
    story.append(Paragraph("Presupuesto de productos — Amazon.es", titulo_style))
    story.append(Paragraph(f"Fecha: {datetime.now().strftime('%d/%m/%Y')}", styles['Normal']))
    story.append(Spacer(1, 0.4*cm))
    story.append(HRFlowable(width="100%", thickness=1, color=colors.orange))
    story.append(Spacer(1, 0.4*cm))

    total_precio = 0
    total_valorado = 0

    for prod in productos:
        # Cabecera del producto
        story.append(Paragraph(f"<b>Producto: {prod['busqueda']}</b>", styles['Heading2']))
        story.append(Spacer(1, 0.2*cm))

        for tipo, emoji, dato in [
            ("Mejor precio", "💰", prod.get("mejor_precio")),
            ("Mejor valorado", "⭐", prod.get("mejor_valorado")),
        ]:
            if not dato:
                continue
            mismo = prod.get("mismo", False)
            if mismo and tipo == "Mejor valorado":
                continue  # evitar duplicar si son el mismo
            
            label = f"{emoji} {tipo}" + (" (misma opción)" if mismo else "")
            story.append(Paragraph(f"<b>{label}</b>", styles['Heading3']))
            
            data = [
                ["Artículo", dato["nombre"]],
                ["Precio", f"{dato['precio']:.2f} €"],
                ["Valoración", f"{dato['estrellas']}/5 ({dato['opiniones']} opiniones)"],
                ["Enlace", dato["url"]],
            ]
            t = Table(data, colWidths=[3.5*cm, 13*cm])
            t.setStyle(TableStyle([
                ('FONTNAME', (0,0), (0,-1), 'Helvetica-Bold'),
                ('FONTSIZE', (0,0), (-1,-1), 9),
                ('VALIGN', (0,0), (-1,-1), 'TOP'),
                ('ROWBACKGROUNDS', (0,0), (-1,-1), [colors.whitesmoke, colors.white]),
                ('GRID', (0,0), (-1,-1), 0.25, colors.lightgrey),
                ('TEXTCOLOR', (1,3), (1,3), colors.blue),
            ]))
            story.append(t)
            story.append(Spacer(1, 0.3*cm))

        story.append(HRFlowable(width="100%", thickness=0.5, color=colors.lightgrey))
        story.append(Spacer(1, 0.4*cm))

        # Acumular totales
        if prod.get("mejor_precio"):
            total_precio += prod["mejor_precio"]["precio"]
        if prod.get("mejor_valorado"):
            total_valorado += prod["mejor_valorado"]["precio"]
        elif prod.get("mejor_precio"):
            total_valorado += prod["mejor_precio"]["precio"]

    # Resumen final
    story.append(Paragraph("<b>Resumen</b>", styles['Heading2']))
    resumen = [
        ["Opción más económica (total estimado):", f"{total_precio:.2f} €"],
        ["Opción mejor valorada (total estimado):", f"{total_valorado:.2f} €"],
    ]
    t2 = Table(resumen, colWidths=[10*cm, 6.5*cm])
    t2.setStyle(TableStyle([
        ('FONTNAME', (0,0), (0,-1), 'Helvetica-Bold'),
        ('FONTSIZE', (0,0), (-1,-1), 10),
        ('BACKGROUND', (0,0), (-1,-1), colors.lightyellow),
        ('GRID', (0,0), (-1,-1), 0.5, colors.grey),
    ]))
    story.append(t2)
    story.append(Spacer(1, 0.5*cm))
    story.append(Paragraph(
        "<i>Precios orientativos obtenidos de Amazon.es. Pueden variar en el momento de la compra.</i>",
        styles['Italic']
    ))

    doc.build(story)
    print(f"PDF generado: {output_path}")
```

### Paso 5 — Entregar el PDF

Guarda el PDF en `/mnt/user-data/outputs/presupuesto_amazon.pdf` y preséntalo al usuario con `present_files`.

---

## Notas importantes

- **Precios dinámicos**: Amazon cambia precios frecuentemente. Indica siempre en el PDF que son orientativos.
- **Productos no encontrados**: Si no encuentras un producto, indícalo claramente en el PDF con "No se encontraron resultados para: [producto]".
- **Muchos productos**: Si el usuario pide más de 8 productos, avísale que la búsqueda puede tardar varios minutos.
- **Variantes**: Si un producto tiene muchas variantes (color, tamaño), busca la configuración más estándar o la más vendida, salvo que el usuario especifique.
- **IVA**: Los precios en Amazon.es ya incluyen IVA. Menciónalo en el resumen.
