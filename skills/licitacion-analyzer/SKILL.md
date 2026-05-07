---
name: licitacion-analyzer
version: 1.0.0
description: Analiza el PDF de bases de una licitación (CompraNet, gobierno estatal/federal, privadas) y devuelve un dossier estructurado con specs técnicas, partidas, fechas, criterios de evaluación, candados y SKUs sugeridos.
triggers:
  - "analiza esta licitación"
  - "lee las bases"
  - "extrae specs licitacion"
  - "que pide la licitacion"
  - "analizame el PDF"
author: Felipe De Alba
---

# Skill: licitacion-analyzer

## Cuándo usarlo
Cuando Felipe pase un PDF de bases de licitación (CompraNet, gobierno, privada) y necesite un análisis exhaustivo antes de cotizar. Conecta con [[provebot]] / [[licitabot-v2]].

## Triggers de input
- Felipe adjunta PDF en Discord/Telegram con texto "analiza esta licitación"
- Felipe da un path local: `analiza /Users/ingfelipe/Downloads/LPN-001-2026.pdf`
- Felipe da una URL CompraNet directa

## Procedimiento

### Paso 1 — Obtener el PDF
- Si es URL: descargar con `curl -o /tmp/licitacion.pdf <url>`
- Si es archivo adjunto Discord/Telegram: el gateway lo guarda local y nos pasa el path
- Si es path local: leer directo

### Paso 2 — OCR + extracción de texto
PDF puede ser texto nativo o escaneado. Estrategia:
1. Intentar `pdftotext <pdf> -`. Si extrae texto razonable (>500 chars), usar eso.
2. Si extrae poco (PDF escaneado), usar OCR via:
   - `analyze_image` tool con cada página convertida a PNG (si Hermes lo permite)
   - O `tesseract` local: `pdftoppm -r 300 <pdf> page-` + `tesseract page-N.png -`

### Paso 3 — Parseo estructurado (LLM)
Pasar el texto extraído al modelo (DeepSeek) con prompt:

```
Eres un analista de licitaciones gubernamentales mexicanas.
Extrae del siguiente PDF de bases:

1. METADATOS:
   - Número de licitación / convocatoria
   - Modalidad (LPN/LPE/INV3/AD/etc)
   - Convocante (dependencia)
   - Tipo (bienes / servicios / obra mixta)
   - Carácter (nacional / internacional)
   - Reducción de plazos (sí/no)

2. FECHAS CLAVE:
   - Publicación
   - Junta de aclaraciones
   - Visita al sitio (si aplica)
   - Apertura de propuestas
   - Fallo
   - Firma de contrato
   - Inicio/fin de servicios

3. PARTIDAS / CONCEPTOS:
   Para CADA partida:
   - Número
   - Descripción
   - Unidad
   - Cantidad
   - Specs técnicas detalladas (marca, modelo, características obligatorias)
   - Plazo de entrega
   - Lugar de entrega

4. CRITERIOS DE EVALUACIÓN:
   - Puntos y porcentajes (si binario, dilo)
   - Capacidad técnica requerida
   - Capacidad económica (capital contable, ventas anuales)
   - Experiencia (años, contratos similares)

5. GARANTÍAS:
   - Seriedad de propuesta (% del monto ofertado)
   - Cumplimiento (% del contrato)
   - Vicios ocultos / calidad
   - Anticipo (si aplica)

6. CANDADOS / RIESGOS:
   - Marca/modelo único especificado (vendor lock)
   - Plazos extremos (entregas en <15 días)
   - Fianzas elevadas (>10%)
   - Capital contable inalcanzable
   - Experiencia muy específica
   - Visitas al sitio obligatorias en ciudad lejana

7. PRESUPUESTO ESTIMADO:
   - Monto autorizado (si publica)
   - Estimar techo basado en partidas

Devuelve TODO en JSON estructurado.
```

### Paso 4 — Cotización automática (opcional)
Para cada SKU/spec extraída, llamar:
```
compare_distributors(sku=<modelo_extraido>)
```
para precio dist real Syscom + comparativa.

### Paso 5 — Generar nota Obsidian
Crear nota en `/Volumes/HIKSEMI 512/ObsidianVault/licitaciones/<numero-licitacion>.md` con:

```markdown
---
type: licitacion
numero: <num>
convocante: <dep>
modalidad: <LPN/LPE/etc>
caracter: <nacional/internacional>
fecha_publicacion: <YYYY-MM-DD>
fecha_apertura: <YYYY-MM-DD>
fecha_fallo: <YYYY-MM-DD>
monto_estimado: <MXN>
status: analizada
tags: [licitacion, <dep>, <ano>]
analyzed_by: hermes
analyzed_at: <YYYY-MM-DD HH:MM>
---

# {Numero} — {Convocante}

## 📋 Metadatos
{tabla con numero/modalidad/caracter/etc}

## 📅 Fechas clave
{tabla}

## 🛒 Partidas
{tabla con # | descripción | unidad | cantidad | especs}

## 💰 Cotización Ocean Tech (Syscom + alternativos)
{tabla con SKU | precio dist | stock | fuente}

## 📊 Criterios de evaluación
{detalle}

## 🛡️ Garantías requeridas
{tabla}

## ⚠️ Candados detectados
{lista de riesgos / vendor lock / plazos extremos}

## 🎯 Recomendación Hermes
{1 párrafo: ir / no ir / condicional. Razones.}

## 🔗 Links
- PDF original: {path o URL}
- CompraNet: {URL si aplica}

## Historial
- {YYYY-MM-DD HH:MM} — Análisis automático Hermes
```

### Paso 6 — Resumen ejecutivo en chat
Devolver al usuario en el chat (Discord/Telegram/CLI):
```
🎯 Licitación {num} — {Convocante}

📌 Resumen
{1-2 líneas}

📅 Fechas críticas
- Apertura: {fecha} ({días restantes})
- Fallo: {fecha}

💰 Estimado total
${monto} MXN

🛒 Top 3 partidas (por monto)
1. ...
2. ...
3. ...

⚠️ Candados detectados ({n})
- ...

🎯 Recomendación: {ir / no ir / condicional}

📝 Análisis completo: [[licitaciones/{numero}]]
```

## Reglas
1. **NO inventes specs** — si el PDF está incompleto, di "no especificado".
2. **Identifica vendor lock SIEMPRE** — Felipe odia que le forcen marca.
3. Si la licitación tiene <10 días para apertura → marcar como "URGENTE" en chat.
4. Si fianza > 10% del monto → flag como riesgo crítico.
5. Para SKUs Hikvision/Bosch/Honeywell etc, llamar `compare_distributors` automáticamente.
6. Si NeoData ya tiene cotización para una partida similar → reutilizar y mencionar `[[NeoData PuPresupuestos]]`.

## Outputs
1. Nota Obsidian completa
2. Resumen ejecutivo en chat
3. (Opcional) Push a [[provebot]] para tracking de pipeline de licitaciones
