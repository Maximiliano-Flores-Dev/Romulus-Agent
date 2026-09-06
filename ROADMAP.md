# Romulus Agent — Plan de Desarrollo

## 1. Clonar Hermes Agent

- Clonar el repositorio de Hermes Agent como base de código inicial.
- Analizar su arquitectura, entrypoints, sistema de herramientas, sistema de memoria y ciclo de vida del agente.
- Identificar qué componentes pueden reutilizarse.
- Identificar qué componentes necesitan ser reemplazados o refactorizados.
- Preservar el historial del proyecto upstream donde sea práctico.

## 2. Crear la Estructura de Proyecto de Romulus

- Adaptar la estructura del proyecto existente para Romulus Agent.
- Separar el Agent Core de las implementaciones específicas de plataforma.
- Establecer interfaces comunes para:
  - Agents
  - Models
  - Tools
  - Memory
  - Runtimes
  - Events
  - Permissions
- Eliminar componentes innecesarios específicos de Hermes.

## 3. Agregar Branding y Assets Personalizados

- Crear la identidad visual de Romulus Agent.
- Agregar logos e iconos personalizados.
- Agregar banners de proyecto y assets de documentación.
- Reemplazar branding de Hermes donde sea apropiado.
- Agregar branding personalizado para CLI/TUI.
- Preparar assets para futuras aplicaciones móviles y de escritorio.

## 4. Implementar la Arquitectura Multi-Dispositivo

- Crear una interfaz `Runtime` común.
- Separar la funcionalidad específica de plataforma del Agent Core.
- Implementar adaptadores de runtime independientes.
- Asegurar que la misma arquitectura de agente pueda operar en diferentes dispositivos.
- Prevenir que la lógica específica de plataforma se filtre en el Agent Core.

### Objetivos de runtime iniciales:

- Android
- Linux
- Windows
- macOS

## 5. Implementar el Sistema de Capacidades de Dispositivo

Crear un sistema que permita a cada runtime exponer sus capacidades disponibles.

### Capacidades de ejemplo:

- `filesystem`
- `shell`
- `notifications`
- `camera`
- `microphone`
- `clipboard`
- `sensors`
- `networking`
- `process_management`
- `application_control`
- `browser_automation`

El agente debe determinar dinámicamente qué capacidades están disponibles en el dispositivo actual.

## 6. Implementar el Sistema de Permisos

Crear un sistema centralizado de permisos para las herramientas del agente.

### Niveles de permiso:

- `READ_ONLY`
- `USER_APPROVAL`
- `PRIVILEGED`
- `SYSTEM`

### Requisitos:

- Requerir aprobación del usuario para operaciones sensibles.
- Prevenir la ejecución de herramientas no autorizadas.
- Aplicar permisos a nivel de runtime.
- Nunca confiar exclusivamente en instrucciones del modelo para seguridad.
- Permitir diferentes políticas de permiso por dispositivo.

## 7. Crear el Runtime de Android

Iniciar la implementación móvil con Android.

### Implementar acceso controlado a:

- Archivos
- Notificaciones
- Portapapeles
- Sensores
- Cámara
- Micrófono
- Red
- Información del dispositivo
- Tareas en segundo plano

### Prioridades:

- Bajo uso de RAM
- Bajo consumo de batería
- Permisos mínimos
- Funcionalidad offline donde sea posible
- Operación confiable bajo condiciones de red limitadas

## 8. Crear el Runtime de Escritorio

Iniciar la implementación de escritorio con Linux.

### Implementar:

- Ejecución de shell
- Acceso al sistema de archivos
- Gestión de procesos
- Lanzamiento de aplicaciones
- Gestión de ventanas
- Portapapeles
- Notificaciones
- Información de hardware
- Información de red
- Integración de modelo local

Preparar la arquitectura de runtime para futuros adaptadores de Windows y macOS.

## 9. Implementar Identidad de Dispositivo

Crear una identidad única para cada instalación de Romulus.

### Cada dispositivo debe exponer metadata tal como:

- Device ID
- Device name
- Platform
- OS version
- Romulus version
- Available capabilities
- Connection status
- Runtime version

Permitir que el agente reconozca y distinga entre sus diferentes dispositivos.

## 10. Implementar Comunicación Multi-Dispositivo

Crear la capa de comunicación Romulus-to-Romulus.

### Implementar:

- Descubrimiento de dispositivos
- Autenticación de dispositivos
- Conexiones seguras
- Emparejamiento de dispositivos
- Intercambio de mensajes
- Transmisión de tareas
- Transmisión de resultados
- Gestión del estado de conexión

El protocolo de comunicación debe estar diseñado para soportar tanto conexiones de red local como remotas.

## 11. Implementar Delegación de Tareas

Permitir que Romulus delegue tareas entre dispositivos.

### Ejemplo:

```
Teléfono
  ↓
Romulus Agent
  ↓
Task Delegation
  ↓
Desktop PC
  ↓
GPU / Recursos Locales
  ↓
Resultado
  ↓
Teléfono
```

- El agente debe determinar automáticamente qué dispositivo es mejor para una tarea.
- Transmitir tareas de forma segura.
- Manejar fallos de red y reconexiones.
- Sincronizar el estado entre dispositivos.
- Permitir al usuario anular decisiones de delegación.

---

## Fases de Implementación

| # | Fase | Estado |
|---|------|--------|
| 1 | Clonar Hermes Agent | ⏳ Pendiente |
| 2 | Estructura de Proyecto | ⏳ Pendiente |
| 3 | Branding y Assets | ⏳ Pendiente |
| 4 | Arquitectura Multi-Dispositivo | ⏳ Pendiente |
| 5 | Sistema de Capacidades | ⏳ Pendiente |
| 6 | Sistema de Permisos | ⏳ Pendiente |
| 7 | Runtime Android | ⏳ Pendiente |
| 8 | Runtime Escritorio (Linux) | ⏳ Pendiente |
| 9 | Identidad de Dispositivo | ⏳ Pendiente |
| 10 | Comunicación Multi-Dispositivo | ⏳ Pendiente |
| 11 | Delegación de Tareas | ⏳ Pendiente |

---

## Consideraciones de Arquitectura

### Principios Fundamentales

- **Separation of Concerns**: Agent Core completamente separado de implementaciones de runtime.
- **Security First**: Los permisos se aplican a nivel de runtime, no de modelo.
- **Device Agnostic**: El mismo agente debe funcionar en cualquier dispositivo compatible.
- **Offline First**: Máximo funcionamiento sin conectividad.
- **Resource Aware**: Adaptarse a limitaciones de recursos del dispositivo.

### Stack Tecnológico Recomendado

- **Agent Core**: Lenguaje agnóstico (Go, Rust, o similar)
- **Android Runtime**: Kotlin/Java + JNI
- **Desktop Runtime**: Go/Rust + bindings
- **Communication**: Protocol Buffers + gRPC o similar
- **Storage**: SQLite (local), con sync a backend

---

## Hitos Principales

- **Hito 1**: Clonar y analizar Hermes Agent ✓
- **Hito 2**: Estructura base de Romulus funcional
- **Hito 3**: Runtime Android MVP
- **Hito 4**: Runtime Desktop MVP
- **Hito 5**: Multi-device communication MVP
- **Hito 6**: Delegación de tareas funcional
- **Hito 7**: Versión 1.0 lista para producción

---

**Última actualización:** Septiembre 6, 2026
