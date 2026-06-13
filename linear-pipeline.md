# Linear-Driven Pipeline — Pirate & Fox Internal Projects

The Linear lifecycle layer for the P&F internal code pipeline. Per-repo agents fetch this
alongside the canonical `pipeline-agent.md` / `plan-agent.md` (which own FlightDesk/session
stage-handler and planning logic — this document only replaces their *source-system* parts).
Repo-specific values live in each repo's `agents/pipeline-config.md` — read that first.

This pipeline runs in **dangerous mode**: specs are pre-vetted by Justin, so there is no
plan-approval gate by default, and approval at review time merges AND deploys (every repo
auto-deploys from its base branch via Railway / CI). The single mandatory human gate is
`In Review → Approved`.

The one exception path is **`Blocked`** — when the pipeline hits something it can't resolve
without a human decision (a red gate from a pre-existing/unrelated issue, a missing credential,
an out-of-scope dependency), it does **not** sit silently in `In Progress`. It moves the issue
to `Blocked`, posts the blocker + the specific question as a comment, and fires a proactive
alert. `Blocked` is a visible "needs you" column with a live comment loop, so the decision
doesn't get buried in a run log.

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
| **In Progress** | pipeline | Per-repo pipeline monitors: FD status, session discovery + branch recovery, PR creation, preview + PR attached to the issue, quality gates (Sonar where wired + Intelligence Check). All green → `In Review`. If a blocker needs a human decision the pipeline **can't** resolve on its own (red gate on a pre-existing/unrelated issue, missing credential/input, out-of-scope dependency), do **not** park here silently → move to `Blocked` (comment + alert). |
| **Blocked** | turn by last comment | Pipeline parked here because it needs a human decision/input. **On entry it must:** (1) post one comment stating the blocker and the specific question/decision, (2) fire a proactive inbox alert (same plumbing as `ROUTE_UNKNOWN`). Then turn-taking by last comment, same as Planning: Shi-authored last comment = waiting on human; human commented last = re-engage — read the decision, act on it (inject into the live session / resume work / open a follow-up), then move to the right next state (`In Progress` if work resumes, `In Review` if it's now ready, `Changes Requested` if the human asked for changes). **Never auto-merge from `Blocked`.** |
| **In Review** | human (Justin) | Skip — Justin tests the preview/PR. He sets `Approved`, or `Changes Requested` with a comment. |
| **Changes Requested** | pipeline | **Pre-merge loop** — read the latest human comment(s), inject into the live session (re-dispatch on the existing branch if it died), comment back when pushed, flip to `In Review`. Repeats. |
| **Approved** | pipeline | **Merge + close out** — merge the PR to the base branch (this IS the deploy: Railway/CI watches the base branch), delete the branch, archive session + FD task, then set `Done` **last**. |
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
- Step 8 equivalent: post the full plan as a **Linear comment** (markdown, one comment).
- Step 9 equivalent: **leave status at `Planning`** — the orchestrator advances it (no `Needs Plan Review` state in dangerous mode; the `plan-gate` label is the opt-in hold).
- Conversational loop: questions are Linear comments; revise the existing plan comment thread incrementally, don't repost the whole plan unless it materially changed.
- Revision mode: if the Qalatra task description says "REVISION of already-deployed work," plan only the delta on a new branch off the current base branch.

## Pipeline-Agent Source-System Overrides (vs canonical pipeline-agent.md)

- Task discovery: the Linear query above (statuses `In Progress`, `Blocked`, `Changes Requested`, `Approved`), filtered to issues whose project routes to *this* repo.
- FD task resolution: the `FlightDesk` attachment on the issue.
- Status writes + comments: GraphQL patterns above. Quality-gate failures inject fix instructions into the session and stay at `In Progress` (no status spam).
- **Blocked handling:** distinguish a *transient* gate failure (the pipeline can fix it itself by injecting instructions — stays `In Progress`) from a *human-decision* blocker (red gate on a pre-existing/unrelated issue, missing credential/input, out-of-scope dependency). On a human-decision blocker:
  1. Move the issue to `Blocked` (`dbacf9b0-d78c-4e44-8f26-aafb19dab41c`).
  2. Post **one** comment stating the blocker + the specific question/decision (no internal provenance — this is Justin-facing).
  3. Fire the proactive alert as a Qalatra inbox task: `create_task({ title: "Blocked: <issue identifier> — <one-line blocker>", description: "<blocker + the decision needed>\n\n<issue url>", context: "coderepos", project: "<repo>", my_priority: 2, inbox: true })`. Dedup on the issue URL first so a still-blocked issue doesn't re-alert every run.
  
  On a later pass, when a human commented last on a `Blocked` issue, read the decision, act on it (inject into the live session / resume / open a follow-up), mark the alert task done, then move to the appropriate next state. **Never auto-merge from `Blocked`.**
- Merge: `gh pr merge <n> --repo <github_slug> --merge --delete-branch`. Merging to the base branch is the deploy — do not run a deploy command unless the repo config specifies one.
- Closeout order: archive session → archive FD task (webhooks usually handle it) → **ship-log entry** → set `Done` last.
- **Ship log (Justin's deploy visibility — Stage 5):** a merge to the base branch is the deploy, so on every closeout append an entry to `$WS/projects/briefs/shipped/YYYY-MM.md` (current month; the file is git-synced to Justin's laptop). Read-modify-write, **newest first**: entries grouped under a `## YYYY-MM-DD` header (create the header if today's isn't already at the top; create the file with a `# Shipped — YYYY-MM` title if absent). One line per merge, in this exact shape:
  ```
  - `HH:MM` · **<repo>** · <PIR-ID> <issue title> · [PR #<n>](<pr url>) · verifier: <MERGE|n/a> · <auto-merged|human-approved>
  ```
  Use the merge time (24h local). `verifier:` = the adversarial verifier's verdict recorded at QA_READY (`n/a` if the repo had no verifier run). `auto-merged` when it merged on `Approved`/verifier sign-off without Justin's manual approval; `human-approved` when Justin set `Approved`. This log is the compensating control for default auto-merge — never skip it on a successful merge. The morning-review agent reads it for the "Shipped Yesterday" digest.
