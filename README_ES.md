# Claude Code (Archivo de Código Fuente Filtrado)

[English](./README.md) | [中文](./README_ZH.md) | [繁體中文](./README_ZH_TW.md) | [조선어](./README_KO.md) | [日本語](./README_JA.md) | Español | [Русский](./README_RU.md)

<p align="center">
  <img src="https://img.shields.io/badge/status-archived%20%2B%20rebuilding-blue" />
  <img src="https://img.shields.io/badge/runtime-Bun%20%2F%20Node-black" />
  <img src="https://img.shields.io/badge/language-TypeScript%20→%20Rust-orange" />
  <img src="https://img.shields.io/badge/focus-Code%20Agent%20Systems-green" />
</p>

---

## 🚨 ACTUALIZACIÓN → Reconstruyendo en Rust

> **Construye mejores herramientas de harness, no solo acumules Claude Code filtrado.  
No colecciones código. Crea resultados.**

Estamos **reconstruyendo activamente Claude Code en Rust**, con el objetivo de crear un **sistema de agentes de código más potente, más fiable y listo para producción**.

👉 Si te interesa la implementación en Rust y el progreso más reciente:  
https://github.com/claw-cli/claw-code-cli

---

## 📦 ¿Qué es este repositorio?

Este repositorio es una **instantánea archivada del código fuente de Claude Code v2.1.88**, que se volvió accesible públicamente a través de un sourcemap incluido en el paquete de npm.

Sirve como:

- 📚 Un **artefacto de investigación** para estudiar sistemas de agentes de IA del mundo real  
- 🔍 Una **implementación de referencia** de arquitectura de tooling LLM a gran escala  
- 🧪 Una **base para reconstruir sistemas mejores**

---

El 31 de marzo de 2026, Chaofan Shou detectó algo inusual: todo el código fuente de Claude Code había sido publicado silenciosamente en npm, oculto dentro de un archivo sourcemap incluido en el paquete.

[![El tweet anunciando la filtración](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)

Este repositorio captura ese momento. Sirve tanto como archivo del código expuesto como análisis de cómo ocurrió la filtración, junto con lo que revela sobre el sistema detrás de escena.

Vamos a ello.

## ¿Cómo ocurrió esto?

Esta es la parte que de verdad te hace detenerte un segundo.

Al publicar un paquete de JavaScript o TypeScript en npm, las canalizaciones de build modernas normalmente generan **archivos source map** (`.map`). Estos archivos actúan como puente entre la salida de producción empaquetada/minificada y el código fuente original. Su propósito es simple: cuando algo falla en producción, los stack traces pueden mapearse a la línea exacta del código original, en lugar de apuntar a una posición ilegible dentro de un bundle comprimido.

Sin embargo, el detalle importante es que **los source maps a menudo incrustan el propio código fuente original**. No referencias: código real en bruto almacenado como cadenas dentro de una estructura JSON.

Un archivo `.map` típico se ve así:

```json
{
  "version": 3,
  "sources": ["../src/main.tsx", "../src/tools/BashTool.ts", "..."],
  "sourcesContent": ["// código fuente original completo por archivo", "..."],
  "mappings": "AAAA,SAAS,OAAO..."
}
````

El campo `sourcesContent` es la clave. Puede contener el contenido completo de cada archivo original: código, comentarios, constantes internas, prompts, todo. Si este archivo se publica, en la práctica se distribuye con él todo el codebase.

Y npm lo distribuirá sin problema. Cualquiera que ejecute `npm pack`, inspeccione el tarball o navegue el contenido del paquete puede acceder a ello directamente.

No es una clase nueva de problema. Ya ha ocurrido varias veces antes y probablemente volverá a ocurrir. La causa raíz suele ser simple: o bien los archivos `*.map` no se excluyen mediante `.npmignore`, o el bundler no está configurado para desactivar la generación de source maps en builds de producción.

En este caso, el proyecto fue construido con Bun, que genera source maps por defecto a menos que se desactiven explícitamente, lo que hace que este tipo de exposición sea fácil de pasar por alto.

![Archivos fuente de Claude Code expuestos en el paquete npm](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)

Lo particularmente irónico aquí es la existencia de un sistema interno llamado “Undercover Mode”, diseñado para evitar que información sensible se filtre a través de salidas generadas como commits o descripciones de PR.

Se invirtió una cantidad considerable de esfuerzo en asegurarse de que la IA no expusiera por accidente detalles internos en texto, y aun así todo el codebase terminó publicándose a través de un artefacto de build.

---

## ¿Qué es Claude por dentro?

Si no has estado viviendo debajo de una roca, Claude Code es la herramienta CLI oficial de Anthropic para programar con Claude y el agente de codificación con IA más popular.

Desde fuera, parece un CLI pulido pero relativamente simple.

Desde dentro, es un punto de entrada **[`main.tsx`](https://github.com/kuberwastaken/claude-code/blob/main/main.tsx) de 785KB**, un renderer de terminal en React personalizado, más de 40 herramientas, un sistema de orquestación multiagente, un motor de consolidación de memoria en segundo plano llamado "dream", y mucho más.

Basta de charla; aquí van algunas partes del código fuente que de verdad me parecieron geniales tras una tarde de exploración a fondo:

---

## BUDDY - Un Tamagotchi dentro de tu terminal

No me lo estoy inventando.

Claude Code tiene un sistema completo de **mascota compañera estilo Tamagotchi** llamado "Buddy". Un **sistema de gacha determinista** con rareza de especies, variantes shiny, estadísticas generadas proceduralmente y una descripción del alma escrita por Claude en la primera eclosión, al estilo OpenClaw.

Todo el sistema vive en [`buddy/`](https://github.com/kuberwastaken/claude-code/tree/main/buddy) y está protegido por el feature flag de compilación `BUDDY`.

### El sistema de gacha

La especie de tu buddy se determina mediante un **Mulberry32 PRNG**, un generador pseudoaleatorio rápido de 32 bits sembrado a partir del hash de tu `userId` con la sal `'friend-2026-401'`:

```typescript
// Mulberry32 PRNG - determinista y reproducible por usuario
function mulberry32(seed: number): () => number {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    var t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  }
}
```

El mismo usuario siempre obtiene el mismo buddy.

### 18 especies (ofuscadas en el código)

Los nombres de las especies están ocultos mediante arrays de `String.fromCharCode()`: está claro que Anthropic no quería que aparecieran en búsquedas de cadenas. Decodificado, el listado completo es:

| Rareza | Especies |
|--------|---------|
| **Común** (60%) | Pebblecrab, Dustbunny, Mossfrog, Twigling, Dewdrop, Puddlefish |
| **Poco común** (25%) | Cloudferret, Gustowl, Bramblebear, Thornfox |
| **Rara** (10%) | Crystaldrake, Deepstag, Lavapup |
| **Épica** (4%) | Stormwyrm, Voidcat, Aetherling |
| **Legendaria** (1%) | Cosmoshale, Nebulynx |

Además, hay un **1% de probabilidad shiny** completamente independiente de la rareza. Así que un Shiny Legendary Nebulynx tiene una probabilidad de **0.01%**. Una locura.

### Stats, ojos, sombreros y alma

Cada buddy obtiene proceduralmente:
- **5 stats**: `DEBUGGING`, `PATIENCE`, `CHAOS`, `WISDOM`, `SNARK` (0-100 cada una)
- **6 estilos posibles de ojos** y **8 opciones de sombrero** (algunas limitadas por rareza)
- **Un "alma"**, como se mencionó, la personalidad generada por Claude en la primera eclosión, escrita en personaje

Los sprites se renderizan como **arte ASCII de 5 líneas de alto y 12 caracteres de ancho** con múltiples frames de animación. Hay animaciones idle, animaciones de reacción y se sientan junto a tu prompt de entrada.

### El lore

El código hace referencia al 1-7 de abril de 2026 como una **ventana teaser** (¿quizás para Pascua?), con el lanzamiento completo bloqueado hasta mayo de 2026. El compañero tiene un system prompt que le dice a Claude:

```
A small {species} named {name} sits beside the user's input box and 
occasionally comments in a speech bubble. You're not {name} - it's a 
separate watcher.
```

Así que no es solo cosmético: el buddy tiene su propia personalidad y puede responder cuando lo llaman por su nombre. De verdad espero que lo lancen.

---

## KAIROS - "Claude siempre activo"

Dentro de [`assistant/`](https://github.com/kuberwastaken/claude-code/tree/main/assistant), hay un modo completo llamado **KAIROS**, es decir, un asistente Claude persistente y siempre en ejecución que no espera a que escribas. Observa, registra y actúa **proactivamente** sobre las cosas que detecta.

Esto está protegido por los feature flags de compilación `PROACTIVE` / `KAIROS` y está completamente ausente en las builds externas.

### Cómo funciona

KAIROS mantiene **archivos de log diarios append-only**: escribe observaciones, decisiones y acciones a lo largo del día. A intervalos regulares, recibe prompts `<tick>` que le permiten decidir si actuar proactivamente o mantenerse en silencio.

El sistema tiene un **presupuesto de bloqueo de 15 segundos**. Cualquier acción proactiva que bloquee el flujo de trabajo del usuario durante más de 15 segundos se difiere. Claude intenta ser útil sin volverse molesto.

### Modo Brief

Cuando KAIROS está activo, existe un modo de salida especial llamado **Brief**, con respuestas extremadamente concisas diseñadas para un asistente persistente que no debería inundar tu terminal. Piénsalo como la diferencia entre un amigo parlanchín y un asistente profesional que solo habla cuando tiene algo valioso que decir.

### Herramientas exclusivas

KAIROS obtiene herramientas que Claude Code normal no tiene:

| Herramienta | Qué hace |
|------|-------------|
| **SendUserFile** | Envía archivos directamente al usuario (notificaciones, resúmenes) |
| **PushNotification** | Envía notificaciones push al dispositivo del usuario |
| **SubscribePR** | Se suscribe y monitoriza actividad de pull requests |

 ---

## ULTRAPLAN - Sesiones remotas de planificación de 30 minutos

Esta es salvaje desde una perspectiva de infraestructura.

**ULTRAPLAN** es un modo en el que Claude Code descarga una tarea compleja de planificación a una **sesión remota de Cloud Container Runtime (CCR)** ejecutando **Opus 4.6**, le da hasta **30 minutos** para pensar y te permite aprobar el resultado desde el navegador.

Flujo básico:

1. Claude Code identifica una tarea que necesita planificación profunda
2. Lanza una sesión remota CCR mediante la configuración `tengu_ultraplan_model`
3. Tu terminal muestra un estado de sondeo, comprobando el resultado cada **3 segundos**
4. Mientras tanto, una UI basada en navegador te permite ver cómo sucede la planificación y aprobarla o rechazarla
5. Cuando se aprueba, hay un valor centinela especial `__ULTRAPLAN_TELEPORT_LOCAL__` que "teletransporta" el resultado de vuelta a tu terminal local

---

## El sistema "Dream" - Claude literalmente sueña

Vale, esto es genuinamente una de las cosas más geniales que hay aquí.

Claude Code tiene un sistema llamado **autoDream** ([`services/autoDream/`](https://github.com/kuberwastaken/claude-code/tree/main/services/autoDream)): un motor de consolidación de memoria en segundo plano que se ejecuta como **subagente bifurcado**. El nombre está muy bien elegido. Claude... está soñando.

Esto es especialmente gracioso porque [la semana pasada tuve la misma idea para LITMUS: subagentes de OpenClaw con tiempo libre para encontrar papers nuevos y divertidos](https://github.com/Kuberwastaken/litmus)

### El disparador de tres compuertas

El dream no se ejecuta cuando le da la gana. Tiene un **sistema de disparo de tres compuertas**:

1. **Compuerta temporal**: 24 horas desde el último dream
2. **Compuerta de sesión**: Al menos 5 sesiones desde el último dream  
3. **Compuerta de bloqueo**: Adquiere un bloqueo de consolidación (evita dreams concurrentes)

Las tres deben cumplirse. Esto evita tanto soñar demasiado como soñar demasiado poco.

### Las cuatro fases

Cuando se ejecuta, el dream sigue cuatro fases estrictas desde el prompt en [`consolidationPrompt.ts`](https://github.com/kuberwastaken/claude-code/blob/main/services/autoDream/consolidationPrompt.ts):

**Fase 1 - Orientarse**: hacer `ls` en el directorio de memoria, leer `MEMORY.md`, hojear archivos de temas existentes para mejorarlos.

**Fase 2 - Reunir señal reciente**: encontrar nueva información que valga la pena persistir. Fuentes por prioridad: logs diarios → memorias desviadas → búsqueda en transcripciones.

**Fase 3 - Consolidar**: escribir o actualizar archivos de memoria. Convertir fechas relativas en absolutas. Eliminar hechos contradichos.

**Fase 4 - Podar e indexar**: mantener `MEMORY.md` por debajo de 200 líneas y ~25KB. Eliminar punteros obsoletos. Resolver contradicciones.

El prompt dice literalmente:

> *"You are performing a dream - a reflective pass over your memory files. Synthesize what you've learned recently into durable, well-organized memories so that future sessions can orient quickly."*

El subagente de dream obtiene **bash de solo lectura**. Puede mirar tu proyecto, pero no modificar nada. Es puramente una pasada de consolidación de memoria.

---

## El registro completo de herramientas - Más de 40 herramientas

El sistema de herramientas de Claude Code vive en [`tools/`](https://github.com/kuberwastaken/claude-code/tree/main/tools). Aquí está la lista completa:

| Herramienta | Qué hace |
|------|-------------|
| **AgentTool** | Genera agentes hijos/subagentes |
| **BashTool** / **PowerShellTool** | Ejecución de shell (con sandbox opcional) |
| **FileReadTool** / **FileEditTool** / **FileWriteTool** | Operaciones de archivos |
| **GlobTool** / **GrepTool** | Búsqueda de archivos (usa `bfs`/`ugrep` nativos cuando están disponibles) |
| **WebFetchTool** / **WebSearchTool** / **WebBrowserTool** | Acceso web |
| **NotebookEditTool** | Edición de notebooks Jupyter |
| **SkillTool** | Invoca skills definidas por el usuario |
| **REPLTool** | Shell VM interactiva (modo bare) |
| **LSPTool** | Comunicación con Language Server Protocol |
| **AskUserQuestionTool** | Solicita input al usuario |
| **EnterPlanModeTool** / **ExitPlanModeV2Tool** | Control del modo Plan |
| **BriefTool** | Sube/resume archivos a claude.ai |
| **SendMessageTool** / **TeamCreateTool** / **TeamDeleteTool** | Gestión de swarm de agentes |
| **TaskCreateTool** / **TaskGetTool** / **TaskListTool** / **TaskUpdateTool** / **TaskOutputTool** / **TaskStopTool** | Gestión de tareas en segundo plano |
| **TodoWriteTool** | Escribe todos (legacy) |
| **ListMcpResourcesTool** / **ReadMcpResourceTool** | Acceso a recursos MCP |
| **SleepTool** | Retrasos asíncronos |
| **SnipTool** | Extracción de fragmentos de historial |
| **ToolSearchTool** | Descubrimiento de herramientas |
| **ListPeersTool** | Lista agentes pares (UDS inbox) |
| **MonitorTool** | Monitoriza servidores MCP |
| **EnterWorktreeTool** / **ExitWorktreeTool** | Gestión de Git worktree |
| **ScheduleCronTool** | Programa trabajos cron |
| **RemoteTriggerTool** | Activa agentes remotos |
| **WorkflowTool** | Ejecuta scripts de workflow |
| **ConfigTool** | Modifica configuración (**solo interno**) |
| **TungstenTool** | Funciones avanzadas (**solo interno**) |
| **SendUserFile** / **PushNotification** / **SubscribePR** | Herramientas exclusivas de KAIROS |

Las herramientas se registran mediante `getAllBaseTools()` y se filtran según feature gates, tipo de usuario, flags de entorno y reglas de denegación de permisos. Hay una **caché de schema de herramientas** ([`toolSchemaCache.ts`](https://github.com/kuberwastaken/claude-code/blob/main/tools/toolSchemaCache.ts)) que almacena schemas JSON en caché para mayor eficiencia de prompts.

---

## El sistema de permisos y seguridad

El sistema de permisos de Claude Code en [`tools/permissions/`](https://github.com/kuberwastaken/claude-code/tree/main/tools/permissions) es mucho más sofisticado que "permitir/denegar":

**Modos de permiso**: `default` (prompts interactivos), `auto` (autoaprobación basada en ML mediante clasificador de transcripciones), `bypass` (omite comprobaciones), `yolo` (lo niega todo, irónicamente)

**Clasificación de riesgo**: cada acción de herramienta se clasifica como riesgo **LOW**, **MEDIUM** o **HIGH**. Hay un **clasificador YOLO**, un sistema rápido de decisión de permisos basado en ML que decide automáticamente.

**Archivos protegidos**: `.gitconfig`, `.bashrc`, `.zshrc`, `.mcp.json`, `.claude.json` y otros están protegidos contra edición automática.

**Prevención de path traversal**: traversals codificados en URL, ataques de normalización Unicode, inyección con barras invertidas, manipulación de rutas insensible a mayúsculas/minúsculas: todo está cubierto.

**Permission Explainer**: una llamada LLM separada explica al usuario los riesgos de una herramienta antes de que apruebe. Cuando Claude dice "este comando modificará tu git config", esa explicación también la genera Claude.

---

## Headers beta ocultos y funciones de API no publicadas

El archivo [`constants/betas.ts`](https://github.com/kuberwastaken/claude-code/blob/main/constants/betas.ts) revela cada feature beta que Claude Code negocia con la API:

```typescript
'interleaved-thinking-2025-05-14'      // Extended thinking
'context-1m-2025-08-07'                // 1M token context window
'structured-outputs-2025-12-15'        // Structured output format
'web-search-2025-03-05'                // Web search
'advanced-tool-use-2025-11-20'         // Advanced tool use
'effort-2025-11-24'                    // Effort level control
'task-budgets-2026-03-13'              // Task budget management
'prompt-caching-scope-2026-01-05'      // Prompt cache scoping
'fast-mode-2026-02-01'                 // Fast mode (Penguin)
'redact-thinking-2026-02-12'           // Redacted thinking
'token-efficient-tools-2026-03-28'     // Token-efficient tool schemas
'afk-mode-2026-01-31'                  // AFK mode
'cli-internal-2026-02-09'             // Internal-only (ant)
'advisor-tool-2026-03-01'              // Advisor tool
'summarize-connector-text-2026-03-13'  // Connector text summarization
```

`redact-thinking`, `afk-mode` y `advisor-tool` tampoco han sido publicados.

---

## Feature gating - Builds internas vs. externas

Esta es una de las partes arquitectónicamente más interesantes del codebase.

Claude Code usa **feature flags en tiempo de compilación** mediante la función `feature()` de Bun desde `bun:bundle`. El bundler hace **constant folding** y **eliminación de código muerto** de las ramas protegidas en las builds externas. La lista completa de flags conocidas:

| Flag | Qué controla |
|------|--------------|
| `PROACTIVE` / `KAIROS` | Modo asistente siempre activo |
| `KAIROS_BRIEF` | Comando Brief |
| `BRIDGE_MODE` | Control remoto vía claude.ai |
| `DAEMON` | Modo demonio en segundo plano |
| `VOICE_MODE` | Entrada por voz |
| `WORKFLOW_SCRIPTS` | Automatización de workflows |
| `COORDINATOR_MODE` | Orquestación multiagente |
| `TRANSCRIPT_CLASSIFIER` | Modo AFK (autoaprobación ML) |
| `BUDDY` | Sistema de mascota compañera |
| `NATIVE_CLIENT_ATTESTATION` | Attestation del cliente |
| `HISTORY_SNIP` | Recorte de historial |
| `EXPERIMENTAL_SKILL_SEARCH` | Descubrimiento de skills |

Además, `USER_TYPE === 'ant'` protege funciones internas de Anthropic: acceso a la API de staging (`claude-ai.staging.ant.dev`), headers beta internos, Undercover mode, el comando `/security-review`, `ConfigTool`, `TungstenTool` y el volcado de prompts de debug en `~/.config/claude/dump-prompts/`.

**GrowthBook** maneja el feature gating en tiempo de ejecución con valores agresivamente cacheados. Los feature flags con prefijo `tengu_` controlan todo, desde fast mode hasta consolidación de memoria. Muchas comprobaciones usan `getFeatureValue_CACHED_MAY_BE_STALE()` para evitar bloquear el bucle principal; se considera aceptable tener datos obsoletos para feature gates.

---

## Otros hallazgos destacables

### El upstream proxy
El directorio [`upstreamproxy/`](https://github.com/kuberwastaken/claude-code/tree/main/upstreamproxy) contiene un relay proxy compatible con contenedores que usa **`prctl(PR_SET_DUMPABLE, 0)`** para evitar ptrace de memoria heap por procesos con el mismo UID. Lee tokens de sesión desde `/run/ccr/session_token` en contenedores CCR, descarga certificados CA e inicia un relay local CONNECT→WebSocket. La API de Anthropic, GitHub, npmjs.org y pypi.org están excluidos explícitamente del proxy.

### Bridge Mode
Un sistema bridge autenticado con JWT en [`bridge/`](https://github.com/kuberwastaken/claude-code/tree/main/bridge) para integrarse con claude.ai. Soporta modos de trabajo: `'single-session'` | `'worktree'` | `'same-dir'`. Incluye tokens de dispositivo confiable para niveles elevados de seguridad.

### Codenombres de modelos en migraciones
El directorio [`migrations/`](https://github.com/kuberwastaken/claude-code/tree/main/migrations) revela el historial interno de codenombres:
- `migrateFennecToOpus` - **"Fennec"** (el zorro) era un codenombre de Opus
- `migrateSonnet1mToSonnet45` - Sonnet con contexto 1M pasó a ser Sonnet 4.5
- `migrateSonnet45ToSonnet46` - Sonnet 4.5 → Sonnet 4.6
- `resetProToOpusDefault` - En algún momento los usuarios Pro fueron restablecidos a Opus por defecto

### Attribution Header
Cada solicitud a la API incluye:
```
x-anthropic-billing-header: cc_version={VERSION}.{FINGERPRINT}; 
  cc_entrypoint={ENTRYPOINT}; cch={ATTESTATION_PLACEHOLDER}; cc_workload={WORKLOAD};
```
La feature `NATIVE_CLIENT_ATTESTATION` permite que la pila HTTP de Bun sobrescriba el marcador `cch=00000` con un hash calculado, esencialmente una comprobación de autenticidad del cliente para que Anthropic pueda verificar que la solicitud provino de una instalación real de Claude Code.

### Computer Use - "Chicago"
Claude Code incluye una implementación completa de Computer Use, con nombre en clave interno **"Chicago"**, construida sobre `@ant/computer-use-mcp`. Proporciona captura de pantalla, entrada de clic/teclado y transformación de coordenadas. Está limitada a suscripciones Max/Pro (con bypass ant para usuarios internos).

### Precios
Para quien se lo pregunte: todos los precios en [`utils/modelCost.ts`](https://github.com/kuberwastaken/claude-code/blob/main/utils/modelCost.ts) coinciden exactamente con [los precios públicos de Anthropic](https://docs.anthropic.com/en/docs/about-claude/models). No hay nada noticiable ahí.

---

## ⚡ TL;DR

- Claude Code es mucho más que un CLI: es una **plataforma de agentes completa**
- Incluye:
  - Orquestación multiagente
  - Memoria en segundo plano ("Dream system")
  - Ecosistema de herramientas (40+ herramientas)
  - Asistente proactivo (KAIROS)
- Hace un uso intensivo de:
  - Feature flags
  - Gating en tiempo de ejecución + compilación
  - Prompt engineering como diseño de sistemas

---

## 🧩 Ideas arquitectónicas clave

### 1. Orquestación multiagente

Claude Code puede operar en **Coordinator Mode**, lanzando múltiples workers:

- Investigación en paralelo
- Planificación centralizada
- Ejecución distribuida
- Bucles de verificación

> El paralelismo se trata como un primitivo de primera clase.

---

### 2. Sistema de tooling (40+ herramientas)

Un ecosistema rico de herramientas que incluye:

- Operaciones de sistema de archivos
- Ejecución de shell
- Navegación web
- Programación de tareas
- Comunicación entre agentes

Todas las herramientas son:
- Guiadas por schema
- Protegidas por permisos
- Habilitadas dinámicamente

---

### 3. Sistema de memoria ("Dream")

Un proceso en segundo plano que:

- Se ejecuta periódicamente
- Consolida conocimiento
- Poda información obsoleta
- Mantiene memoria a largo plazo

> Esto es esencialmente **compresión de memoria LLM + reflexión**.

---

### 4. Agente proactivo (KAIROS)

Un asistente siempre activo que:

- Observa actividad
- Registra comportamiento
- Actúa sin prompts explícitos

Esto es **agent → evolución del sistema**.

---

### 5. Feature gating y estrategia de build

- Flags en tiempo de compilación (vía Bun)
- Eliminación de código muerto
- Builds internas vs externas

Esto permite:
- Funciones ocultas
- Despliegues graduales
- Experimentación interna

---

## 🧠 Lo que aprendimos

- Las herramientas de codificación con IA se están convirtiendo en **sistemas operativos**, no solo asistentes
- Prompt engineering = **arquitectura de sistemas**
- Memoria + herramientas + orquestación = **agentes reales**
- Los sistemas de producción dependen mucho de:
  - Guardrails
  - Permisos
  - Observabilidad

---

## 🛠 ¿Por qué reconstruir en Rust?

Este codebase es potente, pero:

- ❌ Complejo y difícil de mantener
- ❌ Limitaciones del runtime de JS
- ❌ Garantías de rendimiento débiles
- ❌ Difícil razonar sobre concurrencia

Creemos que el sistema de agentes de próxima generación debería ser:

- ⚡ Más rápido (rendimiento nativo)
- 🔒 Más seguro (memoria + ejecución)
- 🧵 Concurrente por diseño
- 📦 Mejor para distribución (CLI + infraestructura)

---

## 🚀 Nuestra dirección

Estamos construyendo:

> Un **runtime de agentes de código de próxima generación**, no solo un wrapper CLI.

Áreas de enfoque:

- Ejecución determinista de agentes
- Mejor sandboxing de herramientas
- Orquestación multiagente de primera clase
- Sistemas de memoria reales (no hacks de prompt)
- Pipeline de distribución amigable con Bun / npm

---

## 📦 Build y distribución

Actualmente trabajando en:

- ✅ Pipeline de build basado en Bun
- 📦 Distribución por npm
- ⚡ Ejecución nativa con Bun

Objetivo:

> Instalación fluida, ejecución instantánea, fricción cero.

---

## ⚠️ Descargo de responsabilidad

Este repositorio es para:

- Investigación
- Educación
- Conocimientos de ingeniería inversa

**No reclamamos la propiedad** del Claude Code original.

---

## ⭐ Nota final

Este repo comenzó como un archivo.

Ahora es una **plataforma de lanzamiento**.

> No colecciones código.  
> Construye sistemas que realmente se entreguen.
