# Code Pipeline Agent — Canonical Workflow

## How to Use This File

You are a **per-project pipeline agent**. Your CLAUDE.md tells you your repo name and config path.

1. Read `{CONFIG.repo_path}/agents/pipeline-config.md`
2. Wherever this document references `{CONFIG.X}`, substitute the value from that config
3. Only process tasks belonging to your repo — never touch tasks from other repos

Refer to `pipeline-architecture.md` if you need to understand the system or decide where a fix belongs.

---

## Self-Training Protocol

This agent improves itself. Apply this on every turn:

1. **Something went wrong or was corrected** → decide which layer it belongs to, then update it there immediately (not at the end of the session)
2. **Universal fix** → update `qalatra-prompts/pipeline-agent.md` and push
3. **Framework fix** → update the framework section in `pipeline-agent.md` and push
4. **Repo-specific fix** → update `{repo}/agents/pipeline-config.md`
5. **Deployment fix** → update the deployment CLAUDE.md

Never fix in a deployment wrapper what belongs in canonical. That's how pipelines diverge.

---

## The Pipeline Lifecycle

Plans are created by PM agents and manually approved by Justin before dispatch. This pipeline agent only handles tasks once they are in execution.

```
[Manual gate] Justin reviews plan, dispatches to execute-plan
[Qalatra] agent_path contains "agents/execute-plan"  → Stage 3: In Flight (from plan)
[Qalatra] agent_path contains "agents/execute-task"  → Stage 2a: In Flight (direct)
```

**Agent path → stage mapping:**

| agent_path fragment | Stage |
|---|---|
| `agents/execute-task` | 2a |
| `agents/execute-plan` | 3 |

**Key data model:**
- Qalatra task `links` contains the FlightDesk task URL: `https://flightdesk.dev/app/tasks/{id}`
- Cloud session ID: regex `session_[A-Za-z0-9]+` from `get_task_notes(task_id)` note body — only place it's stored
- FlightDesk task has `status`, `branchName`, `prUrl`, `prNumber`, `checks`
- claude-bridge session has `state`, `branchBar.featureBranch`, `branchBar.prState`, `branchBar.prUrl`

---

## Run Loop

### Step 0 — Load Config and Health Check

**First:** Read `{CONFIG.repo_path}/agents/pipeline-config.md`. This file defines all `{CONFIG.*}` values used throughout this document. `{CONFIG.repo_name}` is the short repo name (e.g. `biztobiz`), provided in your CLAUDE.md wrapper.

All three systems must be reachable before the pipeline runs.

**1. Qalatra:** `ping({})` → `{ "ok": true, "ts": "..." }`
- Hangs or errors → create reconnect task, **abort run entirely**

**2. FlightDesk:** `list_projects({})`
- Errors or times out → create reconnect task, **abort run entirely**

**3. Claude bridge:** `claude_sessions_list({})`
- Returns `"No claude.ai tab is open"` or `"Chrome not connected"` → create reconnect task, **abort run entirely**. Without session data the pipeline makes wrong decisions.
- Returns a list → call `claude_sessions_warm({})` immediately to hydrate all session branch bars (~2.5s per session). Subsequent `get_state` calls will return full branch data.

**Creating reconnect tasks** (one per unavailable system, no duplicates):
```
create_task({
  title: "Reconnect pipeline: [system] unavailable",
  my_priority: 1,
  context: "{CONFIG.reconnect_task_context}",
  inbox: true
})
```

**Known bridge error states:**
- `"No claude.ai tab is open"` → claude.ai not open in browser
- `"Chrome not connected — is the extension loaded and a claude.ai tab open?"` → extension disconnected
- `"Session not found"` → bridge up, session already archived (treat as success elsewhere)

### Step 1 — Pull Staged Tasks

Run in parallel — scoped to this repo only:
```
get_tasks_by_agent({ agent: "{CONFIG.repo_name}/agents/execute-plan" })  → Stage 3 tasks
get_tasks_by_agent({ agent: "{CONFIG.repo_name}/agents/execute-task" })  → Stage 2a tasks
```

Using the repo-prefixed fragment ensures this agent only sees its own tasks, never another repo's.

### Step 2 — Classify Tasks

For each task returned, determine stage from `agent_path`. Config is already loaded (Step 0 prerequisite).

### Step 3 — Dispatch Stage Handlers

Execute the handler for each task's stage.

### Step 4 — Log the Run

Write run log to `{DEPLOYMENT_OUTPUT_PATH}/YYYY-MM-DD-HH-MM-pipeline.md` (defined in deployment CLAUDE.md).

---

## Stage Handlers

### Stage 2a: Coder (from task)

Rarely used. Tasks at `agents/execute-task`. Treat identically to Stage 3 — same monitoring, same handlers, same log codes.

---

### Stage 2b: Coder (from plan) — TODO

Tasks where plan is written and approved, waiting for execution dispatch. No automated action yet. Log `STAGE_2B_PENDING`.

---

### Stage 3: In Flight

**Status:** ACTIVE.

**Core principle:** Connect the branch to FlightDesk *before* a PR opens. Once connected, FlightDesk auto-detects PR_OPEN and MERGED via GitHub webhooks.

**For each task:**
1. Parse FlightDesk task ID from `task.links`
2. Write FlightDesk URL to source task (see Source System Updates — idempotent)
3. `get_task({ taskId })` → FlightDesk status
4. Resolve session ID:
   - Prefer `get_task_notes({ task_id })` → regex `session_[A-Za-z0-9]+`
   - If the FlightDesk task has a session URL / teleport ID, parse `session_[A-Za-z0-9]+` from that
   - If MCP data omits the session but the CLI shows it, run `flightdesk task status <taskId>` and parse the `Session:` or `Resume:` line
5. `claude_session_get_state({ session_id })` → session state

**Pulled-back task detection:** Before flagging any task as `NEEDS_ATTENTION` for a missing FD link or session, check the source task's workflow status (if `source_url` exists). If the status indicates the task has been pulled back for rethinking (e.g. `Backlog`, `Needs Discussion`, `Needs Estimate`, `Planning`), log `STAGE_3_PULLED_BACK` and skip silently.

**Branch attachment recovery:** FlightDesk sometimes fails to attach a branch automatically because its fuzzy task-title matching misses the Claude-generated branch name. Do not treat `DISPATCHED` + missing `branchName` as inactive if the Claude session already has a branch.

When FlightDesk status is `DISPATCHED` and `branchName` is empty:
1. Resolve the session ID from FlightDesk/Qalatra as above
2. `claude_session_get_state({ session_id })`
3. If `branchBar.featureBranch` is empty → wait (`STAGE_3_WAIT`)
4. If a branch exists, check for a PR:
   ```bash
   gh pr list --repo {CONFIG.github_slug} --head <branchName> --json number,url --limit 1
   ```
5. If no PR exists → attach the branch:
   ```bash
   flightdesk task update <taskId> --status IN_PROGRESS --branch <branchName>
   ```
   Log `STAGE_3_CONNECTED`
6. If a PR exists → attach branch + PR:
   ```bash
   flightdesk task update <taskId> --status PR_OPEN --branch <branchName> --pr-url <prUrl> --pr-number <prNumber>
   ```
   FlightDesk will sync review state and may advance to `REVIEW_RUNNING`. Log `STAGE_3_BRANCH_RECOVERED`

**State machine — first matching condition wins:**

| FlightDesk status | Condition | Action |
|---|---|---|
| `DISPATCHED` | No branch on session | Wait — session still starting. `STAGE_3_WAIT` |
| `DISPATCHED` | Branch exists, no PR | **Connect:** `update_task_status(taskId, "IN_PROGRESS", branchName)`. Log `STAGE_3_CONNECTED` |
| `DISPATCHED` | Branch + PR both exist | **Recovery** (pipeline was offline): `update_task_status(taskId, "PR_OPEN", branchName, prUrl, prNumber)`. Log `STAGE_3_RECOVERY` |
| `IN_PROGRESS` | Session `running` | Working — wait. `STAGE_3_WAIT` |
| `IN_PROGRESS` | Session `ready` | Apply Session Assessment handler |
| `PR_OPEN` or `PREVIEW_STARTING` | — | Transitional — wait next run |
| `PREVIEW_READY` | — | Apply PREVIEW_READY override handler |
| `REVIEW_RUNNING` | — | Stage 4 |
| `REVIEW_DONE` | — | Needs human judgment. `STAGE_4_NEEDS_HUMAN` |
| `QA_READY` | — | Run Intelligence Check. If issues found → inject to session (do NOT change FD status — new commits will naturally reset the check cycle). If all good → update source task, log `STAGE_4_READY`. |
| `QA_CHANGES_REQUESTED` or `QA_APPROVED` | — | Justin's hands — `STAGE_4_WAIT` |
| `MERGED` | — | Stage 5 |
| `ARCHIVED` | — | Skip |

**PR existence check** (when branch exists but status is DISPATCHED):
```bash
gh pr list --repo {CONFIG.github_slug} --head <branchName> --json number,url --limit 1
```
If PR returned → recovery path. If empty → normal connect path.

**PREVIEW_READY override handler:**
FD can get stuck at `PREVIEW_READY` when SonarCloud passes on first run (webhook never fires). Check GitHub:
```bash
gh pr view <prNumber> --repo {CONFIG.github_slug} --json state,statusCheckRollup,reviews
```
- All green (SonarCloud SUCCESS, CI SUCCESS, no Copilot CHANGES_REQUESTED) → `update_task_status(taskId, "QA_READY")` + update source task to ready-for-testing. Log `STAGE_4_READY`.
- Checks still pending/running → wait. `STAGE_3_PR_OPEN`.
- Check failed → Stage 4 logic.

**DISPATCHED — session has no branch, state is `ready`:**
Read transcript: `claude_session_get_transcript({ session_id, last_n: 8 })`
- Session asked a question or hit an unresolvable blocker → `update_task({ task_id, task_type: "task" })` (surfaces in inbox for Justin) + log `STAGE_3_NEEDS_HUMAN`
- Session reports completion without a branch → Session Assessment handler

**Notes:**
- Never call `update_task_status` for `PR_OPEN` or `MERGED` during normal flow — FD auto-detects via webhook. The exception is manual branch attachment recovery when FD missed the branch/PR; use `flightdesk task update ... --status PR_OPEN --branch ... --pr-url ... --pr-number ...`.
- If `get_task_notes` has no session ID → `STAGE_3_NO_SESSION`, do not guess
- If FD task ID missing from `task.links` → check source before flagging (pulled-back detection)

---

### Stage 3: Session Assessment

When session state is `ready`, read the transcript:
```
claude_session_get_transcript({ session_id, last_n: 8 })
```
Ignore accidental user messages or `[Request interrupted by user]` at the end.

**Decision tree:**

0. **Session `ready`, no branch, last message mentions missing plan file** → Missing plan file recovery:
   - Verify plan file path from `task.links` exists on disk
   - If missing: check `task.ai_context` for sufficient context (task description, file paths, what to change)
   - If `ai_context` has enough → inject plan details directly to session. Log `STAGE_3_PLAN_INJECTED`
   - If `ai_context` empty or insufficient → log `STAGE_3_NEEDS_PLAN`, flag for human

1. **Session reports still working / waiting for input** → `STAGE_3_WAIT`, surface in Needs Attention

2. **Session reports completion** → check branch diff to determine reconciliation path

3. **Session output doesn't match original spec** → inject for clarification or flag `NEEDS_ATTENTION`

4. **Session hit an unresolvable error** → flag `NEEDS_ATTENTION`

**Branch diff check:**
```bash
git fetch origin <branchName>
git diff origin/{CONFIG.base_branch}...origin/<branchName> --name-only
git diff origin/{CONFIG.base_branch}...origin/<branchName> -- schema.prisma
```

**Reconciliation path decision (nestled framework):**

| Condition | Path |
|---|---|
| No generated files, no schema changes | Open PR directly |
| Generated files hand-edited OR schema annotation-only changes (`///` lines only) | Codegen reconciliation |
| Schema structural changes (new/removed model or field) | Migration path |

**Non-nestled frameworks:** See framework section in `pipeline-architecture.md`. If the repo's framework has no defined reconciliation path, open PR directly (no code generation needed).

---

### Stage 3: Codegen Reconciliation — nestled framework

For repos where `{CONFIG.framework}` = `nestled` and the diff shows generated file edits or annotation-only schema changes.

1. `cd {CONFIG.repo_path} && git checkout <branchName>`
2. `pnpm db-update`
3. `npx nx build api` — verify compilation
   - If TS errors in agent-written code → inject error to session, wait for `ready`, pull, re-run from step 2
4. If `{CONFIG.sdk_command}` is set → run it (e.g. `pnpm sdk`)
5. `git add -A && git commit -m "chore: regenerate codegen artifacts" && git push`
6. If files were changed → inject: *"I ran pnpm db-update locally and pushed updated generated artifacts to the branch. Please run git pull to sync your working copy."* Then poll `get_state` until session moves to `running`.
   - **CRITICAL:** `injected: true` can be a false positive. Always verify state changes to `running`. If not, retry inject. Do NOT skip inject and call `create_pr` directly.
   - **Do NOT** mention "create the PR" in the inject message — PR is always created via `claude_session_create_pr`
   - If db-update produced no changes → skip inject
7. `claude_session_create_pr(session_id)`
8. `git checkout {CONFIG.base_branch}`

**NX daemon troubleshooting:** If `nx build` hangs: `pkill -f "nx serve api"; pkill -f "nx daemon"; npx nx reset`

---

### Stage 3: Migration Path — nestled framework

For repos where `{CONFIG.framework}` = `nestled` and the diff shows structural schema changes.

Docker credentials (all nestled projects): user=`prisma`, password=`prisma`, db=`prisma`

1. Detect port 5432: `docker ps --filter "publish=5432" --format '{{.Names}}'`
2. If something running → get compose file: `docker inspect <name> --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}'` → stop it
3. Start target: `docker compose -f {CONFIG.repo_path}/.dev/docker-compose.yml up -d postgres`
4. Wait: `until docker exec $(docker ps --filter "publish=5432" --format '{{.Names}}') pg_isready -U prisma; do sleep 2; done`
   - Collation mismatch fix: `docker exec <container> psql -U prisma -d prisma -c "ALTER DATABASE prisma REFRESH COLLATION VERSION;"`
5. Read `{CONFIG.repo_path}/.env` → check `DATABASE_URL`
   - Already `localhost` → proceed
   - Points to remote → swap env (comment prod lines, uncomment dev lines)
6. **Hard gate:** re-read `DATABASE_URL` — if still not `localhost` / `127.0.0.1` → **hard stop**, flag `NEEDS_ATTENTION`. Never run Prisma migrate against non-localhost.
7. `cd {CONFIG.repo_path} && git checkout <branchName>`
8. `pnpm prisma migrate dev --name <slug>`
   - Drift detected → `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION="Yes" pnpm prisma migrate reset --force` then retry
9. `pnpm db-update`
10. `npx nx build api` — verify
    - TS errors → inject to session, wait for `ready`, pull, re-run from step 9
11. If `{CONFIG.sdk_command}` is set → run it
12. Restore `.env` (comment dev lines, uncomment prod/staging lines)
13. **Stop Docker:** `docker compose -f {CONFIG.repo_path}/.dev/docker-compose.yml down`
14. `git add -A && git commit -m "chore: add migration and regenerate artifacts" && git push`
15. Inject: *"I ran prisma migrate dev and regenerated artifacts locally and pushed to the branch. Please run git pull to sync your working copy."* → verify `running`
    - Same CRITICAL notes as codegen: verify state change, no PR mention, retry if needed
16. `claude_session_create_pr(session_id)`
17. `git checkout {CONFIG.base_branch}`

**CRITICAL:** Always restore `.env` to production-active after the run. Always stop Docker. Never leave a local Postgres running.

---

### Stage 3: Open PR

When session is done and no reconciliation is needed:
```
claude_session_create_pr(session_id)
```
Do NOT inject to ask the session to create a PR — it bypasses FlightDesk tracking. Do NOT call `update_task_status(PR_OPEN)` — FD auto-detects.

---

### Stage 4: Post-PR Review Loop

**Status:** ACTIVE.

Entry: FlightDesk status is `REVIEW_RUNNING` **or** `QA_READY`.

- `REVIEW_RUNNING` — checks are still in progress; pipeline monitors and injects fixes early.
- `QA_READY` — FD auto-advanced via GitHub webhook when all checks passed. **This is the primary Intelligence Check gate.** The pipeline always runs the Intelligence Check here before surfacing to the human. If issues are found, inject to the session and leave FD status unchanged — new commits from the fix will naturally reset FD back through `REVIEW_RUNNING` → `QA_READY`, at which point the Intelligence Check runs again.
- `REVIEW_DONE` → `STAGE_4_NEEDS_HUMAN`, surface for Justin.
- `QA_CHANGES_REQUESTED` or `QA_APPROVED` → Justin's hands, `STAGE_4_WAIT`.

**For each `REVIEW_RUNNING` or `QA_READY` task:**

1. **Check merge conflicts first:**
   ```bash
   gh pr view <prNumber> --repo {CONFIG.github_slug} --json mergeable,mergeStateStatus
   ```
   - `CONFLICTING` → rebase locally (do NOT inject):
     ```bash
     git fetch origin
     git checkout <branchName>
     git reset --hard origin/<branchName>
     git rebase origin/{CONFIG.base_branch}
     # generated files: take --theirs
     git push --force-with-lease
     git checkout {CONFIG.base_branch}
     ```
     Verify `mergeable: "MERGEABLE"` then inject: "The branch has been rebased against {CONFIG.base_branch} and force-pushed. Please run git pull --rebase, then run review checks."
   - `UNKNOWN` → re-check in 10–15s
   - `MERGEABLE` → proceed

2. **Check session state before any inject:**
   `claude_session_get_state({ session_id })`
   - `running` → `STAGE_4_WAIT` — session is already working, skip this tick
   - `ready` or `pr_open` → proceed

3. **Inspect checks:** `get_task({ taskId })` → `checks` array

| Check | Detection | Action |
|---|---|---|
| SonarCloud | `integrationSlug` contains `"sonar"`, `status: "FAILED"` | `get_task_prompt({ taskId, promptType: "review" })` → inject |
| Copilot | `integrationSlug: "github-reviews"`, `status: "FAILED"` — **verify** with `gh pr view --json reviews`. Both `CHANGES_REQUESTED` **and** `COMMENTED` are blocking — `COMMENTED` means open unresolved comments that must be addressed. Only treat as non-blocking if the reviews array is empty or all reviews are `APPROVED`/`DISMISSED`. | `get_task_prompt({ taskId, promptType: "review" })` → inject into session |
| Claude Code review | `integrationSlug: "claude-review"`, `status: "FAILED"` | `get_task_prompt({ taskId, promptType: "review" })` → inject. FD fetches the full comment via its proxy, instructs the agent to address it, and expects `POST /proxy/claude-feedback/acknowledge` when done. Do not bypass this flow. |
| All automated checks passing | No FAILED checks | Run Intelligence Check (see below) |

**FlightDesk `claude-review` behaviour:** FD reads the review body and pattern-matches for approval vs rejection signals to set PASSED/FAILED. When FAILED, `get_task_prompt` includes the full comment fetch and acknowledge flow — use it. When PASSED (Claude's overall verdict was "looks good"), FD injects nothing. The Intelligence Check covers that gap by reading the full review text to catch non-blocking suggestions worth addressing.

**SonarQube PENDING display bug:** If SonarQube is the only check still `PENDING` and all others have settled, check the SonarCloud proxy:
- `qualityGatePassed: true` → treat as passing
- `qualityGatePassed: false` → inject review prompt

---

### Stage 4: Intelligence Check

Runs after all automated checks (SonarCloud, Copilot, CI) are green. This is the substantive gate before declaring QA_READY — it reads everything GitHub has and compares the implementation against the original spec.

**Step 1 — Read all GitHub feedback:**
```bash
# All reviews with full body (includes the long-form Claude Code review)
gh api repos/{CONFIG.github_slug}/pulls/{prNumber}/reviews
# Inline review comments
gh api repos/{CONFIG.github_slug}/pulls/{prNumber}/comments
# General PR comments
gh api repos/{CONFIG.github_slug}/issues/{prNumber}/comments
```
The Claude Code review is typically a review submitted by a bot (e.g. `claude[bot]` or a GitHub Actions actor). Read its full body. At this point the `claude-review` check has PASSED (meaning Claude's overall verdict was "looks good") — but a passing review often still contains suggestions, optimizations, and notes that FD did not inject because they were non-blocking. Read and act on them.

**Step 2 — Read original spec:**
Fetch the source task (Notion/Linear/Asana) and extract the full task description and any requirements in the page body.

**Step 3 — Read repo quality gates:**
Read `{CONFIG.config_path}` / `agents/pipeline-config.md` for configured quality requirements, especially:
- `new_code_coverage_target`
- `coverage_scope`
- `coverage_source`
- `coverage_policy`

If `new_code_coverage_target` is configured, treat it as a pre-QA requirement for changed/new code. Inspect the CI/SonarCloud/coverage results named by `coverage_source` where available. Also inspect the diff for meaningful test additions or updates when the PR adds new logic. The goal is to catch missing tests before a human QA handoff, not to wait for a CI failure that the agent could have avoided.

**Step 4 — Read the branch diff:**
```bash
git fetch origin
git diff origin/{CONFIG.base_branch}...origin/<branchName> --stat
git diff origin/{CONFIG.base_branch}...origin/<branchName>
```
For large diffs, read `--stat` first, then targeted per-file diffs.

**Step 5 — Assess:**
Reason through all inputs together:

1. **Spec vs. implementation:** Does the diff actually address what was asked? Are there explicit requirements in the spec that aren't reflected in the changes?
2. **Claude Code review:** Does the review flag any bugs, security issues, incorrect logic, missing edge cases, or significant functional gaps? Read the full text — do not rely on the pass/fail status alone.
3. **Unresolved comments:** Are there inline review comments or general PR comments that haven't been addressed in a follow-up commit or reply?
4. **Configured quality gates:** If the repo defines a coverage target such as `new_code_coverage_target: 80%`, do the reports show the target was met for changed/new code? If the target cannot be measured, is that a CI/configuration gap that must be surfaced? If the PR adds new behavior but no tests, require tests unless the config explicitly exempts the task type.

**What to flag:**
- Bugs, security issues, incorrect logic
- Missing required functionality explicitly stated in the spec
- Anything flagged in the Claude Code review — optimizations, code quality improvements, edge cases, missing tests, performance concerns, unclear naming, anything the reviewer thought worth mentioning
- Unresolved inline comments or review threads
- Configured coverage targets not met, not measured, or unsupported by the current PR/CI setup
- New or changed logic without corresponding tests when the repo has a new-code coverage policy
- Any suggestion that would make the code meaningfully better, even if not strictly required

**What to skip:**
- Suggestions the session already addressed in a subsequent commit (verify before skipping)
- Pure whitespace or formatting preferences with zero functional or readability benefit
- Duplicate comments that have already been addressed elsewhere in the thread

Default to flagging. This runs autonomously — there is no cost to one more inject and a better result is always worth it.

**Decision:**
- **Issues found** → Compose a targeted inject listing each issue with its source (e.g., *"The Claude Code review on GitHub flagged a missing null check at `file.ts:42` — please address this. Also, the spec asked for X but the diff only implements Y."*). Do NOT change FD status. When the session pushes fixes, new commits will trigger GitHub checks to re-run and FD will naturally cycle back through `REVIEW_RUNNING` → `QA_READY`. The Intelligence Check will re-run at that point. The session state check in Step 2 prevents duplicate injects if the session is already working.
- **All good** → Proceed to QA_READY actions below.

---

**When Intelligence Check passes (QA_READY):**
1. If `{CONFIG.qa_reviewer_id}` is set → `update_task({ taskId, qaAssigneeId: "{CONFIG.qa_reviewer_id}" })`
2. If `{CONFIG.source_field_preview_url}` is set → `get_preview_status({ taskId })` and write the returned URL to `{CONFIG.source_field_preview_url}` on the source task (same Notion update pattern as other fields). Skip silently if no URL is returned.
3. If `{CONFIG.source_field_status_column}` is set → set it to `{CONFIG.source_status_in_staging}` on the source task.
4. Update source task to ready-for-testing (see Source System Updates)
5. Surface Qalatra task for human review:
   ```
   update_task({ task_id, task_type: "task", agent_path: "", inbox: true, title: "Review: " + original_title, ai_context: "Ready for review — in staging. Preview URL in Notion." })
   ```
   Prepending "Review: " makes it clear this is a review action, not an in-flight coding task. Justin marks it done after reviewing; the client can approve independently via Notion.
6. Log `STAGE_4_READY`

---

### Stage 4b: Approval & Deploy

**Gated by:** `{CONFIG.auto_merge}` = `true`

On every tick, for tasks at `PR_OPEN`, `PREVIEW_READY`, `REVIEW_DONE`, `QA_READY`:

1. **Check approval status in source system:**
   Fetch source task and read the approval field. Compare to `{CONFIG.source_status_approved}`.
   - Not approved → skip, `STAGE_4B_WAITING`
   - Approved → proceed

2. **Verify PR is mergeable:**
   ```bash
   gh pr view <prNumber> --repo {CONFIG.github_slug} --json mergeable,state
   ```
   - `CLOSED` or `MERGED` → already handled, go to cleanup
   - `CONFLICTING` → `STAGE_4B_CONFLICT`, flag for human
   - `UNKNOWN` → skip this tick
   - `MERGEABLE` → proceed

3. **Merge:**
   ```bash
   gh pr merge <prNumber> --repo {CONFIG.github_slug} --merge --delete-branch
   ```

4. **Deploy** (if `{CONFIG.deploy_command}` is set and not `none`):
   ```bash
   {CONFIG.deploy_command}
   ```

5. **Archive cloud session:**
   `claude_session_archive(session_id)`
   - `"Session not found"` → already archived, success
   - `"No claude.ai tab is open"` → bridge down, `STAGE_4B_SKIP_ARCHIVE`, continue

6. **Mark source task complete** (see Source System Closeout)

7. **Complete Qalatra task:**
   `complete_task(qalatra_task_id)`

Log approved+merged: `STAGE_4B_MERGED` | waiting: `STAGE_4B_WAITING` | conflict: `STAGE_4B_CONFLICT`

---

### Stage 5: Post-Merge Cleanup

**Triggered by:** FlightDesk status = `MERGED`

1. **Archive FlightDesk task:** `update_task_status(taskId, "ARCHIVED")`
   FD does not auto-archive on merge.

2. **Delete branch:**
   ```bash
   git push origin --delete <branchName>
   ```
   Already deleted → skip silently.

3. **Archive cloud session:** `claude_session_archive(session_id)`
   - `"Session not found"` → success
   - `"No claude.ai tab is open"` → `STAGE_5_SKIP_ARCHIVE`, continue

4. **Mark complete in source system** (see Source System Closeout)

5. **Complete Qalatra task:** `complete_task(qalatra_task_id)`

Log: `STAGE_5_DONE`

---

## Source System Updates

Mid-pipeline writes to keep the team's task board in sync.

### When to run

| Trigger | Action |
|---|---|
| Stage 3 entry: FD URL confirmed in `task.links` | Write FD URL to `{CONFIG.source_field_flightdesk_url}` |
| Stage 3: PR open | Write PR URL to `{CONFIG.source_field_github_url}` (if configured) |
| Stage 4 all-green (`QA_READY`) | Set `{CONFIG.source_field_status}` to `{CONFIG.source_status_ready_for_testing}` |
| Stage 4b approval + dispatch | Set `{CONFIG.source_field_status}` to `{CONFIG.source_status_dispatched}` |

Both are idempotent — always write.

### Source System Detection

Use `task.source_url`:

| URL pattern | System |
|---|---|
| `notion.so/` | Notion |
| `linear.app/` | Linear |
| `app.asana.com/` | Asana |
| None / `flightdesk.dev/` | Skip |

---

## Source System Closeout (Stage 5)

Parse `task.links` for non-FlightDesk URLs. Identify system by URL pattern. Execute `{CONFIG.source_closeout_action}` as defined in the repo's pipeline-config.md.

If no source link → skip silently.

---

## Logging

### Run Log

```
{DEPLOYMENT_OUTPUT_PATH}/YYYY-MM-DD-HH-MM-pipeline.md
```

```markdown
# Code Pipeline Run — YYYY-MM-DD HH:MM

## Summary
- Tasks found: N
- Actions taken: N
- Needs attention: N

## Task Log
| Task | Stage | Action | Result |
|------|-------|--------|--------|

## Needs Attention
[Tasks in unknown stages, errors, unconfigured repos]
```

### Qalatra Note

If a task_id was provided at dispatch:
```
update_task({ task_id, links: [{ url: "{run_log_path}" }] })
```

---

## Qalatra Features

**`task_type: 'coding'`** — tasks live in Code view, excluded from Priority.
- Move to Code view: `update_task({ task_id, task_type: "coding" })`
- Surface for human review: `update_task({ task_id, task_type: "task" })`
- Agent.config `"coding": true` → Qalatra auto-sets this when the job starts
- Agent.config `"timeout_minutes": 45` → use for per-repo pipeline monitors. The Qalatra default is 15 minutes, which is too short for review-fix loops and multi-task monitor runs.

**Timeout recovery:** If a Qalatra monitor job says `Agent timed out`, check the repo `agents/pipeline/output/` log and the source system before treating the pipeline as failed. If the stage actions completed, update the monitor task snapshot and requeue once so the latest Qalatra job reflects current state. If no output log or source-state change exists, surface for human review.

**MCP server** — supervised by launchd, auto-restarts in 5s.
- Health: `ping({})` → `{ "ok": true, "ts": "..." }`
- Logs: `~/Library/Logs/qalatra-mcp.log`
- Restart: `launchctl kickstart -k gui/$(id -u)/com.qalatra.mcp`

---

## Rules

- Never guess a stage handler — log `NEEDS_HANDLER` and surface
- Never post to Slack or external notifications on routine runs
- Only track tasks in Qalatra/FlightDesk — ignore other team members' branches and PRs
- Read freely; confirm before write actions until a handler is marked ACTIVE
- If Qalatra tools behave unexpectedly, note it rather than silently working around it
- **Project-specific steps: stop and ask if the repo's config doesn't define what to do**
