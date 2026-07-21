# Linear-Driven Pipeline — Pirate & Fox Internal Projects

The Linear lifecycle layer for the P&F internal code pipeline. Per-repo agents fetch this
alongside the canonical `pipeline-agent.md` / `plan-agent.md` (which own FlightDesk/session
stage-handler and planning logic — this document only replaces their *source-system* parts).
Repo-specific values live in each repo's `agents/pipeline-config.md` — read that first.

This pipeline runs in **dangerous mode**: specs are pre-vetted by Justin, so there is no
plan-approval gate, and **auto-merge is ON by default for all P&F repos** (they're greenfield /
pre-launch — see reverse-graduation in `pipeline-agent.md`). When the quality gates are green and the
adversarial verifier returns `MERGE`, the pipeline merges to the base branch — which **is** the
deploy (Railway / CI) — and closes the issue, with **no human gate**. Human touchpoints are now
exception-only: `Blocked` (a decision the pipeline can't make) and any repo with `auto_merge: false`
in its own `pipeline-config.md` (flipped to require `In Review → Approved` as it matures). Visibility is after-the-fact via the
ship log → morning digest, not pre-merge review.

The one exception path is **`Blocked`** — when the pipeline hits something it can't resolve
without a human decision (a red gate from a pre-existing/unrelated issue, a missing credential,
an out-of-scope dependency), it does **not** sit silently in `In Progress`. It moves the issue
to `Blocked`, posts the blocker + the specific question as a comment, and fires a proactive
alert. `Blocked` is a visible "needs you" column with a live comment loop, so the decision
doesn't get buried in a run log.

## Merge Policy (per-repo `auto_merge` flag)

**The switch is each repo's `auto_merge` flag in its own `pipeline-config.md` — read it and honor it
directly. There is no central override list.**

- **`auto_merge: true`** → the adversarial verifier's `MERGE` verdict **is** the approval. The
  pipeline merges and closes the issue itself, straight from `In Progress`, never stopping at
  `In Review`. This is the default for these greenfield / pre-launch repos.
- **`auto_merge: false`** → human gate: at QA_READY the pipeline sets `In Review` and waits for
  Justin to set `Approved` before merging.

**Current state:** every P&F repo is `auto_merge: true`. (Monroe and Biz to Biz are separate
deployments with their own flavor docs — not governed here.)

Reverse-graduation knob: a repo earns its way *into* human review as its stakes rise (real user
volume, revenue-bearing, production-critical) — flip **its own** `auto_merge` to `false` in its
`pipeline-config.md`, one at a time. Don't gate repos pre-emptively; the verifier + ship log are the
safety net.

---

## Identity & Access

| Item | Value |
|---|---|
| Agent identity | **Shi** — real Linear member, `shi@pirateandfox.com` |
| Shi user ID | `62e2dc3d-544c-428a-ba8f-f9236b91c16e` |
| Justin user ID | `2765d8ea-bacc-4f97-86ae-01e5314f98b5` |
| API token | `~/.config/qalatra/secrets.md` → line `SHI_LINEAR=` (personal API key of the Shi account — all API actions author as Shi) |
| Team | Pirate & Fox (`PIR`) — `f3a51891-3a53-45f1-9a8c-f14bf79fcb43` |

All Linear access is **GraphQL via curl** (headless-safe — never depend on the claude.ai Linear connector):

```bash
KEY=$(grep "^SHI_LINEAR" ~/.config/qalatra/secrets.md | head -1 | cut -d= -f2)
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $KEY" -H "Content-Type: application/json" \
  -d '{"query":"<QUERY>"}'
```

## Workflow State IDs (team Pirate & Fox)

| State | ID | Type |
|---|---|---|
| Backlog | `5c3c23c6-2d62-48ab-8d2d-e70fc57e5007` | backlog |
| Todo | `a17fa00f-576e-48f8-a79b-f4e21f5c03ec` | unstarted |
| Planning | `22a53e52-abea-465b-9383-50cb3068a29c` | started |
| In Progress | `d31fb44f-8e14-40c2-a958-1b5a95af7f51` | started |
| Blocked | `dbacf9b0-d78c-4e44-8f26-aafb19dab41c` | started |
| In Review | `ed40a74a-1572-4c0c-9ea5-f294ab1cd487` | started |
| Changes Requested | `df08f95c-e834-4c88-a723-046c2d6f4327` | started |
| Approved | `4cf763d6-a815-4027-bbbb-29d1c0ea4a40` | started |
| Done | `55a6ef14-35a9-4721-9847-9813b3b8bd55` | completed |
| Canceled | `9d3fc9fd-d731-4b5e-867c-7093fb0a2c23` | canceled |

## Lifecycle (single status field — Linear allows one assignee, so turn-taking is by status and last-comment only)

**Assignment model:** assigning an issue to **Shi** = "the pipeline owns it," and Shi stays the
assignee for the entire lifecycle. Linear has no co-assignment — humans are reached by status
(Justin watches In Review / Needs-input comments), not by assignment. Unassigning Shi is the
manual override that pulls an issue out of the pipeline.

| Status | Owner of the turn | Pipeline action |
|---|---|---|
| Backlog | human | Skip — not a pipeline status |
| **Todo** (+ Shi assigned) | pipeline | **Intake** — dedup, create Qalatra plan task, queue planner → `Planning` |
| **Planning** | turn by last comment | Planner drafts; posts the plan as an issue comment. Needs input → ask in a comment and wait (Shi-authored last comment = waiting on human; human-authored last = re-engage planner). Plan complete → **no approval gate**: orchestrator fires `flightdesk register` immediately → `In Progress`. Exception: if the issue has the **`plan-gate` label**, the orchestrator does NOT auto-execute — Justin reviews the plan comment and removes the label to approve (or comments to steer). |
| **In Progress** | pipeline | Per-repo pipeline monitors: FD status, session discovery + branch recovery, PR creation, preview + PR attached to the issue, quality gates (Sonar where wired + Intelligence Check), thread reconciliation, then the **adversarial verifier**. On verifier `MERGE`: **auto-merge repos (default)** → merge to base branch (= deploy) + closeout → `Done` directly (ship-log `auto-merged`), skipping `In Review`/`Approved`; **human-gated repos** → `In Review`. Verifier `NEEDS_WORK` → inject fixes, stay `In Progress`. If a blocker needs a human decision the pipeline **can't** resolve on its own (red gate on a pre-existing/unrelated issue, missing credential/input, out-of-scope dependency), do **not** park here silently → move to `Blocked` (comment + alert). |
| **Blocked** | turn by last comment | Pipeline parked here because it needs a human decision/input. **On entry it must:** (1) post one comment stating the blocker and the specific question/decision, (2) fire a proactive inbox alert (same plumbing as `ROUTE_UNKNOWN`). Then turn-taking by last comment, same as Planning: Shi-authored last comment = waiting on human; human commented last = re-engage — read the decision, act on it (inject into the live session / resume work / open a follow-up), then move to the right next state (`In Progress` if work resumes, `In Review` if it's now ready, `Changes Requested` if the human asked for changes). **Never auto-merge from `Blocked`.** |
| **In Review** | human (Justin) | Only reached by **human-gated repos**, or when a human manually moves an issue here to intervene — auto-merge repos never stop here (a verifier `MERGE` merges straight from `In Progress`). **Excluded from the discovery query on purpose** (it's the human's column), so the pipeline does not act on it — Justin tests the preview/PR and sets `Approved`, or `Changes Requested` with a comment. An auto-merge issue must never be written here by the pipeline. |
| **Changes Requested** | pipeline | **Pre-merge loop** — read the latest human comment(s), inject into the live session (re-dispatch on the existing branch if it died), comment back when pushed, flip to `In Review`. Repeats. |
| **Approved** | pipeline | **Merge + close out** (human-gated path; auto-merge repos merge straight from `In Progress` and never pass through here) — merge the PR to the base branch (this IS the deploy: Railway/CI watches the base branch), delete the branch, write the ship-log entry (`human-approved`), archive session + FD task, then set `Done` **last**. |
| Done | terminal | Skip. **Reopen path:** a human moves a Done issue back to `Todo` with Shi assigned → intake detects prior FD/PR history and re-plans in **revision mode** on a fresh branch off the current base branch. |
| Canceled / Duplicate | terminal | Skip. If an in-flight issue is canceled, the pipeline archives any live session/FD task on its next pass. |

## GraphQL Patterns

**Discovery — all pipeline-owned issues in one query:**
```graphql
{ issues(filter: {
    assignee: { id: { eq: "62e2dc3d-544c-428a-ba8f-f9236b91c16e" } },
    team: { id: { eq: "f3a51891-3a53-45f1-9a8c-f14bf79fcb43" } },
    state: { name: { in: ["Todo","Planning","In Progress","Blocked","Changes Requested","Approved"] } }
  }, first: 100) {
    nodes {
      id identifier title url description
      state { name }
      project { id name }
      labels { nodes { name } }
      attachments { nodes { title url } }
      comments(last: 5) { nodes { body createdAt user { id name } } }
    }
} }
```

**Set status:** `mutation { issueUpdate(id: "<issue id>", input: { stateId: "<state id>" }) { success } }`

**Comment (plan, questions, change notes):** `mutation { commentCreate(input: { issueId: "<issue id>", body: "<markdown>" }) { success } }`

**Attach FlightDesk / Preview / PR URLs:**
```graphql
mutation { attachmentCreate(input: { issueId: "<issue id>", title: "FlightDesk", url: "<fd url>" }) { success } }
```
Use titles `FlightDesk`, `Preview`, `Pull Request`. Read them back from `attachments.nodes` during
discovery — the FlightDesk attachment is the equivalent of Notion's `Agent URL`.

**Turn-taking check (Planning / Blocked / Changes Requested):** read `comments` — if the most
recent comment's `user.id` is Shi's, the pipeline asked and is waiting on a human; if a human
commented last (or no comments), it's the pipeline's turn to re-engage. This same most-recent-author
rule governs all three conversational states.

## Routing

One Linear project = one repo. The orchestrator owns the `project.id → repo` mapping table
(see `business-tools/code-pipeline/AGENTS.md` in the projects workspace). Issues with no project,
or a project not in the table, are skipped with a `ROUTE_UNKNOWN` log and an inbox alert.

## Portable Paths

Pipeline hosts differ (Mac: `~/IdeaProjects`, Linux boxes: `~/workspaces`). Never hardcode either:

```bash
WS="$(cd "$(git rev-parse --show-toplevel)/.." && pwd)"   # from inside any repo
```

The orchestrator (which runs in the projects workspace) resolves `$WS` the same way and builds
`agent_path` values as `$WS/<repo>/agents/plan` etc. Plan-file links stored in Qalatra use the
absolute path **on the box that wrote them** — the orchestrator re-resolves against its own `$WS`
when the path's prefix doesn't exist locally (swap the prefix up to `/<repo>/` with `$WS/<repo>/`).

## Qalatra Conventions

- Plan tasks: `context: "coderepos"`, `project: "<repo>"`, `task_type: "coding"`, `source_url` = the Linear issue URL, `agent_path` = `$WS/<repo>/agents/plan`.
- Dedup before intake: `search_tasks({ query: "<issue url>" })` — active match = already in pipeline.
- Daily monitor task per repo: `Pipeline — <repo> — YYYY-MM-DD`, agent `$WS/<repo>/agents/pipeline`, cleaned up per the Monroe pattern (never complete running/queued; supersede stale days).
- The planner completes its Qalatra task with the plan file path in `links` (and posts the plan text as a Linear comment). The orchestrator reads the link at execution kickoff.

## Planner Source-System Overrides (vs canonical plan-agent.md)

- Ignore `{EXECUTE_AGENT_PATH}` — execution kickoff is the orchestrator's job.
- Step 8 equivalent: post the full plan as a **Linear comment** (markdown, one comment). **Post it via the `SHI_LINEAR` GraphQL/curl `commentCreate` path from Identity & Access — never the claude.ai Linear connector MCP tool.** The connector authors as **Justin**, not Shi; the orchestrator's Planning turn-taking reads the last comment's author (`62e2dc3d…` Shi = pipeline asked, waiting on human), so a planner comment posted as Justin looks like a human reply and mis-fires the next run — a re-engage loop, or (if the plan task is `done`) an auto-kickoff into an auto-merge repo. **Every** Linear write the planner makes (plan, questions, revision notes) goes through the curl path so it authors as Shi — curl only, no connector.
- Step 9 equivalent: **leave status at `Planning`** — the orchestrator advances it (no `Needs Plan Review` state in dangerous mode; the `plan-gate` label is the opt-in hold).
- Conversational loop: questions are Linear comments; revise the existing plan comment thread incrementally, don't repost the whole plan unless it materially changed.
- Revision mode: if the Qalatra task description says "REVISION of already-deployed work," plan only the delta on a new branch off the current base branch.

## Pipeline-Agent Source-System Overrides (vs canonical pipeline-agent.md)

- Task discovery: the Linear query above (statuses `In Progress`, `Blocked`, `Changes Requested`, `Approved`), filtered to issues whose project routes to *this* repo.
- FD task resolution: the `FlightDesk` attachment on the issue.
- Status writes + comments: GraphQL patterns above. Quality-gate failures inject fix instructions into the session and stay at `In Progress` (no status spam).
- **Blocked handling:** distinguish a *transient* gate failure (the pipeline can fix it itself by injecting instructions — stays `In Progress`) from a *human-decision* blocker (red gate on a pre-existing/unrelated issue, missing credential/input, out-of-scope dependency, **or a cloud session that ends `ready` with an unanswered question / unresolved material decision — Stage 3 Session Assessment**). On a human-decision blocker:
  1. Move the issue to `Blocked` (`dbacf9b0-d78c-4e44-8f26-aafb19dab41c`).
  2. Post **one** comment stating the blocker + the specific question/decision (no internal provenance — this is Justin-facing).
  3. Fire the proactive alert as a Qalatra inbox task: `create_task({ title: "Blocked: <issue identifier> — <one-line blocker>", description: "<blocker + the decision needed>\n\n<issue url>", context: "coderepos", project: "<repo>", my_priority: 2, inbox: true })`. Dedup on the issue URL first so a still-blocked issue doesn't re-alert every run.
  
  On a later pass, when a human commented last on a `Blocked` issue, read the decision, act on it (inject into the live session / resume / open a follow-up), mark the alert task done, then move to the appropriate next state. **Never auto-merge from `Blocked`.**
- **QA_READY + merge trigger (auto-merge repos — default, see Merge Policy):** the verifier `MERGE` verdict **is** the approval. **This overrides canonical Stage 4 QA_READY *and* Stage 4b for these repos:** do **not** write `In Review`/`ready_for_testing` and do **not** surface a human review task — go **straight from `In Progress` to merge** (canonical Stage 4b steps 2–7, skipping the step-1 approval check), then set `Done`. ⚠️ **Never route an auto-merge issue to `In Review`.** That state is deliberately excluded from the discovery query (see GraphQL Patterns) because it's the human-gated parking spot — so an auto-merge issue parked there is **orphaned**: nothing picks it back up to merge. If the PR is not `MERGEABLE` this pass (checks still running, `UNKNOWN`), **leave the issue at `In Progress` and retry next pass**; `CONFLICTING` → `Blocked` + alert. **Human-gated repos** instead set `In Review` at QA_READY and merge only after Justin sets `Approved`. Command: `gh pr merge <n> --repo <github_slug> --merge --delete-branch`. Merging to the base branch is the deploy — do not run a deploy command unless the repo config specifies one.
- Closeout order: archive session → archive FD task (webhooks usually handle it) → **ship-log entry** → set `Done` last.
- **Ship log (Justin's deploy visibility — Stage 5):** performed by **canonical Stage 5 step 5** (now
  unconditional for every flavor) — do **not** write it a second time here. For the P&F/Linear flavor,
  `<workspace>` = `projects`, so the entry lands in `$WS/projects/briefs/shipped/YYYY-MM.md` (git-synced
  to Justin's laptop). The line shape and rules are the canonical ones; the morning-review agent reads
  this file for the "Shipped Yesterday" digest, and it is also the primary source for the nightly fleet
  digest.
