# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | **Português** | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Configuração ideal OpenCode × DeepSeek** —— Leva ao máximo o potencial dos dois modelos DeepSeek V4 (Pro + Flash) dentro do framework multi-agente do OpenCode. Filosofia central: **Eficiência de tokens em primeiro lugar — alcançar o melhor resultado de desenvolvimento com o menor custo de contexto**.

## Visão geral da configuração atual

- Agent principal padrão: `orchestrator`
- Modelo principal: `deepseek/deepseek-v4-pro`, modelo leve: `deepseek/deepseek-v4-flash`
- Hierarquia de agents: `subagent_depth: 3` (suporta até 3 níveis de agents aninhados)
- Isolamento de modelos: bloqueio duplo com `enabled_providers: ["deepseek"]` + `disabled_providers`
- Compartilhamento de sessão: desativado (`share: "disabled"`); snapshots: ativados (`snapshot: true`)
- Baseline de permissões: permitir por padrão, comandos bash destrutivos configurados como `ask`; arquivos sensíveis como `.env` com `deny`; diretórios externos com `ask`
- Compressão de contexto: compressão proativa via DCP (limiar 35K–75K) + compactação nativa do OpenCode como fallback
- Regras globais: `AGENTS.md` (princípios fundamentais, contrato de recusa de tarefas, eficiência de contexto e tokens, autoverificação, antipadrões, etc.)
- Skills: **16** arquivos `SKILL.md` no diretório `skills/`, carregados sob demanda pela ferramenta nativa `skill`
- Plugins: `superpowers` (14 skills de processo), `@tarquinen/opencode-dcp` (poda inteligente de contexto)
- Funcionalidades experimentais: `batch_tool` ativada por padrão

## Configuração dos modelos DeepSeek

### Pré-requisitos

- OpenCode ≥ v1.14.24 (o provider DeepSeek é nativo)
- Chave de API DeepSeek: solicite em [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Método 1: Configuração interativa via TUI (recomendado)

```bash
opencode
# Na TUI, digite: /connect → selecione DeepSeek → cole a API Key
# Em seguida: /models → selecione deepseek-v4-pro
```

A API Key é persistida automaticamente em `~/.local/share/opencode/auth.json`.

### Método 2: Variável de ambiente

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Configuração permanente: adicione `DEEPSEEK_API_KEY` às variáveis de ambiente do sistema.

### Referência de configuração do provider

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

Para ativar thinking/reasoning no modelo Pro, adicione em `provider`:

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

> **Convenção de nomenclatura dos IDs de modelo**: `provider_id/model_id`, ou seja, `deepseek/deepseek-v4-pro` e `deepseek/deepseek-v4-flash`.

## Instalação

### Método 1: Clonar + variável de ambiente (recomendado, multiplataforma)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Em seguida, aponte `OPENCODE_CONFIG_DIR` para o subdiretório `opencode/` dentro do repositório para começar a usar.

**Windows (PowerShell)** —— permanente:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** —— temporário (apenas sessão atual):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— adicionar ao `~/.bashrc` ou `~/.zshrc`:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Método 2: Link simbólico para o diretório de configuração global

**Windows (PowerShell, requer administrador):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Nota de compatibilidade**: `~/.config/opencode` é o caminho padrão de configuração global do OpenCode. Os arquivos de configuração (`agents/`, `skills/`, `AGENTS.md`, etc.) estão dentro do subdiretório `opencode/` deste repositório e seguem o layout convencional do OpenCode. Ao apontar via variável de ambiente ou link simbólico, são reconhecidos automaticamente.

### Verificar instalação

Inicie o OpenCode e confirme:
1. `/models` → o modelo atual é `deepseek/deepseek-v4-pro`
2. A lista de agents deve exibir `orchestrator`, `planner`, `deep-worker` — 10 agents no total
3. Envie qualquer requisição — o Orchestrator analisa automaticamente a intenção e faz o roteamento

## Divisão de funções entre modelos

Este repositório se restringe estritamente aos dois modelos DeepSeek V4, sem introduzir outros modelos:

| Modelo | Finalidade |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planejamento, arquitetura, análise de causa raiz, revisão de código, implementações pesadas, orquestração principal |
| `deepseek/deepseek-v4-flash` | Exploração rápida, buscas externas, tarefas leves, edições simples |

### Estratégia de roteamento

- **Flash primeiro**: tarefas bem definidas como buscas, consultas e edições simples vão preferencialmente para agents flash
- **Pro focado em raciocínio**: planejamento, análise, revisão, implementações complexas — somente com pro
- **Upgrade automático**: quando um agent flash não consegue realizar a tarefa, ocorre upgrade automático para pro (com contexto completo)

## Estrutura de agents

### Agent principal

| Agent | Modelo | Função |
| --- | --- | --- |
| `orchestrator` | v4-pro | Ponto de entrada padrão: gate de intenção (Intent Gate) + roteamento ciente do modelo + cadeia de fallback |

### Subagents

| Agent | Modelo | Permissão | Função |
| --- | --- | --- | --- |
| `planner` | v4-pro | leitura/escrita | Planejamento, arquitetura, decomposição de tarefas |
| `deep-worker` | v4-pro | leitura/escrita | Implementações pesadas, alterações em múltiplos arquivos, depuração complexa |
| `oracle` | v4-pro | **somente leitura** | Análise de causa raiz, compreensão profunda de código |
| `reviewer` | v4-pro | **somente leitura** | Revisão de código em dois eixos (convenções + especificação) + calibração de severidade |
| `ui-builder` | v4-pro | leitura/escrita | Tarefas de frontend e UI |
| `consultant` | v4-pro | leitura/escrita | Discussão de soluções, recomendações de melhores práticas |
| `explore` | v4-flash | **somente leitura** | Busca no codebase, exploração paralela |
| `librarian` | v4-flash | **somente leitura** | Consulta a documentação, busca na web |
| `light-orchestrator` | v4-flash | leitura/escrita | Tarefas leves, edições em arquivo único |

> `deep-worker` e `light-orchestrator` seguem o princípio "proibido pesquisar, proibido delegar" — executam, não exploram; o contexto é fornecido pelo orchestrator.

## Comandos rápidos

### Comandos de roteamento para agents

| Comando | Agent | Finalidade |
| --- | --- | --- |
| `/deep` | `deep-worker` | Implementações pesadas, alterações em múltiplos arquivos |
| `/quick` | `light-orchestrator` | Tarefas leves, edições em arquivo único |
| `/ui` | `ui-builder` | Trabalho de frontend/UI |
| `/review` | `reviewer` (code-review) | Revisão paralela em dois eixos (convenções + especificação) + calibração de severidade |
| `/review-pr` | `reviewer` (code-review + gh-cli) | Revisar PR e postar resultado no GitHub |
| `/plan` | `planner` | Elaborar plano, solução técnica |
| `/search` | `librarian` | Busca externa, consulta a documentação |
| `/oracle` | `oracle` | Análise profunda, rastreamento de causa raiz |
| `/consult` | `consultant` | Consultoria, comparação, recomendações |

### Comandos operacionais

| Comando | Agent | Finalidade |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Gerar mensagem de commit no formato Conventional Commits (inline) |
| `/release` | `deep-worker` (git-release) | Preparar release com tag |
| `/reflect` | `oracle` (reflect) | Identificar atritos → propor otimizações de configuração |
| `/handoff` | `light-orchestrator` (handoff) | Compactar a sessão em um documento de transição |

### Comandos inline

| Comando | Agent | Finalidade |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Gerar mapa da estrutura do repositório |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analisa → light-orchestrator aplica as simplificações |
| `/rmslop` | `deep-worker` (remove-deadcode) | Remover código morto e resíduos (AI slop) |

### Comandos de especificação

| Comando | Agent | Finalidade |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Explorar código → redigir proposta de alteração |
| `/spec-apply` | `deep-worker` (spec-workflow) | Implementar item por item conforme tasks.md → arquivar automaticamente |

## Skills

O OpenCode expõe skills sob demanda pela ferramenta nativa `skill` — os agents só carregam quando necessário, sem ocupar contexto.

| Skill | Função |
| --- | --- |
| `code-review` | Revisão paralela em dois eixos (convenções + especificação) + calibração de severidade |
| `codemap` | Gerar mapa anotado da estrutura do repositório, economizando tokens de exploração |
| `gh-cli` | Referência completa do GitHub CLI v2.96+ (Issues 2.0, copilot, agent-task, gh skill) |
| `git-master` | Operações avançadas de Git: rebase, squash, bisect, reflog, worktree |
| `git-release` | Release com tag: inferência de SemVer, notas de release, comando gh release |
| `handoff` | Compactar sessão em documento de transição (referência por caminho, sem copiar conteúdo) |
| `opencode-config` | Escrever e manter configurações do OpenCode |
| `reflect` | Melhoria contínua: identificar atritos → propor correções mínimas |
| `remove-deadcode` | Encontrar e remover código morto com segurança, verificação LSP antes da exclusão |
| `security-review` | Auditoria de segurança sobre o diff antes do merge |
| `shared-language` | Construir glossário de domínio, economizando drasticamente tokens de contexto |
| `simplify` | Simplificação de código com preservação de comportamento (oracle analisa → light-orchestrator aplica) |
| `spec-workflow` | Alterações leves orientadas a especificação (propose → design → tasks → implement → archive) |
| `verification-planning` | Planejar o caminho mais restrito de verificação antes da implementação |
| `verify-with-docs` | Verificar assinaturas de API na documentação antes de codificar — retrieval-first, prevenindo alucinações |
| `writing-great-skills` | Diretrizes para escrever skills: poda de no-ops, formulação positiva, critérios de conclusão |

## Decisões de design e registro de iterações

A abordagem central foi inspirada nos pontos fortes de [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (gate de intenção, isolamento de somente leitura, antipadrões), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (orquestrador prioritário, cadeia de fallback, contrato de recusa), [anomalyco/opencode](https://github.com/anomalyco/opencode) (schema de configuração, ecossistema de skills), [cli/cli](https://github.com/cli/cli) (conjunto completo de comandos gh), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, atualização de propostas de alteração), [mattpocock/skills](https://github.com/mattpocock/skills) (documentos de transição, depuração estruturada), [pi](https://github.com/earendil-works/pi) (responder primeiro, editar depois; respostas concisas) e [deepreview](https://github.com/mechanai/deepreview) (varredura de entropia, verificação de convergência). Implementação puramente por configuração, zero dependências adicionais.

> **Inspiração, não cópia**: pipelines excessivamente pesados tiveram apenas seus princípios de design leve absorvidos; funcionalidades redundantes são cobertas pelos agents/skills existentes, sem adições. Seguindo o princípio "enxugar antes de adicionar", cada iteração visa a redução líquida de tokens.

### Marcos das iterações

| Fase | Principais alterações |
| --- | --- |
| **v1–v7 (fundação)** | Vinculação de dois modelos, sistema de papéis dos agents, gate de intenção/roteamento por classificação, regras globais AGENTS.md, diretório de skills e aliases de comandos, baseline de permissões |
| **v8–v12 (revisão + especificação)** | Aprimoramento do code-review (classificação/autoverificação/critérios de recusa), criação do spec-workflow (explore→propose→apply→archive), novos skills deepwork/reflect/verification-planning, gh-cli alinhado à v2.96+ |
| **v13–v15 (contrato + enxugamento)** | AGENTS.md ganhou Evidence Discipline / Task Rejection Contract / stop condition; remoção completa de duplicações entre prompts de agents e regras globais; verificação de erros em subagents em segundo plano |
| **v16–v18 (execução eficiente)** | Remoção de nomes mitológicos, fusão de tabelas de roteamento, gh-cli expandido para Issues 2.0, spec-workflow com verify + framework de decisão |
| **v19 (alinhamento upstream)** | Revisão de 6 repositórios upstream; correção de bug nos comentários linha a linha do `/review-pr`; roteamento do code-review alterado de contagem bruta de linhas para volume lógico efetivo de código |
| **v20 (refatoração e otimização)** | `agent/`→`agents/` alinhado à recomendação do OpenCode; AGENTS.md reduzido em 22% (290→227 linhas); novos skills `diagnose` (depuração em 6 fases) + `handoff` (transição de sessão); spec-workflow com `/update`; code-review com varredura de entropia + verificação de convergência; prompts de agents reduzidos em 20% |
| **v21 (reestruturação e enxugamento completo)** | Skills 18→17 (remoção de deepwork/conventional-commits/diagnose, adição de writing-great-skills/shared-language); comandos 29→18 (-38%); AGENTS.md 227→212 linhas (-7%); poda de no-ops frase por frase nos skills. code-review com dois eixos paralelos + mecanismo de arquivo de calibração. Inspirado na experiência prática de 6 repositórios incluindo pi/deepreview/mattpocock. |
| **v22 (validação de schema e enxugamento)** | Validação contra schemas oficiais do OpenCode e DCP: remoção da chave morta `agent.fallback`; confirmação de que todas as chaves em `dcp.jsonc` são válidas (v3.1.14), zero alterações sem necessidade; AGENTS.md com fusão de "Eficiência de tokens" em "Gerenciamento de contexto" e remoção completa de duplicações, correção de referência órfã em `Self-Verification` (212→197 linhas); orchestrator com fusão de três tabelas de roteamento (128→79 linhas, -38%, Intent Gate/Agent Directory/Fallback totalmente preservados); 14 skills tiveram frontmatter `license/compatibility/metadata` ignorado pelo parser removido (-70 linhas); `tool_output` reduzido para economia proativa de tokens (1500 linhas/40KB). |
| **v23 (integração de duas disciplinas + fusão e enxugamento)** | Integração final baseada em 6 repositórios upstream: remoção de `gh-skill` (funcionalidade incorporada na seção Agent Skills do `gh-cli`, -122 linhas); correção de referência morta em `verification-planning`; AGENTS.md com duas disciplinas inspiradas no Pi (responder primeiro + declarar posição, concisão global) + contrato de delegação do slim + job board + IPC por arquivo do deepreview (+15 linhas); tabela do orchestrator reestruturada em subtabelas Pro/Flash, 79→86 linhas; reviewer com postura padrão de recusa do verificador (+6 linhas); seção Agent Skills do gh-cli reforçada (+10 linhas). Redução líquida de ~90 linhas, skills 17→16. |

## Estrutura do repositório

```text
├── opencode/                     # Diretório de configuração do OpenCode (implantável independentemente)
│   ├── .ai/
│   │   └── calibration.yml       # Calibração de severidade do code-review
│   ├── agents/                   # 10 agents especializados
│   │   ├── orchestrator.md       # Ponto de entrada principal: gate de intenção + roteamento ciente do modelo
│   │   ├── planner.md            # pro: arquitetura e planejamento
│   │   ├── deep-worker.md        # pro: implementações pesadas
│   │   ├── oracle.md             # pro: análise profunda de código (somente leitura)
│   │   ├── reviewer.md           # pro: revisão de código em dois eixos (somente leitura)
│   │   ├── consultant.md         # pro: discussão de soluções e recomendações
│   │   ├── ui-builder.md         # pro: frontend e UI
│   │   ├── explore.md            # flash: busca no codebase (somente leitura)
│   │   ├── librarian.md          # flash: buscas externas (somente leitura)
│   │   └── light-orchestrator.md # flash: edições simples
│   ├── skills/                   # 16 skills carregadas sob demanda
│   │   ├── code-review/          # Revisão paralela em dois eixos + calibração de severidade
│   │   ├── codemap/              # Gerar mapa da estrutura do repositório
│   │   ├── gh-cli/               # Referência do GitHub CLI v2.96+
│   │   ├── git-master/           # Operações avançadas de Git
│   │   ├── git-release/          # Release com tag
│   │   ├── handoff/              # Compactar sessão em documento de transição
│   │   ├── opencode-config/      # Meta-skill: escrever configurações deste repositório
│   │   ├── reflect/              # Melhoria contínua
│   │   ├── remove-deadcode/      # Detecção e remoção de código morto
│   │   ├── security-review/      # Checklist de auditoria de segurança
│   │   ├── shared-language/      # Glossário de domínio (economia de tokens)
│   │   ├── simplify/             # Simplificação de código com preservação de comportamento
│   │   ├── spec-workflow/        # Desenvolvimento orientado a especificação
│   │   ├── verification-planning/ # Planejamento do caminho de verificação pré-implementação
│   │   ├── verify-with-docs/     # Verificação de API retrieval-first
│   │   └── writing-great-skills/ # Diretrizes para escrever skills
│   ├── opencode.jsonc            # Configuração principal (18 comandos)
│   ├── AGENTS.md                 # Regras globais (~212 linhas)
│   └── dcp.jsonc                 # Compressão de contexto DCP (DeepSeek 128K)
├── README.md
├── LICENSE
└── README.*.md                   # README em outros idiomas
```

## Guia de uso

### Modo 1: Roteamento automático pelo Orchestrator (padrão)

Descreva sua necessidade em linguagem natural — o Orchestrator analisa automaticamente a intenção, seleciona o agent e o modelo mais adequados para executar.

```text
「Me ajude a investigar o erro desta API de login」     → oracle analisa a causa raiz → retorna relatório de diagnóstico
「Otimize este loop, está com desempenho muito ruim」    → oracle analisa → deep-worker implementa a otimização
「Revise este PR para mim」                               → reviewer faz revisão multidimensional → retorna relatório classificado
「Quero adicionar uma função de exportação ao módulo de usuários」 → planner elabora a solução → deep-worker implementa
「Como usar a API use() do React 19」                     → librarian consulta a documentação → retorna assinatura e exemplos
```

### Modo 2: Acesso direto por alias de comando

| Cenário | Comando |
| --- | --- |
| Implementação complexa / alterações em múltiplos arquivos | `/deep` |
| Alterações leves / edição em arquivo único | `/quick` |
| Elaborar solução técnica / design de arquitetura | `/plan` |
| Investigar bug / análise profunda | `/oracle` |
| Revisão de código | `/review` |
| Busca externa / consulta de API | `/search` |
| Trabalho de frontend / UI | `/ui` |
| Discussão de soluções / comparação e escolha | `/consult` |
| Depuração estruturada | `/oracle` |

### Fluxos de trabalho típicos

**Desenvolver nova funcionalidade (orientado a especificação):**
```text
/spec-propose  → /spec-apply  → /review
```

**Investigar bug:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Revisão de código:**
```text
/review-pr   ← Revisar PR + postar automaticamente
/review      ← Revisão paralela em dois eixos
```

## Filosofia de design

- **Orientado a configuração, zero dependências adicionais** — toda a capacidade é implementada por `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Utilização máxima dos dois modelos DeepSeek V4** — Pro para raciocínio e decisão, Flash para consultas e execução leve
- **Eficiência de tokens em primeiro lugar** — referência por caminho em vez de colar arquivos, skills carregadas sob demanda, compressão com gestão em camadas
- **Plugins que potencializam sem tomar o protagonismo** — superpowers fornece disciplina de processo, DCP faz compressão inteligente em vez de simples truncamento
- **Separação entre execução e exploração** — deep-worker/light-orchestrator proibidos de pesquisar/delegar; explore/librarian proibidos de modificar
- **Melhoria contínua** — mecanismo reflect para identificar atritos, code-review com calibração em dois eixos para garantir qualidade
