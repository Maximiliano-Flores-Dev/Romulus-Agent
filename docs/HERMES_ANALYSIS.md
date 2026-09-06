# Hermes Agent - Análisis de Arquitectura & Plan de Clonación

**Fecha**: 2026-09-06  
**Repositorio Fuente**: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
**Objetivo**: Clonar y adaptar Hermes para Romulus Multi-Device Architecture

---

## 📋 Resumen Ejecutivo

Hermes Agent es un agente de IA automejorable con ciclo de aprendizaje cerrado, soporte multi-plataforma (Telegram, Discord, Slack, etc.) y capacidades avanzadas de memoria y skills. Su arquitectura está optimizada para:

- ✅ **Prompt caching** (conversaciones largas reutilizan prefijos cacheados)
- ✅ **Núcleo estrecho** (capacidades en los bordes, no en el core)
- ✅ **Plugin system** extensible
- ✅ **Multi-gateway** messaging
- ✅ **State management** robusto con SQLite WAL
- ✅ **Lazy dependency loading** (minimize blast radius)

---

## 🏗️ Arquitectura General

### Capas Principales

```
┌─────────────────────────────────────────────────────────────┐
│                    CLI / TUI / Gateway                       │
│    (hermes_cli/, tui_gateway/, gateway/platforms/)           │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│              Agent Runtime Core                              │
│  (run_agent.py, agent/, tools/, model_tools.py)              │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│              State Management Layer                          │
│  (hermes_state*.py — 20+ módulos para persistencia)          │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│           Backend Storage & Services                         │
│  (SQLite WAL, FTS5, Skills, Memory, Sessions)                │
└─────────────────────────────────────────────────────────────┘
```

### Directorio Principal

| Directorio | Propósito | Reutilizable |
|-----------|----------|-------------|
| `agent/` | Core agent loop, transports, model tooling | ✅ Alto |
| `hermes_cli/` | CLI setup wizard, config, auth | ✅ Medio (adaptar UI) |
| `hermes_state*.py` | State persistence, compression, repair | ✅ Alto |
| `tools/` | 40+ herramientas (terminal, files, web, etc) | ✅ Alto |
| `gateway/` | Multi-platform messaging (Telegram, Discord) | ✅ Alto |
| `skills/` | Procedural memory, learned behaviors | ✅ Alto |
| `plugins/` | Plugin system, providers, adapters | ✅ Alto |
| `providers/` | LLM provider registry & profiles | ✅ Alto |
| `tui_gateway/` | Terminal UI | ⚠️ Medio (web-first para Romulus) |
| `web/` | Dashboard (FastAPI + SPA) | ⚠️ Bajo (Romulus will redesign) |
| `tests/` | Test suite | ✅ Alto |
| `nix/` | Nix packaging (skip for Romulus) | ❌ Bajo |

---

## 🔑 Componentes Clave Reutilizables

### 1. **Agent Core** (`agent/`, `run_agent.py`)

**Características**:
- Agent loop con manejo de context compression
- Message history con role alternation enforcement
- Tool calling orchestration
- Model streaming

**Reutilizable**: ✅ **100%** — Mover a `romulus/core/agent/`

```python
# Estructura base esperada
romulus/core/agent/
  ├── agent_loop.py
  ├── transports/
  │   ├── chat_completions.py
  │   └── model_io.py
  └── model_metadata.py
```

---

### 2. **State Management** (`hermes_state*.py`)

**Archivo** | **Líneas** | **Propósito**
---|---|---
`hermes_state.py` | ~2,800 | Main state class, session lifecycle
`hermes_state_messages.py` | ~2,200 | Message storage, compression hooks
`hermes_state_sessions.py` | ~2,400 | Session index, search
`hermes_state_compression.py` | ~1,200 | Context compression w/ LLM
`hermes_state_search.py` | ~2,100 | FTS5 search, semantic indexing
`hermes_state_guard.py` | ~200 | Concurrency guards
`hermes_state_dbfile.py` | ~500 | SQLite lifecycle, WAL mode
`hermes_state_repair.py` | ~2,000 | DB repair & corruption recovery

**Reutilizable**: ✅ **95%** — Adaptar para multi-device state sync

---

### 3. **Tools** (`tools/`)

**40+ herramientas disponibles**:
- Terminal: `run_command`, `read_file`, `write_file`
- Web: `web_search`, `fetch_url`, `browser_use`
- Comms: `send_message`, `email`
- System: `code_execution`, `cron`, `memory`

**Reutilizable**: ✅ **90%** — Keep tool schema, adapt platform handlers

---

### 4. **Multi-Gateway Messaging** (`gateway/`)

**Plataformas soportadas**:
- Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Teams, WeChat, Dingtalk, Feishu, Home Assistant, SMS

**Reutilizable**: ✅ **85%** — Platform-specific logic reusable, adapt authentication

---

### 5. **Skill System** (`skills/`)

**Concepto**: Procedural memory learned from task execution

**Reutilizable**: ✅ **90%** — Move to `romulus/learning/skills/`

---

### 6. **Provider Registry** (`providers/`)

**Concepto**: Single source of truth for LLM provider profiles

**Reutilizable**: ✅ **100%** — Move to `romulus/config/providers/`

---

## 📦 Dependencias Clave

### Core Dependencies (siempre instaladas)

```toml
openai==2.24.0              # LLM transport
pydantic==2.13.4            # Data validation
prompt_toolkit==3.0.52      # CLI UI
croniter==6.0.0             # Cron scheduling
httpx[socks]==0.28.1        # HTTP client
fastapi>=0.104.0,<1         # Web server
```

### Optional (lazy-loaded)

```toml
[messaging]  # Telegram, Discord, Slack
[voice]      # STT (faster-whisper)
[tts-premium] # TTS (elevenlabs)
[mcp]        # MCP servers
[web]        # Dashboard
```

**Estrategia para Romulus**: Mantener separación de core vs. extras, lazy-load en first-use

---

## 🔄 Invariantes de Diseño (CRÍTICOS)

### Invariante 1: Prompt Caching Must Survive

> "A long-lived conversation reuses a cached prefix every turn. Anything that mutates past context, swaps toolsets, reloads memories, or rebuilds the system prompt mid-conversation invalidates that cache."

**Implicación**: 
- ❌ NO mutar system prompt durante sesión
- ❌ NO cambiar tool list mid-session (requiere `--now` flag)
- ✅ Slash commands que usen `/skills install --now` son cache-aware

### Invariante 2: Core is a Narrow Waist

> "Every model tool is sent on every API call, so the bar for a new core tool is high."

**Implicación**:
- ✅ Preferir: Extend existing code → CLI command + skill → service-gated tool → plugin
- ❌ Último recurso: Agregar nuevo core tool

---

## 🧠 Memoria & Learning Loop

### Memoria Persistente

- **SOUL.md**: User persona (manual)
- **MEMORY.md**: Procedural facts (agent-curated)
- **USER.md**: User preferences (manual + agent updates)
- **Skills**: Learned procedures from complex tasks

### Ciclo de Aprendizaje

```
Task Execution
    ↓
Trajectory Logging (agent tracks steps)
    ↓
Post-Task Evaluation (did it work?)
    ↓
Skill Creation (if pattern detected)
    ↓
Skill Refinement (improves on re-use)
    ↓
Next Session (skills are available)
```

---

## 🚀 Entrypoints Principales

```bash
hermes                      # CLI interactive
hermes model               # Model/provider selection
hermes tools               # Tool configuration
hermes gateway setup       # Messaging setup
hermes setup               # Full wizard
hermes update              # Self-update
hermes doctor              # Diagnostics
```

---

## 📊 Configuración

### Archivos de Configuración

```
~/.hermes/
├── config.yaml             # Main config (model, tools, compression)
├── .env                    # Secrets (API keys)
├── sessions.db             # SQLite with WAL mode
├── logs/
│   ├── agent.log
│   └── tool_calls.log
└── skills/                 # User-created & auto-generated
    ├── learned/
    └── openclaw-imports/
```

### Estructura config.yaml

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

## ✅ Próximos Pasos (Phase 2+)

1. **Phase 2: Core Extraction**
   - Mover `agent/` → `romulus/core/agent/`
   - Mover `hermes_state*.py` → `romulus/core/state/`
   - Mover `providers/` → `romulus/config/providers/`
   - Mover `tools/` → `romulus/core/tools/`

2. **Phase 3: Multi-Device Adaptation**
   - Crear `romulus/platform/` para device-specific logic
   - Adaptar state sync para cloud backends
   - Crear `romulus/sync/` para cross-device state

3. **Phase 4: Gateway Redesign**
   - Keep platform adapters (Telegram, Discord, etc.)
   - Redesign messaging loop para async/await consistency
   - Add webhook delivery confirmation

4. **Phase 5: Dashboard & Web UI**
   - Replace Hermes TUI con Romulus web-first design
   - Keep FastAPI backend, redesign SPA frontend
   - Add device management UI

---

## 📚 Referencias Clave

- **AGENTS.md**: Development guide (design philosophy)
- **CONTRIBUTING.md**: Code style, PR process
- **hermes-agent.nousresearch.com/docs**: Full documentation
- **GitHub Issues**: Currently 40K open (lots of feature requests)

---

## 🎯 Decisiones Arquitectónicas para Romulus

| Decisión | Hermes | Romulus |
|----------|--------|---------|
| Storage | SQLite local | SQLite + Cloud sync |
| UI | TUI primary | Web primary + TUI optional |
| Devices | Single | Multi-device with state sync |
| Skills | Local memory | Cloud-backed with versioning |
| Messaging | Gateway loop | Async webhook handlers |
| Caching | Prompt cache only | Cache + ETag versioning |

---

**Documento creado por**: Fase 1 - Hermes Agent Clone Analysis  
**Última actualización**: 2026-09-06  
**Status**: ✅ COMPLETE - Ready for Phase 2: Core Extraction
