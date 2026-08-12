# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | **Deutsch** | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**Die optimale OpenCode × DeepSeek-Konfiguration** — eine Konfiguration, die die Fähigkeiten der beiden DeepSeek V4-Modelle (Pro + Flash) im OpenCode Multi-Agent-Framework voll ausschöpft. Kernphilosophie: **Token-Effizienz zuerst — beste Entwicklungsergebnisse bei minimalen Kontextkosten**.

## Konfigurationsübersicht

- Standard-Hauptagent: `orchestrator`
- Hauptmodell: `deepseek/deepseek-v4-pro`, Leichtgewichtmodell: `deepseek/deepseek-v4-flash`
- Agent-Tiefe: `subagent_depth: 3` (unterstützt 3 Ebenen Agent-Verschachtelung)
- Modellisolierung: doppelte Absicherung durch `enabled_providers: ["deepseek"]` + `disabled_providers`
- Sitzungsfreigabe: deaktiviert (`share: "disabled"`); Snapshots: aktiviert (`snapshot: true`)
- Berechtigungsbasis: standardmäßig erlauben, destruktive Bash-Befehle auf `ask`; sensible Dateien wie `.env` auf `deny`; externe Verzeichnisse auf `ask`
- Kontextkomprimierung: DCP proaktive Komprimierung (35K–75K Schwellenwert) + native OpenCode-Compaction als Fallback
- Globale Regeln: `AGENTS.md` (Kernprinzipien, Task-Rejection-Contract, Kontext- & Token-Effizienz, Selbstverifikation, Anti-Patterns usw.)
- Skills: **17** `SKILL.md`-Skills im Verzeichnis `skills/`, bei Bedarf über das native `skill`-Tool geladen
- Plugins: `superpowers` (14 prozessorientierte Skills), `@tarquinen/opencode-dcp` (intelligentes Context-Trimming)
- Experimentelle Funktion: `batch_tool` standardmäßig aktiviert

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

> **Kompatibilitätshinweis**: `~/.config/opencode` ist der standardmäßige globale Konfigurationspfad von OpenCode. Die Konfigurationsdateien befinden sich im Unterverzeichnis `opencode/` dieses Repositorys und folgen vollständig den OpenCode-Konventionen — durch Verweis via Umgebungsvariable oder symbolischen Link werden sie automatisch erkannt.

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
| `code-review` | Zweiachsiges paralleles Review (Konvention + Spezifikation) + Schweregrad-Kalibrierung |
| `codemap` | Annotiertes Repository-Strukturdiagramm generieren, spart Explorations-Token |
| \`gh-cli\` | Vollständige GitHub CLI v2.97+-Referenz (Issues 2.0, Copilot, Agent-Task, gh skill) + Sicherheitswarnung (Escape-Injection) |
| `git-master` | Fortgeschrittene Git-Operationen: Rebase, Squash, Bisect, Reflog, Worktree |
| \`git-release\` | Tag-Release: SemVer-Ableitung, Release Notes, gh release-Befehl |
| \`resolving-merge-conflicts\` | Merge-Konflikte pro Hunk lösen: ursprüngliche Absicht nachvollziehen, kein neues Verhalten erfinden, niemals --abort |
| \`handoff\` | Sitzung in Übergabedokument komprimieren (Pfadreferenzen, kein Kopieren von Inhalten) |
| `opencode-config` | OpenCode-Konfiguration schreiben und pflegen |
| `reflect` | Kontinuierliche Verbesserung: Reibungspunkte erkennen → minimale Korrekturen vorschlagen |
| `remove-deadcode` | Toten Code sicher finden und löschen, LSP-Verifikation vor dem Löschen |
| `security-review` | Sicherheitsaudit eines Diffs vor dem Merge |
| `shared-language` | Domänen-Glossar aufbauen, spart erheblich Kontext-Token |
| `simplify` | Verhaltenserhaltende Code-Vereinfachung (Oracle analysiert → Light-Orchestrator wendet an) |
| `spec-workflow` | Leichtgewichtiger spezifikationsgetriebener Workflow (propose → design → tasks → implement → archive) |
| `verification-planning` | Engsten Verifikationspfad vor der Implementierung planen |
| `verify-with-docs` | API-Dokumentation vor dem Codieren prüfen, Retrieval-First, Halluzinationen vermeiden |
| `writing-great-skills` | Skill-Authoring-Richtlinien: No-Op-Trimming, positive Formulierung, Abschlusskriterien |

## Designentscheidungen & Iterationsverlauf

Der Kernansatz orientiert sich an [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (Intent Gate, Read-Only-Isolation, Anti-Patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (Scheduler-First, Fallback-Kette, Rejection Contract), [anomalyco/opencode](https://github.com/anomalyco/opencode) (Konfigurationsschema, Skill-System), [cli/cli](https://github.com/cli/cli) (gh v2.97 full command set), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (Delta-Specs, Änderungsvorschlag-Updates), [mattpocock/skills](https://github.com/mattpocock/skills) (conflict resolution discipline, handoff docs), [pi](https://github.com/earendil-works/pi) (erst antworten, dann ändern; prägnante Antworten) und [deepreview](https://github.com/mechanai/deepreview) (novelty-based convergence, effective-size routing) — rein konfigurationsbasiert, null zusätzliche Abhängigkeiten.

> **Inspiriert, nicht kopiert**: Überladene Pipelines werden auf ihre schlanken Designprinzipien reduziert; redundante Funktionen werden durch bestehende Agents/Skills abgedeckt, nichts wird neu hinzugefügt. Dem Prinzip „Vereinfachen vor Hinzufügen" folgend, zielt jede Iteration auf eine Netto-Token-Reduktion.

### Iterations-Meilensteine

25 Iterationen seit v1, kontinuierlich an Upstream-Best Practices ausgerichtet:

- **v1-v7 (Grundlage)**: Dual-Model-Bindung, Agent-Rollensystem, Intent-Gate-Routing, AGENTS.md globale Regeln, Skills-Verzeichnis, Berechtigungsbasis
- **v8-v15 (Review + Specs + Verträge)**: code-review Zwei-Achsen-Kalibrierung, spec-workflow, gh-cli-Ausrichtung, Ablehnungsvertrag, Hintergrundprüfungen
- **v16-v22 (Kontinuierliche Verschlankung)**: Befehle 29→18 (-38%), AGENTS.md 290→211 (-27%), No-Op-Satztrimmen, Schema-Validierung
- **v23-v25 (Ausrichtung + Sicherheit)**: 6 Upstream-Repos integriert, gh-cli v2.97 Escape-Injection-Warnung, procedure-driven Prompt-Verfeinerung, DCP-Fenster-Tuning

## Repository-Struktur

```text
├── opencode/                     # OpenCode-Konfigurationsdateien
│   ├── .ai/
│   │   └── calibration.yml           # Code-Review Schweregrad-Kalibrierung
│   ├── agents/                       # 10 spezialisierte Agenten
│   │   ├── orchestrator.md           # Haupteinstieg: Intent Gate + modellbewusstes Routing
│   │   ├── planner.md                # Pro: Architektur & Planung
│   │   ├── deep-worker.md            # Pro: Heavy-Implementierung
│   │   ├── oracle.md                 # Pro: Tiefgehende Code-Analyse (nur Lesen)
│   │   ├── reviewer.md               # Pro: Zweiachsiges Code-Review (nur Lesen)
│   │   ├── consultant.md             # Pro: Lösungsdiskussion & Beratung
│   │   ├── ui-builder.md             # Pro: Frontend & UI
│   │   ├── explore.md                # Flash: Codebase-Suche (nur Lesen)
│   │   ├── librarian.md              # Flash: Externe Recherche (nur Lesen)
│   │   └── light-orchestrator.md     # Flash: Einfache Bearbeitungen
│   ├── skills/                       # 17 On-Demand-Skills
│   │   ├── code-review/              # Zweiachsiges paralleles Review + Schweregrad-Kalibrierung
│   │   ├── codemap/                  # Repository-Strukturdiagramm generieren
│   │   ├── gh-cli/                   # GitHub CLI v2.97+-Referenz + Sicherheitswarnung
│   │   ├── git-master/               # Fortgeschrittene Git-Operationen
│   │   ├── git-release/              # Tag-Release
│   │   ├── resolving-merge-conflicts/ # Hunk-basierte Merge-Konfliktlösung
│   │   ├── handoff/                  # Sitzung in Übergabedokument komprimieren
│   │   ├── opencode-config/          # Meta-Skill: Konfiguration dieses Repositories
│   │   ├── reflect/                  # Kontinuierliche Verbesserung
│   │   ├── remove-deadcode/          # Toten Code erkennen & löschen
│   │   ├── security-review/          # Sicherheitsaudit-Checkliste
│   │   ├── shared-language/          # Domänen-Glossar (spart Token)
│   │   ├── simplify/                 # Verhaltenserhaltende Code-Vereinfachung
│   │   ├── spec-workflow/            # Spezifikationsgetriebene Entwicklung
│   │   ├── verification-planning/    # Verifikationspfad-Planung vor Implementierung
│   │   ├── verify-with-docs/         # Retrieval-First API-Verifikation
│   │   └── writing-great-skills/     # Skill-Authoring-Richtlinien
│   ├── opencode.jsonc                # Hauptkonfiguration (18 Befehle)
│   ├── AGENTS.md                     # Globale Regeln (~212 Zeilen)
│   └── dcp.jsonc                     # DCP-Kontextkomprimierung (DeepSeek 128K)
├── README.md
├── LICENSE
└── README.*.md
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
- **Plugins ergänzen, dominieren nicht** — Superpowers liefern Prozessdisziplin, DCP intelligente Komprimierung statt einfachem Abschneiden
- **Ausführung von Exploration getrennt** — Deep-Worker/Light-Orchestrator dürfen nicht recherchieren/delegieren, Explore/Librarian dürfen nicht modifizieren
- **Kontinuierliche Verbesserung** — Reflect-Mechanismus zur Erkennung von Reibungspunkten, Code-Review-Zweiachsen-Kalibrierung zur Qualitätssicherung
