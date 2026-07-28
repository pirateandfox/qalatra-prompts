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

### Monitor back-off at human gates

A monitor task exists to advance work. When **every** task in a repo's monitor set is parked on a
human action — an approval, a merge click, an unanswered question — the monitor has nothing to do, but
a fixed-cadence orchestrator relaunches it every pass anyway. Observed in a live deployment: one
monitor resumed 7 times in 89 minutes against a single task whose only remaining move was a human
clicking Approve; six runs were no-ops, the last four byte-identical.

Two costs: **quota** (agent launches producing nothing, on boxes that go dark under usage caps), and
**signal corruption** — monitor tasks run with `agent_resume`, so repeatedly resuming one session to
re-confirm "nothing changed" trains the exact "nothing to do" reflex behind real resume-anchoring
incidents, and makes a genuinely inert monitor indistinguishable from a correctly idle one.

- **Do not dispatch a monitor when every task in that repo's monitor set is parked on a human action.**
  Poll the gate each pass with one cheap read instead — a `gh pr view` on the PR plus the source task's
  status / last-activity field. Dispatch the monitor agent only when that read shows movement: a merge
  landing, a newly blocking merge state, the task leaving its parked status, or a new comment. Log
  `PIPELINE_MONITOR_HELD` carrying the compared values, and refresh the monitor task description so the
  comparison is stateless across passes and `last_touched_ai` stays fresh.

- **Declare the eligible set — never infer it.** Each repo's config declares `human_gate_states`
  (Trello deployments: `human_gate_lists`): the exact status values whose only remaining move is a
  human action. **If the field is absent, the back-off does not engage** and the monitor dispatches as
  it does today. Absence means "not yet classified," never "guess." Status vocabularies are not
  portable across deployments: in the Linear flavor `Approved` is a *machine* turn (the pipeline merges
  on it), so an issue resting there is a **bug signal** and must never be gate-eligible — while
  Linear's actual human parking spot, `In Review`, is excluded from discovery and never enters the
  monitor set at all.

- **Keep a liveness floor of one dispatch per day**, satisfied by the day's first pass dispatching the
  freshly created daily task. Do not use a sub-daily floor: monitor tasks run with `agent_resume`, so
  repeatedly resuming a session that has nothing to do is itself the resume-anchoring failure — a
  byte-identical monitor reply must stay a *signal*, not the expected steady state.

- **Break the hold on age.** If a hold has persisted past `monitor_hold_max_hours` (default `4`),
  dispatch once anyway and log `PIPELINE_MONITOR_HELD_EXPIRED`. Machine failures can leave a task
  parked in a state that only *looks* like a human gate — a plan task stuck `active` after its job
  completed, a manual merge that bypassed closeout — and in those cases the monitor is the thing that
  would have noticed. Without this escape, back-off converts a self-healing stall into a permanent one.

- **Create the daily monitor task fresh; never carry one over.** Detect a carry-over by `created_at`,
  not by title. The daily roll is the only thing that reliably retires an anchored session.

- **Do not trigger dispatch on a page's `updated_at`** — it fires on edits that change nothing
  actionable. Use the newest comment's timestamp instead.

- **Never arm a separate `Monitor` and exit.** A per-repo monitor must resolve its own waits within the
  run: poll `gh pr checks` inline up to its budget, then either complete the hand-off or report the
  outstanding checks explicitly and stop. A monitor that hands off its own waiting has no owner; the
  armed callback dies with the job.

- **Reconcile human comments against current state before any human hand-off status flip.** Before
  moving a source task to a merge/approval hand-off such as `Ready to Merge`, read the newest human
  comment and verify its ask is satisfied by the current head/diff. If the newest human comment is
  newer than the relevant commit or state change and is not visibly answered, treat it as work, not a
  hand-off. Judge by content plus state comparison, never by authorship or board position alone.

- **Back-off applies to monitor dispatch only — never to the orchestrator heartbeat.** Box watchdogs
  key on the `Code Pipeline` heartbeat's job rows and on run-log freshness. The orchestrator must keep
  firing and keep writing its run log every pass, including passes where every repo is held. Raising
  the back-off into the heartbeat would silently blind the dead-man's switch.

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
| `human_gate_states` | optional exact source-system status values where the only remaining move is human; if absent, monitor back-off is disabled |
| `human_gate_lists` | Trello equivalent of `human_gate_states`; exact list names or IDs where the client/human acts |
| `monitor_hold_max_hours` | optional max age for a human-gate hold before dispatching once anyway; default `4` |
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
Applies to (active apps): biztobiz, mi-core, muzebook, flightdesk, cashcast, moceanic-ai, qalatra.com, travel-outlook

Framework/template repos (nestled-based but not pipeline app targets): nestled, nestled-forms, nestled-template, nestled-dev-template

Note: `qalatra` (the desktop app) is NOT nestled-based — it is the `electron` framework. The website is `qalatra.com`, which is nestled.

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
7. Open PR — canonical procedure: `claude_session_create_pr(session_id)` → resolve the PR on GitHub → `flightdesk task update <taskId> --branch ... --pr-url ... --pr-number ...` (+ `--status PR_OPEN` if FD hasn't advanced)
8. `git checkout <base_branch>`

**Migration path steps:**
1. Detect port 5432: `docker ps --filter "publish=5432" --format '{{.Names}}'`
2. Stop anything running (get compose file via `docker inspect`)
3. Start: `docker compose -f {repo}/.dev/docker-compose.yml up -d postgres`
4. Wait: `until docker exec $(docker ps ...) pg_isready -U prisma; do sleep 2; done`
5. Define `LOCAL_DATABASE_URL="postgresql://prisma:prisma@localhost:5432/prisma"` — **never edit `.env`.** Every DB-touching command below is prefixed inline with `DATABASE_URL="$LOCAL_DATABASE_URL"`. **Hard gate: never run Prisma migrate dev/reset against a non-localhost host.**
6. `git checkout <branchName>`
7. `DATABASE_URL="$LOCAL_DATABASE_URL" pnpm prisma migrate dev --name <slug>`
   - If drift detected: `DATABASE_URL="$LOCAL_DATABASE_URL" PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION="Yes" pnpm prisma migrate reset --force` then retry
8. `DATABASE_URL="$LOCAL_DATABASE_URL" pnpm db-update`
9. `DATABASE_URL="$LOCAL_DATABASE_URL" npx nx build api` — verify (inject errors to session if any)
10. Stop Docker: `docker compose -f {repo}/.dev/docker-compose.yml down`
11. `git add -A && git commit -m "chore: add migration and regenerate artifacts" && git push`
12. Inject "please run git pull" → verify `running`
13. Open PR — canonical procedure: `claude_session_create_pr(session_id)` → resolve the PR on GitHub → `flightdesk task update <taskId> --branch ... --pr-url ... --pr-number ...` (+ `--status PR_OPEN` if FD hasn't advanced)
14. `git checkout <base_branch>`

Newer nestled repos also carry a structural guard in `prisma.config.ts` that hard-fails `migrate dev`/`migrate reset` against any non-localhost host. A `BLOCKED: prisma migrate ...` error means the guard fired — fix the URL prefix, never bypass it.

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

## Execution Kickoff — Cloud Prompt Paths (all deployments)

The `--prompt` an orchestrator passes to `flightdesk register` / `claude … --remote` is executed in a
**fresh cloud checkout** whose working directory is the repo root. Any path in that prompt must be
**repo-relative** (e.g. `plans/2026-07-15-….md`), never box-absolute (`$WS/<repo>/…`, `/home/…`,
`~/IdeaProjects/…`). A machine-absolute path does not exist in the cloud environment, so the session
silently finds nothing and produces zero work — the root cause of a DOA kickoff (PIR-206,
2026-07-15, cashcast).

Rule for every flavor's kickoff step: keep box-absolute paths for *local* use only (the `cd` into the
repo, resolving the plan file on disk); strip the `$WS/<repo>/` prefix to the repo-relative tail
before putting a path in any prompt sent to the cloud. Applies to plan references and any other file
the cloud agent is told to open.

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
- `MERGED` comes from FlightDesk's GitHub webhook — never set it by hand.
- `PR_OPEN` is **reported by the pipeline**, not awaited: any time the pipeline opens a PR it runs the canonical Open PR procedure (`flightdesk task update ... --branch --pr-url --pr-number`, then `--status PR_OPEN` only if FD is still at `DISPATCHED`/`IN_PROGRESS`). The webhook is redundant confirmation; it misses PRs often enough to strand tasks. Never write a status that moves FD backwards.
- `DISPATCHED` is ambiguous on its own: it covers "session never started" *and* "session is working and the bridge hasn't caught up." Never resolve that ambiguity from bridge state — see **Liveness & Destructive Actions** in `pipeline-agent.md`. No layer may archive a session/FD task, re-dispatch, or reset a source status to a kickoff state until GitHub has confirmed no branch with commits exists for the task.
- Copilot `status: FAILED` in checks ≠ blocking. Verify with `gh pr view --json reviews` — only block if `state: CHANGES_REQUESTED`.
- SonarQube `PENDING` in FD is a known display bug. Check the SonarCloud proxy for real state.
