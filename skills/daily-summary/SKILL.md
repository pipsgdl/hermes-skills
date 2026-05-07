---
name: daily-summary
version: 1.0.0
description: Genera la daily note del día consolidando los highlights de Felipe — commits Git, notas modificadas, mensajes Telegram/Discord, tareas pendientes — con formato Obsidian estándar.
triggers:
  - "genera daily de hoy"
  - "daily note de hoy"
  - "resumen del dia"
  - "cierra mi dia"
  - "que hice hoy"
author: Felipe De Alba
---

# Skill: daily-summary

## Cuándo usarlo
Al final del día (o cuando Felipe lo pida) para auto-generar la daily note de Obsidian con:
- Commits Git de las últimas 24h en todos los repos respaldados
- Notas Obsidian modificadas hoy
- Resumen ejecutivo de logros del día
- Pendientes detectados (TODO, FIXME, [ ])

## Procedimiento

### Paso 1 — Calcular fecha
Determinar fecha de hoy en formato `YYYY-MM-DD` (zona horaria America/Mexico_City).

### Paso 2 — Verificar si ya existe daily note
```
read_text_file("/Volumes/HIKSEMI 512/ObsidianVault/daily-notes/{fecha}.md")
```
Si existe, **agregar al final** (no sobrescribir). Si no, crear desde scratch.

### Paso 3 — Recolectar commits Git de hoy
Para cada repo en `~/{repo}` y `/Volumes/HIKSEMI 512/{repo}`, ejecutar:
```bash
cd <repo> && git log --since="midnight" --pretty=format:"%h %s" --no-merges
```
Filtrar commits que sean del usuario (no auto-backup salvo que tengan info útil).

### Paso 4 — Notas Obsidian modificadas hoy
```
obsidian_get_recent_changes()  # del MCP obsidian si está disponible
```
o fallback:
```bash
find "/Volumes/HIKSEMI 512/ObsidianVault" -name "*.md" -newer /tmp/today_marker -not -path "*/.obsidian/*"
```

### Paso 5 — Pendientes detectados
Buscar en notas modificadas hoy:
- Líneas que empiecen con `- [ ]`
- TODO, FIXME, PENDIENTE en código
- Callouts `> [!todo]` en Obsidian

### Paso 6 — Síntesis ejecutiva
Pasar la info recolectada a una sola pasada del modelo para que genere:
- 3-5 highlights del día (qué se logró)
- Logros vs pendientes
- 1 frase de cierre (mood/aprendizaje del día)

### Paso 7 — Generar nota con formato estándar
Template (siguiendo el formato de daily notes anteriores en la bóveda):

```markdown
---
type: daily
date: YYYY-MM-DD
tags: [daily, <tags-segun-temas-del-dia>]
---

# {DiaSemana} {Día} de {Mes} de {Año}

> {emoji-mood} {Frase resumen del día}

## ⚡ Resumen ejecutivo
- {highlight 1}
- {highlight 2}
- {highlight 3}

## 🏗 Lo que se construyó (en orden)
{lista cronológica}

## 💻 Commits del día
| Hora | Repo | Mensaje |
|---|---|---|
| ... | ... | ... |

## 📝 Notas Obsidian tocadas
- [[nota-1]]
- [[nota-2]]

## ✅ Tareas completadas
- [x] ...

## ❌ Pendientes detectados
- [ ] ...

## 💭 Reflexión
{1-2 oraciones sobre el día}

## 🪁 Tags relacionados
#daily {otros}
```

### Paso 8 — Guardar
```
obsidian_append_content(filepath="daily-notes/{fecha}.md", content=<nota>)
```
o fallback:
```
write_file(path="/Volumes/HIKSEMI 512/ObsidianVault/daily-notes/{fecha}.md", content=<nota>)
```

## Reglas
1. **NO inventes commits o notas** — solo lo que existe en git log y en la bóveda.
2. **Conciso > exhaustivo** — Felipe tiene TDA, 3-5 highlights máximo.
3. **Wikilinks SIEMPRE** — `[[nota-X]]` para conectar.
4. **Sigue el formato de daily notes anteriores** — lee 1-2 notas anteriores del mes en `/daily-notes/` para alinear estilo.
5. **No duplicar** — si la daily ya tiene contenido, solo agregar la sección "Cierre" al final.

## Output esperado
Una nota daily lista para que Felipe la lea sin editarla, con info precisa y actionable. Si Felipe le hace cambios, los respetamos.
