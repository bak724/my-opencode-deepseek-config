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
- Skills: **16** `SKILL.md`-Skills im Verzeichnis `skills/`, bei Bedarf über das native `skill`-Tool geladen
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

### Methode 1: In globales Konfigurationsverzeichnis klonen (empfohlen)

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
git clone https://github.com/znlgis/my-opencode-deepseek-config.git "$env:USERPROFILE\.config\opencode"
```

**Linux / macOS:**

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git ~/.config/opencode
```

> **Kompatibilitätshinweis**: `~/.config/opencode` ist der standardmäßige globale Konfigurationspfad von OpenCode. Die Dateien `agents/`, `skills/`, `AGENTS.md` usw. in diesem Repository folgen der OpenCode-Konvention und werden nach dem Klonen automatisch erkannt.

### Methode 2: An beliebigen Ort klonen + Umgebungsvariable

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\opencode-config"
opencode
```

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
| `gh-cli` | Vollständige GitHub CLI v2.96+-Referenz (Issues 2.0, Copilot, Agent-Task, gh skill) |
| `git-master` | Fortgeschrittene Git-Operationen: Rebase, Squash, Bisect, Reflog, Worktree |
| `git-release` | Tag-Release: SemVer-Ableitung, Release Notes, gh release-Befehl |
| `handoff` | Sitzung in Übergabedokument komprimieren (Pfadreferenzen, kein Kopieren von Inhalten) |
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

Der Kernansatz orientiert sich an [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (Intent Gate, Read-Only-Isolation, Anti-Patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (Scheduler-First, Fallback-Kette, Rejection Contract), [anomalyco/opencode](https://github.com/anomalyco/opencode) (Konfigurationsschema, Skill-System), [cli/cli](https://github.com/cli/cli) (vollständiger gh-Befehlssatz), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (Delta-Specs, Änderungsvorschlag-Updates), [mattpocock/skills](https://github.com/mattpocock/skills) (Übergabedokumente, strukturiertes Debugging), [pi](https://github.com/earendil-works/pi) (erst antworten, dann ändern; prägnante Antworten) und [deepreview](https://github.com/mechanai/deepreview) (Entropie-Scan, Konvergenz-Prüfung) — rein konfigurationsbasiert, null zusätzliche Abhängigkeiten.

> **Inspiriert, nicht kopiert**: Überladene Pipelines werden auf ihre schlanken Designprinzipien reduziert; redundante Funktionen werden durch bestehende Agents/Skills abgedeckt, nichts wird neu hinzugefügt. Dem Prinzip „Vereinfachen vor Hinzufügen" folgend, zielt jede Iteration auf eine Netto-Token-Reduktion.

### Iterations-Meilensteine

| Phase | Wesentliche Änderungen |
| --- | --- |
| **v1–v7 (Grundlagen)** | Dual-Modell-Bindung, Agent-Rollensystem, Intent Gate/Klassifikations-Routing, AGENTS.md globale Regeln, Skills-Verzeichnis & Befehlsaliase, Berechtigungsbasis |
| **v8–v12 (Review + Specs)** | Verbessertes Code-Review (Stufen/Selbstprüfung/Ablehnungskriterien), Etablierung des spec-workflow (explore→propose→apply→archive), neue Skills deepwork/reflect/verification-planning, gh-cli auf v2.96+ abgestimmt |
| **v13–v15 (Verträge + Straffung)** | AGENTS.md ergänzt um Evidence Discipline / Task Rejection Contract / Stop Condition; vollständige Deduplizierung von Agent-Prompts und globalen Regeln; Fehlerprüfung für Hintergrund-Subagenten |
| **v16–v18 (Effiziente Ausführung)** | Mythische Namen entfernt, Routing-Tabellen konsolidiert, gh-cli auf Issues 2.0 erweitert, spec-workflow um Verify + Entscheidungsframework ergänzt |
| **v19 (Upstream-Abgleich)** | 6 Upstream-Repositories überprüft; `/review-pr` Zeilenkommentar-Bug behoben; Code-Review-Routing von roher Zeilenzahl auf effektive Logikmenge umgestellt |
| **v20 (Refactoring)** | `agent/`→`agents/` an OpenCode-Empfehlung angepasst; AGENTS.md um 22 % gestrafft (290→227 Zeilen); neue Skills `diagnose` (6-Phasen-Debugging) + `handoff` (Sitzungsübergabe); spec-workflow um `/update` ergänzt; Code-Review um Entropie-Scan + Konvergenz-Prüfung erweitert; Agent-Prompts um 20 % dedupliziert |
| **v21 (Umfassende Verschlankung)** | Skills 18→17 (deepwork/conventional-commits/diagnose entfernt, writing-great-skills/shared-language hinzugefügt); Befehle 29→18 (−38 %); AGENTS.md 227→212 Zeilen (−7 %); Skills satzweise auf No-Op getrimmt. Code-Review zweiachsig parallel + Kalibrierungsdatei-Mechanismus. Praxiserkenntnisse aus 6 Repositories (pi/deepreview/mattpocock u. a.) integriert. |
| **v22 (Schema-Validierung & Verschlankung)** | OpenCode- und DCP-Schema abgeglichen: ungültigen Dead-Key `agent.fallback` entfernt; alle `dcp.jsonc`-Schlüssel als gültig bestätigt (v3.1.14), null Änderungen, kein blindes Hinzufügen; AGENTS.md: „Token-Effizienz" in „Kontextmanagement" integriert und vollständig dedupliziert, hängende `Self-Verification`-Referenz behoben (212→197 Zeilen); Orchestrator: drei Routing-Tabellen konsolidiert (128→79 Zeilen, −38 %, Intent Gate/Agent Directory/Fallback vollständig erhalten); 14 Skills: vom Parser ignorierte `license/compatibility/metadata`-Frontmatter entfernt (−70 Zeilen); `tool_output`-Limit auf proaktive Token-Einsparung gesenkt (1500 Zeilen/40 KB). |
| **v23 (Doppelte Disziplin + Konsolidierung)** | Basierend auf 6 Upstream-Repositories finale Integration: `gh-skill` entfernt (Funktionalität in `gh-cli` Agent Skills-Abschnitt integriert, −122 Zeilen); tote Referenz in `verification-planning` behoben; AGENTS.md um zwei von Pi inspirierte Disziplinen ergänzt (erst antworten, dann ändern + Haltung klarstellen; globale Knappheit) + Slim-Delegationsvertrag + Job Board + Deepreview-Datei-IPC (+15 Zeilen); Orchestrator-Tabelle in Pro/Flash-Untertabellen umstrukturiert, 79→86; Reviewer um Default-Ablehnungshaltung des Verifizierers ergänzt (+6 Zeilen); gh-cli Agent Skills-Abschnitt verstärkt (+10 Zeilen). Netto ~−90 Zeilen, Skills 17→16. |

## Repository-Struktur

```text
├── .ai/
│   └── calibration.yml           # Code-Review Schweregrad-Kalibrierung
├── agents/                       # 10 spezialisierte Agenten
│   ├── orchestrator.md           # Haupteinstieg: Intent Gate + modellbewusstes Routing
│   ├── planner.md                # Pro: Architektur & Planung
│   ├── deep-worker.md            # Pro: Heavy-Implementierung
│   ├── oracle.md                 # Pro: Tiefgehende Code-Analyse (nur Lesen)
│   ├── reviewer.md               # Pro: Zweiachsiges Code-Review (nur Lesen)
│   ├── consultant.md             # Pro: Lösungsdiskussion & Beratung
│   ├── ui-builder.md             # Pro: Frontend & UI
│   ├── explore.md                # Flash: Codebase-Suche (nur Lesen)
│   ├── librarian.md              # Flash: Externe Recherche (nur Lesen)
│   └── light-orchestrator.md     # Flash: Einfache Bearbeitungen
├── skills/                       # 16 On-Demand-Skills
│   ├── code-review/              # Zweiachsiges paralleles Review + Schweregrad-Kalibrierung
│   ├── codemap/                  # Repository-Strukturdiagramm generieren
│   ├── gh-cli/                   # GitHub CLI v2.96+-Referenz
│   ├── git-master/               # Fortgeschrittene Git-Operationen
│   ├── git-release/              # Tag-Release
│   ├── handoff/                  # Sitzung in Übergabedokument komprimieren
│   ├── opencode-config/          # Meta-Skill: Konfiguration dieses Repositories
│   ├── reflect/                  # Kontinuierliche Verbesserung
│   ├── remove-deadcode/          # Toten Code erkennen & löschen
│   ├── security-review/          # Sicherheitsaudit-Checkliste
│   ├── shared-language/          # Domänen-Glossar (spart Token)
│   ├── simplify/                 # Verhaltenserhaltende Code-Vereinfachung
│   ├── spec-workflow/            # Spezifikationsgetriebene Entwicklung
│   ├── verification-planning/    # Verifikationspfad-Planung vor Implementierung
│   ├── verify-with-docs/         # Retrieval-First API-Verifikation
│   └── writing-great-skills/     # Skill-Authoring-Richtlinien
├── opencode.jsonc                # Hauptkonfiguration (18 Befehle)
├── AGENTS.md                     # Globale Regeln (~212 Zeilen)
├── dcp.jsonc                     # DCP-Kontextkomprimierung (DeepSeek 128K)
├── LICENSE
└── README.md
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
