# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | **한국어**

**OpenCode × DeepSeek 최적 설정** —— OpenCode 멀티 에이전트 프레임워크에서 DeepSeek V4 듀얼 모델(Pro + Flash)의 성능을 극대화하는 구성 방안입니다. 핵심 이념: **토큰 효율성 우선, 최소한의 컨텍스트 비용으로 최상의 개발 결과를 달성합니다**.

## 현재 구성 개요

- 기본 주 에이전트: `orchestrator`
- 주 모델: `deepseek/deepseek-v4-pro`, 경량 모델: `deepseek/deepseek-v4-flash`
- 에이전트 계층: `subagent_depth: 3`(3단계 에이전트 중첩 지원)
- 모델 격리: `enabled_providers: ["deepseek"]` + `disabled_providers` 이중 잠금
- 세션 공유: 비활성화(`share: "disabled"`); 스냅샷: 활성화(`snapshot: true`)
- 권한 기준: 기본 허용, 파괴적 bash 명령은 `ask`로 설정; `.env`류 민감 파일은 `deny`; 외부 디렉터리는 `ask`
- 컨텍스트 압축: DCP 능동 압축(35K-75K 임계값) + OpenCode 네이티브 compaction 백업
- 전역 규칙: `AGENTS.md`(핵심 원칙, 작업 거부 계약, 컨텍스트와 토큰 효율성, 자체 검증, 안티 패턴 등)
- 스킬: `skills/` 디렉터리 아래 **17개** `SKILL.md` 스킬, 네이티브 `skill` 도구를 통해 필요 시 로드
- 플러그인: `superpowers`(14개 프로세스형 스킬), `@tarquinen/opencode-dcp`(지능형 컨텍스트 가지치기)
- 실험 기능: `batch_tool` 기본 활성화

## DeepSeek 모델 구성

### 사전 조건

- OpenCode ≥ v1.14.24(DeepSeek provider 내장)
- DeepSeek API Key: [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)에서 신청

### 방식 1: TUI 대화형 구성(권장)

```bash
opencode
# TUI에서 입력: /connect → DeepSeek 선택 → API Key 붙여넣기
# 그다음: /models → deepseek-v4-pro 선택
```

API Key는 자동으로 `~/.local/share/opencode/auth.json`에 영구 저장됩니다.

### 방식 2: 환경 변수

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

영구 설정: `DEEPSEEK_API_KEY`를 시스템 환경 변수에 추가합니다.

### Provider 구성 참조

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

Pro 모델에 thinking/reasoning을 활성화하려면 `provider`에 다음을 추가합니다:

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

> **모델 ID 명명 규칙**: `provider_id/model_id`, 즉 `deepseek/deepseek-v4-pro`와 `deepseek/deepseek-v4-flash`입니다.

## 설치 및 배포

### 방식 1: 클론 + 환경 변수(권장, 크로스 플랫폼)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

그런 다음 `OPENCODE_CONFIG_DIR`을 저장소 내 `opencode/` 하위 디렉터리로 지정하면 사용할 수 있습니다.

**Windows(PowerShell)** —— 영구 설정:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows(PowerShell)** —— 임시 설정(현재 세션만):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— `~/.bashrc` 또는 `~/.zshrc`에 추가:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### 방식 2: 전역 구성 디렉터리로 심볼릭 링크

**Windows(PowerShell, 관리자 권한 필요):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **호환성 설명**: `~/.config/opencode`는 OpenCode의 표준 전역 구성 경로입니다. 구성 파일(`agents/`, `skills/`, `AGENTS.md` 등)은 본 저장소의 `opencode/` 하위 디렉터리에 있으며 OpenCode의 규칙적 레이아웃을 따릅니다. 환경 변수나 심볼릭 링크로 지정하면 자동으로 인식됩니다.

### 설치 확인

OpenCode를 시작하여 확인합니다:
1. `/models` → 현재 모델이 `deepseek/deepseek-v4-pro`임
2. 에이전트 목록에 `orchestrator`, `planner`, `deep-worker` 등 10개 에이전트가 표시됨
3. 임의의 요청을 입력하면 Orchestrator가 자동으로 의도를 분석하고 라우팅함

## 모델 분업

본 저장소는 DeepSeek V4 듀얼 모델 내에서만 분업하며, 다른 모델을 도입하지 않습니다:

| 모델 | 용도 |
| --- | --- |
| `deepseek/deepseek-v4-pro` | 계획, 아키텍처, 근본 원인 분석, 코드 리뷰, 중량 구현, 주 제어 스케줄링 |
| `deepseek/deepseek-v4-flash` | 빠른 탐색, 외부 검색, 경량 작업, 간단한 편집 |

### 라우팅 전략

- **Flash 우선**: 검색, 찾기, 간단한 편집 등 명확히 정의된 작업은 우선적으로 flash 에이전트 사용
- **Pro 추론 집중**: 계획, 분석, 리뷰, 복잡한 구현——Pro만 사용
- **자동 승격**: flash 에이전트가 처리할 수 없는 경우 자동으로 pro로 승격(전체 컨텍스트 포함)

## 에이전트 구조

### 주 에이전트(Primary Agent)

| 에이전트 | 모델 | 역할 |
| --- | --- | --- |
| `orchestrator` | v4-pro | 기본 진입점: 의도 게이트(Intent Gate) + 모델 인식 라우팅 + 폴백 체인 |

### 서브 에이전트(Subagents)

| 에이전트 | 모델 | 권한 | 역할 |
| --- | --- | --- | --- |
| `planner` | v4-pro | 읽기/쓰기 | 계획, 아키텍처, 작업 분할 |
| `deep-worker` | v4-pro | 읽기/쓰기 | 중량 구현, 다중 파일 변경, 복잡한 디버깅 |
| `oracle` | v4-pro | **읽기 전용** | 근본 원인 분석, 심층 코드 이해 |
| `reviewer` | v4-pro | **읽기 전용** | 이중 축 코드 리뷰(규범 + 스펙) + 심각도 보정 |
| `ui-builder` | v4-pro | 읽기/쓰기 | 프론트엔드 및 UI 관련 작업 |
| `consultant` | v4-pro | 읽기/쓰기 | 방안 논의, 모범 사례 조언 |
| `explore` | v4-flash | **읽기 전용** | 코드베이스 검색, 병렬 탐색 |
| `librarian` | v4-flash | **읽기 전용** | 문서 검색, 웹 검색 |
| `light-orchestrator` | v4-flash | 읽기/쓰기 | 경량 작업, 단일 파일 편집 |

> `deep-worker`와 `light-orchestrator`는 "연구 금지, 위임 금지" 원칙을 따릅니다——실행만 하고 탐색하지 않으며, 컨텍스트는 orchestrator가 제공합니다.

## 단축 명령

### 에이전트 라우팅 명령

| 명령 | 에이전트 | 용도 |
| --- | --- | --- |
| `/deep` | `deep-worker` | 중량 구현, 다중 파일 변경 |
| `/quick` | `light-orchestrator` | 경량 작업, 단일 파일 편집 |
| `/ui` | `ui-builder` | 프론트엔드/UI 작업 |
| `/review` | `reviewer`(code-review) | 이중 축 병렬 리뷰(규범+스펙) + 심각도 보정 |
| `/review-pr` | `reviewer`(code-review + gh-cli) | PR 리뷰 후 GitHub에 댓글 게시 |
| `/plan` | `planner` | 계획 수립, 기술 방안 |
| `/search` | `librarian` | 외부 검색, 문서 조회 |
| `/oracle` | `oracle` | 심층 분석, 문제 원인 추적 |
| `/consult` | `consultant` | 상담, 비교, 조언 |

### 작업 명령

| 명령 | 에이전트 | 용도 |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Conventional Commits 커밋 메시지 생성(인라인 형식) |
| `/release` | `deep-worker`(git-release) | 태그 릴리스 준비 |
| `/reflect` | `oracle`(reflect) | 마찰 발견 → 구성 최적화 제안 |
| `/handoff` | `light-orchestrator`(handoff) | 세션을 인수인계 문서로 압축 |

### 인라인 명령

| 명령 | 에이전트 | 용도 |
| --- | --- | --- |
| `/codemap` | `explore`(codemap) | 저장소 구조도 생성 |
| `/simplify` | `oracle`(simplify) → `light-orchestrator` | oracle 분석 → light-orchestrator 단순화 적용 |
| `/rmslop` | `deep-worker`(remove-deadcode) | 데드 코드 및 AI slop 정리 |

### 스펙 명령

| 명령 | 에이전트 | 용도 |
| --- | --- | --- |
| `/spec-propose` | `planner`(spec-workflow) | 코드 탐색 → 변경 제안서 초안 |
| `/spec-apply` | `deep-worker`(spec-workflow) | tasks.md에 따라 순차 구현 → 자동 아카이브 |

## 스킬(Skills)

OpenCode는 네이티브 `skill` 도구를 통해 필요 시 스킬을 노출합니다——에이전트는 필요할 때만 로드하며 컨텍스트를 점유하지 않습니다.

| 스킬 | 역할 |
| --- | --- |
| `code-review` | 이중 축 병렬 리뷰(규범 + 스펙) + 심각도 보정 |
| `codemap` | 주석이 포함된 저장소 구조도 생성, 탐색 토큰 절약 |
| `gh-cli` | GitHub CLI v2.97+ 전체 참조(Issues 2.0, copilot, agent-task, gh skill) + 보안 경고 (이스케이프 인젝션) |
| `git-master` | 고급 Git 작업: rebase, squash, bisect, reflog, worktree |
| `git-release` | 태그 릴리스: SemVer 추론, 릴리스 노트, gh release 명령 |
| `resolving-merge-conflicts` | hunk별 병합 충돌 해결: 원래 의도 추적, 새로운 동작 발명 금지, 절대 --abort 사용 안 함 |
| `handoff` | 세션을 인수인계 문서로 압축(경로 참조, 내용 복사 안 함) |
| `opencode-config` | OpenCode 구성 작성 및 유지 관리 |
| `reflect` | 지속적 개선: 마찰 발견 → 최소 수정 제안 |
| `remove-deadcode` | 데드 코드 안전 검색 및 삭제, 삭제 전 LSP 검증 |
| `security-review` | 병합 전 diff 보안 감사 |
| `shared-language` | 도메인 용어집 구축, 컨텍스트 토큰 대폭 절약 |
| `simplify` | 동작 유지 코드 단순화(oracle 분석 → light-orchestrator 적용) |
| `spec-workflow` | 경량 스펙 주도 변경(propose → design → tasks → implement → archive) |
| `verification-planning` | 구현 전 최소 검증 경로 계획 |
| `verify-with-docs` | 코딩 전 API 문서 확인, 검색 우선, 환각 방지 |
| `writing-great-skills` | 스킬 작성 규범: 무작업 가지치기, 긍정적 표현, 완료 기준 |

## 설계 결정과 반복 기록

핵심 아이디어는 [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)(의도 게이트, 읽기 전용 격리, 안티 패턴), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)(스케줄러 우선, 폴백 체인, 거부 계약), [anomalyco/opencode](https://github.com/anomalyco/opencode)(구성 스키마, 스킬 체계), [cli/cli](https://github.com/cli/cli)(gh v2.97), [OpenSpec](https://github.com/Fission-AI/OpenSpec)(델타 스펙, 변경 제안서 업데이트), [mattpocock/skills](https://github.com/mattpocock/skills)(인수인계 문서, 구조화된 디버깅), [pi](https://github.com/earendil-works/pi)(먼저 답변 후 수정, 간결한 응답), [deepreview](https://github.com/mechanai/deepreview)(novelty-based convergence), conflict resolution discipline의 장점을 참고하였으며, 순수 구성으로 구현, 추가 의존성 없음.

> **참고하되 그대로 베끼지 않음**: 지나치게 무거운 파이프라인은 경량 설계 이념만 추출합니다. 중복 기능은 기존 agents/skills가 커버하며, 새로 추가하지 않습니다. "추가보다 간소화 우선" 원칙을 따르며, 매 반복마다 토큰 순감소를 목표로 합니다.

### 반복 마일스톤

v1 이후 25회 반복, 지속적으로 업스트림 모범 사례에 정렬:

- **v1-v7 (기반)**: 듀얼 모델 바인딩, 에이전트 역할 시스템, 의도 게이트 라우팅, AGENTS.md 글로벌 규칙, Skills 디렉토리, 권한 기준
- **v8-v15 (리뷰 + 사양 + 계약)**: code-review 듀얼 축 보정, spec-workflow, gh-cli 정렬, 거부 계약, 백그라운드 확인
- **v16-v22 (지속적 슬림화)**: 명령 29→18 (-38%), AGENTS.md 290→211 (-27%), no-op 트리밍, 스키마 검증
- **v23-v25 (정렬 + 보안)**: 6개 업스트림 저장소 통합, gh-cli v2.97 이스케이프 인젝션 경고, procedure-driven 프롬프트 개선, DCP 창 튜닝

## 저장소 구조

```text
├── opencode/                     # OpenCode 구성 디렉터리(독립 배포 가능)
│   ├── .ai/
│   │   └── calibration.yml       # code-review 심각도 보정
│   ├── agents/                   # 10개 전담 에이전트
│   │   ├── orchestrator.md       # 주 진입점: 의도 게이트 + 모델 인식 라우팅
│   │   ├── planner.md            # pro: 아키텍처와 계획
│   │   ├── deep-worker.md        # pro: 중량 구현
│   │   ├── oracle.md             # pro: 심층 코드 분석(읽기 전용)
│   │   ├── reviewer.md           # pro: 이중 축 코드 리뷰(읽기 전용)
│   │   ├── consultant.md         # pro: 방안 논의와 조언
│   │   ├── ui-builder.md         # pro: 프론트엔드와 UI
│   │   ├── explore.md            # flash: 코드베이스 검색(읽기 전용)
│   │   ├── librarian.md          # flash: 외부 검색(읽기 전용)
│   │   └── light-orchestrator.md # flash: 간단한 편집
│   ├── skills/                   # 17개 필요 시 로드 스킬
│   │   ├── code-review/          # 이중 축 병렬 리뷰 + 심각도 보정
│   │   ├── codemap/              # 저장소 구조도 생성
│   │   ├── gh-cli/               # GitHub CLI v2.97+ 참조 + 보안 경고
│   │   ├── git-master/           # 고급 Git 작업
│   │   ├── git-release/          # 태그 릴리스
│   │   ├── handoff/              # 세션을 인수인계 문서로 압축
│   │   ├── opencode-config/      # 메타 스킬: 본 저장소 구성 작성
│   │   ├── reflect/              # 지속적 개선
│   │   ├── remove-deadcode/      # 데드 코드 감지 및 삭제
│   │   ├── resolving-merge-conflicts/ # hunk별 병합 충돌 해결 규율
│   │   ├── security-review/      # 보안 감사 체크리스트
│   │   ├── shared-language/      # 도메인 용어집(토큰 절약)
│   │   ├── simplify/             # 동작 유지 코드 단순화
│   │   ├── spec-workflow/        # 스펙 주도 개발
│   │   ├── verification-planning/ # 구현 전 검증 경로 계획
│   │   ├── verify-with-docs/     # 검색 우선 API 검증
│   │   └── writing-great-skills/ # 스킬 작성 규범
│   ├── opencode.jsonc            # 주 구성(18개 명령)
│   ├── AGENTS.md                 # 전역 규칙(~212줄)
│   └── dcp.jsonc                 # DCP 컨텍스트 압축(DeepSeek 128K)
├── README.md
├── LICENSE
└── README.*.md                   # 기타 언어 README
```

## 사용 가이드

### 모드 1: Orchestrator 자동 라우팅(기본)

자연어로 요구사항을 설명하면, Orchestrator가 자동으로 의도를 분석하고 가장 적합한 에이전트와 모델을 선택하여 실행합니다.

```text
「이 로그인 인터페이스의 오류를 분석해 주세요」   → oracle 근본 원인 분석 → 진단 보고서 반환
「이 루프를 최적화해 주세요, 성능이 너무 나쁩니다」 → oracle 분석 → deep-worker 최적화 구현
「이 PR 리뷰를 부탁합니다」                     → reviewer 다차원 리뷰 → 등급별 보고서 반환
「사용자 모듈에 내보내기 기능을 추가하고 싶습니다」 → planner 방안 수립 → deep-worker 구현
「React 19의 use() API 사용법을 알려주세요」      → librarian 문서 조회 → 시그니처와 예제 반환
```

### 모드 2: 명령 별칭 직접 사용

| 시나리오 | 명령 |
| --- | --- |
| 복잡한 구현 / 다중 파일 변경 | `/deep` |
| 경량 수정 / 단일 파일 편집 | `/quick` |
| 기술 방안 수립 / 아키텍처 설계 | `/plan` |
| 버그 조사 / 심층 분석 | `/oracle` |
| 코드 리뷰 | `/review` |
| 외부 검색 / API 조회 | `/search` |
| 프론트엔드 / UI 작업 | `/ui` |
| 방안 논의 / 비교 및 선택 | `/consult` |
| 구조화된 디버깅 | `/oracle` |

### 일반적인 워크플로우

**새 기능 개발(스펙 주도):**
```text
/spec-propose  → /spec-apply  → /review
```

**버그 조사:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**코드 리뷰:**
```text
/review-pr   ← PR 리뷰 + 자동 댓글 게시
/review      ← 이중 축 병렬 리뷰
```

## 설계 철학

- **순수 구성 주도, 추가 의존성 없음** —— 모든 기능은 `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`로 구현
- **DeepSeek V4 듀얼 모델 극한 활용** —— Pro는 추론과 의사 결정을, Flash는 조회와 경량 실행을 담당
- **토큰 효율성 우선** —— 파일 붙여넣기 대신 경로 참조, 스킬 필요 시 로드, 압축 계층적 관리
- **플러그인은 효율을 높이되 주객전도 금지** —— superpowers는 프로세스 규율 제공, DCP 지능형 압축으로 단순 절단 대체
- **실행과 탐색 분리** —— deep-worker/light-orchestrator는 연구/위임 금지, explore/librarian은 수정 금지
- **지속적 개선** —— reflect로 마찰 체계적 발견, code-review 이중 축 보정으로 품질 보장
