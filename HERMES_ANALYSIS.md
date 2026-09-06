# Hermes Agent - Análisis de Arquitectura & Plan de Clonación

**Fecha**: 2026-09-06  
**Repositorio Fuente**: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
**Objetivo**: Clonar y adaptar Hermes para Romulus Multi-Device Architecture

---

## 📋 Resumen Ejecutivo

Hermes Agent es un agente de IA automejorable con ciclo de aprendizaje cerrado, soporte multi-plataforma (Telegram, Discord, Slack, etc.) y capacidades avanzadas de memoria y skills.

### Características Clave Identificadas

✅ **Prompt Caching**: Conversaciones largas reutilizan prefijos cacheados  
✅ **Núcleo Estrecho**: Capacidades en los bordes, no en el core  
✅ **Plugin System**: Extensible architecture  
✅ **Multi-Gateway**: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Teams  
✅ **State Management**: SQLite WAL mode con FTS5 search  
✅ **Lazy Loading**: Minimize blast radius de dependencias  

---

## 🏗️ Componentes Reutilizables para Romulus

| Componente | Líneas | Reutilizable | Destino Romulus |
|-----------|--------|-------------|-----------------|
| `agent/` | ~5K | ✅ 100% | `romulus/core/agent/` |
| `hermes_state*.py` | ~20K | ✅ 95% | `romulus/core/state/` |
| `tools/` | ~15K | ✅ 90% | `romulus/core/tools/` |
| `gateway/` | ~8K | ✅ 85% | `romulus/platform/gateway/` |
| `skills/` | ~3K | ✅ 90% | `romulus/learning/skills/` |
| `providers/` | ~2K | ✅ 100% | `romulus/config/providers/` |
| `plugins/` | ~4K | ✅ 100% | `romulus/plugins/` |

---

## 📦 Stack Tecnológico

### Core Dependencies
- `openai==2.24.0` — LLM transport
- `pydantic==2.13.4` — Data validation
- `fastapi>=0.104.0,<1` — Web server
- `croniter==6.0.0` — Cron scheduling
- `prompt_toolkit==3.0.52` — CLI UI

### Optional (lazy-loaded)
- `messaging` — Telegram, Discord, Slack
- `voice` — STT (faster-whisper)
- `tts-premium` — TTS (elevenlabs)
- `mcp` — MCP servers
- `web` — Dashboard

---

## 🔄 Invariantes de Diseño (CRÍTICOS)

### ⚡ Invariante 1: Prompt Caching Must Survive
Long-lived conversations reuse cached prefixes every turn. Anything that mutates past context, swaps toolsets mid-session, or rebuilds the system prompt invalidates that cache.

**Implicación**:
- ❌ NO mutar system prompt durante sesión
- ❌ NO cambiar tool list mid-session (requiere `--now` flag)
- ✅ Slash commands son cache-aware

### 🎯 Invariante 2: Core is a Narrow Waist
Every model tool is sent on every API call, so the bar for a new core tool is HIGH.

**Implicación**:
- ✅ Preferir: Extend existing → CLI command → Skill → Service-gated tool → Plugin
- ❌ Último recurso: Agregar nuevo core tool

---

## 🧠 Ciclo de Aprendizaje

```
Task Execution
    ↓
Trajectory Logging
    ↓
Post-Task Evaluation
    ↓
Skill Creation (if pattern detected)
    ↓
Skill Refinement (improves on re-use)
    ↓
Next Session (skills disponibles)
```

---

## 🚀 Entrypoints Principales

```bash
hermes                 # CLI interactive
hermes model          # Model/provider selection
hermes tools          # Tool configuration
hermes gateway setup  # Messaging setup
hermes setup          # Full wizard
hermes update         # Self-update
hermes doctor         # Diagnostics
```

---

## 📊 Estructura de Configuración

```yaml
model:
  provider: openrouter
  base_url: https://openrouter.ai/api/v1
  default: nous/hermes-3-405b

agent:
  max_turns: 150
  reasoning_effort: medium

compression:
  enabled: true
  threshold: 0.50

display:
  tool_progress: all

session_reset:
  mode: none
```

---

## ✅ Roadmap: Phase 2-5

### Phase 2: Core Extraction
- Mover `agent/` → `romulus/core/agent/`
- Mover `hermes_state*.py` → `romulus/core/state/`
- Mover `providers/` → `romulus/config/providers/`
- Mover `tools/` → `romulus/core/tools/`

### Phase 3: Multi-Device Adaptation
- Crear `romulus/platform/` para device-specific logic
- Adaptar state sync para cloud backends
- Crear `romulus/sync/` para cross-device state

### Phase 4: Gateway Redesign
- Keep platform adapters
- Async/await consistency
- Webhook delivery confirmation

### Phase 5: Dashboard & Web UI
- Web-first design (vs TUI-first en Hermes)
- FastAPI backend + SPA redesign
- Device management UI

---

## 🎯 Decisiones Arquitectónicas: Hermes vs Romulus

| Decisión | Hermes | Romulus |
|----------|--------|---------|
| **Storage** | SQLite local | SQLite + Cloud sync |
| **UI Primary** | TUI | Web |
| **Dispositivos** | Single | Multi-device |
| **Skills** | Local | Cloud-backed + versioning |
| **Messaging** | Gateway loop sync | Async webhooks |
| **Caching** | Prompt cache | Cache + ETag versioning |

---

**Status**: ✅ COMPLETE - Ready for Phase 2: Core Extraction  
**Próximo Paso**: Crear estructura de directorios `romulus/` e iniciar extracción de componentes core
