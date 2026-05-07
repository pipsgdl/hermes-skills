---
name: search-everywhere
version: 1.0.0
description: Busca un término o concepto en TODOS los proyectos de Felipe — repos GitHub locales + bóveda Obsidian — y devuelve un resumen consolidado.
triggers:
  - "busca en todo"
  - "busca everywhere"
  - "buscar en repos y obsidian"
  - "qué tengo de"
  - "dónde menciono"
author: Felipe De Alba
---

# Skill: search-everywhere

## Cuándo usarlo
Cuando Felipe pida buscar un término, concepto, persona o tema y esa info pueda estar dispersa entre:
- **Notas Obsidian** (proyectos, daily, ideas, personas, briefings)
- **Repos GitHub locales** (código, READMEs, docs, scripts)
- **Memoria global Claude** (`~/.claude/memory/`)

## Cómo se usa
Activado por triggers como:
- *"Busca en todo lo que tengo sobre DEXTRA"*
- *"¿Dónde menciono a Germán Alexis?"*
- *"Qué tengo de licitaciones"*

## Procedimiento (instrucciones para el agente)

### Paso 1 — Buscar en bóveda Obsidian
Usa el tool `obsidian_simple_search` (si está disponible) o `search_files` del MCP filesystem con la consulta del usuario.

```
obsidian_simple_search(query="<TERMINO>")
```

Si el MCP `obsidian` no está disponible (Obsidian.app cerrado), cae al filesystem:
```
search_files(path="/Volumes/HIKSEMI 512/ObsidianVault", pattern="<TERMINO>")
```

### Paso 2 — Buscar en repos GitHub locales
Usa el tool `terminal` o `code_execution` para ejecutar grep recursivo:

```bash
# Repos a buscar (excluyendo binarios, node_modules, .git, venv)
for dir in ~/remote-desktop ~/syscom-odoo-sync ~/oceanwatch-security-dashboard \
          ~/DesktopCommanderMCP ~/openclaw-config ~/oceantech-legal \
          ~/oracle-vps-setup ~/presupuestos-inteligentes ~/ocean-tech-marketing \
          "/Volumes/HIKSEMI 512/neodata-control"; do
  if [ -d "$dir" ]; then
    grep -rIl --exclude-dir={.git,node_modules,venv,.venv,__pycache__,dist,build} \
         -i "<TERMINO>" "$dir" 2>/dev/null | head -5
  fi
done
```

### Paso 3 — Buscar en memoria global Claude
```
search_files(path="/Users/ingfelipe/.claude/memory", pattern="<TERMINO>")
```

### Paso 4 — Sintetizar resultados
Devuelve al usuario una respuesta estructurada en 3 bloques:

```markdown
## Resultados de búsqueda: "<TERMINO>"

### 📝 Bóveda Obsidian (N matches)
1. **[[nota1]]** — contexto breve (1 línea)
2. **[[nota2]]** — contexto breve

### 💻 Repos GitHub locales (N matches)
1. `repo/path/file.py:LINEA` — contexto
2. `repo/path/file.md:LINEA` — contexto

### 🧠 Memoria global (N matches)
1. `project_X.md` — contexto

### 🎯 Resumen ejecutivo
[1-2 oraciones consolidando hallazgos clave]
```

## Reglas
1. Si más de 10 matches por fuente, mostrar solo los 5 más relevantes con nota "(N totales)".
2. Si NO hay matches, sugerir términos alternativos usando sinónimos.
3. Si el término es un nombre de persona, también buscar en `Personas/` de Obsidian.
4. NO leer el archivo completo — solo el contexto del match (5-10 palabras antes/después).
5. NUNCA inventes resultados — si no hay matches reales, dilo.
6. Sugiere wikilinks `[[nombre-nota]]` para que Felipe pueda saltar directo en Obsidian.

## Output esperado
Concise, estructurado, con paths/wikilinks para que Felipe pueda profundizar manualmente si quiere.
