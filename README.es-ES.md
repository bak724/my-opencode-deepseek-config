# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | **Español** | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Configuración óptima de OpenCode × DeepSeek** —— Un esquema de configuración que maximiza el potencial de los modelos duales DeepSeek V4 (Pro + Flash) dentro del framework multi-agente de OpenCode. Principio central: **eficiencia de tokens primero, lograr el mejor resultado de desarrollo con el menor coste de contexto**.

## Resumen de la configuración actual

- Agente principal predeterminado: `orchestrator`
- Modelo principal: `deepseek/deepseek-v4-pro`, modelo ligero: `deepseek/deepseek-v4-flash`
- Jerarquía de agentes: `subagent_depth: 3` (soporta 3 niveles de anidamiento de subagentes)
- Aislamiento de modelos: bloqueo dual `enabled_providers: ["deepseek"]` + `disabled_providers`
- Compartición de sesión: desactivada (`share: "disabled"`); instantáneas: activadas (`snapshot: true`)
- Línea base de permisos: permitir por defecto, comandos bash destructivos configurados como `ask`; archivos sensibles tipo `.env` en `deny`; directorios externos en `ask`; lista blanca de bash para agentes de solo lectura (denegar todo por defecto + permitir solo subcomandos de lectura)
- Compresión de contexto: compresión proactiva DCP con umbral del 60% + auto compaction nativa de OpenCode como respaldo ante un desbordamiento inminente, dos capas complementarias (prune recorta salidas antiguas de herramientas)
- Reglas globales: `AGENTS.md` (principios fundamentales, contrato de rechazo de tareas, autoverificación, antipatrones, etc.; la disciplina de contexto/tokens se ha trasladado al `orchestrator`)
- Habilidades: **18** `SKILL.md` en el directorio `skills/`, cargadas bajo demanda mediante la herramienta nativa `skill`
- Plugins: `superpowers` (v6.3.0, habilidades de proceso), `@tarquinen/opencode-dcp` (poda inteligente de contexto)

## Configuración del modelo DeepSeek

### Requisitos previos

- OpenCode ≥ v1.14.24 (el proveedor DeepSeek viene integrado)
- DeepSeek API Key: solicítala en [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Opción 1: Configuración interactiva TUI (recomendada)

```bash
opencode
# En la TUI, introduce: /connect → selecciona DeepSeek → pega la API Key
# Luego: /models → selecciona deepseek-v4-pro
```

La API Key se almacena automáticamente en `~/.local/share/opencode/auth.json`.

### Opción 2: Variable de entorno

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Configuración permanente: añade `DEEPSEEK_API_KEY` a las variables de entorno del sistema.

### Referencia de configuración del proveedor

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

Si necesitas activar thinking/reasoning para el modelo Pro, añade en `provider`:

```jsonc
"provider": {
  "deepseek": {
    "models": {
      "deepseek-v4-pro": {
        "options": {
          "thinking": { "type": "enabled" }
        }
      }
    }
  }
}
```

> **Convención de nombres de ID de modelo**: `provider_id/model_id`, es decir, `deepseek/deepseek-v4-pro` y `deepseek/deepseek-v4-flash`.

## Instalación

### Opción 1: Clonar + variable de entorno (recomendado, multiplataforma)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Luego apunta `OPENCODE_CONFIG_DIR` al subdirectorio `opencode/` dentro del repositorio para empezar a usarlo.

**Windows (PowerShell)** —— permanente:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** —— temporal (solo sesión actual):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— añadir a `~/.bashrc` o `~/.zshrc`:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Opción 2: Enlace simbólico al directorio de configuración global

**Windows (PowerShell, requiere administrador):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Nota de compatibilidad**: `~/.config/opencode` es la ruta de configuración global estándar de OpenCode. Los archivos de configuración (`agents/`, `skills/`, `AGENTS.md`, etc.) se encuentran dentro del subdirectorio `opencode/` de este repositorio y siguen el diseño convencional de OpenCode. Al apuntar mediante variable de entorno o enlace simbólico, son reconocidos automáticamente.

### Verificar la instalación

Inicia OpenCode y confirma:
1. `/models` → el modelo actual es `deepseek/deepseek-v4-pro`
2. La lista de agentes debe mostrar 10 agentes: `orchestrator`, `planner`, `deep-worker`, etc.
3. Introduce cualquier petición; el Orchestrator analiza automáticamente la intención y enruta

## División de roles entre modelos

Este repositorio se limita estrictamente a los dos modelos DeepSeek V4, sin introducir otros:

| Modelo | Uso |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planificación, arquitectura, análisis de causa raíz, revisión de código, implementación pesada, orquestación principal |
| `deepseek/deepseek-v4-flash` | Exploración rápida, búsqueda externa, tareas ligeras, ediciones simples |

### Estrategia de enrutamiento

- **Flash primero**: tareas claramente definidas como búsqueda, localización y ediciones simples se asignan preferentemente a agentes flash
- **Pro enfocado al razonamiento**: planificación, análisis, revisión, implementación compleja — solo Pro
- **Actualización automática**: cuando un agente flash no puede manejar la tarea, se escala automáticamente a Pro (con contexto completo)

## Estructura de agentes

### Agente principal

| Agente | Modelo | Función |
| --- | --- | --- |
| `orchestrator` | v4-pro | Punto de entrada predeterminado: puerta de intención (Intent Gate) + enrutamiento consciente del modelo + cadena de respaldo |

### Subagentes

| Agente | Modelo | Permisos | Función |
| --- | --- | --- | --- |
| `planner` | v4-pro | Lectura/escritura | Planificación, arquitectura, descomposición de tareas |
| `deep-worker` | v4-pro | Lectura/escritura | Implementación pesada, cambios multiarchivo, depuración compleja |
| `oracle` | v4-pro | **Solo lectura** | Análisis de causa raíz, comprensión profunda del código |
| `reviewer` | v4-pro | **Solo lectura** | Revisión de código en dos ejes (convenciones + especificación) + calibración de severidad |
| `ui-builder` | v4-pro | Lectura/escritura | Tareas de frontend y UI |
| `consultant` | v4-pro | Lectura/escritura | Discusión de enfoques, recomendaciones de buenas prácticas |
| `explore` | v4-flash | **Solo lectura** | Búsqueda en el código base, exploración paralela |
| `librarian` | v4-flash | **Solo lectura** | Recuperación de documentación, búsqueda web |
| `light-orchestrator` | v4-flash | Lectura/escritura | Tareas ligeras, edición de un solo archivo |

> `deep-worker` y `light-orchestrator` siguen el principio de "prohibido investigar, prohibido delegar" — ejecutan, no exploran; el contexto lo proporciona el orchestrator.
>
> Los agentes de solo lectura (`oracle`/`reviewer`/`explore`/`librarian`) son de solo lectura real: `edit: deny` + lista blanca de bash (denegar todo por defecto, permitiendo solo subcomandos de lectura como `git status/diff/log/show/blame/grep`, `rg`, etc.; `oracle`/`reviewer` también permiten `gh pr view/diff`, `gh issue view`, `gh api` para soportar la publicación de comentarios de `/review-pr`).

## Comandos rápidos

### Comandos de enrutamiento de agentes

| Comando | Agente | Uso |
| --- | --- | --- |
| `/deep` | `deep-worker` | Implementación pesada, cambios multiarchivo |
| `/quick` | `light-orchestrator` | Tareas ligeras, edición de un solo archivo |
| `/ui` | `ui-builder` | Trabajo de frontend/UI |
| `/review` | `reviewer` (code-review) | Revisión paralela en dos ejes (convenciones + especificación) + calibración de severidad |
| `/review-pr` | `reviewer` (code-review + gh-cli) | Revisar PR y publicar comentarios en GitHub |
| `/plan` | `planner` | Elaborar plan, propuesta técnica |
| `/search` | `librarian` | Búsqueda externa, consulta de documentación |
| `/oracle` | `oracle` | Análisis profundo, trazabilidad de problemas |
| `/consult` | `consultant` | Consulta, comparación, recomendaciones |

### Comandos operativos

| Comando | Agente | Uso |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Generar mensaje de commit en formato Conventional Commits (inline) |
| `/release` | `deep-worker` (git-release) | Preparar publicación de tag |
| `/reflect` | `oracle` (reflect) | Detectar fricción → proponer optimización de configuración |
| `/handoff` | `light-orchestrator` (handoff) | Comprimir sesión en documento de traspaso |

### Comandos inline

| Comando | Agente | Uso |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Generar mapa de estructura del repositorio |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analiza → light-orchestrator aplica la simplificación |
| `/rmslop` | `deep-worker` (remove-deadcode) | Limpiar código muerto y AI slop |

### Comandos de especificación

| Comando | Agente | Uso |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Explorar código → redactar propuesta de cambio |
| `/spec-apply` | `deep-worker` (spec-workflow) | Implementar paso a paso según tasks.md → archivar automáticamente |

## Habilidades (Skills)

OpenCode expone las habilidades bajo demanda mediante la herramienta nativa `skill` — los agentes solo las cargan cuando las necesitan, sin ocupar contexto.

| Skill | Función |
| --- | --- |
| `code-review` | Revisión de código multidimensional con ahorro de tokens: informe clasificado por dimensión + severidad, puntos de acuerdo marcados con máxima confianza, nunca modifica código por iniciativa propia |
| `codemap` | Generar mapa anotado de la estructura del repositorio para una orientación rápida, ahorrando tokens de exploración |
| `gh-cli` | Referencia de GitHub CLI v2.97+: paginación, localización de repositorios, discussions/projects/rulesets/skills, rate limit, CI agéntico gh-aw, fallback a gh api |
| `git-master` | Operaciones avanzadas de Git: rebase, squash, fixup, bisect, reflog, arqueología de código, worktree |
| `git-release` | Publicación de tags: notas de versión, inferencia de SemVer, comando gh release |
| `resolving-merge-conflicts` | Resolver conflictos de merge por hunk: rastrear la intención original, nunca inventar comportamiento nuevo, nunca --abort |
| `handoff` | Comprimir la sesión en un documento de traspaso (referencias por ruta, sin copiar contenido) |
| `opencode-config` | Escribir y mantener la configuración de OpenCode de este repositorio (agents/skills/commands/permissions) |
| `reflect` | Mejora continua: detectar fricción → proponer reparaciones mínimas y mantenibles |
| `remove-deadcode` | Buscar y eliminar código muerto de forma segura, con verificación previa mediante cadena de herramientas/LSP |
| `security-review` | Revisión de seguridad antes de fusionar (inyección/XSS/SSRF/secretos/deserialización/path traversal), solo informa, no modifica |
| `shared-language` | Construir glosario de dominio (CONTEXT.md), ahorrando drásticamente tokens |
| `simplify` | Simplificación de código con preservación de comportamiento (oracle analiza → aplicar) |
| `spec-workflow` | Cambio ligero guiado por especificación: proposal → specs → design → tasks → archive |
| `verification-planning` | Planificar la ruta de verificación más acotada antes de implementar |
| `verify-with-docs` | Verificar la documentación de la API antes de codificar, recuperación primero, evita alucinaciones |
| `grilling` | Entrevista de alineación de requisitos: una pregunta a la vez, prioridad a opción múltiple, actuar solo tras converger la ambigüedad |
| `tech-debt-audit` | Auditoría de deuda técnica en 9 dimensiones (código muerto/duplicación/deriva de nombres/complejidad/dependencias/manejo de errores/pruebas/documentación/seguridad), informe de solo lectura sin modificar código |

## Decisiones de diseño y registro de iteraciones

La idea central se inspira en las ventajas de [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (puerta de intención, aislamiento de solo lectura, antipatrones), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (prioridad del planificador, cadena de respaldo, contrato de rechazo, seguridad de caché de prompts, impact×confidence÷cost), [anomalyco/opencode](https://github.com/anomalyco/opencode) (esquema de configuración, sistema de habilidades), [cli/cli](https://github.com/cli/cli) (conjunto de comandos gh v2.97, rate limit, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, flujo de acciones OPSX update/verify/cuatro preguntas), [mattpocock/skills](https://github.com/mattpocock/skills) (disciplina de resolución de conflictos, documentos de traspaso), [pi](https://github.com/earendil-works/pi) (responder primero y luego modificar, respuestas concisas, recopilación de contexto en sesiones independientes) y [deepreview](https://github.com/mechanai/deepreview) (convergencia por clasificación de novedad, enrutamiento por tamaño efectivo, Points of Agreement). Implementación pura por configuración, cero dependencias adicionales.

> **Inspiración, no copia**: de los pipelines demasiado pesados solo se extraen conceptos de diseño ligero; las funciones redundantes las cubren los agentes/habilidades existentes, sin añadir nada nuevo. Se sigue el principio de "simplificar antes que añadir"; cada iteración apunta a una reducción neta de tokens.
>
> **Origen de los mecanismos de esta ronda (v27)**: el flujo de acciones OPSX (update/verify/cuatro preguntas) se internaliza en spec-workflow; la recopilación de contexto en sesiones independientes y la seguridad de caché de prompts (prefijos estáticos estables, contenido variable al final del payload) se inspiran en pi y oh-my-opencode-slim; el control iterativo impact×confidence÷cost entra en deep-worker; Points of Agreement (marcar puntos de acuerdo con máxima confianza) se inspira en deepreview; gh-cli añade rate limit y gh-aw de cli/cli v2.97.
>
> **Evaluado y descartado**: la divulgación progresiva y wait-what de mattpocock/skills (la carga perezosa de habilidades existente ya cubre su valor); superpowers no tiene opciones de configuración, se mantiene inyectado como cadena de plugin.

### Hitos de iteración

27 iteraciones desde v1, continuamente alineadas con las mejores prácticas de los repositorios upstream:

- **v1-v7 (Fundación)**: Vinculación dual-modelo, sistema de roles de agentes, enrutamiento por intención, reglas globales AGENTS.md, directorio skills, base de permisos
- **v8-v15 (Revisión + Specs + Contratos)**: code-review calibración de doble eje, spec-workflow, alineación gh-cli, contrato de rechazo, verificaciones en segundo plano
- **v16-v22 (Adelgazamiento continuo)**: Comandos 29→18 (-38%), AGENTS.md 290→211 (-27%), recorte no-op, validación de esquema
- **v23-v25 (Alineación + Seguridad)**: 6 repos upstream integrados, gh-cli v2.97 advertencia de inyección de escape, refinamiento procedure-driven de prompts, ajuste DCP
- **v26 (Adelgazamiento de esta ronda)**: prune:true y endurecimiento de tool_output 800/20480, DCP cambia a umbrales porcentuales 60%/30%, grilling introducido en sustitución de writing-great-skills, opencode-config simplificado 131→64, code-review clasificación de hallazgos + validator, gh-cli añade gh status, AGENTS.md añade User Override, disciplina de coste de delegación en orchestrator, 7 archivos de agentes con reducción neta de 22 líneas
- **v27 (Eliminación/Migración/Novedades)**: elimina la configuración muerta batch_tool, el inútil `write: deny` de los agentes de solo lectura y 3 redundancias de bash; la sección de gestión de contexto se traslada a la subsección exclusiva del orchestrator; lista blanca de bash para agentes de solo lectura y `read` complementado con `.env`; nueva habilidad tech-debt-audit; 15 descripciones de habilidades reducidas un 30-40%; gh-cli añade rate limit/alojamiento de gh skill/gh-aw entre otros 5 puntos, code-review añade Points of Agreement, spec-workflow añade dos preguntas de update, orchestrator añade recopilación en sesiones independientes + seguridad de caché de prompts, deep-worker añade impact×confidence÷cost

## Estructura del repositorio

```text
├── opencode/                     # Directorio de configuración de OpenCode (implementable de forma independiente)
│   ├── .ai/
│   │   └── calibration.yml       # Calibración de severidad de code-review
│   ├── agents/                   # 10 agentes especializados
│   │   ├── orchestrator.md       # Entrada principal: puerta de intención + enrutamiento consciente del modelo
│   │   ├── planner.md            # pro: arquitectura y planificación
│   │   ├── deep-worker.md        # pro: implementación pesada
│   │   ├── oracle.md             # pro: análisis profundo de código (solo lectura)
│   │   ├── reviewer.md           # pro: revisión de código en dos ejes (solo lectura)
│   │   ├── consultant.md         # pro: discusión de enfoques y recomendaciones
│   │   ├── ui-builder.md         # pro: frontend y UI
│   │   ├── explore.md            # flash: búsqueda en el código base (solo lectura)
│   │   ├── librarian.md          # flash: búsqueda externa (solo lectura)
│   │   └── light-orchestrator.md # flash: ediciones simples
│   ├── skills/                   # 18 habilidades cargadas bajo demanda
│   │   ├── code-review/          # Revisión paralela en dos ejes + calibración de severidad
│   │   ├── codemap/              # Generar mapa de estructura del repositorio
│   │   ├── gh-cli/               # Referencia de GitHub CLI v2.97+ + advertencia seguridad
│   │   ├── git-master/           # Operaciones avanzadas de Git
│   │   ├── git-release/          # Publicación de tags
│   │   ├── handoff/              # Compresión de sesión en documento de traspaso
│   │   ├── opencode-config/      # Meta-habilidad: escritura de configuración de este repositorio
│   │   ├── reflect/              # Mejora continua
│   │   ├── remove-deadcode/      # Detección y eliminación de código muerto
│   │   ├── resolving-merge-conflicts/ # Resolución de conflictos de merge por hunk
│   │   ├── security-review/      # Lista de verificación de seguridad
│   │   ├── shared-language/      # Glosario de dominio (ahorro de tokens)
│   │   ├── simplify/             # Simplificación de código con preservación de comportamiento
│   │   ├── spec-workflow/        # Desarrollo guiado por especificación
│   │   ├── tech-debt-audit/      # Auditoría de deuda técnica (9 dimensiones, informe de solo lectura)
│   │   ├── verification-planning/ # Planificación de ruta de verificación antes de implementar
│   │   ├── verify-with-docs/     # Verificación de API con prioridad de recuperación
│   │   └── grilling/             # Entrevista de alineación de requisitos
│   ├── opencode.jsonc            # Configuración principal (18 comandos)
│   ├── AGENTS.md                 # Reglas globales
│   └── dcp.jsonc                 # Compresión de contexto DCP (DeepSeek 128K, umbrales porcentuales 60%/30%)
├── README.md
├── LICENSE
└── README.*.md                   # README en otros idiomas
```

## Guía de uso

### Modo 1: Enrutamiento automático del Orchestrator (predeterminado)

Describe tus necesidades en lenguaje natural; el Orchestrator analiza automáticamente la intención, selecciona el agente y modelo más adecuado y ejecuta.

```text
«Ayúdame a diagnosticar el error de esta API de login»     → oracle analiza la causa raíz → devuelve informe de diagnóstico
«Optimiza este bucle, el rendimiento es pésimo»              → oracle analiza → deep-worker implementa la optimización
«Revisa este PR por favor»                                    → reviewer realiza revisión multidimensional → devuelve informe clasificado
«Quiero añadir una función de exportación al módulo de usuarios» → planner elabora el plan → deep-worker implementa
«¿Cómo se usa la API use() de React 19?»                      → librarian consulta la documentación → devuelve firma y ejemplos
```

### Modo 2: Acceso directo por alias de comando

| Escenario | Comando |
| --- | --- |
| Implementación compleja / cambios multiarchivo | `/deep` |
| Modificación ligera / edición de un solo archivo | `/quick` |
| Elaborar propuesta técnica / diseño de arquitectura | `/plan` |
| Diagnosticar bugs / análisis profundo | `/oracle` |
| Revisión de código | `/review` |
| Búsqueda externa / consulta de API | `/search` |
| Trabajo de frontend / UI | `/ui` |
| Discusión de enfoques / comparación y decisión | `/consult` |
| Depuración estructurada | `/oracle` |

### Flujos de trabajo típicos

**Desarrollar una nueva funcionalidad (guiado por especificación):**
```text
/spec-propose  → /spec-apply  → /review
```

**Diagnosticar un bug:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Revisión de código:**
```text
/review-pr   ← Revisar PR + publicar comentarios automáticamente
/review      ← Revisión paralela en dos ejes
```

## Filosofía de diseño

- **Puramente basado en configuración, cero dependencias adicionales** —— Todas las capacidades se implementan mediante `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Máximo aprovechamiento de los dos modelos DeepSeek V4** —— Pro para razonamiento y decisiones, Flash para consultas y ejecución ligera
- **Eficiencia de tokens primero** —— Referencias por ruta en lugar de pegar archivos, habilidades bajo demanda, gestión de compresión por niveles
- **Plugins que potencian sin eclipsar** —— superpowers proporciona disciplina de proceso, DCP aporta compresión inteligente en lugar de simple truncamiento (umbrales porcentuales adaptativos, compactación nativa como respaldo)
- **Separación de ejecución y exploración** —— deep-worker/light-orchestrator tienen prohibido investigar/delegar, explore/librarian tienen prohibido modificar
- **Mejora continua** —— reflect detecta fricción de forma sistemática, code-review calibra en dos ejes para garantizar la calidad
