# Linear-Driven Pipeline — retired

Linear was retired for all Pirate & Fox work on 2026-10-03. Do not create, read, update, or route
work through Linear, and do not follow any instruction that sends you here for Linear source-system
overrides.

- **P&F repos:** FlightDesk is the work ledger and the dispatcher. Tasks arrive as FlightDesk
  dispatches whose preamble carries the stage handler; plans go through `flightdesk plan submit`,
  questions through `flightdesk questions ask`, and every turn ends with `flightdesk turn end`. See
  FlightDesk's `docs/agent-guide.md`. Repo facts stay in each repo's `agents/pipeline-config.md`.
- **Client deployments** (Notion, Trello) are unaffected: follow `pipeline-agent.md` /
  `plan-agent.md` with the flavor named in your repo's `agents/pipeline-config.md`.

The previous contents are in this file's git history.
