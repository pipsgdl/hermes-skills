---
name: health-self-check
version: 1.0.0
description: Auto-diagnóstico de la salud del propio Hermes y de las demás instancias del stack — verifica conectividad, sockets WebSocket, gateway alive, modelos LLM disponibles, MCPs cargados. Si detecta algo zombie, sugiere restart.
triggers:
  - "estás bien"
  - "como estas tu"
  - "auto-diagnóstico"
  - "health check"
  - "estatus tuyo"
  - "estás vivo"
  - "salud del sistema"
author: Felipe De Alba
---

# Skill: health-self-check

## Cuándo usarlo
Cuando Felipe pregunta por la salud del sistema Hermes (cualquier instancia) o quiere un diagnóstico rápido antes de empezar trabajo crítico.

## Procedimiento

### Paso 1 — Autodiagnóstico local
```bash
# 1.1) ¿Mi proceso está activo?
systemctl --user is-active hermes-gateway.service 2>/dev/null

# 1.2) ¿Cuántos sockets SSL ESTABLISHED tengo?
PID=$(pgrep -f "hermes_cli.main gateway" | head -1)
ss -tnp 2>/dev/null | grep "pid=$PID" | grep ESTAB | grep -c ":443"
# Esperado: ≥ 2 si Discord + Telegram conectados

# 1.3) ¿Cuándo procesé el último mensaje real (no auto-ping)?
grep -E "inbound message" ~/.hermes/logs/gateway.log | grep -v "healthcheck health_" | tail -1
```

### Paso 2 — Verificar las OTRAS instancias Hermes (si aplica)
Si soy Hermes Mac Mini (Obsidian) → preguntar a Rogger/Ozzy via SSH:
```bash
ssh ocean-ai 'systemctl --user is-active hermes-gateway.service'
ssh ocean-worker 'systemctl --user is-active hermes-gateway.service'
```

Si soy Rogger (Zack) → verificar Mac Mini + Ozzy (mismo método).

### Paso 3 — Verificar modelos LLM
```bash
# DeepSeek API
curl -s -m 5 -H "Authorization: Bearer $OPENAI_API_KEY" \
    https://api.deepseek.com/v1/models | grep -o '"id":"[^"]*"' | head -3

# Ollama local (solo en Rogger)
curl -s -m 3 http://127.0.0.1:11434/api/tags | python3 -c "import json,sys; print(len(json.load(sys.stdin)['models']))"
```

### Paso 4 — Verificar MCPs cargados
```bash
hermes mcp list 2>&1 | grep -cE "enabled|connected"
```

### Paso 5 — Reportar en formato unificado

```markdown
## 🩺 Health check — {Instance Name}

### Mi estado
- Service: {active/inactive}
- Sockets SSL activos: {N}
- Último mensaje real: {timestamp}
- MCPs cargados: {N}
- Modelo LLM principal: {responde/no responde}

### Otras instancias
- **Rogger (Zack)**: {OK/DOWN}
- **Ozzy (Mr Crowley)**: {OK/DOWN}
- **Mac Mini (Obsidian)**: {OK/DOWN}

### Servicios externos
- Telegram getMe: {OK/FAIL}
- Discord gateway: {OK/FAIL}

### Watchdog state
- Zombie detector (Mac Mini cron 5min): última corrida {timestamp}
- Self-ping (cron 1h): último round-trip {timestamp}
- Restart preventivo (Rogger 3am): último {date}

### Veredicto
{🟢 Todo OK / 🟡 Algo degradado / 🔴 Crítico}

### Acción sugerida
{ninguna / restart hermes-gateway / investigar logs / contactar a Felipe}
```

## Reglas
1. **NO restartear nunca el propio proceso** desde este skill (sería suicidio del agente). Si detectas que tú mismo estás mal, escríbele a Felipe.
2. Para restartear OTRA instancia: SÍ está permitido vía SSH si detectas zombie.
3. Si Telegram getMe falla pero los sockets internos están vivos: avisa, no restartees (puede ser blip de red).
4. Si todos los checks pasan: respuesta breve "🟢 todo OK" + 1 línea de stats.
5. Si algo crítico: respuesta detallada con próximo paso accionable.

## Output esperado
1 párrafo breve si todo bien · dashboard detallado si hay problemas.
