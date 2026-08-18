# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | **Español** | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Configuración óptima de OpenCode × DeepSeek** — Un esquema de configuración que exprime al máximo el potencial de los dos modelos DeepSeek V4 (Pro + Flash) dentro del framework multi-agente de OpenCode. Principio central: **primero la eficiencia de tokens; lograr el mejor resultado de desarrollo con el menor coste de contexto**.

## Resumen de la configuración actual

- Agente principal predeterminado: `orchestrator`
- Modelo principal: `deepseek/deepseek-v4-pro`, modelo ligero: `deepseek/deepseek-v4-flash`
- Profundidad de agentes: `subagent_depth: 3` (admite 3 niveles de anidamiento de subagentes)
- Aislamiento de modelos: bloqueo único `enabled_providers: ["deepseek"]`
- Compartir sesión: desactivado (`share: "disabled"`); instantáneas: activadas (`snapshot: true`)
- Línea base de permisos: permitir por defecto; comandos bash destructivos configurados como `ask`; archivos sensibles tipo `.env` con `deny`; directorios externos con `ask`; lista blanca de bash para agentes de solo lectura (denegar todo por defecto + permitir solo subcomandos de solo lectura)
- Compresión de contexto: el compaction integrado (opencode.jsonc) gestiona el disparo automático + el prune de salidas de herramientas antiguas; DCP (dcp.jsonc) gestiona la deduplicación activa + el umbral de compresión; ambos se complementan
- Reglas globales: `AGENTS.md` (principios básicos, contrato de rechazo de tareas, autoverificación, antipatrones, etc.; la disciplina de contexto/tokens se ha delegado al `orchestrator`)
- Skills: **23** `SKILL.md` bajo el directorio `skills/`, cargados bajo demanda mediante la herramienta nativa `skill`
- Plugins: `superpowers` (v6.3.0, skills de proceso), `@tarquinen/opencode-dcp` (recorte inteligente de contexto)

## Configuración del modelo DeepSeek

### Requisitos previos

- OpenCode ≥ v1.18.x (el provider de DeepSeek está integrado)
- API Key de DeepSeek: solicítala en [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Opción 1: configuración interactiva vía TUI (recomendada)

```bash
opencode
# En la TUI escribe: /connect → selecciona DeepSeek → pega la API Key
# Luego: /models → selecciona deepseek-v4-pro
```

La API Key se guardará automáticamente en `~/.local/share/opencode/auth.json`.

### Opción 2: variables de entorno

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Configuración permanente: añade `DEEPSEEK_API_KEY` a las variables de entorno del sistema.

### Referencia de configuración del provider

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"]
}
```

Para habilitar thinking/reasoning en el modelo Pro, añade dentro de `provider`:

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

> **Convención de nombres de modelos**: `provider_id/model_id`, es decir, `deepseek/deepseek-v4-pro` y `deepseek/deepseek-v4-flash`.

## Instalación y despliegue

### Opción 1: clonar + variable de entorno (recomendada, multiplataforma)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Luego apunta `OPENCODE_CONFIG_DIR` al subdirectorio `opencode/` del repositorio y listo.

**Windows (PowerShell)** — permanente:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** — temporal (solo la sesión actual):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** — añade a `~/.bashrc` o `~/.zshrc`:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Opción 2: enlace simbólico al directorio de configuración global

**Windows (PowerShell, requiere administrador):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Nota de compatibilidad**: `~/.config/opencode` es la ruta global estándar de OpenCode. El subdirectorio `opencode/` de este repositorio contiene `agents/`, `skills/`, `AGENTS.md` y demás archivos, con una estructura que sigue exactamente las convenciones de OpenCode; al apuntar a ella mediante la variable de entorno o un enlace simbólico, se detectará automáticamente.

### Verificar la instalación

Inicia OpenCode y confirma:
1. `/models` → el modelo actual es `deepseek/deepseek-v4-pro`
2. La lista de agentes debe mostrar 10 agentes: `orchestrator`, `planner`, `deep-worker`, etc.
3. Escribe cualquier solicitud: el Orchestrator analizará la intención y enrutará automáticamente

## Reparto de trabajo entre modelos

Este repositorio se limita estrictamente a repartir el trabajo entre los dos modelos DeepSeek V4; no se introducen otros modelos:

| Modelo | Uso |
| --- | --- |
| `deepseek/deepseek-v4-pro` | planificación, arquitectura, análisis de causa raíz, revisión de código, implementación pesada, orquestación principal |
| `deepseek/deepseek-v4-flash` | exploración rápida, búsquedas externas, tareas ligeras, ediciones sencillas |

### Estrategia de enrutamiento

- **Flash primero**: las tareas bien definidas (búsqueda, localización, ediciones sencillas) van primero al agente flash
- **Pro centrado en razonar**: planificación, análisis, revisión, implementación compleja — solo pro
- **Escalado automático**: cuando el agente flash no da abasto, escala automáticamente a pro (con el contexto completo)

## Estructura de agentes

### Primary Agent

| Agente | Modelo | Rol |
| --- | --- | --- |
| `orchestrator` | v4-pro | entrada predeterminada: control de intención (Intent Gate) + enrutamiento consciente del modelo + cadena de respaldo |

### Subagents

| Agente | Modelo | Permisos | Rol |
| --- | --- | --- | --- |
| `planner` | v4-pro | lectura/escritura | planificación, arquitectura, descomposición de tareas |
| `deep-worker` | v4-pro | lectura/escritura | implementación pesada, cambios en múltiples archivos, depuración compleja |
| `oracle` | v4-pro | **solo lectura** | análisis de causa raíz, comprensión profunda del código |
| `reviewer` | v4-pro | **solo lectura** | revisión de código de doble eje (normas + especificaciones) + calibración de severidad |
| `ui-builder` | v4-pro | lectura/escritura | tareas de frontend y UI |
| `consultant` | v4-pro | lectura/escritura | discusión de soluciones, recomendaciones de buenas prácticas |
| `explore` | v4-flash | **solo lectura** | búsqueda en el código, exploración paralela |
| `librarian` | v4-flash | **solo lectura** | consulta de documentación, búsqueda web |
| `light-orchestrator` | v4-flash | lectura/escritura | tareas ligeras, edición de un solo archivo |

> `deep-worker` y `light-orchestrator` siguen el principio de "no investigar, no delegar" — ejecutan, no exploran; el contexto lo proporciona el orchestrator.
>
> Los agentes de solo lectura (`oracle`/`reviewer`/`explore`/`librarian`) son de verdadera solo lectura: `edit: deny` + lista blanca de bash (denegar todo por defecto, permitir solo subcomandos de solo lectura como `git status/diff/log/show/blame/grep`, `rg`, etc.; `oracle`/`reviewer` además pueden usar `gh pr view/diff`, `gh issue view`, `gh api` para responder en `/review-pr`).

## Comandos rápidos

### Comandos de enrutamiento de agentes

| Comando | Agente | Uso |
| --- | --- | --- |
| `/deep` | `deep-worker` | implementación pesada, cambios en múltiples archivos |
| `/quick` | `light-orchestrator` | tareas ligeras, edición de un solo archivo |
| `/ui` | `ui-builder` | trabajo de frontend/UI |
| `/review` | `reviewer` (code-review) | revisión paralela de doble eje (normas+especificaciones) + calibración de severidad |
| `/review-pr` | `reviewer` (code-review + gh-cli) | revisa el PR y responde en GitHub |
| `/plan` | `planner` | elaborar planes, soluciones técnicas |
| `/search` | `librarian` | búsqueda externa, consulta de documentación |
| `/oracle` | `oracle` | análisis profundo, rastreo de problemas |
| `/consult` | `consultant` | consultas, comparaciones, recomendaciones |

### Comandos de operación

| Comando | Agente | Uso |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | genera el mensaje de commit en formato Conventional Commits (formato inline) |
| `/release` | `deep-worker` (git-release) | prepara el lanzamiento con Tag |
| `/reflect` | `oracle` (reflect) | detecta fricciones → propone optimizaciones de configuración |
| `/handoff` | `light-orchestrator` (handoff) | comprime la sesión en un documento de traspaso |

### Comandos inline

| Comando | Agente | Uso |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | genera el mapa de estructura del repositorio |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analiza → light-orchestrator aplica la simplificación |
| `/rmslop` | `deep-worker` (remove-deadcode) | limpia código muerto y AI slop |

### Comandos de especificaciones

| Comando | Agente | Uso |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | explora el código → redacta la propuesta de cambio |
| `/spec-apply` | `deep-worker` (spec-workflow) | implementa uno a uno según tasks.md → archiva automáticamente |

## Habilidades (Skills)

OpenCode expone las skills bajo demanda mediante la herramienta nativa `skill` — los agentes solo las cargan cuando las necesitan, sin ocupar contexto.

| Skill | Descripción |
| --- | --- |
| `code-review` | revisión de código multidimensional que ahorra tokens: informe por dimensiones+severidad, los puntos coincidentes marcados con la máxima confianza, autofalsificación de deepreview, nunca reescribe código sin permiso |
| `codemap` | genera un mapa de estructura del repositorio con anotaciones, orientación rápida, ahorra tokens de exploración |
| `gh-cli` | referencia de GitHub CLI v2.97+: paginación, localización de repositorios, discussions/projects/rulesets/skills, rate limit, CI agentic gh-aw, fallback a gh api |
| `git-master` | operaciones avanzadas de Git: rebase, squash, fixup, bisect, reflog, arqueología de código, worktree |
| `git-release` | publicación con Tag: notas de lanzamiento, inferencia de SemVer, comando gh release |
| `resolving-merge-conflicts` | resolución de conflictos de merge hunk a hunk: rastrear la intención original, nunca inventar comportamiento nuevo, nunca --abort |
| `handoff` | comprime la sesión en un documento de traspaso (referencias por ruta, no copia contenido) |
| `opencode-config` | escribir y mantener la configuración de OpenCode de este repositorio (agents/skills/commands/permissions) |
| `reflect` | mejora continua: detectar fricciones → proponer arreglos mínimos y mantenibles |
| `remove-deadcode` | busca y elimina código muerto de forma segura, verificado con toolchain/LSP antes de borrar |
| `security-review` | auditoría de seguridad antes del merge (inyección/XSS/SSRF/secretos/deserialización/path traversal); solo informa, no corrige |
| `shared-language` | construye un glosario de términos del dominio (CONTEXT.md), ahorra muchos tokens |
| `simplify` | simplificación de código que preserva el comportamiento (análisis de oracle → aplicación) |
| `spec-workflow` | cambios ligeros guiados por especificaciones: proposal → specs → design → tasks → archive |
| `verification-planning` | planifica la ruta de verificación más estrecha antes de implementar |
| `verify-with-docs` | verifica la documentación de la API antes de codificar, recuperación primero, contra las alucinaciones |
| `grilling` | entrevista de alineación de requisitos: una pregunta a la vez, opción múltiple primero, converge la ambigüedad antes de actuar |
| `tech-debt-audit` | auditoría de deuda técnica en 9 dimensiones (código muerto/duplicación/deriva de nombres/complejidad/dependencias/manejo de errores/tests/documentación/seguridad), informe de solo lectura sin tocar el código |
| `wait-what` | cuando el mensaje del usuario es difícil de entender, primero lo reformula en una frase para confirmar y luego actúa |
| `writing-for-agents` | palancas de escritura para documentación dirigida a agentes (skill/AGENTS.md/documentos puntero) |
| `to-questionnaire` | cuestionario único fuera de canal (relleno asíncrono), a diferencia de la entrevista en vivo de grilling |
| `research` | investigación profunda de temas abiertos, produce Markdown con citas, a diferencia de la verificación puntual de verify-with-docs |
| `wizard` | asistente paso a paso guiado por humanos (script bash, validado con `bash -n`), guía a la persona en los pasos que solo ella puede hacer |

## Decisiones de diseño e historial de iteraciones

El enfoque central toma ideas de [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (control de intención, aislamiento de solo lectura, antipatrones), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (prioridad del despachador, cadena de respaldo, contrato de rechazo, seguridad de la caché de prompts, impacto×confianza÷coste), [anomalyco/opencode](https://github.com/anomalyco/opencode) (schema de configuración, sistema de skills), [cli/cli](https://github.com/cli/cli) (conjunto de comandos gh v2.97, rate limit, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, flujo de acciones OPSX update/verify/cuatro preguntas), [mattpocock/skills](https://github.com/mattpocock/skills) (disciplina de resolución de conflictos, documentos de traspaso), [pi](https://github.com/earendil-works/pi) (responder antes de editar, respuestas concisas, recopilación en sesión independiente) y [deepreview](https://github.com/mechanai/deepreview) (convergencia de clasificación por novedad, enrutamiento por tamaño efectivo, Points of Agreement); implementación 100% por configuración, cero dependencias adicionales.

> **Inspiración, no copia**: de los pipelines demasiado pesados solo se toma la filosofía de diseño ligero; las funcionalidades redundantes las cubren los agents/skills existentes, no se añaden otros nuevos. Se sigue el principio de «reducir antes que añadir», y cada iteración busca reducir tokens netos.
>
> **Origen de los mecanismos de esta ronda (v28)**: disciplina de caché+thinking de DeepSeek, scope-first + delegación primero, TODO atómico incorporado a AGENTS.md; 5 skills nuevas (wait-what/writing-for-agents/to-questionnaire/research/wizard) hasta llegar a 23; gh-cli añade 4 entradas de seguridad GHSA; code-review integra la autofalsificación de deepreview; eliminado .ai/calibration.yml (las reglas de calibración se integran en code-review).
>
> **Evaluado y descartado**: las demás skills de proceso de mattpocock/skills (code-review, tdd, implement, etc. — se solapan con superpowers/las skills existentes); superpowers no tiene perillas de configuración, se mantiene inyectado como cadena del plugin.

### Hitos de iteración

28 iteraciones desde v1, en continua comparación con las mejores prácticas de los repositorios de referencia:

- **v1-v7 (fundamentos)**: vinculación de doble modelo, sistema de roles de agentes, enrutamiento por control de intención, reglas globales en AGENTS.md, directorio de skills, línea base de permisos
- **v8-v15 (revisión+especificaciones+contrato)**: calibración de doble eje de code-review, spec-workflow, alineación con gh-cli, contrato de rechazo, verificación en segundo plano
- **v16-v22 (adelgazamiento continuo)**: comandos 29→18 (-38%), AGENTS.md 290→211 (-27%), poda de no-ops frase a frase, validación de schema sin claves muertas
- **v23-v25 (alineación+seguridad)**: integración de 6 repositorios de referencia, sección de seguridad sobre inyección por escapes en gh-cli v2.97, refinamiento de prompts orientados a procedimientos, ajuste de las ventanas de DCP
- **v26 (adelgazamiento de esta ronda)**: prune:true y endurecimiento de tool_output 800/20480, DCP cambia a umbrales porcentuales 60%/30%, grilling reemplaza a writing-great-skills, opencode-config 131→64, code-review con niveles+validator, gh-cli añade gh status, AGENTS.md añade User Override, disciplina de coste de delegación en orchestrator, 7 archivos de agentes con -22 líneas netas
- **v27 (borrado/migración/adición)**: eliminada la configuración muerta batch_tool, el inútil `write: deny` de los agentes de solo lectura, 3 entradas bash redundantes; la sección Context Management se mueve a una subsección propia del orchestrator; lista blanca de bash para agentes de solo lectura, read añade `.env`; nueva skill tech-debt-audit; descripciones de 15 skills adelgazadas 30-40%; gh-cli añade 5 puntos (rate limit/hosting de la skill gh/gh-aw), code-review añade Points of Agreement, spec-workflow añade las dos preguntas de update, orchestrator añade recopilación en sesión independiente + seguridad de la caché de prompts, deep-worker añade impacto×confianza÷coste
- **v28 (refactor de disciplina)**: disciplina de caché+thinking, scope-first + delegación primero, TODO atómico incorporado a AGENTS.md; 5 skills nuevas hasta 23; gh-cli añade 4 entradas GHSA; code-review integra la autofalsificación de deepreview; eliminado .ai/calibration.yml (reglas integradas en code-review); README sincronizado en 10 idiomas

## Estructura del repositorio

```text
├── opencode/                     # directorio de configuración de OpenCode (desplegable por separado)
│   ├── agents/                   # 10 agentes especializados
│   │   ├── orchestrator.md       # entrada principal: control de intención + enrutamiento consciente del modelo
│   │   ├── planner.md            # pro: arquitectura y planificación
│   │   ├── deep-worker.md        # pro: implementación pesada
│   │   ├── oracle.md             # pro: análisis profundo de código (solo lectura)
│   │   ├── reviewer.md           # pro: revisión de código de doble eje (solo lectura)
│   │   ├── consultant.md         # pro: discusión de soluciones y recomendaciones
│   │   ├── ui-builder.md         # pro: frontend y UI
│   │   ├── explore.md            # flash: búsqueda en el código (solo lectura)
│   │   ├── librarian.md          # flash: búsquedas externas (solo lectura)
│   │   └── light-orchestrator.md # flash: ediciones sencillas
│   ├── skills/                   # 23 skills de carga bajo demanda
│   │   ├── code-review/          # revisión paralela de doble eje + calibración de severidad
│   │   ├── codemap/              # genera el mapa de estructura del repositorio
│   │   ├── gh-cli/               # referencia de GitHub CLI v2.97+ + advertencias de seguridad
│   │   ├── git-master/           # operaciones avanzadas de Git
│   │   ├── git-release/          # publicación con Tag
│   │   ├── handoff/              # comprime la sesión en un documento de traspaso
│   │   ├── opencode-config/      # meta-skill: escritura de la configuración de este repositorio
│   │   ├── reflect/              # mejora continua
│   │   ├── remove-deadcode/      # detección y eliminación de código muerto
│   │   ├── resolving-merge-conflicts/ # disciplina de resolución de conflictos hunk a hunk
│   │   ├── security-review/      # lista de verificación de seguridad
│   │   ├── shared-language/      # glosario de términos del dominio (ahorra tokens)
│   │   ├── simplify/             # simplificación de código que preserva el comportamiento
│   │   ├── spec-workflow/        # desarrollo guiado por especificaciones
│   │   ├── tech-debt-audit/      # auditoría de deuda técnica (9 dimensiones, informe de solo lectura)
│   │   ├── verification-planning/ # planificación de la ruta de verificación antes de implementar
│   │   ├── verify-with-docs/     # verificación de API con recuperación primero
│   │   ├── grilling/             # entrevista de alineación de requisitos
│   │   ├── research/             # investigación profunda de temas abiertos (con citas)
│   │   ├── to-questionnaire/     # cuestionario único fuera de canal
│   │   ├── wait-what/            # reformula en una frase los mensajes confusos y confirma
│   │   ├── wizard/               # asistente paso a paso guiado por humanos (validado con bash -n)
│   │   └── writing-for-agents/   # escritura de documentación dirigida a agentes
│   ├── opencode.jsonc            # configuración principal (18 comandos)
│   ├── AGENTS.md                 # reglas globales
│   └── dcp.jsonc                 # compresión de contexto DCP (DeepSeek 128K, umbrales porcentuales 60%/30%)
├── README.md
├── LICENSE
└── README.*.md                   # README en otros idiomas
```

## Guía de uso

### Modo 1: enrutamiento automático del Orchestrator (predeterminado)

Describe tus necesidades en lenguaje natural; el Orchestrator analizará automáticamente la intención y elegirá el agente y el modelo más adecuados para ejecutarlas.

```text
«Ayúdame a diagnosticar el error de este endpoint de login»     → oracle analiza la causa raíz → devuelve un informe de diagnóstico
«Optimiza este bucle, el rendimiento es muy malo»                → oracle analiza → deep-worker implementa la optimización
«Revisa este PR por favor»                                       → reviewer hace una revisión multidimensional → devuelve un informe por niveles
«Quiero añadir una función de exportación al módulo de usuarios» → planner elabora el plan → deep-worker lo implementa
«¿Cómo se usa la API use() de React 19?»                         → librarian consulta la documentación → devuelve firma y ejemplos
```

### Modo 2: acceso directo mediante alias de comandos

| Escenario | Comando |
| --- | --- |
| Implementación compleja / cambios en múltiples archivos | `/deep` |
| Modificación ligera / edición de un solo archivo | `/quick` |
| Elaborar solución técnica / diseño de arquitectura | `/plan` |
| Depurar bugs / análisis profundo | `/oracle` |
| Revisión de código | `/review` |
| Búsqueda externa / consulta de APIs | `/search` |
| Trabajo de frontend / UI | `/ui` |
| Discusión de soluciones / análisis de trade-offs | `/consult` |
| Depuración estructurada | `/oracle` |

### Flujos de trabajo típicos

**Desarrollar una nueva función (guiado por especificaciones):**
```text
/spec-propose  → /spec-apply  → /review
```

**Depurar un bug:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Revisión de código:**
```text
/review-pr   ← revisar el PR + respuesta automática en GitHub
/review      ← revisión paralela de doble eje
```

## Filosofía de diseño

- **100% configuración, cero dependencias adicionales** — todas las capacidades se implementan con `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Máximo aprovechamiento de los dos modelos DeepSeek V4** — Pro para razonamiento y decisiones, Flash para consultas y ejecución ligera
- **Primero la eficiencia de tokens** — referencias por ruta en lugar de pegar archivos, skills de carga bajo demanda, compresión con gestión por niveles
- **Los plugins suman sin quitar protagonismo** — superpowers aporta disciplina de proceso, DCP (dcp.jsonc) deduplicación activa + umbral de compresión, el compaction integrado (opencode.jsonc) disparo automático + respaldo con prune
- **Separación entre ejecución y exploración** — deep-worker/light-orchestrator tienen prohibido investigar/delegar; explore/librarian tienen prohibido modificar
- **Disciplina de caché y thinking** — prefijos estáticos estables para acertar en la caché de prompts de DeepSeek; temperatura 0 en tareas de codificación; thinking solo para tareas de razonamiento, desactivado en tareas simples/de búsqueda
- **Scope First + Delegate Always** — define primero el alcance (tareas de 2+ pasos / múltiples archivos / cambios de arquitectura pasan primero por planner), luego delega la ejecución; los tokens del nivel superior se reservan solo para el enrutamiento y los problemas difíciles
- **TODO atómico** — en tareas de varios pasos escribe primero un TODO ordenado, uno a uno in_progress→completed; formato `path: action for scenario — verify by check`
- **Mejora continua** — reflect mecaniza la detección de fricciones, la calibración de doble eje de code-review garantiza la calidad
