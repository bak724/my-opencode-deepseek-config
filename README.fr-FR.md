# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | **Français** | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Configuration optimale OpenCode × DeepSeek** — un schéma de configuration qui exploite au maximum la puissance des deux modèles DeepSeek V4 (Pro + Flash) au sein du framework multi-agent OpenCode. Idée directrice : **la priorité à l'efficacité des tokens — obtenir les meilleurs résultats de développement avec un coût de contexte minimal**.

## Aperçu de la configuration actuelle

- Agent principal par défaut : `orchestrator`
- Modèle principal : `deepseek/deepseek-v4-pro`, modèle léger : `deepseek/deepseek-v4-flash`
- Hiérarchie d'agents : `subagent_depth: 3` (3 niveaux d'imbrication de sous-agents)
- Isolation des modèles : verrou unique `enabled_providers: ["deepseek"]`
- Partage de session : désactivé (`share: "disabled"`) ; instantanés : activés (`snapshot: true`)
- Base des permissions : tout autorisé par défaut, commandes bash destructrices en `ask` ; fichiers sensibles de type `.env` en `deny` ; répertoires externes en `ask` ; liste blanche bash des agents en lecture seule (tout `deny` par défaut + seules les sous-commandes en lecture sont autorisées)
- Compression du contexte : compaction intégrée (opencode.jsonc) pour le déclenchement automatique + prune des anciennes sorties d'outils, DCP (dcp.jsonc) pour la déduplication proactive + les seuils de compression — complémentaires l'un de l'autre
- Règles globales : `AGENTS.md` (principes fondamentaux, contrat de refus de tâche, auto-vérification, anti-patterns, etc. ; la discipline contexte/tokens est déléguée à `orchestrator`)
- Skills : **23** skills `SKILL.md` dans le répertoire `skills/`, chargés à la demande via l'outil natif `skill`
- Plugins : `superpowers` (v6.3.0, skills orientés processus), `@tarquinen/opencode-dcp` (élagage intelligent du contexte)

## Configuration du modèle DeepSeek

### Prérequis

- OpenCode ≥ v1.18.x (le provider DeepSeek est intégré)
- Clé API DeepSeek : demandez-la sur [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Méthode 1 : configuration interactive via le TUI (recommandée)

```bash
opencode
# Dans la TUI, saisir : /connect → sélectionner DeepSeek → coller la clé API
# Puis : /models → sélectionner deepseek-v4-pro
```

La clé API est automatiquement persistée dans `~/.local/share/opencode/auth.json`.

### Méthode 2 : variables d'environnement

Windows PowerShell :
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Configuration permanente : ajoutez `DEEPSEEK_API_KEY` aux variables d'environnement système.

### Référence de configuration du provider

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"]
}
```

Pour activer thinking/reasoning sur le modèle Pro, ajoutez ce qui suit dans `provider` :

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

> **Règle de nommage des ID de modèles** : `provider_id/model_id`, soit `deepseek/deepseek-v4-pro` et `deepseek/deepseek-v4-flash`.

## Installation et déploiement

### Méthode 1 : clone + variable d'environnement (recommandée, multiplateforme)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Puis pointez `OPENCODE_CONFIG_DIR` vers le sous-répertoire `opencode/` du dépôt.

**Windows (PowerShell)** — effet permanent :

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** — effet temporaire (session courante uniquement) :

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** — ajoutez à `~/.bashrc` ou `~/.zshrc` :

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Méthode 2 : lien symbolique vers le répertoire de configuration global

**Windows (PowerShell, droits administrateur requis) :**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS :**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Note de compatibilité** : `~/.config/opencode` est le chemin de configuration global standard d'OpenCode. Le sous-répertoire `opencode/` de ce dépôt contient `agents/`, `skills/`, `AGENTS.md`, etc., avec une organisation entièrement conforme aux conventions d'OpenCode ; pointé via une variable d'environnement ou un lien symbolique, il est automatiquement reconnu.

### Vérification de l'installation

Lancez OpenCode et vérifiez :
1. `/models` → le modèle courant est `deepseek/deepseek-v4-pro`
2. La liste des agents doit montrer 10 agents : `orchestrator`, `planner`, `deep-worker`, etc.
3. Envoyez une requête quelconque : l'Orchestrator analyse l'intention et route automatiquement

## Répartition des modèles

Ce dépôt s'en tient strictement aux deux modèles DeepSeek V4, sans introduire d'autres modèles :

| Modèle | Usage |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planification, architecture, analyse des causes racines, revue de code, implémentations lourdes, orchestration principale |
| `deepseek/deepseek-v4-flash` | Exploration rapide, recherche externe, tâches légères, éditions simples |

### Stratégie de routage

- **Flash d'abord** : les tâches bien définies — recherche, consultation, édition simple — vont d'abord aux agents flash
- **Pro pour le raisonnement** : planification, analyse, revue, implémentation complexe — uniquement en pro
- **Montée en gamme automatique** : un agent flash incapable de mener la tâche est automatiquement promu vers pro (avec le contexte complet)

## Structure des agents

### Agent principal (Primary)

| Agent | Modèle | Rôle |
| --- | --- | --- |
| `orchestrator` | v4-pro | Entrée par défaut : contrôle d'intention (Intent Gate) + routage sensible au modèle + chaîne de repli |

### Sous-agents (Subagents)

| Agent | Modèle | Permissions | Rôle |
| --- | --- | --- | --- |
| `planner` | v4-pro | lecture/écriture | Planification, architecture, découpage des tâches |
| `deep-worker` | v4-pro | lecture/écriture | Implémentations lourdes, modifications multi-fichiers, débogage complexe |
| `oracle` | v4-pro | **lecture seule** | Analyse des causes racines, compréhension approfondie du code |
| `reviewer` | v4-pro | **lecture seule** | Revue de code à deux axes (conventions + spécifications) + calibration de la sévérité |
| `ui-builder` | v4-pro | lecture/écriture | Tâches frontend et UI |
| `consultant` | v4-pro | lecture/écriture | Discussion de solutions, conseils de bonnes pratiques |
| `explore` | v4-flash | **lecture seule** | Recherche dans la base de code, exploration en parallèle |
| `librarian` | v4-flash | **lecture seule** | Recherche documentaire, recherche web |
| `light-orchestrator` | v4-flash | lecture/écriture | Tâches légères, édition d'un seul fichier |

> `deep-worker` et `light-orchestrator` suivent le principe « ni recherche, ni délégation » — exécuter, pas explorer ; le contexte est fourni par l'orchestrator.
>
> Les agents en lecture seule (`oracle`/`reviewer`/`explore`/`librarian`) sont réellement limités à la lecture : `edit: deny` + liste blanche bash (tout `deny` par défaut, seules les sous-commandes en lecture telles que `git status/diff/log/show/blame/grep`, `rg`, etc. sont autorisées ; `oracle`/`reviewer` autorisent en plus `gh pr view/diff`, `gh issue view`, `gh api` pour répondre via `/review-pr`).

## Commandes rapides

### Commandes de routage d'agents

| Commande | Agent | Usage |
| --- | --- | --- |
| `/deep` | `deep-worker` | Implémentations lourdes, modifications multi-fichiers |
| `/quick` | `light-orchestrator` | Tâches légères, édition d'un seul fichier |
| `/ui` | `ui-builder` | Travail frontend / UI |
| `/review` | `reviewer` (code-review) | Revue parallèle à deux axes (conventions + spécifications) + calibration de la sévérité |
| `/review-pr` | `reviewer` (code-review + gh-cli) | Revue d'une PR et réponse sur GitHub |
| `/plan` | `planner` | Élaboration de plans, solutions techniques |
| `/search` | `librarian` | Recherche externe, consultation de la documentation |
| `/oracle` | `oracle` | Analyse approfondie, remontée à la source des problèmes |
| `/consult` | `consultant` | Conseil, comparaison, recommandations |

### Commandes d'opérations

| Commande | Agent | Usage |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Génère un message de commit Conventional Commits (format inline) |
| `/release` | `deep-worker` (git-release) | Prépare une version taguée |
| `/reflect` | `oracle` (reflect) | Identifie les frictions → propose des optimisations de configuration |
| `/handoff` | `light-orchestrator` (handoff) | Compresse la session en document de passation |

### Commandes inline

| Commande | Agent | Usage |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Génère une carte de la structure du dépôt |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analyse → light-orchestrator applique la simplification |
| `/rmslop` | `deep-worker` (remove-deadcode) | Nettoie le code mort et l'« AI slop » |

### Commandes de spécification

| Commande | Agent | Usage |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Explore le code → rédige une proposition de changement |
| `/spec-apply` | `deep-worker` (spec-workflow) | Implémente une à une les tâches de tasks.md → archivage automatique |

## Skills

OpenCode expose les skills à la demande via l'outil natif `skill` — les agents ne les chargent qu'en cas de besoin, sans consommer de contexte.

| Skill | Rôle |
| --- | --- |
| `code-review` | Revue de code multidimensionnelle économe en tokens : rapport classé par dimension + sévérité, points d'accord marqués avec une confiance maximale, auto-falsification deepreview, ne modifie jamais le code sans demande |
| `codemap` | Génère une carte annotée de la structure du dépôt, orientation rapide, économise les tokens d'exploration |
| `gh-cli` | Référence GitHub CLI v2.97+ : pagination, ciblage des dépôts, discussions/projects/rulesets/skills, rate limit, CI agentique gh-aw, repli gh api |
| `git-master` | Opérations Git avancées : rebase, squash, fixup, bisect, reflog, archéologie du code, worktree |
| `git-release` | Publication de version taguée : notes de version, inférence SemVer, commande gh release |
| `resolving-merge-conflicts` | Résolution des conflits de merge hunk par hunk : remonter à l'intention d'origine, ne jamais inventer de nouveau comportement, jamais de --abort |
| `handoff` | Compresse la session en document de passation (références par chemin, pas de copie du contenu) |
| `opencode-config` | Rédige et maintient la configuration OpenCode de ce dépôt (agents/skills/commands/permissions) |
| `reflect` | Amélioration continue : identifie les frictions → propose des correctifs minimaux et durables |
| `remove-deadcode` | Trouve et supprime le code mort en toute sécurité, vérifié via la chaîne d'outils / LSP avant suppression |
| `security-review` | Audit de sécurité avant merge (injection/XSS/SSRF/secrets/désérialisation/path traversal), signale sans corriger |
| `shared-language` | Construit un glossaire du domaine (CONTEXT.md), économise fortement les tokens |
| `simplify` | Simplification du code préservant le comportement (analyse par oracle → application) |
| `spec-workflow` | Changement léger piloté par spécifications : proposal → specs → design → tasks → archive |
| `verification-planning` | Planifie le chemin de vérification le plus étroit avant l'implémentation |
| `verify-with-docs` | Vérifie la documentation de l'API avant de coder, recherche d'abord, anti-hallucination |
| `grilling` | Entretien d'alignement des besoins : une question à la fois, choix multiples d'abord, agir une fois l'ambiguïté levée |
| `tech-debt-audit` | Audit de dette technique en 9 dimensions (code mort/duplication/dérive de nommage/complexité/dépendances/gestion d'erreurs/tests/documentation/sécurité), rapport en lecture seule sans modification du code |
| `wait-what` | Quand le message de l'utilisateur est difficile à comprendre, reformuler d'abord en une phrase pour confirmation avant d'agir |
| `writing-for-agents` | Leviers d'écriture pour les documents destinés aux agents (skills/AGENTS.md/documents pointeurs) |
| `to-questionnaire` | Questionnaire unique hors canal (remplissage asynchrone), à distinguer de l'entretien en temps réel de grilling |
| `research` | Recherche approfondie sur un sujet ouvert, produit un Markdown cité, à distinguer de la vérification ponctuelle de verify-with-docs |
| `wizard` | Assistant pas à pas pour un humain (script bash, vérifié par `bash -n`), guide l'humain à travers les étapes qu'il est seul à pouvoir accomplir |

## Décisions de conception et journal d'itération

L'idée directrice emprunte aux points forts de [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (contrôle d'intention, isolation en lecture seule, anti-patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (priorité au dispatcher, chaîne de repli, contrat de refus, sécurité du cache de prompts, impact×confidence÷cost), [anomalyco/opencode](https://github.com/anomalyco/opencode) (schéma de configuration, système de skills), [cli/cli](https://github.com/cli/cli) (jeu de commandes gh v2.97, rate limit, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, flux d'actions OPSX update/verify/quatre questions), [mattpocock/skills](https://github.com/mattpocock/skills) (discipline de résolution des conflits, documents de passation), [pi](https://github.com/earendil-works/pi) (répondre avant de modifier, réponses concises, collecte en sessions indépendantes) et [deepreview](https://github.com/mechanai/deepreview) (convergence par classification novelty, routage par taille effective, Points of Agreement) — implémentation 100 % configuration, zéro dépendance supplémentaire.

> **S'inspirer, pas copier** : des pipelines trop lourds, on ne retient que les idées d'allègement ; les fonctionnalités redondantes sont couvertes par les agents/skills existants, sans ajout. Fidèle au principe « la simplification avant l'ajout », chaque itération vise une réduction nette des tokens.
>
> **Sources des mécanismes de cette itération (v28)** : discipline cache + thinking DeepSeek, scope-first + priorité à la délégation, TODO atomiques descendus dans AGENTS.md ; 5 nouveaux skills (wait-what/writing-for-agents/to-questionnaire/research/wizard) portent le total à 23 ; gh-cli enrichi de 4 entrées de sécurité GHSA ; code-review intègre l'auto-falsification deepreview ; suppression de .ai/calibration.yml (règles de calibration inlinées dans code-review).
>
> **Évalués puis rejetés** : les autres skills de processus de mattpocock/skills (code-review, tdd, implement, etc. — ils font double emploi avec superpowers / les skills existants) ; superpowers sans bouton de réglage, conservé tel quel sous forme de chaîne de plugin.

### Jalons d'itération

28 itérations depuis la v1, en alignement continu sur les meilleures pratiques des dépôts amont :

- **v1-v7 (fondations)** : liaison des deux modèles, système de rôles des agents, routage par contrôle d'intention, règles globales AGENTS.md, répertoire Skills, base des permissions
- **v8-v15 (revue + spécifications + contrats)** : calibration à deux axes de code-review, spec-workflow, alignement gh-cli, contrat de refus, vérifications en arrière-plan
- **v16-v22 (amaigrissement continu)** : commandes 29→18 (-38 %), AGENTS.md 290→211 (-27 %), élagage phrase par phrase des no-op, validation du schéma pour supprimer les clés mortes
- **v23-v25 (alignement + sécurité)** : intégration de 6 dépôts amont, section sécurité sur l'injection par échappement de gh-cli v2.97, affinage des prompts procedure-driven, réglage de la fenêtre DCP
- **v26 (amaigrissement de cette itération)** : prune:true et resserrement de tool_output 800/20480, DCP basculé sur des seuils en pourcentage 60 %/30 %, introduction de grilling en remplacement de writing-great-skills, opencode-config réduit 131→64, code-review avec classification + validator, ajout de gh status dans gh-cli, ajout de User Override dans AGENTS.md, discipline du coût de délégation de l'orchestrator, 22 lignes nettes en moins sur 7 fichiers d'agents
- **v27 (suppression/migration/ajout)** : suppression de la configuration morte batch_tool, du `write: deny` inefficace des agents en lecture seule, de 3 redondances bash ; section Context Management déplacée dans la sous-section dédiée de l'orchestrator ; liste blanche bash des agents en lecture seule, lecture de `.env` complétée ; ajout du skill tech-debt-audit ; descriptions de 15 skills allégées de 30-40 % ; 5 ajouts dans gh-cli (rate limit, hébergement du skill gh, gh-aw, etc.), Points of Agreement ajoutés à code-review, spec-workflow complété de deux questions update, collecte en sessions indépendantes + sécurité du cache de prompts ajoutées à l'orchestrator, impact×confidence÷cost ajouté à deep-worker
- **v28 (refonte de la discipline)** : discipline cache + thinking, scope-first + priorité à la délégation, TODO atomiques descendus dans AGENTS.md ; 5 nouveaux skills portent le total à 23 ; gh-cli enrichi de 4 entrées GHSA ; code-review intègre l'auto-falsification deepreview ; suppression de .ai/calibration.yml (règles inlinées dans code-review) ; README synchronisé en dix langues

## Structure du dépôt

```text
├── opencode/                     # Répertoire de configuration OpenCode (déployable indépendamment)
│   ├── agents/                   # 10 agents spécialisés
│   │   ├── orchestrator.md       # Point d'entrée principal : filtrage d'intention + routage adapté au modèle
│   │   ├── planner.md            # pro : architecture et planification
│   │   ├── deep-worker.md        # pro : implémentation lourde
│   │   ├── oracle.md             # pro : analyse de code approfondie (lecture seule)
│   │   ├── reviewer.md           # pro : revue de code sur deux axes (lecture seule)
│   │   ├── consultant.md         # pro : discussion de solutions et conseils
│   │   ├── ui-builder.md         # pro : frontend et UI
│   │   ├── explore.md            # flash : recherche dans le code source (lecture seule)
│   │   ├── librarian.md          # flash : recherche externe (lecture seule)
│   │   └── light-orchestrator.md # flash : modifications simples
│   ├── skills/                   # 23 compétences chargées à la demande
│   │   ├── code-review/          # Revue parallèle sur deux axes + calibration de sévérité
│   │   ├── codemap/              # Génération de carte de structure de dépôt
│   │   ├── gh-cli/               # Référence GitHub CLI v2.97+ + avertissement sécurité
│   │   ├── git-master/           # Opérations Git avancées
│   │   ├── git-release/          # Release avec tag
│   │   ├── handoff/              # Compression de session en document de passation
│   │   ├── opencode-config/      # Méta-compétence : rédaction de configuration de ce dépôt
│   │   ├── reflect/              # Amélioration continue
│   │   ├── remove-deadcode/      # Détection et suppression de code mort
│   │   ├── resolving-merge-conflicts/ # Résolution de conflits de merge par hunk
│   │   ├── security-review/      # Checklist d'audit de sécurité
│   │   ├── shared-language/      # Glossaire de domaine (économie de tokens)
│   │   ├── simplify/             # Simplification de code préservant le comportement
│   │   ├── spec-workflow/        # Développement piloté par spécification
│   │   ├── tech-debt-audit/      # Audit de dette technique (9 dimensions, rapport en lecture seule)
│   │   ├── verification-planning/ # Planification du chemin de vérification avant implémentation
│   │   ├── verify-with-docs/     # Vérification API avec recherche prioritaire
│   │   ├── grilling/             # Interview d'alignement des besoins
│   │   ├── research/             # Recherche approfondie sur un sujet ouvert (avec citations)
│   │   ├── to-questionnaire/     # Questionnaire unique hors canal
│   │   ├── wait-what/            # Reformulation en une phrase des messages difficiles avant d'agir
│   │   ├── wizard/               # Assistant pas à pas pour un humain (vérifié par bash -n)
│   │   └── writing-for-agents/   # Écriture pour les documents destinés aux agents
│   ├── opencode.jsonc            # Configuration principale (18 commandes)
│   ├── AGENTS.md                 # Règles globales
│   └── dcp.jsonc                 # Compression de contexte DCP (DeepSeek 128K, seuils en pourcentage 60%/30%)
├── README.md
├── LICENSE
└── README.*.md                   # Autres README linguistiques
```

## Guide d'utilisation

### Mode 1 : routage automatique de l'Orchestrator (défaut)

Décrivez vos besoins en langage naturel : l'Orchestrator analyse l'intention et choisit l'agent et le modèle les plus adaptés.

```text
« Aide-moi à diagnostiquer l'erreur de cette API de connexion »     → oracle analyse la cause racine → retourne un rapport de diagnostic
« Optimise cette boucle, les performances sont trop mauvaises »     → oracle analyse → deep-worker implémente l'optimisation
« Peux-tu revoir cette PR pour moi »                                 → reviewer revue multi-dimensionnelle → retourne un rapport gradué
« Je veux ajouter une fonction d'export au module utilisateur »      → planner élabore un plan → deep-worker implémente
« Comment utiliser l'API use() de React 19 »                        → librarian consulte la doc → retourne signature et exemples
```

### Mode 2 : accès direct par alias de commandes

| Scénario | Commande |
| --- | --- |
| Implémentation complexe / modifications multi-fichiers | `/deep` |
| Modification légère / édition d'un seul fichier | `/quick` |
| Élaboration d'une solution technique / conception d'architecture | `/plan` |
| Débogage / analyse approfondie | `/oracle` |
| Revue de code | `/review` |
| Recherche externe / consultation d'API | `/search` |
| Travail frontend / UI | `/ui` |
| Discussion de solutions / arbitrages | `/consult` |
| Débogage structuré | `/oracle` |

### Workflows typiques

**Développer une nouvelle fonctionnalité (pilotée par spécifications) :**
```text
/spec-propose  → /spec-apply  → /review
```

**Déboguer un bug :**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Revue de code :**
```text
/review-pr   ← Revue de PR + publication automatique du commentaire
/review      ← Revue parallèle sur deux axes
```

## Philosophie de conception

- **100 % configuration, zéro dépendance supplémentaire** — toutes les capacités sont réalisées par `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Exploitation maximale des deux modèles DeepSeek V4** — Pro pour le raisonnement et les décisions, Flash pour les requêtes et l'exécution légère
- **Priorité à l'efficacité des tokens** — références par chemin au lieu de coller des fichiers, skills chargés à la demande, compression gérée par niveaux
- **Les plugins renforcent sans prendre le dessus** — superpowers apporte la discipline de processus, DCP (dcp.jsonc) la déduplication proactive + les seuils de compression, la compaction intégrée (opencode.jsonc) le déclenchement automatique + le filet prune
- **Séparation exécution / exploration** — deep-worker/light-orchestrator : ni recherche ni délégation ; explore/librarian : aucune modification
- **Discipline cache + thinking** — préfixes statiques stables pour toucher le cache de prompts DeepSeek ; température 0 pour les tâches de codage ; thinking activé uniquement pour les tâches de raisonnement, désactivé pour les tâches simples / de recherche
- **Scope First + Delegate Always** — d'abord délimiter le périmètre (2+ étapes / multi-fichiers / changement d'architecture → passer d'abord par planner), ensuite déléguer l'exécution ; les tokens du niveau supérieur sont réservés au routage et aux problèmes difficiles
- **TODO atomiques** — pour toute tâche multi-étapes, écrire d'abord une liste TODO ordonnée, chaque élément passant l'un après l'autre de in_progress à completed ; format `path: action for scenario — verify by check`
- **Amélioration continue** — reflect institutionnalise la détection des frictions, la calibration à deux axes de code-review garantit la qualité