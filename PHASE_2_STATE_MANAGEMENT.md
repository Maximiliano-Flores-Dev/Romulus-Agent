# Phase 2: Multi-Device State Management Architecture

**Fecha**: 2026-09-06  
**Objetivo**: Adaptar `hermes_state*.py` para arquitectura multi-dispositivo con sincronización cloud  
**Status**: 🚀 IN PROGRESS

---

## 📋 Resumen Ejecutivo

Hermes usa SQLite local con WAL mode y FTS5. Para Romulus, necesitamos:

1. **Local SQLite** — Mantener rendimiento local
2. **Cloud Sync Layer** — Replicar cambios a backend
3. **Conflict Resolution** — Resolver divergencias entre dispositivos
4. **Offline Support** — Funcionar sin conectividad
5. **Change Tracking** — Registrar qué cambió, quién, cuándo

---

## 🏗️ Hermes State Architecture (Actual)

```
┌─────────────────────────────────────────────┐
│    Application (agent, CLI, gateway)        │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│  HermesState (hermes_state.py)              │
│  ├── Session Management                     │
│  ├── Message Storage                        │
│  ├── Compression                            │
│  └── Search (FTS5)                          │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│  SQLite Database (local only)               │
│  ├── sessions table (index)                 │
│  ├── messages table (searchable)            │
│  ├── memory table (user facts)              │
│  └── WAL mode (crash-safe)                  │
└─────────────────────────────────────────────┘
```

### Módulos de State en Hermes

```
hermes_state.py              # Main orchestrator
hermes_state_messages.py     # Message CRUD + compression hooks
hermes_state_sessions.py     # Session lifecycle (create, resume, list, delete)
hermes_state_search.py       # FTS5 full-text search + semantic search
hermes_state_compression.py  # Context compression with LLM
hermes_state_dbfile.py       # SQLite lifecycle (open, close, WAL, checkpoint)
hermes_state_repair.py       # DB corruption recovery
hermes_state_guard.py        # Concurrency guards (locks, mutexes)
hermes_state_portability.py  # Export/import sessions
hermes_state_wal.py          # WAL mode management
hermes_state_registry.py     # Session registry / indexing
```

---

## 🎯 Romulus Multi-Device State Architecture (NUEVA)

```
┌──────────────────────────────────────────────────────────────┐
│         Application (agent, web, mobile, CLI)                │
└────────────────┬─────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────┐
│  RomulusState (romulus/core/state/state_manager.py)          │
│  ├── Local State Layer (↓)                                    │
│  ├── Sync Controller (↔)                                      │
│  ├── Conflict Resolution (⚔)                                  │
│  └── Change Tracking (+)                                      │
└────────────────┬─────────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
┌───────▼────────┐  ┌─────▼──────────────┐
│  Local SQLite  │  │  Sync Controller   │
│  ├── sessions  │  │  ├── Queue         │
│  ├── messages  │  │  ├── Conflict Res  │
│  ├── memory    │  │  └── Change Log    │
│  └── metadata  │  └─────┬──────────────┘
│                │        │
│  WAL mode      │   ┌────▼──────────────┐
│  Offline-safe  │   │  Cloud Backend    │
└───────────────┘   │  ├── MongoDB       │
                    │  ├── Firebase      │
                    │  └── Custom API    │
                    └───────────────────┘
```

---

## 🔧 Componentes Nuevos Necesarios

### 1. **Change Tracking Layer**

Registrar TODAS las mutaciones:
- Qué cambió (tabla, fila, columna)
- Cuándo cambió (timestamp + version vector)
- Quién lo cambió (device_id + user_id)
- Por qué cambió (operation type: INSERT/UPDATE/DELETE)

```python
# romulus/core/state/change_tracking.py

class ChangeLog:
    """Track mutations for sync"""
    session_id: str
    timestamp: datetime
    device_id: str                    # ← Device que hizo el cambio
    operation: Literal["INSERT", "UPDATE", "DELETE"]
    table_name: str
    row_id: str
    old_values: dict                  # ← Para conflict resolution
    new_values: dict
    vector_clock: dict                # ← Lamport clock para causalidad
    synced: bool = False              # ← Flag para saber qué falta subir
```

### 2. **Sync Queue**

FIFO de cambios pendientes:

```python
# romulus/core/state/sync_queue.py

class SyncQueue:
    """Queue of changes waiting to be synced"""
    
    async def enqueue(self, change: ChangeLog) -> None:
        """Add local change to queue"""
        
    async def dequeue(self, batch_size: int = 100) -> list[ChangeLog]:
        """Get changes ready to send"""
        
    async def mark_synced(self, change_ids: list[str]) -> None:
        """Mark changes as successfully synced"""
        
    async def get_pending_count(self) -> int:
        """How many changes waiting to sync"""
```

### 3. **Conflict Resolution Engine**

Resolver divergencias cuando 2+ dispositivos editan lo mismo:

```python
# romulus/core/state/conflict_resolution.py

class ConflictResolver:
    """Resolve conflicts between device states"""
    
    STRATEGIES = {
        "last-write-wins": lambda local, remote: remote if remote.timestamp > local.timestamp else local,
        "device-priority": lambda local, remote, device_order: ...,
        "custom": lambda local, remote, metadata: custom_resolver(local, remote, metadata),
    }
    
    async def resolve(
        self,
        local_value: Any,
        remote_value: Any,
        strategy: str = "last-write-wins",
        metadata: dict = None
    ) -> tuple[Any, bool]:
        """
        Resolve conflict and return (resolved_value, did_merge)
        """
        
    async def detect_conflict(
        self,
        local_change: ChangeLog,
        remote_change: ChangeLog
    ) -> bool:
        """Detect if two changes conflict"""
```

### 4. **Device-Aware State Layer**

Rastrear qué dispositivo hace qué:

```python
# romulus/core/state/device_aware.py

class DeviceAwareState:
    """State layer aware of multi-device context"""
    
    device_id: str                    # UUID of this device
    device_name: str                  # "Max's MacBook Pro"
    device_type: str                  # "web", "mobile", "cli", "desktop"
    last_sync: datetime
    sync_version: int                 # Vector clock version
    
    async def get_device_context(self) -> dict:
        """Get current device metadata for sync"""
        
    async def set_device_context(self, name: str, type: str) -> None:
        """Update device metadata"""
```

### 5. **Sync Orchestrator**

Coordinar sincronización:

```python
# romulus/core/state/sync_orchestrator.py

class SyncOrchestrator:
    """Coordinate sync between local and cloud"""
    
    async def sync_up(self) -> SyncResult:
        """
        Push local changes to cloud:
        1. Get pending changes from sync queue
        2. Send to cloud API
        3. Handle conflicts
        4. Mark as synced
        """
        
    async def sync_down(self) -> SyncResult:
        """
        Pull remote changes from cloud:
        1. Fetch changes since last_sync
        2. Detect local conflicts
        3. Resolve conflicts
        4. Apply to local DB
        5. Update last_sync timestamp
        """
        
    async def full_sync(self) -> SyncResult:
        """Bidirectional sync (up then down)"""
        
    async def watch_for_changes(self) -> AsyncIterator[ChangeLog]:
        """Stream of local changes (for real-time sync)"""
```

---

## 🔄 Hermes → Romulus State Migration Strategy

### Paso 1: Crear Layer de Abstracción

```python
# romulus/core/state/persistence.py

class PersistenceBackend(ABC):
    """Abstract backend for state storage"""
    
    @abstractmethod
    async def write_session(self, session: Session) -> None: ...
    
    @abstractmethod
    async def read_session(self, session_id: str) -> Session: ...
    
    @abstractmethod
    async def write_message(self, msg: Message) -> None: ...
    
    # etc...

class LocalSQLiteBackend(PersistenceBackend):
    """Direct port from Hermes"""
    async def write_session(self, session: Session) -> None:
        # Same as hermes_state_sessions.py
        
class CloudSyncBackend(PersistenceBackend):
    """Cloud-first with local cache"""
    async def write_session(self, session: Session) -> None:
        # Write to local cache
        # Queue for cloud sync
        # Return immediately
```

### Paso 2: Adaptar hermes_state Modules

```
romulus/core/state/
├── __init__.py
├── state_manager.py              # Main orchestrator (← hermes_state.py)
├── persistence.py                # Abstract backend
├── backends/
│   ├── sqlite_local.py           # Direct port from Hermes
│   └── cloud_sync.py             # New: Cloud + local cache
├── change_tracking.py            # NEW
├── sync_queue.py                 # NEW
├── sync_orchestrator.py          # NEW
├── conflict_resolution.py        # NEW
├── device_aware.py               # NEW
├── messages.py                   # ← hermes_state_messages.py
├── sessions.py                   # ← hermes_state_sessions.py
├── search.py                     # ← hermes_state_search.py
├── compression.py                # ← hermes_state_compression.py
├── repair.py                     # ← hermes_state_repair.py
├── guard.py                      # ← hermes_state_guard.py
└── portability.py                # ← hermes_state_portability.py
```

---

## 💾 Schema Additions for Multi-Device

### Nueva tabla: `device_sync_metadata`

```sql
CREATE TABLE device_sync_metadata (
    id TEXT PRIMARY KEY,
    device_id TEXT NOT NULL,
    device_name TEXT,
    device_type TEXT,              -- web, mobile, cli, desktop
    last_sync TIMESTAMP,
    last_sync_version INTEGER,     -- Vector clock
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    UNIQUE(device_id)
);
```

### Nueva tabla: `change_log`

```sql
CREATE TABLE change_log (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    device_id TEXT NOT NULL,
    operation TEXT NOT NULL,        -- INSERT, UPDATE, DELETE
    table_name TEXT NOT NULL,
    row_id TEXT NOT NULL,
    old_values JSON,
    new_values JSON,
    vector_clock JSON,              -- Lamport clock
    synced BOOLEAN DEFAULT FALSE,
    synced_at TIMESTAMP,
    created_at TIMESTAMP,
    FOREIGN KEY(session_id) REFERENCES sessions(id),
    FOREIGN KEY(device_id) REFERENCES device_sync_metadata(device_id),
    INDEX(session_id),
    INDEX(synced),
    INDEX(created_at)
);
```

### Nueva tabla: `sync_queue`

```sql
CREATE TABLE sync_queue (
    id TEXT PRIMARY KEY,
    change_log_id TEXT NOT NULL,
    priority INTEGER DEFAULT 0,     -- Higher = sync sooner
    retry_count INTEGER DEFAULT 0,
    last_error TEXT,
    queued_at TIMESTAMP,
    FOREIGN KEY(change_log_id) REFERENCES change_log(id),
    INDEX(priority),
    INDEX(queued_at)
);
```

### Nueva tabla: `conflict_resolution`

```sql
CREATE TABLE conflict_resolution (
    id TEXT PRIMARY KEY,
    session_id TEXT,
    table_name TEXT,
    row_id TEXT,
    local_version JSON,
    remote_version JSON,
    resolution TEXT,                -- "local", "remote", "merged", "manual"
    resolved_at TIMESTAMP,
    resolved_by_device_id TEXT,
    created_at TIMESTAMP,
    FOREIGN KEY(session_id) REFERENCES sessions(id)
);
```

---

## 🔐 Offline-First Architecture

### Cuando NO hay conectividad:

```python
# romulus/core/state/offline_mode.py

class OfflineMode:
    """Continue working without cloud connectivity"""
    
    async def start_offline(self) -> None:
        """Switch to offline mode"""
        self.is_offline = True
        # Disable auto-sync
        # Continue using local DB
        
    async def write_message(self, msg: Message) -> None:
        """Write locally, queue for sync when back online"""
        await self.local_db.write(msg)
        await self.sync_queue.enqueue(ChangeLog(...))
        
    async def get_sessions(self) -> list[Session]:
        """Read from local cache only"""
        return await self.local_db.read_sessions()
        
    async def resume_online(self) -> SyncResult:
        """When connectivity returns, catch up"""
        # 1. Push pending changes
        # 2. Pull remote changes
        # 3. Resolve conflicts
        # 4. Update state
```

---

## 🧪 Testing Strategy for Multi-Device

### Test: Concurrent edits on two devices

```python
# tests/test_state_sync.py

@pytest.mark.asyncio
async def test_concurrent_edits_conflict_resolution():
    """Two devices edit same message concurrently"""
    
    # Setup
    device1 = await create_device("web", "Max's Browser")
    device2 = await create_device("mobile", "Max's Phone")
    
    # Device 1: Edit message at 12:00:00
    await device1.update_message(msg_id="msg-123", text="Edited on web")
    change1 = await device1.get_pending_changes()[0]
    assert change1.device_id == device1.device_id
    
    # Device 2: Edit SAME message at 12:00:01
    await device2.update_message(msg_id="msg-123", text="Edited on mobile")
    change2 = await device2.get_pending_changes()[0]
    
    # Sync with conflict resolution (last-write-wins)
    result = await sync_with_conflict_resolution([change1, change2])
    
    # Expect: Device 2's version wins (later timestamp)
    assert result.resolved_message.text == "Edited on mobile"
    assert result.conflict_log is not None
    assert result.conflict_log.resolution == "last-write-wins"
```

### Test: Offline → Online transition

```python
@pytest.mark.asyncio
async def test_offline_to_online_sync():
    """Device works offline, then syncs when back online"""
    
    # Go offline
    await state.start_offline()
    assert state.is_offline == True
    
    # Make 50 changes offline
    for i in range(50):
        await state.add_message(Message(text=f"Offline message {i}"))
    
    # Check queue has 50 pending
    pending = await state.sync_queue.get_pending_count()
    assert pending == 50
    
    # Come back online
    result = await state.resume_online()
    
    # Verify all 50 synced
    assert result.synced_count == 50
    assert await state.sync_queue.get_pending_count() == 0
    assert state.is_offline == False
```

---

## 🚀 Fase 3: Cloud Backend Integration

### API Endpoints Requeridos

```
POST   /api/sync/init                    # Start sync session
POST   /api/sync/changes                 # Upload changes
GET    /api/sync/changes                 # Download changes
PUT    /api/sync/conflicts/{conflict_id} # Resolve conflict
GET    /api/sync/status                  # Sync status
DELETE /api/sync/session/{session_id}    # Archive session
```

### Ejemplo: Upload Changes Endpoint

```python
# romulus_backend/api/sync.py

@router.post("/sync/changes")
async def upload_changes(
    device_id: str,
    changes: list[ChangeLog],
    vector_clock: dict
) -> SyncResponse:
    """
    Device sends local changes to cloud
    
    1. Validate device signature
    2. Check for conflicts with other devices
    3. Apply conflict resolution
    4. Persist to backend storage
    5. Return ack + remote changes
    """
    
    for change in changes:
        # Detect conflict
        conflicts = await detect_conflicts(change)
        
        if conflicts:
            # Store for manual resolution
            await store_conflict(change, conflicts)
        else:
            # Apply directly
            await apply_change(change)
    
    # Return remote changes to pull
    remote_changes = await get_remote_changes_since(vector_clock)
    
    return SyncResponse(
        acked_count=len(changes),
        remote_changes=remote_changes,
        conflicts=conflicts,
    )
```

---

## 📊 Implementation Roadmap

| Fase | Componente | Timeline | Owner |
|------|-----------|----------|-------|
| 2.1 | Change Tracking Layer | Week 1 | - |
| 2.2 | Sync Queue | Week 1 | - |
| 2.3 | Local SQLite Backend | Week 2 | - |
| 2.4 | Conflict Resolution | Week 2 | - |
| 2.5 | Device-Aware State | Week 2 | - |
| 2.6 | Offline Mode | Week 3 | - |
| 2.7 | Sync Orchestrator | Week 3 | - |
| 3.1 | Cloud API Design | Week 4 | - |
| 3.2 | Cloud Sync Backend | Week 4-5 | - |
| 3.3 | Integration Tests | Week 5 | - |

---

## ✅ Success Criteria

- ✅ Local SQLite continues to work exactly like Hermes
- ✅ All changes tracked with metadata (device, timestamp, reason)
- ✅ Conflicts detected and resolved automatically
- ✅ Works offline (queue builds up, syncs when online)
- ✅ Device A edits message → Device B sees change in <2s
- ✅ No data loss on either device
- ✅ Causal consistency maintained (change ordering preserved)

---

**Próximo Paso**: ¿Comenzamos con **Change Tracking Layer** (2.1)?
