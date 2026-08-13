# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | **Deutsch** | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Die optimale OpenCode × DeepSeek-Konfiguration** — eine Konfiguration, die die Fähigkeiten der beiden DeepSeek V4-Modelle (Pro + Flash) im OpenCode Multi-Agent-Framework voll ausschöpft. Kernphilosophie: **Token-Effizienz zuerst — beste Entwicklungsergebnisse bei minimalen Kontextkosten**.

## Konfigurationsübersicht

- Standard-Hauptagent: `orchestrator`
- Hauptmodell: `deepseek/deepseek-v4-pro`, Leichtgewichtmodell: `deepseek/deepseek-v4-flash`
- Agent-Tiefe: `subagent_depth: 3` (unterstützt 3 Ebenen Agent-Verschachtelung)
- Modellisolierung: doppelte Absicherung durch `enabled_providers: ["deepseek"]` + `disabled_providers`
- Sitzungsfreigabe: deaktiviert (`share: "disabled"`); Snapshots: aktiviert (`snapshot: true`)
- Berechtigungsbasis: standardmäßig erlauben, destruktive Bash-Befehle auf `ask`; sensible Dateien wie `.env` auf `deny`; externe Verzeichnisse auf `ask`; Bash-Whitelist für Read-only-Agenten (standardmäßig alles deny + nur Read-only-Unterbefehle erlaubt)
- Kontextkomprimierung: DCP proaktive Komprimierung bei 60%-Schwelle + native OpenCode-Auto-Compaction als Absicherung nahe dem Überlauf, zwei komplementäre Ebenen (prune kappt alte Tool-Ausgaben)
- Globale Regeln: `AGENTS.md` (Kernprinzipien, Task-Rejection-Contract, Selbstverifikation, Anti-Patterns usw.; Kontext-/Token-Disziplin in den `orchestrator` verlagert)
- Skills: **18** `SKILL.md`-Skills im Verzeichnis `skills/`, bei Bedarf über das native `skill`-Tool geladen
- Plugins: `superpowers` (v6.3.0, prozessorientierte Skills), `@tarquinen/opencode-dcp` (intelligentes Context-Trimming)

## DeepSeek-Modellkonfiguration

### Voraussetzungen

- OpenCode ≥ v1.14.24 (DeepSeek-Provider ist integriert)
- DeepSeek API Key: unter [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) beantragen

### Methode 1: Interaktive TUI-Konfiguration (empfohlen)

```bash
opencode
# In der TUI: /connect → DeepSeek auswählen → API Key einfügen
# Dann: /models → deepseek-v4-pro auswählen
```

Der API Key wird automatisch in `~/.local/share/opencode/auth.json` persistiert.

### Methode 2: Umgebungsvariable

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Dauerhafte Einrichtung: `DEEPSEEK_API_KEY` zu den Systemumgebungsvariablen hinzufügen.

### Provider-Konfigurationsreferenz

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

Um Thinking/Reasoning für das Pro-Modell zu aktivieren, in `provider` ergänzen:

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

> **Modell-ID-Benennung**: `provider_id/model_id`, d. h. `deepseek/deepseek-v4-pro` und `deepseek/deepseek-v4-flash`.

## Installation und Deployment

### Methode 1: Klonen + Umgebungsvariable (empfohlen, plattformübergreifend)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Dann `OPENCODE_CONFIG_DIR` auf das Unterverzeichnis `opencode/` des Repositorys setzen.

**Windows (PowerShell)** — dauerhaft:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** — temporär (nur aktuelle Sitzung):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** — in `~/.bashrc` oder `~/.zshrc` eintragen:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Methode 2: Symbolischer Link zum globalen Konfigurationsverzeichnis

**Windows (PowerShell, Administratorrechte erforderlich):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Kompatibilitätshinweis**: `~/.config/opencode` ist der standardmäßige globale Konfigurationspfad von OpenCode. Das Unterverzeichnis `opencode/` dieses Repositorys enthält `agents/`, `skills/`, `AGENTS.md` usw. und folgt vollständig den OpenCode-Konventionen — durch Verweis via Umgebungsvariable oder symbolischen Link werden sie automatisch erkannt.

### Installation überprüfen

OpenCode starten und prüfen:
1. `/models` → aktuelles Modell ist `deepseek/deepseek-v4-pro`
2. Die Agent-Liste sollte `orchestrator`, `planner`, `deep-worker` und alle 10 Agenten enthalten
3. Eine beliebige Anfrage stellen — der Orchestrator analysiert automatisch die Absicht und leitet sie weiter

## Modellaufteilung

Dieses Repository beschränkt sich strikt auf die beiden DeepSeek V4-Modelle — keine weiteren Modelle:

| Modell | Verwendung |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planung, Architektur, Ursachenanalyse, Code-Review, Heavy-Implementierung, Orchestrierung |
| `deepseek/deepseek-v4-flash` | Schnelle Exploration, externe Recherche, leichte Aufgaben, einfache Bearbeitungen |

### Routing-Strategie

- **Flash zuerst**: klar definierte Aufgaben wie Suche, Lookup, einfache Bearbeitungen gehen bevorzugt an Flash-Agenten
- **Pro für Reasoning**: Planung, Analyse, Review, komplexe Implementierung — nur Pro
- **Automatisches Upgrade**: wenn ein Flash-Agent eine Aufgabe nicht bewältigen kann, automatisches Upgrade auf Pro (mit vollständigem Kontext)

## Agent-Struktur

### Primärer Agent

| Agent | Modell | Rolle |
| --- | --- | --- |
| `orchestrator` | v4-pro | Standardeinstieg: Intent Gate + modellbewusstes Routing + Fallback-Kette |

### Subagenten

| Agent | Modell | Berechtigung | Rolle |
| --- | --- | --- | --- |
| `planner` | v4-pro | Lesen/Schreiben | Planung, Architektur, Aufgabenzerlegung |
| `deep-worker` | v4-pro | Lesen/Schreiben | Heavy-Implementierung, Multi-Datei-Änderungen, komplexes Debugging |
| `oracle` | v4-pro | **Nur Lesen** | Ursachenanalyse, tiefgehendes Code-Verständnis |
| `reviewer` | v4-pro | **Nur Lesen** | Zweiachsiges Code-Review (Konvention + Spezifikation) + Schweregrad-Kalibrierung |
| `ui-builder` | v4-pro | Lesen/Schreiben | Frontend- und UI-Aufgaben |
| `consultant` | v4-pro | Lesen/Schreiben | Lösungsdiskussion, Best-Practice-Beratung |
| `explore` | v4-flash | **Nur Lesen** | Codebase-Suche, parallele Exploration |
| `librarian` | v4-flash | **Nur Lesen** | Dokumentationssuche, Websuche |
| `light-orchestrator` | v4-flash | Lesen/Schreiben | Leichte Aufgaben, Einzeldatei-Bearbeitungen |

> `deep-worker` und `light-orchestrator` folgen dem Prinzip „Keine Recherche, keine Delegation" — ausführen statt erkunden, Kontext wird vom Orchestrator bereitgestellt.
>
> Read-only-Agenten (`oracle`/`reviewer`/`explore`/`librarian`) sind wirklich read-only: `edit: deny` + Bash-Whitelist (standardmäßig alles deny, nur Read-only-Unterbefehle wie `git status/diff/log/show/blame/grep`, `rg` erlaubt; `oracle`/`reviewer` dürfen zusätzlich `gh pr view/diff`, `gh issue view`, `gh api` nutzen, um `/review-pr`-Antworten zu unterstützen).

## Kurzbefehle

### Agent-Routing-Befehle

| Befehl | Agent | Verwendung |
| --- | --- | --- |
| `/deep` | `deep-worker` | Heavy-Implementierung, Multi-Datei-Änderungen |
| `/quick` | `light-orchestrator` | Leichte Aufgaben, Einzeldatei-Bearbeitungen |
| `/ui` | `ui-builder` | Frontend/UI-Arbeiten |
| `/review` | `reviewer` (code-review) | Zweiachsiges paralleles Review (Konvention + Spezifikation) + Schweregrad-Kalibrierung |
| `/review-pr` | `reviewer` (code-review + gh-cli) | PR reviewen und auf GitHub kommentieren |
| `/plan` | `planner` | Plan erstellen, technische Lösung entwerfen |
| `/search` | `librarian` | Externe Suche, Dokumentation nachschlagen |
| `/oracle` | `oracle` | Tiefenanalyse, Problemherkunft ermitteln |
| `/consult` | `consultant` | Beratung, Vergleich, Empfehlungen |

### Operationsbefehle

| Befehl | Agent | Verwendung |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Conventional-Commits-Nachricht generieren (Inline-Format) |
| `/release` | `deep-worker` (git-release) | Tag-Release vorbereiten |
| `/reflect` | `oracle` (reflect) | Reibungspunkte erkennen → Konfigurationsoptimierung vorschlagen |
| `/handoff` | `light-orchestrator` (handoff) | Sitzung in Übergabedokument komprimieren |

### Inline-Befehle

| Befehl | Agent | Verwendung |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Repository-Strukturdiagramm generieren |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | Oracle analysiert → Light-Orchestrator wendet Vereinfachung an |
| `/rmslop` | `deep-worker` (remove-deadcode) | Toten Code und AI-Slop bereinigen |

### Spezifikationsbefehle

| Befehl | Agent | Verwendung |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Code explorieren → Änderungsvorschlag erstellen |
| `/spec-apply` | `deep-worker` (spec-workflow) | Schrittweise nach tasks.md implementieren → automatisch archivieren |

## Skills

OpenCode stellt Skills über das native `skill`-Tool bei Bedarf bereit — Agenten laden sie nur, wenn sie benötigt werden, ohne Kontext zu belegen.

| Skill | Funktion |
| --- | --- |
| `code-review` | Token-sparender multidimensionaler Code-Review: gestufter Bericht nach Dimension + Schweregrad, Übereinstimmungspunkte mit höchster Konfidenz markiert, ändert niemals selbstständig Code |
| `codemap` | Annotiertes Repository-Strukturdiagramm generieren, schnelle Orientierung, spart Explorations-Token |
| `gh-cli` | GitHub-CLI-v2.97+-Referenz: Paginierung, Repository-Lokalisierung, Discussions/Projects/Rulesets/Skills, Rate Limit, gh-aw Agentic CI, gh api-Fallback |
| `git-master` | Fortgeschrittene Git-Operationen: Rebase, Squash, Fixup, Bisect, Reflog, Code-Archäologie, Worktree |
| `git-release` | Tag-Release: Release Notes, SemVer-Ableitung, gh release-Befehl |
| `resolving-merge-conflicts` | Merge-Konflikte pro Hunk lösen: ursprüngliche Absicht nachvollziehen, niemals neues Verhalten erfinden, niemals --abort |
| `handoff` | Sitzung in Übergabedokument komprimieren (Pfadreferenzen, kein Kopieren von Inhalten) |
| `opencode-config` | OpenCode-Konfiguration dieses Repositories schreiben und pflegen (agents/skills/commands/permissions) |
| `reflect` | Kontinuierliche Verbesserung: Reibungspunkte erkennen → minimale wartbare Korrekturen vorschlagen |
| `remove-deadcode` | Toten Code sicher finden und löschen, vor dem Löschen über Toolchain/LSP verifiziert |
| `security-review` | Sicherheitsaudit vor dem Merge (Injection/XSS/SSRF/Secrets/Deserialisierung/Path-Traversal), berichtet nur, ändert nicht |
| `shared-language` | Domänen-Glossar aufbauen (CONTEXT.md), spart erheblich Token |
| `simplify` | Verhaltenserhaltende Code-Vereinfachung (Oracle analysiert → angewendet) |
| `spec-workflow` | Leichtgewichtige spezifikationsgetriebene Änderungen: proposal → specs → design → tasks → archive |
| `verification-planning` | Engsten Verifikationspfad vor der Implementierung planen |
| `verify-with-docs` | API-Dokumentation vor dem Codieren prüfen, Retrieval-First, Halluzinationen vermeiden |
| `grilling` | Anforderungsabgleich-Interview: eine Frage nach der anderen, Multiple-Choice bevorzugt, erst nach Mehrdeutigkeits-Konvergenz handeln |
| `tech-debt-audit` | Technische-Schulden-Audit in 9 Dimensionen (toter Code/Duplikate/Namensdrift/Komplexität/Abhängigkeiten/Fehlerbehandlung/Tests/Dokumentation/Sicherheit), Read-only-Bericht, ändert keinen Code |

## Designentscheidungen & Iterationsverlauf

Der Kernansatz übernimmt die Vorteile von [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (Intent Gate, Read-Only-Isolation, Anti-Patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (Scheduler-First, Fallback-Kette, Rejection-Contract, Prompt-Cache-Sicherheit, impact×confidence÷cost), [anomalyco/opencode](https://github.com/anomalyco/opencode) (Konfigurationsschema, Skill-System), [cli/cli](https://github.com/cli/cli) (gh-v2.97-Befehlssatz, Rate Limit, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (Delta-Specs, OPSX-Aktionsfluss update/verify/vier Fragen), [mattpocock/skills](https://github.com/mattpocock/skills) (Konfliktlösungsdisziplin, Übergabedokumente), [pi](https://github.com/earendil-works/pi) (erst antworten, dann ändern; prägnante Antworten; unabhängige Sitzungssammlung) und [deepreview](https://github.com/mechanai/deepreview) (Novelty-Klassifikations-Konvergenz, Effektivgrößen-Routing, Points of Agreement) — rein konfigurationsbasiert, null zusätzliche Abhängigkeiten.

> **Inspiriert, nicht kopiert**: Überladene Pipelines werden auf ihre schlanken Designprinzipien reduziert; redundante Funktionen werden durch bestehende Agents/Skills abgedeckt, nichts Neues hinzugefügt. Dem Prinzip „Vereinfachen vor Hinzufügen" folgend, zielt jede Iteration auf eine Netto-Token-Reduktion.
>
> **Mechanismusquellen dieser Runde (v27)**: OPSX-Aktionsfluss (update/verify/vier Fragen) in spec-workflow verinnerlicht; unabhängige Sitzungssammlung für Kontext, Prompt-Cache-Sicherheit (stabiler statischer Präfix, volatile Inhalte ans Ende der Payload) entlehnt aus pi und oh-my-opencode-slim; impact×confidence÷cost als Iterations-Gate in deep-worker; Points of Agreement (Übereinstimmungspunkte mit höchster Konfidenz markiert) entlehnt aus deepreview; gh-cli um Rate Limit und gh-aw aus cli/cli v2.97 ergänzt.
>
> **Nach Bewertung nicht übernommen**: progressive disclosure und wait-what aus mattpocock/skills (Lazy-Loading bestehender Skills deckt deren Wert bereits ab); superpowers bietet keine Konfigurationsknöpfe, wird weiterhin als Plugin-String injiziert.

### Iterations-Meilensteine

27 Iterationen seit v1, kontinuierlich an Upstream-Best Practices ausgerichtet:

- **v1-v7 (Grundlage)**: Dual-Model-Bindung, Agent-Rollensystem, Intent-Gate-Routing, AGENTS.md globale Regeln, Skills-Verzeichnis, Berechtigungsbasis
- **v8-v15 (Review + Specs + Verträge)**: code-review Zwei-Achsen-Kalibrierung, spec-workflow, gh-cli-Ausrichtung, Ablehnungsvertrag, Hintergrundprüfungen
- **v16-v22 (Kontinuierliche Verschlankung)**: Befehle 29→18 (-38%), AGENTS.md 290→211 (-27%), No-Op-Satztrimmen, Schema-Validierung entfernt tote Schlüssel
- **v23-v25 (Ausrichtung + Sicherheit)**: 6 Upstream-Repos integriert, gh-cli v2.97 Escape-Injection-Warnung, procedure-driven Prompt-Verfeinerung, DCP-Fenster-Tuning
- **v26 (Verschlankung dieser Runde)**: prune:true und tool_output 800/20480 verschärft, DCP auf 60%/30%-Prozentschwellen umgestellt, grilling ersetzt writing-great-skills, opencode-config 131→64 verschlankt, code-review Stufung + Validator, gh-cli um gh status ergänzt, AGENTS.md um User Override erweitert, Delegationskosten-Disziplin im Orchestrator, 7 Agentendateien netto −22 Zeilen
- **v27 (Löschen/Migration/Neu)**: Totkonfiguration batch_tool gelöscht, unwirksames `write: deny` bei Read-only-Agenten und 3 redundante Bash-Zeilen entfernt; Context-Management-Abschnitt in orchestrator-eigenen Unterabschnitt verlagert; Bash-Whitelist für Read-only-Agenten, `read` um `.env` ergänzt; neuer Skill tech-debt-audit; 15 Skill-Beschreibungen um 30–40% verschlankt; gh-cli um 5 Punkte ergänzt (Rate Limit, gh-skill-Host, gh-aw u. a.), code-review um Points of Agreement erweitert, spec-workflow um zwei Update-Fragen ergänzt, Orchestrator um unabhängige Sitzungssammlung + Prompt-Cache-Sicherheit erweitert, deep-worker um impact×confidence÷cost ergänzt

## Repository-Struktur

```text
├── opencode/                     # OpenCode-Konfigurationsverzeichnis (einzeln deploybar)
│   ├── .ai/
│   │   └── calibration.yml       # Schweregrad-Kalibrierung für code-review
│   ├── agents/                   # 10 spezialisierte Agenten
│   │   ├── orchestrator.md       # Haupteinstieg: Intent Gate + modellbewusstes Routing
│   │   ├── planner.md            # Pro: Architektur & Planung
│   │   ├── deep-worker.md        # Pro: Heavy-Implementierung
│   │   ├── oracle.md             # Pro: tiefgehende Code-Analyse (nur Lesen)
│   │   ├── reviewer.md           # Pro: zweiachsiges Code-Review (nur Lesen)
│   │   ├── consultant.md         # Pro: Lösungsdiskussion & Beratung
│   │   ├── ui-builder.md         # Pro: Frontend & UI
│   │   ├── explore.md            # Flash: Codebase-Suche (nur Lesen)
│   │   ├── librarian.md          # Flash: externe Recherche (nur Lesen)
│   │   └── light-orchestrator.md # Flash: einfache Bearbeitungen
│   ├── skills/                   # 18 On-Demand-Skills
│   │   ├── code-review/          # Zweiachsiges paralleles Review + Schweregrad-Kalibrierung
│   │   ├── codemap/              # Repository-Strukturdiagramm generieren
│   │   ├── gh-cli/               # GitHub-CLI-v2.97+-Referenz + Sicherheitswarnung
│   │   ├── git-master/           # Fortgeschrittene Git-Operationen
│   │   ├── git-release/          # Tag-Release
│   │   ├── handoff/              # Sitzung in Übergabedokument komprimieren
│   │   ├── opencode-config/      # Meta-Skill: Konfiguration dieses Repositories
│   │   ├── reflect/              # Kontinuierliche Verbesserung
│   │   ├── remove-deadcode/      # Toten Code erkennen & löschen
│   │   ├── resolving-merge-conflicts/ # Konfliktlösungsdisziplin pro Hunk
│   │   ├── security-review/      # Sicherheitsaudit-Checkliste
│   │   ├── shared-language/      # Domänen-Glossar (spart Token)
│   │   ├── simplify/             # Verhaltenserhaltende Code-Vereinfachung
│   │   ├── spec-workflow/        # Spezifikationsgetriebene Entwicklung
│   │   ├── tech-debt-audit/      # Technische-Schulden-Audit (9 Dimensionen, Read-only-Bericht)
│   │   ├── verification-planning/ # Verifikationspfad-Planung vor der Implementierung
│   │   ├── verify-with-docs/     # Retrieval-First API-Verifikation
│   │   └── grilling/             # Anforderungsabgleich-Interview
│   ├── opencode.jsonc            # Hauptkonfiguration (18 Befehle)
│   ├── AGENTS.md                 # Globale Regeln
│   └── dcp.jsonc                 # DCP-Kontextkomprimierung (DeepSeek 128K, 60%/30%-Prozentschwellen)
├── README.md
├── LICENSE
└── README.*.md                   # READMEs in anderen Sprachen
```

## Benutzerhandbuch

### Modus 1: Automatisches Orchestrator-Routing (Standard)

Beschreiben Sie Ihre Anforderung in natürlicher Sprache — der Orchestrator analysiert automatisch die Absicht und wählt den geeignetsten Agenten und das passende Modell für die Ausführung.

```text
„Untersuche den Fehler dieser Login-Schnittstelle"       → Oracle analysiert Ursache → Diagnosebericht
„Optimiere diese Schleife, die Performance ist schlecht"  → Oracle analysiert → Deep-Worker implementiert Optimierung
„Review mal diesen PR"                                     → Reviewer führt mehrdimensionales Review durch → gestuften Bericht
„Ich möchte eine Exportfunktion zum Benutzermodul hinzufügen" → Planner entwirft Lösung → Deep-Worker implementiert
„Wie benutzt man die use() API von React 19?"             → Librarian schlägt Dokumentation nach → Signatur & Beispiele
```

### Modus 2: Direktzugriff über Befehlsaliase

| Szenario | Befehl |
| --- | --- |
| Komplexe Implementierung / Multi-Datei-Änderungen | `/deep` |
| Leichte Änderungen / Einzeldatei-Bearbeitungen | `/quick` |
| Technischen Plan / Architektur entwerfen | `/plan` |
| Bug untersuchen / Tiefenanalyse | `/oracle` |
| Code-Review | `/review` |
| Externe Suche / API nachschlagen | `/search` |
| Frontend / UI-Arbeiten | `/ui` |
| Lösungsdiskussion / Vergleich & Abwägung | `/consult` |
| Strukturiertes Debugging | `/oracle` |

### Typische Workflows

**Neue Funktion entwickeln (spezifikationsgetrieben):**
```text
/spec-propose  → /spec-apply  → /review
```

**Bug untersuchen:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Code-Review:**
```text
/review-pr   ← PR reviewen + automatisch auf GitHub posten
/review      ← Zweiachsiges paralleles Review
```

## Designphilosophie

- **Rein konfigurationsbasiert, null zusätzliche Abhängigkeiten** — alle Fähigkeiten werden durch `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md` realisiert
- **Maximale Nutzung der beiden DeepSeek V4-Modelle** — Pro für Reasoning & Entscheidungen, Flash für Abfragen & leichte Ausführung
- **Token-Effizienz zuerst** — Pfadreferenzen statt Dateiinhalte einfügen, Skills on-demand laden, gestufte Komprimierungsverwaltung
- **Plugins ergänzen, dominieren nicht** — Superpowers liefern Prozessdisziplin, DCP intelligente Komprimierung statt einfachem Abschneiden (adaptive Prozentschwellen, native Compaction als Fallback)
- **Ausführung von Exploration getrennt** — Deep-Worker/Light-Orchestrator dürfen nicht recherchieren/delegieren, Explore/Librarian dürfen nicht modifizieren
- **Kontinuierliche Verbesserung** — Reflect-Mechanismus zur Erkennung von Reibungspunkten, Code-Review-Zweiachsen-Kalibrierung zur Qualitätssicherung
