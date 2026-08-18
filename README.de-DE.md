# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | **Deutsch** | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Die optimale OpenCode × DeepSeek-Konfiguration** — eine Konfiguration, die die Fähigkeiten der beiden DeepSeek-V4-Modelle (Pro + Flash) im OpenCode-Multi-Agent-Framework voll ausschöpft. Kernphilosophie: **Token-Effizienz zuerst — beste Entwicklungsergebnisse bei minimalen Kontextkosten**.

## Konfigurationsübersicht

- Standard-Hauptagent: `orchestrator`
- Hauptmodell: `deepseek/deepseek-v4-pro`, leichtes Modell: `deepseek/deepseek-v4-flash`
- Agent-Verschachtelung: `subagent_depth: 3` (unterstützt 3 Ebenen verschachtelter Agenten)
- Modell-Isolation: `enabled_providers: ["deepseek"]` als einzige Sperre
- Sitzungsfreigabe: deaktiviert (`share: "disabled"`); Snapshots: aktiviert (`snapshot: true`)
- Berechtigungs-Baseline: standardmäßig erlauben; destruktive bash-Befehle auf `ask`; sensible Dateien wie `.env` auf `deny`; externe Verzeichnisse auf `ask`; bash-Whitelist für schreibgeschützte Agenten (standardmäßig alles `deny` + nur schreibgeschützte Unterbefehle erlaubt)
- Kontextkomprimierung: eingebaute Komprimierung (opencode.jsonc) für automatische Auslösung + Prune alter Tool-Ausgaben; DCP (dcp.jsonc) für aktive Deduplizierung + Komprimierungsschwellen — beides ergänzt sich
- Globale Regeln: `AGENTS.md` (Kernprinzipien, Ablehnungsvertrag für Aufgaben, Selbstverifikation, Anti-Muster usw.; Kontext-/Token-Disziplin in `orchestrator` verlagert)
- Skills: **23** `SKILL.md`-Skills im Verzeichnis `skills/`, bei Bedarf über das native `skill`-Tool geladen
- Plugins: `superpowers` (v6.3.0, prozessorientierte Skills), `@tarquinen/opencode-dcp` (intelligente Kontextbereinigung)

## DeepSeek-Modellkonfiguration

### Voraussetzungen

- OpenCode ≥ v1.18.x (DeepSeek-Provider ist integriert)
- DeepSeek-API-Key: unter [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) beantragen

### Variante 1: Interaktive TUI-Konfiguration (empfohlen)

```bash
opencode
# In der TUI eingeben: /connect → DeepSeek wählen → API-Key einfügen
# Danach: /models → deepseek-v4-pro wählen
```

Der API-Key wird automatisch unter `~/.local/share/opencode/auth.json` gespeichert.

### Variante 2: Umgebungsvariable

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Dauerhaft einrichten: `DEEPSEEK_API_KEY` zu den Systemumgebungsvariablen hinzufügen.

### Provider-Konfigurationsreferenz

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"]
}
```

Um Thinking/Reasoning für das Pro-Modell zu aktivieren, kann in `provider` Folgendes ergänzt werden:

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

> **Namenskonvention für Modell-IDs**: `provider_id/model_id`, also `deepseek/deepseek-v4-pro` und `deepseek/deepseek-v4-flash`.

## Installation

### Variante 1: Klonen + Umgebungsvariable (empfohlen, plattformübergreifend)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Danach `OPENCODE_CONFIG_DIR` auf das Unterverzeichnis `opencode/` im Repository zeigen lassen — fertig.

**Windows (PowerShell)** — dauerhaft:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** — temporär (nur aktuelle Sitzung):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** — an `~/.bashrc` oder `~/.zshrc` anhängen:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Variante 2: Symlink auf das globale Konfigurationsverzeichnis

**Windows (PowerShell, Administratorrechte erforderlich):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Kompatibilitätshinweis**: `~/.config/opencode` ist der standardmäßige globale Konfigurationspfad von OpenCode. Das Unterverzeichnis `opencode/` dieses Repositorys enthält `agents/`, `skills/`, `AGENTS.md` und weitere Dateien und folgt vollständig den OpenCode-Konventionen — nach dem Verweis per Umgebungsvariable oder Symlink wird es automatisch erkannt.

### Installation verifizieren

OpenCode starten und prüfen:
1. `/models` → aktuelles Modell ist `deepseek/deepseek-v4-pro`
2. in der Agent-Liste sollten die 10 Agenten `orchestrator`, `planner`, `deep-worker` usw. sichtbar sein
3. beliebige Anfrage eingeben — der Orchestrator analysiert die Absicht automatisch und routet

## Modellaufteilung

Dieses Repository beschränkt sich strikt auf die beiden DeepSeek-V4-Modelle, ohne weitere Modelle einzuführen:

| Modell | Zweck |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planung, Architektur, Ursachenanalyse, Code-Review, schwere Implementierung, zentrale Steuerung |
| `deepseek/deepseek-v4-flash` | schnelle Exploration, externe Recherche, leichte Aufgaben, einfache Bearbeitungen |

### Routing-Strategie

- **Flash zuerst**: klar definierte Aufgaben wie Suche, Nachschlagen und einfache Bearbeitungen gehen zuerst an Flash-Agenten
- **Pro fürs Denken**: Planung, Analyse, Review, komplexe Implementierung — ausschließlich Pro
- **Automatisches Upgrade**: wenn ein Flash-Agent nicht ausreicht, automatisches Upgrade auf Pro (mit vollem Kontext)

## Agent-Struktur

### Hauptagent

| Agent | Modell | Zweck |
| --- | --- | --- |
| `orchestrator` | v4-pro | Standardeinstieg: Intent Gate + modellbewusstes Routing + Fallback-Kette |

### Subagenten

| Agent | Modell | Berechtigungen | Zweck |
| --- | --- | --- | --- |
| `planner` | v4-pro | Lesen/Schreiben | Planung, Architektur, Aufgabenzerlegung |
| `deep-worker` | v4-pro | Lesen/Schreiben | schwere Implementierungen, Änderungen über mehrere Dateien, komplexes Debugging |
| `oracle` | v4-pro | **Nur Lesen** | Ursachenanalyse, tiefes Codeverständnis |
| `reviewer` | v4-pro | **Nur Lesen** | zweiachsiger Code-Review (Konventionen + Spezifikation) + Schweregrad-Kalibrierung |
| `ui-builder` | v4-pro | Lesen/Schreiben | Frontend- und UI-Aufgaben |
| `consultant` | v4-pro | Lesen/Schreiben | Lösungsdiskussion, Best-Practice-Empfehlungen |
| `explore` | v4-flash | **Nur Lesen** | Codebase-Suche, parallele Exploration |
| `librarian` | v4-flash | **Nur Lesen** | Dokumentenrecherche, Websuche |
| `light-orchestrator` | v4-flash | Lesen/Schreiben | leichte Aufgaben, Einzeldatei-Bearbeitung |

> `deep-worker` und `light-orchestrator` folgen dem Prinzip „keine Recherche, keine Delegation" — Ausführen statt Erkunden, der Kontext wird vom Orchestrator bereitgestellt.
>
> Schreibgeschützte Agenten (`oracle`/`reviewer`/`explore`/`librarian`) sind wirklich schreibgeschützt: `edit: deny` + bash-Whitelist (standardmäßig alles `deny`, nur schreibgeschützte Unterbefehle wie `git status/diff/log/show/blame/grep`, `rg` usw. erlaubt; `oracle`/`reviewer` dürfen zusätzlich `gh pr view/diff`, `gh issue view`, `gh api` für Antworten auf `/review-pr`).

## Schnellbefehle

### Agent-Routing-Befehle

| Befehl | Agent | Zweck |
| --- | --- | --- |
| `/deep` | `deep-worker` | schwere Implementierungen, Änderungen über mehrere Dateien |
| `/quick` | `light-orchestrator` | leichte Aufgaben, Einzeldatei-Bearbeitung |
| `/ui` | `ui-builder` | Frontend-/UI-Arbeit |
| `/review` | `reviewer` (code-review) | zweiachsiger paralleler Review (Konventionen + Spezifikation) + Schweregrad-Kalibrierung |
| `/review-pr` | `reviewer` (code-review + gh-cli) | PR reviewen und auf GitHub antworten |
| `/plan` | `planner` | Pläne und technische Lösungen erstellen |
| `/search` | `librarian` | externe Suche, Dokumentation nachschlagen |
| `/oracle` | `oracle` | Tiefenanalyse, Ursachenforschung |
| `/consult` | `consultant` | Beratung, Vergleiche, Empfehlungen |

### Aktionsbefehle

| Befehl | Agent | Zweck |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Conventional-Commits-Commit-Message generieren (Inline-Format) |
| `/release` | `deep-worker` (git-release) | Tag-Release vorbereiten |
| `/reflect` | `oracle` (reflect) | Reibungspunkte finden → Konfigurationsoptimierung vorschlagen |
| `/handoff` | `light-orchestrator` (handoff) | Sitzung zu Übergabedokument komprimieren |

### Inline-Befehle

| Befehl | Agent | Zweck |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Strukturplan des Repositorys generieren |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analysiert → light-orchestrator wendet die Vereinfachung an |
| `/rmslop` | `deep-worker` (remove-deadcode) | toten Code und AI-Slop entfernen |

### Spezifikationsbefehle

| Befehl | Agent | Zweck |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Code erkunden → Änderungsvorschlag entwerfen |
| `/spec-apply` | `deep-worker` (spec-workflow) | Schritt für Schritt nach tasks.md umsetzen → automatisch archivieren |

## Skills

OpenCode stellt Skills bei Bedarf über das native `skill`-Tool bereit — Agenten laden sie nur, wenn sie sie brauchen, ohne Kontext zu belegen.

| Skill | Zweck |
| --- | --- |
| `code-review` | token-sparender, mehrdimensionaler Code-Review: Bericht nach Dimension + Schweregrad, höchste Konfidenz für übereinstimmende Punkte, Selbst-Falsifikation à la deepreview, ändert nie eigenmächtig Code |
| `codemap` | annotierter Strukturplan des Repositorys, schnelle Orientierung, spart Explorations-Token |
| `gh-cli` | GitHub-CLI-v2.97+-Referenz: Paginierung, Repository-Auswahl, discussions/projects/rulesets/skills, Rate Limits, gh-aw-Agentic-CI, gh-api-Fallback |
| `git-master` | fortgeschrittene Git-Operationen: rebase, squash, fixup, bisect, reflog, Code-Archäologie, worktree |
| `git-release` | Tag-Release: Release-Notes, SemVer-Ableitung, gh-release-Befehl |
| `resolving-merge-conflicts` | Merge-Konflikte hunk-weise lösen: ursprüngliche Absicht nachverfolgen, nie neues Verhalten erfinden, nie --abort |
| `handoff` | Sitzung zu Übergabedokument komprimieren (Pfadverweise, kein Kopieren von Inhalten) |
| `opencode-config` | OpenCode-Konfiguration dieses Repositorys schreiben und pflegen (agents/skills/commands/permissions) |
| `reflect` | kontinuierliche Verbesserung: Reibungspunkte finden → minimale, wartbare Korrekturen vorschlagen |
| `remove-deadcode` | toten Code sicher finden und löschen, vor jedem Löschen per Toolchain/LSP verifiziert |
| `security-review` | Sicherheitsprüfung vor dem Merge (Injection/XSS/SSRF/Secrets/Deserialisierung/Pfad-Traversal), berichtet nur, ändert nichts |
| `shared-language` | Domänen-Glossar aufbauen (CONTEXT.md), spart massiv Token |
| `simplify` | verhaltenserhaltende Code-Vereinfachung (oracle analysiert → Anwendung) |
| `spec-workflow` | leichtgewichtige, spezifikationsgetriebene Änderungen: proposal → specs → design → tasks → archive |
| `verification-planning` | schmalsten Verifikationspfad vor der Implementierung planen |
| `verify-with-docs` | API-Dokumentation vor dem Codieren prüfen, Retrieval zuerst, gegen Halluzinationen |
| `grilling` | Interview zur Anforderungsklärung: eine Frage nach der anderen, Multiple-Choice bevorzugt, erst handeln, wenn Mehrdeutigkeiten geklärt sind |
| `tech-debt-audit` | Technische-Schulden-Audit über 9 Dimensionen (toter Code/Duplikate/Namensdrift/Komplexität/Abhängigkeiten/Fehlerbehandlung/Tests/Dokumentation/Sicherheit), reiner Bericht ohne Codeänderungen |
| `wait-what` | unverständliche Nutzernachricht erst in einem Satz paraphrasieren und bestätigen, dann handeln |
| `writing-for-agents` | Schreibhebel für Dokumentation, die Agenten lesen (skill/AGENTS.md/Verweisdokumente) |
| `to-questionnaire` | einmaliger Offline-Fragebogen (asynchrone Beantwortung), im Unterschied zum Live-Interview von grilling |
| `research` | vertiefte Recherche offener Themen, Ergebnis als Markdown mit Zitaten, im Unterschied zur punktuellen Prüfung von verify-with-docs |
| `wizard` | schrittweiser Assistent für Menschen (bash-Skript, per `bash -n` verifiziert), führt durch Schritte, die nur der Mensch ausführen kann |

## Designentscheidungen und Iterationsprotokoll

Die Kernideen sind inspiriert von [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (Intent Gate, schreibgeschützte Isolation, Anti-Muster), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (Scheduler zuerst, Fallback-Kette, Ablehnungsvertrag, Prompt-Cache-Sicherheit, impact×confidence÷cost), [anomalyco/opencode](https://github.com/anomalyco/opencode) (Config-Schema, Skill-System), [cli/cli](https://github.com/cli/cli) (gh-v2.97-Befehlsliste, Rate Limits, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (Delta-Specs, OPSX-Aktionsfluss update/verify/vier Fragen), [mattpocock/skills](https://github.com/mattpocock/skills) (Disziplin beim Konfliktlösen, Übergabedokumente), [pi](https://github.com/earendil-works/pi) (erst antworten, dann ändern; knappe Antworten; unabhängige Sitzungssammlung) und [deepreview](https://github.com/mechanai/deepreview) (Novelty-Klassifikation, Effektivgrößen-Routing, Points of Agreement) — reine Konfiguration, null zusätzliche Abhängigkeiten.

> **Inspiriert, nicht kopiert**: von zu schwergewichtigen Pipelines werden nur die Leichtbau-Ideen übernommen; redundante Funktionen decken die vorhandenen Agents/Skills ab, nichts Neues kommt hinzu. Nach dem Prinzip „Verschlankung vor Ergänzung" zielt jede Iteration auf Netto-Token-Reduktion.
>
> **Quellen der Mechanismen dieser Runde (v28)**: DeepSeek-Cache+Thinking-Disziplin, scope-first+Delegation zuerst, atomare TODOs in AGENTS.md verlagert; 5 neue Skills (wait-what/writing-for-agents/to-questionnaire/research/wizard) — jetzt 23; gh-cli um 4 GHSA-Sicherheitseinträge ergänzt; code-review um deepreview-Selbst-Falsifikation erweitert; .ai/calibration.yml gelöscht (Kalibrierungsregeln in code-review eingebettet).
>
> **Geprüft, aber verworfen**: die übrigen Prozess-Skills aus mattpocock/skills (code-review, tdd, implement usw. — sie überschneiden sich mit superpowers/vorhandenen Skills); superpowers hat keine Konfigurationsknöpfe und bleibt als Plugin-String eingebunden.

### Iterations-Meilensteine

Seit v1 wurden 28 Iterationen durchlaufen, kontinuierlich abgeglichen mit den Best Practices der Upstream-Repositories:

- **v1-v7 (Fundament)**: Bindung der beiden Modelle, Agent-Rollensystem, Intent-Gate-Routing, globale Regeln in AGENTS.md, Skills-Verzeichnis, Berechtigungs-Baseline
- **v8-v15 (Review + Spec + Vertrag)**: zweiachsige code-review-Kalibrierung, spec-workflow, gh-cli-Angleichung, Ablehnungsvertrag, Hintergrundprüfung
- **v16-v22 (kontinuierliche Verschlankung)**: Befehle 29→18 (−38 %), AGENTS.md 290→211 (−27 %), satzweises No-op-Pruning, Schema-Validierung gegen tote Schlüssel
- **v23-v25 (Angleichung + Sicherheit)**: 6 Upstream-Repositories integriert, gh-cli-v2.97-Abschnitt zu Escaping/Injection-Sicherheit, procedure-driven-Prompt-Verfeinerung, DCP-Fenster-Feintuning
- **v26 (Verschlankung dieser Runde)**: prune:true und tool_output auf 800/20480 verschärft, DCP auf 60 %/30 %-Prozentschwellen umgestellt, grilling ersetzt writing-great-skills, opencode-config 131→64 gestrafft, code-review mit Stufen + Validator, gh-cli um gh status ergänzt, AGENTS.md um User Override erweitert, Kostendisziplin bei Orchestrator-Delegation, 7 Agent-Dateien netto 22 Zeilen weniger
- **v27 (Löschen/Migration/Ergänzung)**: tote batch_tool-Konfiguration, unwirksames `write: deny` bei schreibgeschützten Agenten und 3 redundante bash-Einträge gelöscht; Abschnitt Context Management in den Orchestrator-Abschnitt verschoben; bash-Whitelist für schreibgeschützte Agenten, read um `.env` ergänzt; Skill tech-debt-audit neu; 15 Skill-Descriptions um 30–40 % gestrafft; gh-cli um 5 Punkte ergänzt (Rate Limits, gh-Skill-Host, gh-aw usw.), code-review um Points of Agreement erweitert, spec-workflow um zwei update-Fragen ergänzt, Orchestrator um unabhängige Sitzungssammlung + Prompt-Cache-Sicherheit erweitert, deep-worker um impact×confidence÷cost erweitert
- **v28 (Disziplin-Refactoring)**: Cache+Thinking-Disziplin, scope-first+Delegation zuerst, atomare TODOs in AGENTS.md verlagert; 5 neue Skills — jetzt 23; gh-cli um 4 GHSA-Einträge ergänzt; code-review um deepreview-Selbst-Falsifikation erweitert; .ai/calibration.yml gelöscht (Regeln in code-review eingebettet); README in zehn Sprachen synchronisiert

## Repository-Struktur

```text
├── opencode/                     # OpenCode-Konfigurationsverzeichnis (eigenständig einsetzbar)
│   ├── agents/                   # 10 spezialisierte Agenten
│   │   ├── orchestrator.md       # Haupteinstieg: Intent Gate + modellbewusstes Routing
│   │   ├── planner.md            # pro: Architektur und Planung
│   │   ├── deep-worker.md        # pro: schwere Implementierungen
│   │   ├── oracle.md             # pro: tiefe Code-Analyse (nur lesen)
│   │   ├── reviewer.md           # pro: zweiachsiger Code-Review (nur lesen)
│   │   ├── consultant.md         # pro: Lösungsdiskussion und Empfehlungen
│   │   ├── ui-builder.md         # pro: Frontend und UI
│   │   ├── explore.md            # flash: Codebase-Suche (nur lesen)
│   │   ├── librarian.md          # flash: externe Recherche (nur lesen)
│   │   └── light-orchestrator.md # flash: einfache Bearbeitungen
│   ├── skills/                   # 23 bei Bedarf geladene Skills
│   │   ├── code-review/          # zweiachsiger paralleler Review + Schweregrad-Kalibrierung
│   │   ├── codemap/              # Strukturplan des Repositorys generieren
│   │   ├── gh-cli/               # GitHub-CLI-v2.97+-Referenz + Sicherheitswarnungen
│   │   ├── git-master/           # fortgeschrittene Git-Operationen
│   │   ├── git-release/          # Tag-Release
│   │   ├── handoff/              # Sitzungen zu Übergabedokumenten komprimieren
│   │   ├── opencode-config/      # Meta-Skill: Konfiguration dieses Repositorys schreiben
│   │   ├── reflect/              # kontinuierliche Verbesserung
│   │   ├── remove-deadcode/      # toten Code erkennen und löschen
│   │   ├── resolving-merge-conflicts/ # Disziplin für hunk-weises Lösen von Konflikten
│   │   ├── security-review/      # Sicherheits-Checkliste
│   │   ├── shared-language/      # Domänen-Glossar (spart Token)
│   │   ├── simplify/             # verhaltenserhaltende Code-Vereinfachung
│   │   ├── spec-workflow/        # spezifikationsgetriebene Entwicklung
│   │   ├── tech-debt-audit/      # Technische-Schulden-Audit (9 Dimensionen, reiner Bericht)
│   │   ├── verification-planning/ # Verifikationspfad vor der Implementierung planen
│   │   ├── verify-with-docs/     # retrieval-first API-Verifikation
│   │   ├── grilling/             # Interview zur Anforderungsklärung
│   │   ├── research/             # vertiefte Recherche offener Themen (mit Zitaten)
│   │   ├── to-questionnaire/     # einmaliger Offline-Fragebogen
│   │   ├── wait-what/            # unverständliche Nachrichten erst in einem Satz bestätigen
│   │   ├── wizard/               # schrittweiser Assistent für Menschen (per bash -n verifiziert)
│   │   └── writing-for-agents/   # Dokumentation für Agenten schreiben
│   ├── opencode.jsonc            # Hauptkonfiguration (18 Befehle)
│   ├── AGENTS.md                 # globale Regeln
│   └── dcp.jsonc                 # DCP-Kontextkomprimierung (DeepSeek 128K, 60 %/30 %-Prozentschwellen)
├── README.md
├── LICENSE
└── README.*.md                   # READMEs in anderen Sprachen
```

## Benutzung

### Modus 1: Automatisches Routing durch den Orchestrator (Standard)

Anforderungen in natürlicher Sprache beschreiben — der Orchestrator analysiert die Absicht automatisch, wählt den passenden Agenten und das passende Modell und führt aus.

```text
„Hilf mir, den Fehler in der Login-API zu untersuchen"     → oracle analysiert die Ursache → gibt Diagnosebericht zurück
„Optimiere diese Schleife, die Performance ist zu schlecht" → oracle analysiert → deep-worker setzt die Optimierung um
„Bitte review diesen PR für mich"                          → reviewer prüft mehrdimensional → gibt abgestuften Bericht zurück
„Ich möchte dem User-Modul eine Exportfunktion hinzufügen" → planner entwirft den Plan → deep-worker implementiert
„Wie nutzt man die use()-API von React 19?"                → librarian schlägt die Doku nach → gibt Signatur und Beispiele zurück
```

### Modus 2: Direkt über Befehlsaliase

| Szenario | Befehl |
| --- | --- |
| komplexe Implementierung / Änderungen über mehrere Dateien | `/deep` |
| leichte Änderung / Einzeldatei-Bearbeitung | `/quick` |
| technische Lösung / Architekturentwurf | `/plan` |
| Bug-Untersuchung / Tiefenanalyse | `/oracle` |
| Code-Review | `/review` |
| externe Suche / API nachschlagen | `/search` |
| Frontend-/UI-Arbeit | `/ui` |
| Lösungsdiskussion / Abwägung von Alternativen | `/consult` |
| strukturiertes Debugging | `/oracle` |

### Typische Workflows

**Neue Funktionen entwickeln (spezifikationsgetrieben):**
```text
/spec-propose  → /spec-apply  → /review
```

**Bug-Untersuchung:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Code-Review:**
```text
/review-pr   ← PR reviewen + automatisch antworten
/review      ← zweiachsiger paralleler Review
```

## Designphilosophie

- **Reine Konfiguration, null zusätzliche Abhängigkeiten** — alle Fähigkeiten werden über `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md` umgesetzt
- **Maximale Nutzung der beiden DeepSeek-V4-Modelle** — Pro übernimmt Denken und Entscheiden, Flash Abfragen und leichte Ausführung
- **Token-Effizienz zuerst** — Pfadverweise statt eingefügter Dateien, On-Demand-Laden der Skills, gestufte Komprimierung
- **Plugins verstärken, ohne zu dominieren** — superpowers liefert Prozessdisziplin, DCP (dcp.jsonc) aktive Deduplizierung + Komprimierungsschwellen, eingebaute Komprimierung (opencode.jsonc) automatische Auslösung + Prune als Auffangnetz
- **Trennung von Ausführung und Exploration** — deep-worker/light-orchestrator ohne Recherche/Delegation, explore/librarian ohne Änderungen
- **Cache- und Thinking-Disziplin** — stabile statische Präfixe treffen den DeepSeek-Prompt-Cache; Temperatur 0 bei Coding-Aufgaben; Thinking nur für Denkaufgaben, aus bei einfachen/Recherche-Aufgaben
- **Scope First + Delegate Always** — erst den Umfang festlegen (bei 2+ Schritten/mehreren Dateien/Architekturänderungen zuerst zum Planner), dann Ausführung delegieren; Top-Level-Token bleiben für Routing und schwierige Fragen
- **Atomare TODOs** — bei mehrstufigen Aufgaben zuerst eine geordnete TODO-Liste schreiben, einzeln in_progress→completed; Format `path: action for scenario — verify by check`
- **Kontinuierliche Verbesserung** — reflect findet Reibungspunkte systematisch, code-review sichert Qualität durch zweiachsige Kalibrierung
