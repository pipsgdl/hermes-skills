---
name: project-status
version: 1.0.0
description: Da el estado actual de un proyecto/repo de Felipe — combinando README, última actividad git, issues/PRs abiertos, y nota Obsidian asociada — en un dashboard one-pager.
triggers:
  - "estado del proyecto"
  - "como va"
  - "status de"
  - "donde quedo"
  - "que falta de"
author: Felipe De Alba
---

# Skill: project-status

## Cuándo usarlo
Cuando Felipe pregunte cómo va un proyecto específico:
- *"¿Cómo va presupuestos-inteligentes?"*
- *"Status de bluax"*
- *"¿Dónde quedé con LiDAR?"*
- *"Qué falta de COE"*

## Procedimiento

### Paso 1 — Identificar el proyecto/repo
A partir del input del usuario, encontrar el repo correspondiente:
- Match exacto: si dice "presupuestos-inteligentes" → `~/presupuestos-inteligentes`
- Match parcial: "presupuestos" → buscar repos cuyo nombre contenga "presupuestos"
- Match por sinónimo: "marketing" → `ocean-tech-marketing`, "vault"/"obsidian" → `ObsidianVault`

Si hay múltiples matches, pregúntale a Felipe cuál.

### Paso 2 — README del repo
```bash
cat ~/<repo>/README.md  # primeras 50 líneas
```
o si no existe local:
```
gh api /repos/pipsgdl/<repo>/readme -H "Accept: application/vnd.github.raw"
```

### Paso 3 — Actividad Git reciente
```bash
cd ~/<repo>
git log --since="2 weeks ago" --pretty=format:"%h %cr %s" --no-merges | head -10
```

### Paso 4 — Estado Git (dirty, ahead/behind)
```bash
git status -sb
git stash list  # cambios stashados
```

### Paso 5 — Issues/PRs abiertos en GitHub
```bash
gh issue list --repo pipsgdl/<repo> --state open --limit 5
gh pr list --repo pipsgdl/<repo> --state open --limit 5
```

### Paso 6 — Nota Obsidian asociada
```
obsidian_simple_search(query="<repo>")
# o
search_files("/Volumes/HIKSEMI 512/ObsidianVault/proyectos", "<repo>.md")
```
Leer la nota si existe — extraer "## Pendientes" o "## Próximos pasos".

### Paso 7 — Output unificado one-pager

```markdown
## 📊 Estado: <repo>

### 🎯 Qué es
<1-2 líneas del README>

### 🚦 Status
- **Última commit:** {fecha} ({tiempo_relativo})
- **Estado git:** {dirty/limpio} · {ahead N / behind M}
- **Stash:** {cantidad}
- **Issues abiertos:** {N}
- **PRs abiertos:** {N}

### 📈 Actividad reciente (2 semanas)
| Fecha | Commit |
|---|---|
| ... | ... |

### 📝 Nota Obsidian: [[<nota>]]
**Pendientes documentados:**
- [ ] ...
- [ ] ...

### 🎬 Siguiente paso sugerido
<inferido del contexto: si dirty → commit · si stale → revisar · si issues → priorizar>

### 🔗 Links
- Repo: https://github.com/pipsgdl/<repo>
- Local: `~/<repo>` o `/Volumes/HIKSEMI 512/<repo>`
```

## Reglas
1. **No inventes data** — si no hay issues/PRs/commits recientes, dilo.
2. Si el repo no existe (match falla), sugiere los más cercanos.
3. **Una sola pasada** — recolecta todo + sintetiza, no hagas back-and-forth.
4. Si Felipe pregunta por un proyecto sin repo (solo nota Obsidian), enfócate en la nota.
5. Tiempo relativo: "hace 3 días", "hace 2 semanas" (no fechas absolutas en el resumen).

## Output esperado
Un one-pager consumible en 30 segundos que le diga a Felipe:
1. Qué es el proyecto
2. Cuál es el estado actual
3. Qué falta hacer
4. Cuál es el siguiente paso lógico
