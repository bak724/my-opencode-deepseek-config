# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | **Français** | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Configuration optimale OpenCode × DeepSeek** — Une configuration qui exploite pleinement la puissance des deux modèles DeepSeek V4 (Pro + Flash) au sein du framework multi-agent OpenCode. Principe fondamental : **Priorité à l'efficacité des tokens, obtenir les meilleurs résultats de développement avec un coût de contexte minimal**.

## Aperçu de la configuration actuelle

- Agent principal par défaut : `orchestrator`
- Modèle principal : `deepseek/deepseek-v4-pro`, modèle léger : `deepseek/deepseek-v4-flash`
- Niveaux hiérarchiques d'agents : `subagent_depth: 3` (prend en charge 3 niveaux d'imbrication d'agents)
- Isolation des modèles : double verrouillage `enabled_providers: ["deepseek"]` + `disabled_providers`
- Partage de session : désactivé (`share: "disabled"`) ; Instantanés : activés (`snapshot: true`)
- Base de permissions : autorisation par défaut, commandes bash destructrices réglées sur `ask` ; fichiers sensibles de type `.env` réglés sur `deny` ; répertoires externes réglés sur `ask`
- Compression de contexte : compression proactive DCP (seuils en pourcentage 60%/30%, adaptés à la fenêtre du modèle) + compaction native OpenCode en filet de sécurité (prune écarte les anciennes sorties d'outils)
- Règles globales : `AGENTS.md` (principes fondamentaux, contrat de refus de tâche, contexte et efficacité des tokens, auto-vérification, anti-patterns, etc.)
- Compétences : **17** `SKILL.md` dans le répertoire `skills/`, chargées à la demande via l'outil `skill` natif
- Plugins : `superpowers` (14 compétences orientées processus), `@tarquinen/opencode-dcp` (élagage intelligent du contexte)
- Fonctionnalités expérimentales : `batch_tool` activé par défaut

## Configuration du modèle DeepSeek

### Prérequis

- OpenCode ≥ v1.14.24 (le fournisseur DeepSeek est intégré)
- Clé API DeepSeek : demander sur [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Méthode 1 : Configuration interactive via TUI (recommandée)

```bash
opencode
# Dans la TUI, saisir : /connect → sélectionner DeepSeek → coller la clé API
# Puis : /models → sélectionner deepseek-v4-pro
```

La clé API est automatiquement persistée dans `~/.local/share/opencode/auth.json`.

### Méthode 2 : Variable d'environnement

Windows PowerShell :
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Configuration permanente : ajouter `DEEPSEEK_API_KEY` aux variables d'environnement système.

### Référence de configuration du fournisseur

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

Pour activer le thinking/reasoning sur le modèle Pro, ajouter dans `provider` :

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

> **Règle de nommage des ID de modèles** : `provider_id/model_id`, c'est-à-dire `deepseek/deepseek-v4-pro` et `deepseek/deepseek-v4-flash`.

## Installation et déploiement

### Méthode 1 : Cloner + variable d'environnement (recommandée, multiplateforme)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Puis pointer `OPENCODE_CONFIG_DIR` vers le sous-répertoire `opencode/` du dépôt.

**Windows (PowerShell)** — permanent :

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** — temporaire (session courante uniquement) :

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** — ajouter à `~/.bashrc` ou `~/.zshrc` :

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Méthode 2 : Lien symbolique vers le répertoire de configuration global

**Windows (PowerShell, administrateur requis) :**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS :**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Note de compatibilité** : `~/.config/opencode` est le chemin standard de configuration globale d'OpenCode. Les fichiers de configuration se trouvent dans le sous-répertoire `opencode/` de ce dépôt, leur structure respecte entièrement les conventions d'OpenCode — en pointant via variable d'environnement ou lien symbolique, ils sont automatiquement reconnus.

### Vérifier l'installation

Lancer OpenCode et confirmer :
1. `/models` → le modèle actuel est `deepseek/deepseek-v4-pro`
2. La liste des agents doit afficher 10 agents : `orchestrator`, `planner`, `deep-worker`, etc.
3. Saisir n'importe quelle requête, l'Orchestrator analyse automatiquement l'intention et route

## Répartition des modèles

Ce dépôt limite strictement la répartition aux deux modèles DeepSeek V4, sans introduire d'autres modèles :

| Modèle | Usage |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planification, architecture, analyse de cause racine, revue de code, implémentation lourde, orchestration principale |
| `deepseek/deepseek-v4-flash` | Exploration rapide, recherche externe, tâches légères, modifications simples |

### Stratégie de routage

- **Flash en priorité** : les tâches bien définies comme la recherche, la consultation et les modifications simples sont traitées en priorité par un agent flash
- **Pro dédié au raisonnement** : planification, analyse, revue, implémentation complexe — Pro uniquement
- **Montée en gamme automatique** : quand un agent flash ne peut pas accomplir une tâche, montée automatique vers Pro (avec contexte complet)

## Structure des agents

### Agent principal

| Agent | Modèle | Rôle |
| --- | --- | --- |
| `orchestrator` | v4-pro | Point d'entrée par défaut : filtrage d'intention (Intent Gate) + routage adapté au modèle + chaîne de repli |

### Sous-agents

| Agent | Modèle | Permissions | Rôle |
| --- | --- | --- | --- |
| `planner` | v4-pro | Lecture/écriture | Planification, architecture, décomposition des tâches |
| `deep-worker` | v4-pro | Lecture/écriture | Implémentation lourde, modifications multi-fichiers, débogage complexe |
| `oracle` | v4-pro | **Lecture seule** | Analyse de cause racine, compréhension approfondie du code |
| `reviewer` | v4-pro | **Lecture seule** | Revue de code sur deux axes (conventions + spécifications) + calibration de sévérité |
| `ui-builder` | v4-pro | Lecture/écriture | Tâches frontend et UI |
| `consultant` | v4-pro | Lecture/écriture | Discussion de solutions, conseils en bonnes pratiques |
| `explore` | v4-flash | **Lecture seule** | Recherche dans le code source, exploration parallèle |
| `librarian` | v4-flash | **Lecture seule** | Recherche documentaire, recherche Web |
| `light-orchestrator` | v4-flash | Lecture/écriture | Tâches légères, édition de fichier unique |

> `deep-worker` et `light-orchestrator` suivent le principe « pas de recherche, pas de délégation » — exécuter, pas explorer, le contexte est fourni par l'orchestrator.

## Commandes rapides

### Commandes de routage d'agent

| Commande | Agent | Usage |
| --- | --- | --- |
| `/deep` | `deep-worker` | Implémentation lourde, modifications multi-fichiers |
| `/quick` | `light-orchestrator` | Tâches légères, édition de fichier unique |
| `/ui` | `ui-builder` | Travail frontend/UI |
| `/review` | `reviewer` (code-review) | Revue parallèle sur deux axes (conventions + spécifications) + calibration de sévérité |
| `/review-pr` | `reviewer` (code-review + gh-cli) | Revue de PR et publication du commentaire sur GitHub |
| `/plan` | `planner` | Élaboration de plan, solution technique |
| `/search` | `librarian` | Recherche externe, consultation de documentation |
| `/oracle` | `oracle` | Analyse approfondie, traçage de problème |
| `/consult` | `consultant` | Consultation, comparaison, recommandation |

### Commandes opérationnelles

| Commande | Agent | Usage |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Générer un message de commit Conventional Commits (format inline) |
| `/release` | `deep-worker` (git-release) | Préparer une release avec tag |
| `/reflect` | `oracle` (reflect) | Détecter les frictions → proposer des optimisations de configuration |
| `/handoff` | `light-orchestrator` (handoff) | Compresser la session en document de passation |

### Commandes inline

| Commande | Agent | Usage |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Générer une carte de la structure du dépôt |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analyse → light-orchestrator applique la simplification |
| `/rmslop` | `deep-worker` (remove-deadcode) | Nettoyer le code mort et le slop IA |

### Commandes de spécification

| Commande | Agent | Usage |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Explorer le code → rédiger une proposition de changement |
| `/spec-apply` | `deep-worker` (spec-workflow) | Implémenter étape par étape selon tasks.md → archiver automatiquement |

## Compétences (Skills)

OpenCode expose les compétences à la demande via l'outil `skill` natif — les agents ne les chargent que lorsqu'elles sont nécessaires, sans consommer de contexte.

| Compétence | Rôle |
| --- | --- |
| `code-review` | Revue parallèle sur deux axes (conventions + spécifications) + calibration de sévérité + classification des constatations confirmed/plausible (trivial renvoyé au doc drift) et validation de sortie par le pass Validator |
| `codemap` | Générer une carte annotée de la structure du dépôt, économisant des tokens d'exploration |
| `gh-cli` | Référence complète GitHub CLI v2.97+ (Issues 2.0, copilot, agent-task, gh skill) + avertissement de sécurité (injection d'échappement) + inventaire des tâches via gh status |
| `git-master` | Opérations Git avancées : rebase, squash, bisect, reflog, worktree |
| `git-release` | Release avec tag : inférence SemVer, notes de release, commande gh release |
| `resolving-merge-conflicts` | Résoudre les conflits de merge par hunk : tracer l'intention originale, ne jamais inventer de comportement, jamais --abort |
| `handoff` | Compresser la session en document de passation (référence par chemin, pas de copie de contenu) |
| `opencode-config` | Rédiger et maintenir la configuration OpenCode |
| `reflect` | Amélioration continue : détecter les frictions → proposer des corrections minimales |
| `remove-deadcode` | Trouver et supprimer le code mort en toute sécurité, vérification LSP avant suppression |
| `security-review` | Audit de sécurité du diff avant fusion |
| `shared-language` | Construire un glossaire de domaine, économisant considérablement les tokens de contexte |
| `simplify` | Simplification de code préservant le comportement (oracle analyse → light-orchestrator applique) |
| `spec-workflow` | Changement piloté par spécification léger (propose → design → tasks → implement → archive) |
| `verification-planning` | Planifier le chemin de vérification le plus étroit avant l'implémentation |
| `verify-with-docs` | Vérifier la documentation API avant de coder, recherche prioritaire, prévention des hallucinations |
| `grilling` | Interview d'alignement des besoins : une question à la fois, choix multiples privilégiés, agir seulement après convergence des ambiguïtés (en phase avec la discipline de questionnement d'AGENTS.md) |

## Décisions de conception et historique des itérations

L'approche fondamentale s'inspire des meilleures pratiques de [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (filtrage d'intention, isolation lecture seule, anti-patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (priorité au planificateur, chaîne de repli, contrat de refus), [anomalyco/opencode](https://github.com/anomalyco/opencode) (schéma de configuration, système de compétences), [cli/cli](https://github.com/cli/cli) (ensemble complet de commandes gh), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (spécifications delta, mise à jour des propositions de changement), [mattpocock/skills](https://github.com/mattpocock/skills) (document de passation, débogage structuré, résolution de conflits de merge), [pi](https://github.com/earendil-works/pi) (répondre d'abord puis modifier, réponses concises) et [deepreview](https://github.com/mechanai/deepreview) (convergence basée sur la nouveauté, scan mécanique). Implémentation purement via configuration, zéro dépendance supplémentaire.

> **S'inspirer, pas copier** : des pipelines trop lourds, seuls les concepts de conception légers sont extraits ; les fonctionnalités redondantes sont couvertes par les agents/compétences existants, sans en ajouter de nouvelles. Suivant le principe « simplifier plutôt qu'ajouter », chaque itération vise une réduction nette des tokens.
> Origines des mécanismes de cette itération : grilling implémenté à partir de mattpocock/skills ; classification des constatations / table de mots-signaux / annotation des points de consensus empruntés aux mécanismes légers de deepreview ; gh status complété d'après le manuel cli/cli v2.97.

### Jalons d'itération

26 itérations depuis la v1, continuellement alignées sur les meilleures pratiques :

- **v1-v7 (Fondation)** : Liaison double modèle, système de rôles d'agents, routage par intention, règles globales AGENTS.md, répertoire skills, base de permissions
- **v8-v15 (Revue + Specs + Contrats)** : code-review calibration double axe, spec-workflow, alignement gh-cli, contrat de rejet, vérifications en arrière-plan
- **v16-v22 (Amincissement continu)** : Commandes 29→18 (-38%), AGENTS.md 290→211 (-27%), élagage no-op, validation de schéma
- **v23-v25 (Alignement + Sécurité)** : 6 dépôts upstream intégrés, gh-cli v2.97 avertissement d'injection, raffinement des prompts procedure-driven, tuning DCP
- **v26 (amincissement de ce cycle)** : prune:true et resserrement de tool_output 800/20480, DCP bascule sur des seuils en pourcentage 60%/30%, introduction de grilling en remplacement de writing-great-skills, opencode-config réduit de 131 à 64 lignes, code-review classification des constatations + validator, gh-cli complété par gh status, ajout de User Override dans AGENTS.md, discipline de coût de délégation de l'orchestrator, 7 fichiers d'agents allégés de 22 lignes nettes

## Structure du dépôt

```text
├── opencode/                     # Fichiers de configuration OpenCode
│   ├── .ai/
│   │   └── calibration.yml           # Calibration de sévérité code-review
│   ├── agents/                       # 10 agents spécialisés
│   │   ├── orchestrator.md           # Point d'entrée principal : filtrage d'intention + routage adapté au modèle
│   │   ├── planner.md                # pro : architecture et planification
│   │   ├── deep-worker.md            # pro : implémentation lourde
│   │   ├── oracle.md                 # pro : analyse de code approfondie (lecture seule)
│   │   ├── reviewer.md               # pro : revue de code sur deux axes (lecture seule)
│   │   ├── consultant.md             # pro : discussion de solutions et conseils
│   │   ├── ui-builder.md             # pro : frontend et UI
│   │   ├── explore.md                # flash : recherche dans le code source (lecture seule)
│   │   ├── librarian.md              # flash : recherche externe (lecture seule)
│   │   └── light-orchestrator.md     # flash : modifications simples
│   ├── skills/                       # 17 compétences chargées à la demande
│   │   ├── code-review/              # Revue parallèle sur deux axes + calibration de sévérité
│   │   ├── codemap/                  # Génération de carte de structure de dépôt
│   │   ├── gh-cli/                   # Référence GitHub CLI v2.97+ + avertissement sécurité
│   │   ├── git-master/               # Opérations Git avancées
│   │   ├── git-release/              # Release avec tag
│   │   ├── resolving-merge-conflicts/ # Résolution de conflits de merge par hunk
│   │   ├── handoff/                  # Compression de session en document de passation
│   │   ├── opencode-config/          # Méta-compétence : rédaction de configuration de ce dépôt
│   │   ├── reflect/                  # Amélioration continue
│   │   ├── remove-deadcode/          # Détection et suppression de code mort
│   │   ├── security-review/          # Checklist d'audit de sécurité
│   │   ├── shared-language/          # Glossaire de domaine (économie de tokens)
│   │   ├── simplify/                 # Simplification de code préservant le comportement
│   │   ├── spec-workflow/            # Développement piloté par spécification
│   │   ├── verification-planning/    # Planification du chemin de vérification avant implémentation
│   │   ├── verify-with-docs/         # Vérification API avec recherche prioritaire
│   │   └── grilling/             # Interview d'alignement des besoins
│   ├── opencode.jsonc                # Configuration principale (18 commandes)
│   ├── AGENTS.md                     # Règles globales (206 lignes)
│   └── dcp.jsonc                     # Compression de contexte DCP (DeepSeek 128K, seuils en pourcentage 60%/30%)
├── README.md
├── LICENSE
└── README.*.md
```

## Guide d'utilisation

### Mode 1 : Routage automatique par l'Orchestrator (par défaut)

Décrivez vos besoins en langage naturel, l'Orchestrator analyse automatiquement l'intention et sélectionne l'agent et le modèle les plus adaptés pour l'exécution.

```text
« Aide-moi à diagnostiquer l'erreur de cette API de connexion »     → oracle analyse la cause racine → retourne un rapport de diagnostic
« Optimise cette boucle, les performances sont trop mauvaises »     → oracle analyse → deep-worker implémente l'optimisation
« Peux-tu revoir cette PR pour moi »                                 → reviewer revue multi-dimensionnelle → retourne un rapport gradué
« Je veux ajouter une fonction d'export au module utilisateur »      → planner élabore un plan → deep-worker implémente
« Comment utiliser l'API use() de React 19 »                        → librarian consulte la doc → retourne signature et exemples
```

### Mode 2 : Alias de commandes directes

| Scénario | Commande |
| --- | --- |
| Implémentation complexe / modifications multi-fichiers | `/deep` |
| Modification légère / édition de fichier unique | `/quick` |
| Élaboration de solution technique / conception d'architecture | `/plan` |
| Diagnostic de bug / analyse approfondie | `/oracle` |
| Revue de code | `/review` |
| Recherche externe / consultation d'API | `/search` |
| Travail frontend / UI | `/ui` |
| Discussion de solutions / comparaison et arbitrage | `/consult` |
| Débogage structuré | `/oracle` |

### Workflows types

**Développer une nouvelle fonctionnalité (piloté par spécification) :**
```text
/spec-propose  → /spec-apply  → /review
```

**Diagnostiquer un bug :**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Revue de code :**
```text
/review-pr   ← Revue de PR + publication automatique du commentaire
/review      ← Revue parallèle sur deux axes
```

## Philosophie de conception

- **Piloté par la configuration pure, zéro dépendance supplémentaire** — toutes les capacités sont implémentées par `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Exploitation optimale des deux modèles DeepSeek V4** — Pro pour le raisonnement et la décision, Flash pour la recherche et l'exécution légère
- **Priorité à l'efficacité des tokens** — référence par chemin plutôt que collage de fichiers, compétences chargées à la demande, compression à plusieurs niveaux
- **Plugins pour amplifier l'efficacité sans prendre le dessus** — superpowers fournit la discipline de processus, DCP remplace la simple troncature par une compression intelligente (seuils en pourcentage auto-adaptatifs, compaction native en filet de sécurité)
- **Séparation de l'exécution et de l'exploration** — deep-worker/light-orchestrator interdits de recherche/délégation, explore/librarian interdits de modification
- **Amélioration continue** — reflect mécanise la détection de frictions, code-review avec calibration sur deux axes garantit la qualité
