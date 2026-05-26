# Code Pipeline — Architecture Guide

Read this before troubleshooting, improving, or extending the pipeline. It explains the full system, how the layers relate, and where to put any change you make.

---

## What the Pipeline Does

An autonomous agent runs on a schedule (~30 min) and advances coding tasks through a multi-system lifecycle:

```
Qalatra (task management)
  → FlightDesk (PR/review/QA tracking)
  → Claude Code cloud sessions (the actual coding)
```

**Human gates — the pipeline never skips these:**
1. **Plan review** — you review the plan before dispatching to execute-plan
2. **Approval** — the project's PM system has an "approved" status; the pipeline detects it and merges + deploys automatically

Everything else is autonomous.

---

## Qalatra Task Roles

Qalatra has two distinct roles in the pipeline. Keep them separate:

1. **Planning tasks** are one-per-source-task per routed repo. They dispatch the plan agent and must be completed when the plan is written to the repo and source system.
2. **Pipeline monitor tasks** are scheduler handles for per-repo pipeline agents. They are not the source task and should not accumulate unbounded context.

For scheduled monitor tasks, use a bounded window:

- Recommended default: **one active monitor task per repo per day** for deployments that run every 10-30 minutes.
- Requeue the same daily task during that day instead of creating one task per run.
- Write detailed run output to `output/YYYY-MM-DD-HH-MM-*.md`; do not append routine "checked, still waiting" notes to Qalatra forever.
- Keep the monitor task description as the current task snapshot. Add Qalatra notes/context only for notable events: failures, handoffs, cleanup, or human attention.
- Per-repo pipeline agents can exceed Qalatra's default 15-minute runner timeout while fixing review comments or advancing multiple tasks. Set `"timeout_minutes": 45` in pipeline `agent.config` files unless the deployment has a stronger reason to keep the default.
- On the first run of a new day, complete stale active monitor tasks whose latest job is not `running` or `queued`.
- If duplicate active monitor tasks exist for the same repo/day, keep the `running`/`queued` one if present, otherwise keep the newest; complete the stale duplicates.

No Qalatra feature change is required for this pattern. Existing primitives are enough: `get_tasks_by_agent`, `create_task`, `update_task`, `queue_agent_job`, and `complete_task`.

---

## Three-Layer Architecture

### Layer 1 — Canonical (`qalatra-prompts`)

**Repo:** `https://github.com/pirateandfox/qalatra-prompts`  
**Local:** `~/IdeaProjects/qalatra-prompts/`

Files:
- `pipeline-agent.md` — full stage handler logic
- `pipeline-architecture.md` — this document

**Changes here apply to every pipeline deployment.** Only put things here if they genuinely apply across all projects regardless of framework, team, or source system. When this file improves, every deployment benefits on its next run.

### Layer 2 — Per-Repo Config

**Location:** `~/IdeaProjects/{repo}/agents/pipeline-config.md`

One file per code repo. Defines everything specific to that codebase:

| Field | Description |
|---|---|
| `framework` | `nestled` \| `shopify` \| `electron` \| `other` |
| `github_slug` | e.g. `BizToBiz/biztobiz` |
| `base_branch` | e.g. `develop`, `live` |
| `flightdesk_project_id` | UUID |
| `flightdesk_subprojects` | optional table (Monroe pattern) |
| `source_system` | `notion` \| `linear` \| `asana` |
| `notion_mcp_prefix` | e.g. `mcp__claude_ai_Notion__` or `mcp__notion-monroe__` |
| `notion_database_id` | UUID |
| `notion_field_*` | field names for status, FlightDesk URL, GitHub URL |
| `notion_status_*` | exact status string values |
| `notion_pickup_statuses` | statuses that mean "ready to be worked on" |
| `notion_routing_field` | for multi-repo deployments (Monroe pattern) |
| `qa_reviewer_id` | FlightDesk user ID, or `none` |
| `auto_merge` | `true` = merge+deploy on approval; `false` = skip |
| `deploy_command` | e.g. `pnpm stage && pnpm run deploy`, or `none` |
| `reconnect_task_context` | Qalatra context for health-check alert tasks |
| `new_code_coverage_target` | Optional coverage target for changed/new code, e.g. `80%` |

**Changes here apply to that repo only.** When you learn a new field name, a new status value, a different deploy command — update this file.

### Quality Gates

Repo-specific quality requirements belong in `{repo}/agents/pipeline-config.md` and should also be enforced by CI when possible. Do not rely on CI alone: put the rule in config so agents know the target before coding.

Example:

| Field | Value |
|---|---|
| `new_code_coverage_target` | `80%` |
| `coverage_scope` | New or changed code introduced by the PR |
| `coverage_source` | CI/SonarCloud/coverage report |
| `coverage_policy` | Agents must add or update tests with implementation work; pipeline treats unmet or unmeasured configured coverage as blocking before QA handoff. |

If CI cannot currently measure the target, the pipeline should flag the missing measurement as a setup gap instead of silently declaring the task QA-ready.

### Layer 3 — Deployment Wrapper

**Location:** `{workspace}/business-tools/code-pipeline/CLAUDE.md`

One per physical machine/deployment. Contains:
- Which repos this instance monitors
- Reference to `pipeline-agent.md` for the canonical logic
- Instruction to read `pipeline-config.md` per task for repo-specific config
- Any deployment-level overrides (rare)

**Changes here apply to one machine only.** Different computers, different Qalatra instances.

---

## Known Frameworks

### `nestled`
Applies to: biztobiz, mi-core, muzebook, nestled, nestled-forms, nestled-template, flightdesk, qalatra

Post-session diff assessment:

| Diff condition | Path |
|---|---|
| No generated files, no schema changes | Open PR directly |
| Generated files hand-edited OR schema annotation-only changes | Codegen reconciliation |
| Schema structural changes (new/removed model or field) | Migration path |

**Codegen reconciliation steps:**
1. `git checkout <branchName>`
2. `pnpm db-update`
3. `npx nx build api` — verify compilation (use `build` not `serve`)
4. If build fails with TS errors → inject error to session, wait for `ready`, pull, re-run
5. `git add -A && git commit -m "chore: regenerate codegen artifacts" && git push`
6. If files changed: inject "please run git pull" → verify session moves to `running`
7. `claude_session_create_pr(session_id)`
8. `git checkout <base_branch>`

**Migration path steps:**
1. Detect port 5432: `docker ps --filter "publish=5432" --format '{{.Names}}'`
2. Stop anything running (get compose file via `docker inspect`)
3. Start: `docker compose -f {repo}/.dev/docker-compose.yml up -d postgres`
4. Wait: `until docker exec $(docker ps ...) pg_isready -U prisma; do sleep 2; done`
5. Check `.env` — `DATABASE_URL` must be `localhost`. **Hard gate: never run Prisma migrate against non-localhost.**
6. `git checkout <branchName>`
7. `pnpm prisma migrate dev --name <slug>`
   - If drift detected: `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION="Yes" pnpm prisma migrate reset --force` then retry
8. `pnpm db-update`
9. `npx nx build api` — verify (inject errors to session if any)
10. Restore `.env` to production-active
11. Stop Docker: `docker compose -f {repo}/.dev/docker-compose.yml down`
12. `git add -A && git commit -m "chore: add migration and regenerate artifacts" && git push`
13. Inject "please run git pull" → verify `running`
14. `claude_session_create_pr(session_id)`
15. `git checkout <base_branch>`

**NX daemon troubleshooting:** If `nx build` hangs: `pkill -f "nx serve api"; pkill -f "nx daemon"; npx nx reset`

Docker credentials (all nestled projects): user=`prisma`, password=`prisma`, db=`prisma`

### `shopify`
Applies to: tmi-shopify-3.0

- No codegen, no migrations, no local API
- Post-session: diff check only — if changes look right, open PR directly
- Base branch: `develop`
- Before execution kickoff, sync `develop` with `live`: merge `origin/live` into `develop` and push. If the sync conflicts or fails, do not register the task.
- Deploy: Shopify pre-push hook handles theme sync automatically

### `electron`
Applies to: qalatra (desktop app)

- No codegen, no migrations, no local web API
- Build/test TBD — define in `pipeline-config.md` when first encountered

### `other`
Anything not matched above. Log `FRAMEWORK_UNKNOWN` and flag for human before proceeding.

---

## Known Deployments

### P&F Pipeline
- Agent: `~/IdeaProjects/projects/business-tools/code-pipeline/`
- Handles: biztobiz, muzebook, nestled, nestled-forms, nestled-template, nestledforms.com, nestledjs.com, flightdesk, silvermouse, qalatra, qalatra.com, moceanic-ai
- Qalatra context: `coderepos`
- Reconnect task context: `internal`
- Auto-merge: per-project config

### Monroe Pipeline
- Agent: `~/IdeaProjects/mi-projects/business-tools/code-pipeline/`
- Handles: mi-core, tmi-shopify-3.0
- Qalatra context: `monroe`
- Reconnect task context: `monroe`
- Multi-repo routing: Notion `Project Type` field
- QA reviewer: Edgar Gutierrez-Mata (resolve ID via `list_members` on first run, cache for session)
- Auto-merge: true (Stage 4b triggers on `Workflow Status = "Approved"`)

---

## Improvement Protocol

When you fix a bug or improve behavior, decide which layer it belongs to before writing it anywhere:

| Question | Layer | Where to write it |
|---|---|---|
| Does this apply to all projects regardless of framework or team? | Canonical | `qalatra-prompts/pipeline-agent.md` |
| Does this apply to all projects using a specific framework? | Canonical (framework section) | `qalatra-prompts/pipeline-agent.md` under the framework block |
| Does this apply to one repo? | Per-repo | `{repo}/agents/pipeline-config.md` |
| Does this apply to one physical machine/deployment? | Deployment | `{workspace}/business-tools/code-pipeline/CLAUDE.md` |

**The rule:** if you're fixing something in a deployment wrapper that should be in the canonical file, it will drift again. Fix it in the right layer once.

After updating `qalatra-prompts/`, commit and push — both deployments pick it up on next run.

---

## Agent Path Reference (post-2026-05-09 rename)

| Stage | agent_path fragment |
|---|---|
| Planning | `agents/plan` |
| Execute from plan | `agents/execute-plan` |
| Execute from task | `agents/execute-task` |

---

## FlightDesk Status Flow

```
PENDING → DISPATCHED → IN_PROGRESS → PR_OPEN
  → PREVIEW_STARTING → PREVIEW_READY
  → REVIEW_RUNNING → REVIEW_DONE → QA_READY
  → QA_CHANGES_REQUESTED / QA_APPROVED
  → MERGED / ARCHIVED
```

Key facts:
- `PREVIEW_READY` is transient — FD can get stuck here if SonarCloud passes on first run (no webhook fires). Check GitHub directly and advance manually if all green.
- `QA_READY` = all checks passed. Use this as the Stage 4 completion signal.
- `REVIEW_DONE` = a check submitted `NEEDS_HUMAN_REVIEW`. Cannot auto-resolve — surface for human.
- Never call `update_task_status` for `PR_OPEN` or `MERGED` — FlightDesk detects these via GitHub webhooks.
- Copilot `status: FAILED` in checks ≠ blocking. Verify with `gh pr view --json reviews` — only block if `state: CHANGES_REQUESTED`.
- SonarQube `PENDING` in FD is a known display bug. Check the SonarCloud proxy for real state.
