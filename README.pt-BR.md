# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | **Português** | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Configuração ideal OpenCode × DeepSeek** —— Leva ao máximo o potencial dos dois modelos DeepSeek V4 (Pro + Flash) dentro do framework multi-agente do OpenCode. Filosofia central: **Eficiência de tokens em primeiro lugar — alcançar o melhor resultado de desenvolvimento com o menor custo de contexto**.

## Visão geral da configuração atual

- Agente principal padrão: `orchestrator`
- Modelo principal: `deepseek/deepseek-v4-pro`, modelo leve: `deepseek/deepseek-v4-flash`
- Hierarquia de agentes: `subagent_depth: 3` (suporta 3 níveis de aninhamento de agentes)
- Isolamento de modelos: `enabled_providers: ["deepseek"]` como trava única
- Compartilhamento de sessão: desativado (`share: "disabled"`); snapshots: ativados (`snapshot: true`)
- Baseline de permissões: liberado por padrão; comandos bash destrutivos definidos como `ask`; arquivos sensíveis do tipo `.env` com `deny`; diretórios externos com `ask`; whitelist de bash para agentes somente leitura (negar tudo por padrão + liberar apenas subcomandos de leitura)
- Compressão de contexto: compactação integrada (opencode.jsonc) gerencia disparo automático + prune de saídas antigas de ferramentas; DCP (dcp.jsonc) gerencia deduplicação ativa + limites de compactação — os dois se complementam
- Regras globais: `AGENTS.md` (princípios centrais, contrato de rejeição de tarefas, autoverificação, antipadrões etc.; a disciplina de contexto/tokens foi movida para o `orchestrator`)
- Skills: **23** skills `SKILL.md` no diretório `skills/`, carregadas sob demanda via ferramenta nativa `skill`
- Plugins: `superpowers` (v6.3.0, skills de processo), `@tarquinen/opencode-dcp` (poda inteligente de contexto)

## Configuração do modelo DeepSeek

### Pré-requisitos

- OpenCode ≥ v1.18.x (o provider DeepSeek é integrado)
- Chave de API DeepSeek: solicite em [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Método 1: configuração interativa via TUI (recomendado)

```bash
opencode
# Na TUI, digite: /connect → escolha DeepSeek → cole sua API Key
# Depois: /models → selecione deepseek-v4-pro
```

A API Key é persistida automaticamente em `~/.local/share/opencode/auth.json`.

### Método 2: variáveis de ambiente

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
  "enabled_providers": ["deepseek"]
}
```

Se quiser habilitar thinking/reasoning para o modelo Pro, acrescente em `provider`:

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

> **Regra de nomenclatura dos IDs de modelo**: `provider_id/model_id`, ou seja, `deepseek/deepseek-v4-pro` e `deepseek/deepseek-v4-flash`.

## Instalação

### Método 1: clone + variável de ambiente (recomendado, multiplataforma)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Depois aponte `OPENCODE_CONFIG_DIR` para o subdiretório `opencode/` do repositório e pronto.

**Windows (PowerShell)** —— permanente:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** —— temporário (apenas a sessão atual):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— acrescente ao `~/.bashrc` ou `~/.zshrc`:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Método 2: link simbólico para o diretório de configuração global

**Windows (PowerShell, requer admin):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Nota de compatibilidade**: `~/.config/opencode` é o caminho global padrão do OpenCode. O subdiretório `opencode/` deste repositório contém `agents/`, `skills/`, `AGENTS.md` etc., com layout que segue as convenções do OpenCode — basta apontá-lo via variável de ambiente ou link simbólico para ser reconhecido automaticamente.

### Verificar a instalação

Inicie o OpenCode e confirme:
1. `/models` → o modelo atual é `deepseek/deepseek-v4-pro`
2. A lista de agentes deve mostrar os 10 agentes: `orchestrator`, `planner`, `deep-worker` etc.
3. Envie qualquer solicitação e o Orchestrator analisa a intenção automaticamente e faz o roteamento

## Divisão de trabalho entre os modelos

Este repositório limita estritamente a divisão de trabalho aos dois modelos DeepSeek V4, sem introduzir outros modelos:

| Modelo | Uso |
| --- | --- |
| `deepseek/deepseek-v4-pro` | planejamento, arquitetura, análise de causa raiz, revisão de código, implementação pesada, orquestração principal |
| `deepseek/deepseek-v4-flash` | exploração rápida, busca externa, tarefas leves, edições simples |

### Estratégia de roteamento

- **Flash primeiro**: buscas, consultas e edições simples — tarefas bem definidas vão primeiro para agentes flash
- **Pro focado em raciocínio**: planejamento, análise, revisão, implementação complexa — somente pro
- **Upgrade automático**: quando um agente flash não dá conta, sobe automaticamente para pro (com contexto completo)

## Estrutura de agentes

### Agente principal

| Agente | Modelo | Papel |
| --- | --- | --- |
| `orchestrator` | v4-pro | entrada padrão: gate de intenção (Intent Gate) + roteamento ciente de modelo + cadeia de fallback |

### Subagentes

| Agente | Modelo | Permissões | Papel |
| --- | --- | --- | --- |
| `planner` | v4-pro | leitura/escrita | planejamento, arquitetura, decomposição de tarefas |
| `deep-worker` | v4-pro | leitura/escrita | implementação pesada, mudanças em múltiplos arquivos, depuração complexa |
| `oracle` | v4-pro | **somente leitura** | análise de causa raiz, compreensão profunda de código |
| `reviewer` | v4-pro | **somente leitura** | revisão de código em dois eixos (convenções + especificação) + calibração de severidade |
| `ui-builder` | v4-pro | leitura/escrita | tarefas de frontend e UI |
| `consultant` | v4-pro | leitura/escrita | discussão de abordagens, recomendações de boas práticas |
| `explore` | v4-flash | **somente leitura** | busca na base de código, exploração paralela |
| `librarian` | v4-flash | **somente leitura** | consulta a documentação, busca na web |
| `light-orchestrator` | v4-flash | leitura/escrita | tarefas leves, edição em arquivo único |

> `deep-worker` e `light-orchestrator` seguem o princípio "sem pesquisa, sem delegação" — executam em vez de explorar; o contexto é fornecido pelo orchestrator.
>
> Os agentes somente leitura (`oracle`/`reviewer`/`explore`/`librarian`) são realmente somente leitura: `edit: deny` + whitelist de bash (negar tudo por padrão, liberar apenas subcomandos de leitura como `git status/diff/log/show/blame/grep`, `rg` etc.; `oracle`/`reviewer` também podem usar `gh pr view/diff`, `gh issue view`, `gh api` para responder em `/review-pr`).

## Comandos rápidos

### Comandos de roteamento de agentes

| Comando | Agente | Uso |
| --- | --- | --- |
| `/deep` | `deep-worker` | implementação pesada, mudanças em múltiplos arquivos |
| `/quick` | `light-orchestrator` | tarefas leves, edição em arquivo único |
| `/ui` | `ui-builder` | trabalho de frontend/UI |
| `/review` | `reviewer` (code-review) | revisão paralela em dois eixos (convenções+especificação) + calibração de severidade |
| `/review-pr` | `reviewer` (code-review + gh-cli) | revisa PR e responde no GitHub |
| `/plan` | `planner` | elabora planos e abordagens técnicas |
| `/search` | `librarian` | busca externa, consulta a documentação |
| `/oracle` | `oracle` | análise profunda, rastreamento de problemas |
| `/consult` | `consultant` | consultoria, comparações, recomendações |

### Comandos de operação

| Comando | Agente | Uso |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | gera mensagem de commit em Conventional Commits (formato inline) |
| `/release` | `deep-worker` (git-release) | prepara publicação de tag |
| `/reflect` | `oracle` (reflect) | descobre fricções → propõe otimizações de configuração |
| `/handoff` | `light-orchestrator` (handoff) | compacta a sessão em documento de handoff |

### Comandos inline

| Comando | Agente | Uso |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | gera mapa estrutural do repositório |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analisa → light-orchestrator aplica a simplificação |
| `/rmslop` | `deep-worker` (remove-deadcode) | limpa código morto e AI slop |

### Comandos de especificação

| Comando | Agente | Uso |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | explora o código → rascunha proposta de mudança |
| `/spec-apply` | `deep-worker` (spec-workflow) | implementa item por item conforme tasks.md → arquiva automaticamente |

## Skills

O OpenCode expõe as skills sob demanda via ferramenta nativa `skill` — o agente só as carrega quando precisa, sem ocupar contexto.

| Skill | O que faz |
| --- | --- |
| `code-review` | revisão de código multidimensional e econômica em tokens: relatórios por dimensão+severidade, pontos de concordância com confiança máxima, autofalsificação deepreview, nunca reescreve código sem pedido |
| `codemap` | gera mapa estrutural do repositório com anotações, orientação rápida, economiza tokens de exploração |
| `gh-cli` | referência do GitHub CLI v2.97+: paginação, localização de repositórios, discussions/projects/rulesets/skills, rate limit, CI agêntico gh-aw, fallback gh api |
| `git-master` | operações avançadas de Git: rebase, squash, fixup, bisect, reflog, arqueologia de código, worktree |
| `git-release` | publicação de tag: notas de release, inferência SemVer, comando gh release |
| `resolving-merge-conflicts` | resolução de conflitos de merge hunk a hunk: rastreia a intenção original, nunca inventa comportamento novo, nunca usa --abort |
| `handoff` | compacta a sessão em documento de handoff (referências por caminho, sem copiar conteúdo) |
| `opencode-config` | escreve e mantém a configuração OpenCode deste repositório (agents/skills/commands/permissions) |
| `reflect` | melhoria contínua: descobre fricções → propõe correções mínimas e sustentáveis |
| `remove-deadcode` | localiza e remove código morto com segurança, verificando via toolchain/LSP antes de apagar |
| `security-review` | revisão de segurança antes do merge (injeção/XSS/SSRF/segredos/desserialização/path traversal), apenas reporta, não corrige |
| `shared-language` | constrói glossário de domínio (CONTEXT.md), grande economia de tokens |
| `simplify` | simplificação de código que preserva comportamento (oracle analisa → aplica) |
| `spec-workflow` | mudanças leves orientadas a especificação: proposal → specs → design → tasks → archive |
| `verification-planning` | planeja o caminho de verificação mais estreito antes de implementar |
| `verify-with-docs` | confere a documentação da API antes de codificar, retrieval-first, evita alucinação |
| `grilling` | entrevista de alinhamento de requisitos: uma pergunta por vez, múltipla escolha preferencial, só age após convergir ambiguidades |
| `tech-debt-audit` | auditoria de dívida técnica em 9 dimensões (código morto/duplicação/deriva de nomes/complexidade/dependências/tratamento de erros/testes/documentação/segurança), relatório somente leitura, não altera código |
| `wait-what` | quando a mensagem do usuário é difícil de entender, reformula em uma frase para confirmar antes de agir |
| `writing-for-agents` | alavancas de escrita para documentos que agentes leem (skill/AGENTS.md/documentos ponteiro) |
| `to-questionnaire` | questionário único fora do canal (preenchimento assíncrono), diferente da entrevista em tempo real do grilling |
| `research` | pesquisa profunda de temas abertos, produz Markdown com citações, diferente da verificação pontual do verify-with-docs |
| `wizard` | assistente passo a passo guiado por humano (script bash, validado com `bash -n`), conduz o humano pelas etapas que só ele pode executar |

## Decisões de design e registro de iterações

A ideia central toma emprestado de [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (gate de intenção, isolamento somente leitura, antipadrões), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (scheduler primeiro, cadeia de fallback, contrato de rejeição, segurança de cache de prompt, impact×confidence÷cost), [anomalyco/opencode](https://github.com/anomalyco/opencode) (schema de configuração, sistema de skills), [cli/cli](https://github.com/cli/cli) (conjunto de comandos gh v2.97, rate limit, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, fluxo de ações OPSX update/verify/quatro perguntas), [mattpocock/skills](https://github.com/mattpocock/skills) (disciplina de resolução de conflitos, documento de handoff), [pi](https://github.com/earendil-works/pi) (responder antes de alterar, respostas enxutas, coleta em sessões independentes) e [deepreview](https://github.com/mechanai/deepreview) (convergência de classificação de novidade, roteamento por tamanho efetivo, Points of Agreement), implementado puramente via configuração, com zero dependências extras.

> **Emprestar, não copiar**: dos pipelines pesados extraímos apenas as ideias de design leve; funcionalidades redundantes são cobertas pelos agents/skills existentes, sem criar novos. Seguimos o princípio "enxugar antes de adicionar" — cada iteração mira redução líquida de tokens.
>
> **Fontes dos mecanismos desta rodada (v28)**: disciplina de cache+thinking do DeepSeek, scope-first+delegação primeiro, TODO atômico descido para o AGENTS.md; 5 novas skills (wait-what/writing-for-agents/to-questionnaire/research/wizard) totalizando 23; gh-cli ganhou 4 itens de segurança GHSA; code-review incorporou autofalsificação do deepreview; removido .ai/calibration.yml (regras de calibração inlined no code-review).
>
> **Avaliado e não adotado**: as demais skills de processo do mattpocock/skills (code-review, tdd, implement etc. — sobrepõem-se ao superpowers/às skills existentes); superpowers não tem botões de configuração, mantido como string de plugin injetada.

### Marcos de iteração

Desde a v1, foram 28 iterações, continuamente alinhadas às melhores práticas dos repositórios upstream:

- **v1-v7 (fundação)**: vínculo dos dois modelos, sistema de papéis de agentes, roteamento por gate de intenção, regras globais do AGENTS.md, diretório Skills, baseline de permissões
- **v8-v15 (revisão+especificação+contrato)**: calibração em dois eixos do code-review, spec-workflow, alinhamento do gh-cli, contrato de rejeição, verificação em background
- **v16-v22 (enxugamento contínuo)**: comandos 29→18 (-38%), AGENTS.md 290→211 (-27%), poda de no-ops frase a frase, validação de schema para remover chaves mortas
- **v23-v25 (alinhamento+segurança)**: integração de 6 repositórios upstream, seção de segurança de escape injection do gh-cli v2.97, refinamento de prompts procedure-driven, ajuste de janelas do DCP
- **v26 (enxugamento desta rodada)**: prune:true e tool_output 800/20480 mais rígidos, DCP trocado para limites percentuais 60%/30%, grilling introduzido no lugar de writing-great-skills, opencode-config enxugado de 131→64, code-review com níveis+validator, gh-cli ganhou gh status, AGENTS.md ganhou User Override, disciplina de custo de delegação do orchestrator, 7 arquivos de agentes com redução líquida de 22 linhas
- **v27 (remoção/migração/adição)**: removida config morta batch_tool, `write: deny` ineficaz em agentes somente leitura, 3 redundâncias de bash; seção Context Management migrada para a subseção exclusiva do orchestrator; whitelist de bash para agentes somente leitura, read complementado com `.env`; nova skill tech-debt-audit; descriptions de 15 skills enxugadas em 30-40%; gh-cli ganhou rate limit/hospedagem de gh skill/gh-aw entre 5 pontos, code-review ganhou Points of Agreement, spec-workflow ganhou duas perguntas de update, orchestrator ganhou coleta em sessões independentes+segurança de cache de prompt, deep-worker ganhou impact×confidence÷cost
- **v28 (refatoração de disciplina)**: disciplina de cache+thinking, scope-first+delegação primeiro, TODO atômico descido ao AGENTS.md; 5 novas skills totalizando 23; gh-cli ganhou 4 itens GHSA; code-review incorporou autofalsificação do deepreview; removido .ai/calibration.yml (regras inlined no code-review); README sincronizado em dez idiomas

## Estrutura do repositório

```text
├── opencode/                     # Diretório de configuração do OpenCode (implantável independentemente)
│   ├── agents/                   # 10 agentes especializados
│   │   ├── orchestrator.md       # entrada principal: gate de intenção + roteamento ciente de modelo
│   │   ├── planner.md            # pro: arquitetura e planejamento
│   │   ├── deep-worker.md        # pro: implementação pesada
│   │   ├── oracle.md             # pro: análise profunda de código (somente leitura)
│   │   ├── reviewer.md           # pro: revisão de código em dois eixos (somente leitura)
│   │   ├── consultant.md         # pro: discussão de abordagens e recomendações
│   │   ├── ui-builder.md         # pro: frontend e UI
│   │   ├── explore.md            # flash: busca na base de código (somente leitura)
│   │   ├── librarian.md          # flash: busca externa (somente leitura)
│   │   └── light-orchestrator.md # flash: edições simples
│   ├── skills/                   # 23 skills carregadas sob demanda
│   │   ├── code-review/          # revisão paralela em dois eixos + calibração de severidade
│   │   ├── codemap/              # gera mapa estrutural do repositório
│   │   ├── gh-cli/               # referência GitHub CLI v2.97+ + avisos de segurança
│   │   ├── git-master/           # operações avançadas de Git
│   │   ├── git-release/          # publicação de tag
│   │   ├── handoff/              # compacta a sessão em documento de handoff
│   │   ├── opencode-config/      # meta-skill: escrita de configuração deste repositório
│   │   ├── reflect/              # melhoria contínua
│   │   ├── remove-deadcode/      # detecção e remoção de código morto
│   │   ├── resolving-merge-conflicts/ # disciplina de resolução de conflitos hunk a hunk
│   │   ├── security-review/      # checklist de revisão de segurança
│   │   ├── shared-language/      # glossário de domínio (economia de tokens)
│   │   ├── simplify/             # simplificação de código que preserva comportamento
│   │   ├── spec-workflow/        # desenvolvimento orientado a especificação
│   │   ├── tech-debt-audit/      # auditoria de dívida técnica (9 dimensões, relatório somente leitura)
│   │   ├── verification-planning/ # planejamento do caminho de verificação antes de implementar
│   │   ├── verify-with-docs/     # verificação de API com retrieval-first
│   │   ├── grilling/             # entrevista de alinhamento de requisitos
│   │   ├── research/             # pesquisa profunda de temas abertos (com citações)
│   │   ├── to-questionnaire/     # questionário único fora do canal
│   │   ├── wait-what/            # reformula mensagens confusas em uma frase antes de agir
│   │   ├── wizard/               # assistente passo a passo guiado por humano (validado com bash -n)
│   │   └── writing-for-agents/   # escrita de documentação voltada a agentes
│   ├── opencode.jsonc            # configuração principal (18 comandos)
│   ├── AGENTS.md                 # regras globais
│   └── dcp.jsonc                 # compressão de contexto DCP (DeepSeek 128K, limites percentuais 60%/30%)
├── README.md
├── LICENSE
└── README.*.md                   # READMEs em outros idiomas
```

## Guia de uso

### Modo 1: roteamento automático do Orchestrator (padrão)

Descreva a necessidade em linguagem natural; o Orchestrator analisa a intenção, escolhe o agente e o modelo mais adequados e executa.

```text
"Me ajude a investigar o erro neste endpoint de login"     → oracle analisa a causa raiz → devolve relatório de diagnóstico
"Otimize este loop, o desempenho está péssimo"             → oracle analisa → deep-worker implementa a otimização
"Revise este PR para mim"                                  → reviewer revisa em múltiplas dimensões → devolve relatório por níveis
"Quero adicionar exportação ao módulo de usuários"         → planner define a abordagem → deep-worker implementa
"Como uso a API use() do React 19"                         → librarian consulta a documentação → devolve assinaturas e exemplos
```

### Modo 2: atalhos de comando diretos

| Cenário | Comando |
| --- | --- |
| Implementação complexa / mudanças em múltiplos arquivos | `/deep` |
| Alterações leves / edição em arquivo único | `/quick` |
| Abordagem técnica / design de arquitetura | `/plan` |
| Investigação de bugs / análise profunda | `/oracle` |
| Revisão de código | `/review` |
| Busca externa / consulta de API | `/search` |
| Trabalho de frontend / UI | `/ui` |
| Discussão de abordagens / trade-offs | `/consult` |
| Depuração estruturada | `/oracle` |

### Fluxos de trabalho típicos

**Desenvolvimento de nova feature (orientado a especificação):**
```text
/spec-propose  → /spec-apply  → /review
```

**Investigação de bugs:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Revisão de código:**
```text
/review-pr   ← revisa PR + responde automaticamente
/review      ← revisão paralela em dois eixos
```

## Filosofia de design

- **Puramente orientado a configuração, zero dependências extras** —— todas as capacidades são implementadas por `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Uso extremo dos dois modelos DeepSeek V4** —— Pro para raciocínio e decisões, Flash para consultas e execução leve
- **Eficiência de tokens em primeiro lugar** —— referências por caminho em vez de colar arquivos, skills carregadas sob demanda, compressão em camadas
- **Plugins que somam sem roubar a cena** —— superpowers traz disciplina de processo, DCP (dcp.jsonc) deduplica ativamente + limites de compactação, compactação integrada (opencode.jsonc) dispara automaticamente + prune como rede de segurança
- **Separação entre execução e exploração** —— deep-worker/light-orchestrator proibidos de pesquisar/delegar, explore/librarian proibidos de modificar
- **Disciplina de cache e thinking** —— prefixos estáticos estáveis para acertar o cache de prompt do DeepSeek; temperatura 0 para tarefas de codificação; thinking habilitado apenas para tarefas de raciocínio, desativado para tarefas simples/de busca
- **Scope First + Delegate Always** —— primeiro defina o escopo (2+ passos/múltiplos arquivos/mudança de arquitetura passam primeiro pelo planner), depois delegue a execução; tokens do nível superior ficam só para roteamento e problemas difíceis
- **TODO atômico** —— tarefas de múltiplos passos começam com TODO ordenado, item a item in_progress→completed; formato `path: action for scenario — verify by check`
- **Melhoria contínua** —— reflect mecaniza a descoberta de fricções, code-review calibra a qualidade em dois eixos
