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
- Hangs or errors → trip the breaker (below)

**2. FlightDesk:** `list_projects({})`
- Errors or times out → trip the breaker (below)

**3. Claude bridge:** `claude_sessions_list({})`
- Returns `"No claude.ai tab is open"` or `"Chrome not connected"` → trip the breaker (below). Without session data the pipeline makes wrong decisions.
- Returns `[]` (empty, **no error**) — this is the trap: the daemon answers cleanly even when the claude.ai tab is stale/logged-out, so "empty" ≠ "healthy + idle." **Discriminator:** if FlightDesk shows any active/`DISPATCHED` task for this repo (or one dispatched in the last ~90 min) but the bridge lists nothing / `get_state` returns "Session not found" for it, the bridge is **DOWN**, not idle → trip the breaker. Only treat `[]` as a genuine idle state when FD also shows nothing active.
- Returns a non-empty list → call `claude_sessions_warm({})` immediately to hydrate all session branch bars (~2.5s per session). Subsequent `get_state` calls will return full branch data.

**Trip the breaker** — when any critical system above is down, halting just this run isn't
enough: the heartbeat re-fires every ~15 min and burns an agent spin-up re-discovering the
same failure. Trip a circuit breaker instead — create a fix task, **pause the pipeline**,
and stop running until a human resets it. Pipeline state lives in the source system / FD,
not in the run, so resuming loses nothing.

1. **Dedup:** if a `Pipeline paused:` task is already open in `{CONFIG.reconnect_task_context}`, just exit (breaker already tripped).
2. **Create one fix task:**
   ```
   create_task({
     title: "Pipeline paused: [system] down",
     description: "[the specific failure + what to check]. Resume: re-enable the `Code Pipeline` heartbeat once fixed.",
     my_priority: 1,
     context: "{CONFIG.reconnect_task_context}",
     inbox: true
   })
   ```
3. **Pause:** disable the `Code Pipeline` Qalatra heartbeat — `update_heartbeat({ id: <heartbeat id>, active: false })`. This call is idempotent: it sets state rather than flipping it, so it is safe to call any number of times and a second disable from a still-in-flight run cannot re-arm an already-disabled heartbeat. A disabled heartbeat spins up **no agent at all** — that's the token saver. **Do not** retry, fall back to git/FD, or assess any session: a blind run produces confident-but-wrong state (the v1 "no branch" misread is the evidence).
4. **Exit the run.**

**Resume:** a human fixes the component, completes the fix task, and re-enables the `Code Pipeline` heartbeat — `update_heartbeat({ id: <heartbeat id>, active: true })`, equally idempotent. (An optional Shi-box watchdog that refreshes a stale claude.ai tab may clear the most common bridge failure and re-enable the heartbeat itself — a convenience on top of the breaker, never a replacement.)

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

**Core principle:** Connect the branch to FlightDesk *before* a PR opens, and **report the PR to FlightDesk the moment the pipeline opens one** (canonical *Stage 3: Open PR*). FlightDesk's GitHub webhook is a redundant confirmation path, not the mechanism the pipeline relies on — a missed webhook must never be able to strand a task. `MERGED` still comes from the webhook.

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
- Session asked a question or hit an unresolvable blocker → **relay the question to where the human is already looking.** If the source system has a conversational blocked state (Linear — see source-system overrides), post the question onto the *issue* (as the pipeline identity) + move it to that state; the Qalatra `update_task({ task_id, task_type: "task" })` inbox task is the **alert**, not where the question lives. Log `STAGE_3_NEEDS_HUMAN`. (A question must never die in a transcript only the bridge can read.)
- Session reports completion without a branch → Session Assessment handler

**Notes:**
- **PR_OPEN is reported, not awaited.** Whenever the pipeline opens a PR it tells FlightDesk directly — see the canonical *Stage 3: Open PR* procedure (`flightdesk task update <taskId> --branch ... --pr-url ... --pr-number ...`, then `--status PR_OPEN` only if FD is still at `DISPATCHED`/`IN_PROGRESS`). The webhook is redundant confirmation, not the source of truth. The same command is the branch-attachment recovery path when FD missed a PR the pipeline didn't open.
- `MERGED` still comes from FD's webhook — do not set it by hand during normal flow.
- Never write a FlightDesk status that moves the task backwards from where FD already is.
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

**Before nudging or escalating, read what the session actually said — and check for produced work.**
Always pair `claude_session_get_transcript({ session_id, last_n: 8 })` with `claude_session_get_state({ session_id })`. The bridge transcript is **eventually consistent** — assistant turns can lag the cloud session by *hours*, so "only user turns visible" / "no assistant turns" is **not** evidence the session is idle, unresponsive, or wedged. Cross-check before acting:
- `branchBar.featureBranch` populated (`additions`/`deletions` > 0) **or** a remote branch exists (`gh api repos/{CONFIG.github_slug}/branches/<branch>`) → **the session has produced work; it is not wedged.** Do not kill, re-dispatch, or raise a "stalled session" alert. Route to PR creation below.
- The last assistant turn names a blocker → respond to **that** blocker. Never re-send a generic "are you stuck?" nudge over the top of an answer you haven't read.

**Branch pushed, no PR → open the PR on-box; never nudge the session to open it.**
If a branch is pushed and no PR exists — **including** when the session reports it is blocked opening its own PR (e.g. its cloud-side GitHub MCP is disconnected, the common case) — PR creation is the pipeline's job: `claude_session_create_pr({ session_id })`. Do **not** inject "open the PR" / "create the PR" to the session (canonical rule — see *Stage 3: Open PR*); the cloud env's git/PR tooling is unreliable and asking it to do the pipeline's job is how a finished session gets misread as stuck. Log `STAGE_3_PR_OPENED_ONBOX`.

**Never kill + re-dispatch while a feature branch with commits exists on the remote** — that discards pushed work and risks two sessions double-pushing the same branch. Recovery = adopt the existing branch (attach + open PR), not restart. Kill + re-dispatch is only for a confirmed-dead session with **no** pushed branch.

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
5. **Inspect the regen diff before committing.** If db-update *reverted* something the agent hand-added to a generated file (e.g. a new `@Field` on a generated model class), do **not** commit the wipe and do **not** keep the hand edit — both lose: committing breaks the feature, keeping it means the next regen silently deletes it. This is an architecture problem, not a codegen artifact: `git checkout` the reverted file, then inject a rework pointing the session at the repo's codegen-safe pattern for computed fields (nestled: an extension resolver — `@Resolver(() => Model)` + `@ResolveField` in a custom plugin, e.g. `user-extension.resolver.ts`). Wait for `ready`, pull, re-run from step 2.
6. `git add -A && git commit -m "chore: regenerate codegen artifacts" && git push`
7. If files were changed → inject: *"I ran pnpm db-update locally and pushed updated generated artifacts to the branch. Please run git pull to sync your working copy."* Then poll `get_state` until session moves to `running`.
   - **CRITICAL:** `injected: true` can be a false positive. Always verify state changes to `running`. If not, retry inject. Do NOT skip inject and call `create_pr` directly.
   - **Do NOT** mention "create the PR" in the inject message — PR is always created via `claude_session_create_pr`
   - If db-update produced no changes → skip inject
8. Open the PR via the canonical **Stage 3: Open PR** procedure (`create_pr` → resolve the PR on GitHub → report branch/PR to FlightDesk with `flightdesk task update`). Never stop at `create_pr`.
9. `git checkout {CONFIG.base_branch}`

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
5. Define the local DB URL — **never edit `.env`**; the ambient env file stays untouched no matter what it points at:
   ```bash
   LOCAL_DATABASE_URL="postgresql://prisma:prisma@localhost:5432/prisma"
   ```
   Every DB-touching command below gets `DATABASE_URL="$LOCAL_DATABASE_URL"` prefixed inline. dotenv does not override already-set env vars, so the inline value always wins over whatever `.env` contains.
6. **Hard gate:** the host in `$LOCAL_DATABASE_URL` must be `localhost` / `127.0.0.1`, and every migrate/db-update/build command below must carry the explicit `DATABASE_URL=` prefix. If a command would run without it → **hard stop**, flag `NEEDS_ATTENTION`. Never run `prisma migrate dev`/`migrate reset` against a remote host.
7. `cd {CONFIG.repo_path} && git checkout <branchName>`
8. `DATABASE_URL="$LOCAL_DATABASE_URL" pnpm prisma migrate dev --name <slug>`
   - Drift detected → `DATABASE_URL="$LOCAL_DATABASE_URL" PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION="Yes" pnpm prisma migrate reset --force` then retry
9. `DATABASE_URL="$LOCAL_DATABASE_URL" pnpm db-update`
10. `DATABASE_URL="$LOCAL_DATABASE_URL" npx nx build api` — verify
    - TS errors → inject to session, wait for `ready`, pull, re-run from step 9
11. If `{CONFIG.sdk_command}` is set → run it (same `DATABASE_URL=` prefix)
12. **Stop Docker:** `docker compose -f {CONFIG.repo_path}/.dev/docker-compose.yml down`
13. `git add -A && git commit -m "chore: add migration and regenerate artifacts" && git push`
14. Inject: *"I ran prisma migrate dev and regenerated artifacts locally and pushed to the branch. Please run git pull to sync your working copy."* → verify `running`
    - Same CRITICAL notes as codegen: verify state change, no PR mention, retry if needed
15. Open the PR via the canonical **Stage 3: Open PR** procedure (`create_pr` → resolve the PR on GitHub → report branch/PR to FlightDesk with `flightdesk task update`). Never stop at `create_pr`.
16. `git checkout {CONFIG.base_branch}`

**CRITICAL:** Never edit `.env` — the explicit `DATABASE_URL=` prefix replaces the old swap/restore approach. Always stop Docker. Never leave a local Postgres running.

**Structural guard (newer nestled repos):** repos carrying the template's `prisma.config.ts` guard hard-fail `migrate dev`/`migrate reset` whenever the resolved `DATABASE_URL` host isn't localhost. If you see `BLOCKED: prisma migrate ...`, the guard fired and saved you — fix the `DATABASE_URL=` prefix on the command; never attempt to bypass or edit the guard.

---

### Stage 3: Open PR — canonical PR-open procedure

**Every** PR the pipeline opens runs this procedure: the direct path (session done, no reconciliation needed), the tail of Codegen Reconciliation (step 8), and the tail of the Migration Path (step 15). Opening a PR is not done until FlightDesk has been told about it.

1. **Open it.**
   ```
   claude_session_create_pr(session_id)
   ```
   Do NOT inject to ask the session to create a PR — it bypasses FlightDesk tracking.

2. **Resolve the PR from GitHub.** The bridge click is asynchronous — never assume it landed:
   ```bash
   gh pr list --repo {CONFIG.github_slug} --head <branchName> --json number,url --limit 1
   ```
   Poll ~15s apart for up to ~2 min. Still empty → log `STAGE_3_PR_NOT_FOUND` and flag `NEEDS_ATTENTION`. Do not re-click `create_pr` blindly, and do not inject "open the PR" to the session.

3. **Report it to FlightDesk immediately — do not wait for the webhook.** This is the step that keeps FD status honest; the GitHub→FD webhook is best-effort and silently misses PRs (fuzzy title matching, missed deliveries), which strands the task at `DISPATCHED`/`IN_PROGRESS` forever.
   ```bash
   flightdesk task update <taskId> --branch <branchName> --pr-url <prUrl> --pr-number <prNumber>
   ```
   Metadata-only, always safe, idempotent — send it whether or not the webhook already fired.

4. **Advance status only if FD hasn't already.** Re-read status (`get_task({ taskId })`, or `flightdesk task status <taskId>`):
   - Still `DISPATCHED` or `IN_PROGRESS` → `flightdesk task update <taskId> --status PR_OPEN`. Log `STAGE_3_PR_OPENED`.
   - Already `PR_OPEN` / `PREVIEW_*` / `REVIEW_*` / `QA_*` / `MERGED` → the webhook beat you; step 3 was enough. Log `STAGE_3_PR_REPORTED`. **Never write a status that moves FD backwards.**

5. Optional, when review state matters right away: `flightdesk task sync <taskId>` pulls Copilot/Claude/human review state from GitHub onto the task.

**Rule:** the pipeline reports PR-open to FlightDesk as a write-through, and treats the webhook as a redundant confirmation — not the source of truth.

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
| Copilot | `integrationSlug: "github-reviews"`, `status: "FAILED"` — **verify** with `gh pr view --json reviews` **and** the `reviewThreads` GraphQL query (isResolved/isOutdated). `CHANGES_REQUESTED` and `COMMENTED` are blocking *only while review threads are still unresolved* — the review object itself stays `COMMENTED` even after every thread is handled, so do not key off the verdict alone or you'll inject forever. Non-blocking when: reviews array is empty, all reviews are `APPROVED`/`DISMISSED`, **or every review thread is resolved** (i.e. addressed + reconciled per Stage 4 Step 6). | Unresolved threads with real, still-present findings → `get_task_prompt({ taskId, promptType: "review" })` → inject. Threads already addressed by the current head → reply + resolve (Step 6), do not inject. |
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

**Step 2 — Read original spec AND the committed plan:**
Fetch the source task (Notion/Linear/Asana) and extract the full task description and any requirements in the page body. **Also read the committed plan** (`git show origin/{CONFIG.base_branch}:agents/plan/plans/<plan-file>.md`, path from the source task's links / kickoff prompt) — the plan's acceptance criteria / Definition of Done is the authoritative scope contract, and is usually more explicit than the issue. If no plan was committed, note it: the author may have reconstructed scope blind, so scrutinize coverage harder below.

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

1. **Scope coverage (enumerate, don't eyeball):** List every discrete requirement from the spec **and the plan** — each acceptance criterion, Definition-of-Done bullet, numbered scope item — and for each, confirm where the diff satisfies it or mark it missing/partial. Treat a missing or half-done requirement as blocking, the same as a bug. A multi-part task (e.g. "fix X **and** add Y") where the diff only covers the headline part is the classic blind-reconstruction miss — the headline ships, the secondary requirement silently drops. Inject the missing items.
2. **Claude Code review:** Does the review flag any bugs, security issues, incorrect logic, missing edge cases, or significant functional gaps? Read the full text — do not rely on the pass/fail status alone. **Verify each flagged item against the *current* branch head before treating it as real.** Long-form reviews (the Claude review especially) are generated against the commit that existed when they ran, and go stale the moment a later commit addresses the finding — GitHub marks such threads `isOutdated`. A finding the current code already handles is *not* a real issue: reply on the thread noting it's already addressed and resolve it (Step 6), do **not** re-inject it into the session.
3. **Unresolved comments:** Are there inline review comments or general PR comments that haven't been addressed in a follow-up commit or reply?
4. **Configured quality gates:** If the repo defines a coverage target such as `new_code_coverage_target: 80%`, do the reports show the target was met for changed/new code? If the target cannot be measured, is that a CI/configuration gap that must be surfaced? If the PR adds new behavior but no tests, require tests unless the config explicitly exempts the task type.

**What to flag:**
- Bugs, security issues, incorrect logic
- Missing required functionality explicitly stated in the spec **or plan** (any unimplemented acceptance criterion / DoD item — partial scope is blocking)
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
- **All good** → Proceed to Step 6, then the Adversarial Verifier, then QA_READY actions below.

---

**Step 6 — Reconcile and resolve every review thread (required before QA_READY):**

Fixing a comment in code is not enough — the pipeline must close the loop *on GitHub* so no thread
is left dangling. This is mandatory, not optional: it's what keeps a Copilot `COMMENTED` review from
blocking forever, and it's the difference between "the code is right" and "the reviewer can see it's
been handled." For **every** open review thread on the PR (Copilot, Claude, other bots, humans):

1. Determine its disposition against the **current branch head**:
   - *Already addressed* — the current code already handles it (the thread is often `isOutdated`
     because a later commit changed the line). Common for the long-form Claude review.
   - *Fixed this pass* — you just injected a fix for it.
   - *Won't fix* — intentionally not actioned (out of scope, incorrect suggestion, by-design).
2. Post **one** reply on the thread stating the disposition in plain terms — what was changed (name
   the mechanism/commit) or why it isn't being actioned.
3. Mark the thread resolved:
   ```bash
   # List open threads + node IDs
   gh api graphql -f query='{ repository(owner:"<owner>",name:"<repo>"){ pullRequest(number:<n>){
     reviewThreads(first:50){ nodes{ id isResolved isOutdated comments(first:10){ nodes{ author{login} body path } } } } } } }'
   # Reply, then resolve (per thread)
   gh api graphql -f query='mutation($t:ID!,$b:String!){ addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$t,body:$b}){ comment{ url } } }' -f t=<threadId> -f b="<reply>"
   gh api graphql -f query='mutation($t:ID!){ resolveReviewThread(input:{threadId:$t}){ thread{ isResolved } } }' -f t=<threadId>
   ```
   (Requires the box's `gh` identity to have triage/write on the repo.)

**A QA_READY handoff requires zero unresolved review threads.** If a thread genuinely can't be
resolved because it needs a *human* decision (not a code fix the agent can make), that is a
`Blocked`-class signal — leave it unresolved, state why in the reply, and follow the source system's
blocked/needs-human path (P&F: move the Linear issue to `Blocked` + inbox alert) rather than
surfacing a half-reconciled PR as ready.

---

### Stage 4: Adversarial Verifier — independent sign-off (required before QA_READY)

The agent that wrote the code must not be the only judge of whether it ships. Everything above is
the *author* assessing its own work — a self-graded gate. After Step 6, before declaring QA_READY,
an **independent verifier with fresh context** must sign off in writing. This is a different failure
mode from the mechanical gates: the gates check *facts* (threads resolved, CI green); the verifier
checks *judgment* (is the fix actually correct, complete, and free of new bugs?). The door is never
the verifier.

**Skip-if-unchanged:** record the verified head SHA with each `MERGE` verdict. If `git rev-parse
origin/<branchName>` still equals the last verified SHA, the diff hasn't changed — reuse the prior
verdict, don't re-run (avoids re-verifying and re-commenting on an unchanged PR every tick).

**Independence is the whole point — run it as a fresh one-shot process, never as in-session
reasoning** (a nested `claude -p`, or an equivalent fresh-context subagent — the required property
is *zero shared context* with the author session; it sees only facts, not the author's narrative):

```bash
HEAD_SHA=$(git rev-parse origin/<branchName>)
SPEC=$(…source task description + body…)
# The committed plan is the authoritative Definition of Done — read it from the base branch, NOT the
# author's narrative. Resolve the plan path from the source task's links / the kickoff prompt.
PLAN=$(git show origin/{CONFIG.base_branch}:agents/plan/plans/<plan-file>.md 2>/dev/null)   # empty if no plan was committed
DIFF=$(git diff origin/{CONFIG.base_branch}...origin/<branchName>)
REVIEW=$(gh api repos/{CONFIG.github_slug}/pulls/<prNumber>/reviews --jq '.[].body')

VERDICT=$(claude -p "You are an INDEPENDENT reviewer. The author of this PR is biased toward
shipping; your job is to find concrete reasons it should NOT merge. Trust nothing in the author's
description — verify every claim against the actual diff and current code. Decide MERGE or
NEEDS_WORK and return ONLY JSON:
{\"verdict\":\"MERGE|NEEDS_WORK\",\"blocking\":[{\"file\":\"\",\"line\":0,\"issue\":\"\",\"evidence\":\"\"}],\"summary\":\"\"}

Check, against the CURRENT head only:
1. SCOPE COVERAGE (do this first, explicitly). Enumerate EVERY discrete requirement in the SPEC and
   PLAN — each acceptance-criterion, Definition-of-Done bullet, and numbered scope item — and for each
   one, point to where the DIFF satisfies it or mark it MISSING/PARTIAL with evidence. A spec/plan
   with N requirements where the diff implements only some is NEEDS_WORK — **partial scope is a
   blocking failure, not a nit.** Authors that reconstructed scope without the plan (e.g. inferred it
   from the branch name) routinely ship the headline fix and silently drop a secondary requirement —
   a requested toggle, a second view, an acceptance criterion. If PLAN is empty AND the SPEC implies
   more than the diff delivers, flag the missing scope and that no plan was committed to verify against.
2. Real bugs, security holes, broken edge cases, or data-loss risk introduced by THIS diff.
3. Claimed fixes — actually present in the current code, or merely asserted?
4. Any Copilot/Claude review finding genuinely still unaddressed in the current head (ignore ones
   the current code already handles — those are stale).
Ignore style/nits. Only list issues you can point to with file + concrete evidence. None → verdict=MERGE.

SPEC:
$SPEC

PLAN (authoritative Definition of Done; if empty, no plan was committed — scrutinize scope harder):
$PLAN

DIFF:
$DIFF

REVIEW FEEDBACK (cross-check only; may be stale):
$REVIEW")
```

**Sign-off is written evidence** (this is what makes the verdict auditable and seeds the
trust-architecture track record): post the verifier's `summary` + verdict as a comment on the source
task/issue (per source system — Linear `commentCreate`, Notion comment, etc.), clearly in the
verifier's voice (prefix `🔍 Independent verification — <verdict>:`). Then record `HEAD_SHA` as the
verified SHA on the task.

**Gate:**
- `verdict: MERGE` → proceed to the QA_READY actions below.
- `verdict: NEEDS_WORK` → inject the `blocking` list into the author session (each item with its
  file + evidence). Do **not** change FD status — the fix produces new commits, checks re-run, and
  the verifier runs again on the new head. Same loop as Intelligence Check findings. If the verifier
  and author ping-pong on the same point across 3 cycles without converging, that's a `Blocked`-class
  signal — stop and route to the human (P&F: `Blocked` + inbox alert) rather than looping.

**Risk-tiered per repo via `{CONFIG.auto_merge}` — and graduation runs in *reverse* of the usual
trust model.** These are mostly greenfield / internal / pre-launch repos, so a bad merge costs
almost nothing: the default is **auto-merge on a `MERGE` verdict** — the verifier is the gate, cheap
and fast. A repo does **not** earn its way *out* of human review; it graduates *into* mandatory human
review when its stakes rise (real user volume, revenue-bearing, production-critical), at which point
you deliberately flip `auto_merge: false` and pay for a human gate to buy down financial risk. The
verifier's per-repo track record (and incident history) is the signal for *when* a repo has crossed
that line (see trust-architecture).

**Scope:** "verifier `MERGE` = approval, merge on it" is the **Linear flavor's** model
(`linear-pipeline.md`). Other deployments keep their own approval gate even when `auto_merge: true`:
Monroe is human-merge, and Biz to Biz auto-merges only on its `Ready for Production` status — the
verifier is an *added* quality gate for them, **not** the merge trigger. The executable rule is
Stage 4b step 1: skip the approval check only for Linear `auto_merge: true`; every other deployment
reads its configured approval status.

---

**When Intelligence Check passes (mechanical gates green + zero unresolved threads + verifier `MERGE`) → QA_READY:**

> **Does your source-system flavor doc override QA_READY routing?** Only the **Linear flavor with
> `auto_merge: true`** does (`linear-pipeline.md` → Merge Policy): there the verifier `MERGE` verdict
> *is* the approval, so **skip steps 3–5 below** (no `ready_for_testing` write, no review task), go
> straight to Stage 4b, and keep the issue in the work-in-progress state — never a review state — if
> the merge can't complete this pass. **Every other deployment uses the numbered steps below
> unchanged**: Notion/Monroe and Biz to Biz both still write `ready_for_testing` and wait for their
> configured approval status. Do not skip unless your own flavor doc explicitly says so.

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
   **Skip this check only for the Linear flavor with `auto_merge: true`** (`linear-pipeline.md`),
   where the verifier `MERGE` already *is* the approval → proceed to step 2. **All other deployments
   do this check** — Notion/Monroe, Biz to Biz (whose gate is its `Ready for Production` status), and
   any `auto_merge: false` repo: fetch source task and read the approval field. Compare to
   `{CONFIG.source_status_approved}`.
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

5. **Ship-log entry (every flavor — always):** a merge to the base branch *is* the deploy, so this is
   the moment something went live. **Always** append one line to the ship log in **the agent workspace
   this pipeline runs under** — `$WS/<workspace>/briefs/shipped/YYYY-MM.md` (current month), where
   `<workspace>` is this box's own projects-style workspace: `projects` (P&F), `mi-projects` (Monroe),
   `moceanic-projects` (Moceanic), `biz-projects` (Biz to Biz). This is the same workspace whose
   `briefs/` the box's other agents write to; if unsure, it is the workspace the orchestrator resolved
   `$WS` from. Read-modify-write, **newest first**: group entries under a `## YYYY-MM-DD` header (create
   the header if today's isn't already at the top; create the file with a `# Shipped — YYYY-MM` title if
   absent). One line per merge, in this exact shape:
   ```
   - `HH:MM` · **<repo>** · <issue id> <issue title> · [PR #<n>](<pr url>) · verifier: <MERGE|n/a> · <auto-merged|human-approved>
   ```
   Merge time in 24h local; `verifier:` = the adversarial verifier's verdict at QA_READY (`n/a` if none
   ran); `auto-merged` when it merged on verifier/`Approved` sign-off without Justin's manual review,
   `human-approved` when Justin set `Approved`. This is the compensating control that makes default
   auto-merge safe (after-the-fact visibility), **and** the primary source for the nightly fleet digest
   each box emails to Shi (`shi/tools/fleet-alerting`) — never skip it on a successful merge. See
   `linear-pipeline.md` for any P&F-specific notes.

6. **Complete Qalatra task:** `complete_task(qalatra_task_id)`

Log: `STAGE_5_DONE`

---

## Source System Updates

Mid-pipeline writes to keep the team's task board in sync.

### When to run

| Trigger | Action |
|---|---|
| Stage 3 entry: FD URL confirmed in `task.links` | Write FD URL to `{CONFIG.source_field_flightdesk_url}` |
| Stage 3: PR open | Write PR URL to `{CONFIG.source_field_github_url}` (if configured) |
| Stage 4 all-green (`QA_READY`) | Set `{CONFIG.source_field_status}` to `{CONFIG.source_status_ready_for_testing}`. **Skip only for the Linear flavor with `auto_merge: true`** — it merges straight to `Done` and must never write a review state (`In Review` is excluded from Linear discovery, so writing it orphans the issue — see `linear-pipeline.md`). Notion/Monroe and Biz to Biz still write it. |
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
