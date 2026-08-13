---
name: gh-cli
description: Patterns for invoking the GitHub CLI (gh v2.97.0+) from agents. Use when the task mentions GitHub, gh, pull requests/PRs, issues, releases, gists, Actions/workflow runs, forks, repo cloning, reviews, or you need exact gh commands. Covers structured output, pagination, repo targeting, search vs list, issue types/sub-issues, discussions, projects, rulesets, agent skills, AI commands (copilot, agent-task), repo read-file/read-dir, and gh api fallback.
---

# GitHub CLI (`gh`) agent patterns

Authoritative patterns for driving the official `gh` CLI (v2.97.0) from agents,
based on [cli/cli](https://github.com/cli/cli) trunk. Prefer `gh` over raw `curl`
or `gh api` — `gh` handles auth, pagination, and JSON output automatically.

## Security Advisory — escape-sequence injection (v2.97.0)

v2.97.0 fixed 4 escape-sequence injection vulnerabilities. These commands can
inject ANSI control sequences (cursor movement, screen clearing, clipboard
exfiltration) when output is rendered to a terminal:

| Affected command | Risk |
|---|---|
| `gh gist view` | Untrusted gist content → terminal |
| `gh api` | Untrusted API response → terminal |
| `gh pr diff` | Untrusted PR content → terminal |
| `gh release download --output -` | Untrusted release artifact → stdout/terminal |
| `gh codespace logs` | Untrusted container output → terminal |
| `gh agent-task view` / `create` | Untrusted task description/output → terminal |

**Agent rules:**
- **Never** pipe output to a terminal renderer. Prefer `--json` for structured
  output; use `> file` for raw content.
- **Never** use `gh release download --output -` (stdout) — always save to a file
  with `--output <path>`.
- For `gh repo read-file`, binary content is auto-refused and ANSI is stripped
  by default since v2.97; use `--allow-escape-sequences` only when you need raw
  escapes and understand the risk.
- When fetching issues/PRs/comments from untrusted repos, prefer `--json` over
  human-readable output — JSON is not vulnerable to escape injection.

## Interactivity policy

`gh` does the right thing in non-TTY contexts: skips the pager, strips ANSI
color, and errors fast instead of prompting (e.g. `must provide --title and --body
when not running interactively`).

- Set `GH_PROMPT_DISABLED=1` to force `gh` to fail instead of prompting.
- A few commands still need explicit flags non-interactively: `gh pr merge`
  (`--squash`/`--merge`/`--rebase`), `gh release create` (`--notes` or
  `--generate-notes`), `gh pr create` (`--fill` or explicit `--title`/`--body`).
- Exit codes: `0` success, `1` failure, `2` cancelled, `4` auth required. Check
  `gh auth status` first; missing auth returns `4`, not `1`.

### CI / non-interactive mode

Essential environment variables for agent-driven scripting:

| Variable | Effect |
|---|---|
| `GH_PROMPT_DISABLED=1` | Fail instead of prompting interactively |
| `GH_PAGER=cat` | Disable pager (already auto in non-TTY) |
| `GH_NO_UPDATE_NOTIFIER=1` | Skip version check (saves a request) |
| `GH_FORCE_TTY=1` | Force colored/formatted output even when piped |
| `NO_COLOR=1` | Strip ANSI color from output |
| `GH_DEBUG=api` | Log HTTP request/response for debugging |
| `GH_TELEMETRY=false` | Disable telemetry (opt-out; default enabled since v2.91.0). `gh telemetry` is a help topic, not a command group. |

Token/auth: `GH_TOKEN` or `GITHUB_TOKEN` for non-interactive use. `@me` resolves to the authenticated user: `--assignee @me`, `--author @me`, `--owner @me`. Use `gh auth status --json` to verify the active session.

## Parsing JSON

- `--json field1,field2,...` for structured output
- `--json` with no field list prints available fields — use this first
- `--jq '<expr>'` to filter without piping through `jq`
- `--template '<go-template>'` for shaped text output. Note: `-T`/`--template`
  collides with `gh pr create`/`gh issue create` body-template flag.
- Template helpers: `tablerow`, `tablerender`, `timeago`, `truncate`, `hyperlink`,
  `pluck`, `join`, `color`, `autocolor`, `regexMatch`, `contains`.

## Pagination

List commands cap results silently:

- `gh pr list`, `gh issue list`, `gh search ...`: use `-L N` (`--limit N`).
  Default is 30. No `totalCount` via `--json` — use `gh api graphql` for true
  totals.
- `gh api --paginate <path>` concatenates each page's JSON. For `[...]` responses
  that yields multiple arrays — add `--slurp` to wrap into one array.
- `gh api --cache 30m <path>` caches responses to avoid repeat hits.

## Repo targeting

`gh` infers the repo from cwd git remotes. Pass `--repo OWNER/REPO` (`-R`) to
override. Set `GH_REPO=OWNER/REPO` for session-wide default.

## Search vs list

- `gh search issues|prs|code|repos|commits` uses GitHub's search index with full
  syntax. Each qualifier is its own bare token — do NOT quote them as one string:
  `gh search issues repo:cli/cli is:open author:monalisa` works,
  `gh search issues "repo:cli/cli is:open"` fails. Quote only multi-word free text.
- `gh issue list --search "..."` / `gh pr list --search "..."` take one quoted
  string, scoped to one repo.
- Bots author as GitHub Apps: `--author dependabot` fails. Use `--app dependabot`
  (on `pr`/`issue list` and `search prs|issues`).
- Exclude qualifiers with `--` stop-parser: `gh search issues -- "error -label:bug"`.
- Advanced search syntax (v2.79.0+): `gh search issues` and `gh search prs` support
  the full GitHub search qualifier syntax — `author:`, `label:`, `milestone:`,
  `assignee:`, `review:`, `status:`, `base:`, `head:`, `merged:`,
  `created:`, `updated:`, `closed:`, `comments:`, `interactions:`, `reactions:`.
  Combine multiple: `gh search prs --repo cli/cli --author monalisa --label bug --state open`.

## Issue types, sub-issues, and relationships (v2.94.0+)

- `gh issue create`: `--type Bug|Feature|Task` (or custom org types on GHES 3.17+),
  `--parent <number|url>`, `--blocked-by <number|url,...>`,
  `--blocking <number|url,...>`.
- `gh issue edit <n>`: `--type/--remove-type`, `--parent/--remove-parent`,
  `--add-sub-issue <n>/--remove-sub-issue <n>`,
  `--add-blocked-by <n>/--remove-blocked-by <n>`,
  `--add-blocking <n>/--remove-blocking <n>`.
- `gh issue list --type Bug|Feature|Task` (filter by type).
- `gh issue close <n> --duplicate-of <number|url>` — closes as a duplicate
  (sets `--reason duplicate` and links the canonical issue).
- JSON fields: `issueType`, `parent`, `subIssues` (`nodes` + `totalCount`),
  `subIssuesSummary`, `blockedBy` (`nodes` + `totalCount`), `blocking` (`nodes` +
  `totalCount`). Compare `nodes.length` vs `totalCount` to detect truncation
  (subIssues capped at 100, blocked/blocking at 50).
- GHES availability: issue **types** require GHES 3.17+, **sub-issues** (parent/sub)
  require GHES 3.17+, **blocked-by/blocking** relationships require GHES 3.19+. On
  older hosts these flags error — fall back to labels for types, linked issues for
  relationships.

## Discussions (`gh discussion`) — v2.94.0 preview

- `gh discussion list`: `--state open|closed|all`, `--category <name>`,
  `--author <login>`, `--label <name>`, `--answered` (tri-state for Q&A),
  `--search <query>`, `--sort`/`--order`, `--limit`, `--json`
- `gh discussion view {<n>|<url>}`: `--comments`, `--order`, `--limit`, `--json`
- `gh discussion create`: `--title`, `--body`, `--category` (required
  non-interactively)
- `gh discussion edit <n>`: `--title`, `--body`, `--add-label <name>`,
  `--remove-label <name>`, `--category`
- `gh discussion comment {<n>|<url>}`: `--body "text"`
  `--edit <comment-id>`, `--delete <comment-id>` (`--yes`
  to skip confirmation prompt)

## Projects V2 (`gh project`)

Full project management with 19 subcommands. Key patterns:

```bash
# List and view
gh project list --owner "@me"
gh project view <number> --json title,url,fields

# Items — add issues/PRs to projects
gh project item-add <number> --url <issue-or-pr-url>
gh project item-list <number> --owner "@me" --limit 50
gh project item-list <number> --owner "@me" --query '."Status" == "Todo"'     # filter by field value (v2.87.0+)
gh project item-edit <item-id> --field-id <field-id> --text "value"
gh project item-edit <item-id> --field-id <field-id> --iteration-id <id>
gh project item-edit <item-id> --field-id <field-id> --single-select-option-id <id>
gh project item-archive <number> <item-id>

# Custom fields
gh project field-list <number> --json name,id,dataType
gh project field-create <number> --name "Status" --data-type SINGLE_SELECT \
  --single-select-options "Todo,In Progress,Done"

# CRUD
gh project create --title "My Project" --owner "@me"
gh project copy <number> --source-owner <org> --target-owner "@me" --title "Copy"
gh project close <number>
gh project delete <number>
```

# By-name field editing (v2.97.0+)
```bash
gh project item-edit <item-id> --field "Status" --value "In Progress"
gh project item-edit <item-id> --field "Priority" --clear
gh project item-list <number> --owner "@me" --field "Status" --field "Priority"
```

The old `--field-id`/`--text`/`--single-select-option-id` flow still works for scripting but the by-name path is simpler for humans.

All `gh project` commands accept `--owner` (user or org). Use `--format json` +
`--jq` for structured output on `item-list`, `field-list`.

## Rulesets (`gh ruleset`)

```bash
gh ruleset list -R owner/repo             # --json for structured
gh ruleset view <id> -R owner/repo
gh ruleset check -b main -R owner/repo    # check branch compliance
```

## Actions cache (`gh cache`)

```bash
gh cache list -R owner/repo -L 50 --json key,sizeInBytes,ref
gh cache delete <key> -R owner/repo
gh cache delete --all -R owner/repo --succeed-on-no-caches   # --succeed-on-no-caches avoids non-zero exit when no caches exist
gh cache delete --ref refs/heads/main -R owner/repo            # delete by ref (v2.86.0)
gh cache delete --all --ref refs/heads/main -R owner/repo      # all caches for a ref
```
- `gh cache delete --succeed-on-no-caches` (v2.86.0+) exits 0 when no caches match, preventing scripting breaks.

## Reading files and directories (`gh repo read-file` / `read-dir`) — v2.95.0 preview

Read repo contents over API without cloning. Use `-R`/`--repo` and
`--ref <branch|tag|commit>` to target any repo/ref.

```bash
# Read a file (prints raw content to stdout)
gh repo read-file README.md --repo cli/cli
gh repo read-file go.mod --ref v2.94.0 --output ./go.mod --clobber

# Read a directory listing
gh repo read-dir script --repo cli/cli
gh repo read-dir --ref main --json name,path,size,type
```

- Piped/stdout output is **raw bytes** — no JSON wrapper. Binary files are
  auto-detected and refused; use `--output` to save binary files.
- `--json` fields for `read-dir`: `name`, `path`, `gitSHA`, `size`, `type`
  (`file`/`dir`/`submodule`/`symlink`), `encoding`, `content`.
- `--clobber` overwrites existing output files; without it, writing to an
  existing file is an error.

## Reviewing PRs (`gh pr review` vs inline comments)

`gh pr review <n>` submits **only** a top-level review — one verdict + one body:

```bash
gh pr review <n> --approve  --body "LGTM"
gh pr review <n> --comment  --body "notes…"        # -c
gh pr review <n> --request-changes --body "…"       # -r (body required)
gh pr review <n> --approve --body-file review.md    # -F, use - for stdin
```

It has **no** flag for per-line comments — a common agent mistake. To attach
findings to specific lines, post one pending review via the REST API with a
`comments[]` array (new-side line numbers, inside changed hunks):

```bash
gh api repos/{owner}/{repo}/pulls/<n>/reviews --method POST \
  -f event=COMMENT -f body="overall summary" \
  -F 'comments[][path]=src/app.go' -F 'comments[][line]=42' \
  -F 'comments[][body]=this needs a nil check' \
  -F 'comments[][path]=src/app.go' -F 'comments[][line]=88' \
  -F 'comments[][body]=off-by-one here'
```

`event`: `APPROVE` | `REQUEST_CHANGES` | `COMMENT`, or omit for a `PENDING`
draft. Never auto-`APPROVE` from an agent — leave the verdict to a human.

## `gh api` — the universal fallback

When no porcelain command covers what you need:

### GraphQL (preferred for complex queries)

```bash
# Basic
gh api graphql -f query='query { viewer { login } }'

# With variables (use -F for typed: numbers, booleans, @file)
gh api graphql -F owner='cli' -F name='cli' -f query='
  query($name: String!, $owner: String!) {
    repository(owner: $owner, name: $name) {
      releases(last: 3) { nodes { tagName } }
    }
  }'

# Paginated GraphQL ($endCursor is auto-managed by --paginate)
gh api graphql --paginate -f query='
  query($endCursor: String) {
    search(query: "is:pr is:merged", type: ISSUE, first: 100, after: $endCursor) {
      nodes { ... on PullRequest { number title } }
      pageInfo { hasNextPage endCursor }
    }
  }'
```

### REST

```bash
gh api repos/{owner}/{repo}/releases
gh api repos/{owner}/{repo}/issues/123/comments -f body='Hello'
gh api --paginate --slurp repos/{owner}/{repo}/issues --jq 'map(.number)'
gh api --cache 30m repos/{owner}/{repo}
```

`{owner}/{repo}` placeholders auto-fill from detected remotes.
`-f key=value` sends strings; `-F key=value` parses numbers/booleans/`@file`.

## Authentication

- `gh auth status --json` — active host, user, auth source
- `GH_TOKEN` / `GITHUB_TOKEN` env vars for non-interactive/CI use
- `GH_ENTERPRISE_TOKEN` for GHES, `GH_HOST` for enterprise instances
- Never paste tokens on the command line; use `--with-token < file` or env vars

## Gists (`gh gist`)

```bash
gh gist create file.txt -d "description"               # public by default
gh gist create file.txt -d "desc" --public              # explicit public
gh gist create file.txt -d "desc" --secret              # unlisted
gh gist create -f "name=content" -d "snippet"           # inline content (-f)
gh gist list -L 20 --public                              # your public gists
gh gist list -L 20 --secret                              # your secret gists
gh gist view <id> --raw                                  # raw content
gh gist view <id> --files                                # list files
gh gist edit <id> -a "new content" -f file.txt           # append content
gh gist edit <id> -f "file.txt=new content"               # replace content
gh gist delete <id>                                       # delete (needs confirmation)
gh gist clone <id> [dir]                                  # clone to local dir
```

## Secrets and Variables (`gh secret`, `gh variable`)

Actions secrets and variables — distinct commands, both scoped to repo/org/env:

```bash
# Secrets (encrypted)
gh secret set SECRET_NAME -b "value" -R owner/repo         # from string (-b body)
gh secret set SECRET_NAME < secret.txt -R owner/repo       # from file
gh secret set SECRET_NAME -b "$(cmd)" -R owner/repo        # from command output
gh secret set SECRET_NAME -b "val" --org org                # org-level
gh secret set SECRET_NAME -b "val" --env production -R owner/repo  # env-level
gh secret set SECRET_NAME -b "val" --app agents -R owner/repo     # Copilot agent secret (v2.93.0+)
gh secret list -R owner/repo                               # list names (not values)
gh secret remove SECRET_NAME -R owner/repo

# Variables (plaintext)
gh variable set VAR_NAME -b "value" -R owner/repo
gh variable set VAR_NAME -b "val" --org org
gh variable set VAR_NAME -b "val" --env staging -R owner/repo
gh variable list -R owner/repo
gh variable remove VAR_NAME -R owner/repo
```

## Codespaces (`gh codespace`)

```bash
gh codespace list --json name,state,machine
gh codespace create -R owner/repo -b main -m basicLinux32gb
gh codespace create -R owner/repo -b main --devcontainer-path .devcontainer/dev.json
gh codespace stop -c <name>
gh codespace delete -c <name>
gh codespace logs -c <name>
gh codespace ssh -c <name>                                # SSH in
gh codespace ports visibility 3000:public -c <name>       # port forwarding
```

## Config (`gh config`)

```bash
gh config set editor "code --wait"
gh config set git_protocol ssh
gh config set prompt disabled                             # disable interactivity
gh config get git_protocol
gh config list
gh config set browser ""                                   # disable browser opening
```

## Extensions (`gh extension`)

```bash
gh extension install owner/repo                           # install from GitHub (no auth required since v2.90.0)
gh extension install /path/to/local                       # install from local dir
gh extension list                                          # list installed
gh extension upgrade owner/repo                           # upgrade one
gh extension upgrade --all                                # upgrade all
gh extension remove owner/repo                            # uninstall (alias for `uninstall`, v2.94.0+)
gh extension exec <name> [args]                           # run by name
gh extension search <query>                               # search public extensions (top 30 by stars default)
gh extension browse                                       # interactive TUI to browse/add/remove extensions
gh extension create [<name>]                              # scaffold a new extension (--precompiled go|other)
```

Note: `gh extension install` no longer requires authentication (v2.90.0+). The
CLI downloads public extensions without checking for an active session.

## Aliases (`gh alias`)

```bash
gh alias set myprs 'pr list --author @me --state open -L 10'
gh alias set --shell review 'gh pr diff $1 | code -'     # pipe to editor
gh alias list
gh alias delete myprs
```

## Agent Skills (`gh skill`) — v2.94.0+

First-class command group for discovering, installing, and publishing Agent Skills.
Replaces the standalone `gh-skill` skill (deleted in v23).

```bash
# Discovery (v2.94.0+)
gh skill search <query>                                 # search for skills (no --agent flag)
gh skill search <query> --owner <org-or-user>            # filter by owner
gh skill list --json skillName,description,installed    # list installed skills
gh skill preview <skill-id>                              # preview a skill before installing
gh skill preview <skill-id> --allow-hidden-dirs          # discover in hidden dirs

# Installation (v2.94.0+)
gh skill install owner/repo --agent opencode             # install with agent binding
gh skill install owner/repo --agent opencode --pin v1.2  # pin to version tag
gh skill install author/skill@v1.2.0 --agent opencode    # namespaced with version
gh skill install --all --agent opencode                  # install all matching
gh skill install --from-local ./local-repo --agent opencode # from local dir
gh skill install owner/repo --force --agent opencode     # force overwrite
gh skill install owner/repo --dir ./custom-path          # install to custom dir

# Management (v2.94.0+)
gh skill update <skill-id>                               # update installed skill (no --agent)
gh skill update --all --dry-run                           # preview updates
gh skill update --all --force --unpin                     # force update pinned skills

# Publishing (v2.94.0+)
gh skill publish                                          # publish skill from cwd
```

`gh skill install --agent opencode` installs skills to `~/.config/opencode/skills/<name>/SKILL.md`. Default scope is `project`, default agent is `github-copilot` — always pass `--agent opencode`. Skills installed via `gh skill` appear in the agent's available skills list automatically — no config change needed.

## AI-integrated commands (`gh copilot`, `gh agent-task`)

### `gh copilot` — native built-in (v2.86.0+)

The native `gh copilot` command replaces the deprecated `gh copilot` extension
(Oct 2025). It is a built-in passthrough that downloads and runs the standalone
Copilot CLI binary (`~/.config/gh/copilot/`).

```bash
# Prompt-based assistance
gh copilot "explain this error: cannot find module 'lodash'"
gh copilot suggest "how to squash the last 3 commits"
gh copilot "write a bash script to find files larger than 100MB"

# Interactive mode (starts a Copilot session)
gh copilot
```

`gh copilot` is agent-driven and human-in-the-loop — it prompts for clarification
before making changes. Prefer it for interactive development workflows, not
unattended scripts.

### `gh agent-task` — delegate coding tasks (v2.80.0+)

Aliases: `gh agent`, `gh agents`. Delegates a coding task to a GitHub coding agent
(preview). Requires an OAuth token from `gh auth login` (`gho_` prefix) — plain
PAT/`GH_TOKEN` is rejected.

```bash
gh agent-task create "Fix the login redirect"                       # positional description
gh agent-task create "Fix the login redirect" --base main           # set base branch
gh agent-task create "Add pagination" -F task-description.md        # from file
gh agent-task create --custom-agent my-agent "Add pagination"       # specific agent (v2.83.0+)
gh agent-task create --follow "Fix login"                           # follow progress
gh agent-task list --json id,name,state,repository                  # list tasks
gh agent-task view <id> --json id,name,state                        # view task details
```

- `--custom-agent` / `-a` flag (v2.83.0+) selects a named custom agent instead of the default.
- `--json` / `--jq` / `--template` support for `list` and `view` (v2.88.0+).

### Copilot as reviewer / assignee

Request Copilot reviews on PRs (v2.88.0+, github.com + GHES 3.15+):

```bash
gh pr create --reviewer @copilot --fill
gh pr edit <n> --add-reviewer @copilot
```

Assign Copilot to issues (v2.73.0+, github.com):

```bash
gh issue edit <n> --add-assignee @copilot
gh issue create --assignee @copilot            # v2.76.0+
```

## Release verification (`gh release verify`) — v2.75.0+

Verify release artifact attestations (Sigstore supply-chain). No auth needed for
public repos.

```bash
gh release verify -R cli/cli                  # verify attestation for latest release
gh release verify v2.96.0 -R cli/cli          # verify a specific release
gh release verify-asset cli.zip -R cli/cli    # verify a specific asset (v2.81.0+)
```

Note: Releases created with v2.93.0+ are immutable — JSON output includes an
`isImmutable` field. Use `gh release download <tag>` (no auth for public repos,
v2.96.0+) to fetch artifacts.

## Recent commands worth knowing

- `gh pr revert <n>` — open a revert PR (`--draft`, `--title`, `--body-file`).
- `gh pr update-branch <n>` — sync PR branch from base (`--rebase`).
- `gh pr checkout <n>` — alias `gh co` (v2.81.0+).
- `gh pr create --fill-first` — use only the first commit's message as body (vs `--fill` which uses all commits, `--fill-verbose` which uses all commit msg+bodies).
- `gh pr create --dry-run` — preview without creating.
- `gh pr create --recover <token>` — recover from a crashed create session.
- `gh pr diff --exclude <pattern>` — exclude specific file paths from diff (v2.88.0+). Useful for filtering out generated files or vendor dirs.
- `gh repo clone --no-upstream` — clone without adding a remote named `upstream` (v2.88.0+).
- `gh repo edit --squash-merge-commit-message <message>` — set the default message for `--squash` merges (v2.88.0+).
- `gh browse --blame` — open GitHub's blame view for the current file (v2.88.0+).
- `gh browse --actions` — open the Actions tab (v2.85.0+).
- `gh run watch <id> --exit-status` — block until run finishes, exit non-zero on
  failure. `--compact` (v2.74.0+) shows only failed/relevant steps. Fine-grained
  PATs lack `checks:read` — use `GITHUB_TOKEN` in Actions or a classic PAT.
- `gh run cancel <id> --force` — immediately cancel without waiting for in-progress steps (v2.78.0+).
- `gh run rerun <id> --failed` — rerun only failed jobs.
- `gh attestation verify|download file.bin -R owner/repo` — Sigstore supply-chain.
- `gh attestation trusted-root` — export a `trusted_root.jsonl` for offline verification (pass via `--custom-trusted-root`; `--tuf-url`/`--tuf-root` for a custom TUF mirror).
- `gh release download <tag>` — no auth needed on public repos (v2.96+).
- Telemetry: set `GH_TELEMETRY=false` or `DO_NOT_TRACK=true` env vars to disable (v2.91.0+).
- `gh skill` — see the Agent Skills section above and the `gh-skill` skill for the full workflow.
- `--json` with NO value: `gh pr list --json` prints all available JSON field names — use this to discover fields before querying. Works on all list/view commands.
- `gh issue develop <n>` — manage linked branches for an issue (`--list`, `--checkout`, `--name`, `--base`).
- `gh org list` — list organizations for the authenticated user (`-L` to limit).
- `gh label clone` — clone labels from a source repo to the current repo (`--force` overwrites existing).
- `gh preview prompter` — experimental; unstable, may change at any time — do not depend on it.

## New in v2.97.0

- `gh project` by-name field editing (see above)
- Security: `gh api`, `gh gist view`, `gh pr diff`, `gh repo read-file` now strip ANSI escapes by default; use `--allow-escape-sequences` to preserve them

## Quick reference

Most common commands agents need:

```bash
# Issues with types (v2.94.0+)
gh issue create --type Bug --title "..." --body "..."
gh issue list --type Bug --assignee @me -L 20 --json number,title,state,issueType
gh issue close <n> --duplicate-of <n>
gh issue edit <n> --add-assignee @copilot

# PRs
gh pr list --state open --label bug -L 20 --json number,title,state,headRefName
gh pr view <n> --json state,mergeable,reviewDecision,statusCheckRollup
gh pr create --fill --base main
gh pr create --title "Fix X" --body "Closes #12" --base main
gh pr create --reviewer @copilot --fill
gh pr diff <n>
gh pr diff <n> --exclude '*.generated.*'
gh pr merge <n> --squash --delete-branch
gh pr revert <n>
gh pr checks <n>
gh pr ready <n>
gh co <n>                             # shorthand for gh pr checkout (v2.81.0+)

# Discussions (v2.94.0 preview)
gh discussion list --state open --json number,title,category
gh discussion create --title "..." --body "..." --category "..."
gh discussion comment <n> --body "..."

# Actions
gh run list --workflow ci.yml --branch main --limit 20
gh run view <id> --log-failed          # failed-step logs only (--log for full; the two are mutually exclusive)
gh run watch <id> --exit-status --compact
gh run cancel <id> --force
gh run rerun <id> --failed

# Releases
gh release create v1.2.0 --generate-notes
gh release download <tag>                                   # no auth for public repos
gh release verify -R owner/repo                             # verify latest release attestation

# Search
gh search prs --author @me --state open --label bug --repo OWNER/REPO
gh search issues --repo OWNER/REPO --search "error in:title"
gh search repos --language go --stars ">5000"

# Copilot & agent tasks
gh copilot "explain this error"
gh agent-task create "Fix login"
gh agent-task list --json id,name,state

# Skills & file reading
gh project item-edit <id> --field "Status" --value "Done"   # new in v2.97.0
gh skill search <query>
gh skill install owner/repo --agent opencode
gh skill update <skill-id>
gh repo read-file README.md --repo cli/cli
gh repo read-dir script --repo cli/cli

# Status
gh status                                # assigned issues/PRs, review requests, mentions, repo activity
gh status --org <org>                    # scope to one org; --exclude owner/repo to skip repos

# Auth & config
gh config set git_protocol ssh
gh auth status --json
# Telemetry controlled via GH_TELEMETRY=false env var
```
