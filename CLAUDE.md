# CLAUDE.md — App Portfolio cooperation rules

Governs how work proceeds on Jira project **PA** (`Portfolio Apps`, `https://lukaszkot03.atlassian.net`). Applies to this repo and to `platform`. Merge with any global CLAUDE.md instructions; where they conflict, the more specific/recent instruction from the user wins.

## Tooling

Specific tools to reach for instead of guessing or scraping documentation — a real amount of the prototype build's pain came from exactly that (six rounds of Terraform trial-and-error that a schema lookup would have caught the first time):

- `mcp__azure__azureterraform` — real AzureRM/AzAPI provider resource schemas before writing any Terraform. Use this, not WebSearch/WebFetch against scraped docs.
- `mcp__azure__azureterraformbestpractices` / `mcp__azure__get_azure_bestpractices` — call before generating any Terraform/Bicep for a given Azure service.
- `mcp__azure__deploy` — deployment plan / CI-CD pipeline / architecture guidance with Azure-specific constraints.
- `mcp__azure__advisor` — real cost/optimization recommendations against the actual subscription; check periodically given how cost-conscious this project is.
- `context7` — current library docs, used _before_ writing code against the MCP C# SDK, Microsoft Graph SDK, MSAL.NET (Entra External ID), or EF Core — these move faster than training-data knowledge stays current.
- `mssql` MCP — schema inspection and query/index analysis once the real Azure SQL database exists.
- `playwright` MCP — the actual runtime for UI end-to-end tests, not a hand-rolled alternative.
- **GitHub Dependabot** (a repo setting, not an MCP) — enabled on both repos; pairs directly with the very-high-autonomy package-upgrade tier below — it opens the PR, an agent just verifies tests pass and merges.

## C4 diagrams (Structurizr)

- `workspace.dsl` (Structurizr DSL) is the version-controlled source of truth for this repo's C4 diagrams — same philosophy as ADRs: text, in the repo, not a diagram tool's proprietary state.
- View/edit locally via **Structurizr Lite** (free, self-hosted, Docker) — it also renders `docs/adr/`.
- This repo is **private**, so its diagrams are viewed locally only — no Structurizr cloud workspace.
- **Whenever a ticket's architecture step introduces a new container, external system integration, or significant component, update** `workspace.dsl` in the same PR. Same discipline as ADRs — not written once and left stale.

## Ticket structure

- Jira **epics are containers only** — work is never executed at the epic level. Each epic gets broken into **stories/tasks** (and separately, **bugs**, always their own ticket even if found mid-work on something else). One PR per story/task/bug-level ticket, never one PR per epic.
- Issue types (the Jira site is in Polish, so its names are in parentheses):
  - **Epic** (`Epik`) groups related features. Every epic and its children carry the repo name as a label: `platform`, or the app's repo (e.g. `email-triage`, `meal-math`).
  - **Story** is one feature, implemented end to end.
  - **Task** (`Zadanie`) is a technical problem or prerequisite.
  - **Bug** (`Błąd w programie`) is always its own ticket.

## Branching and commits

- Branch naming: `<type>/PA-<n>-<short-slug>`, where `<type>` is `feature`, `task`, or `fix` (matching the ticket type).
- Every commit message references its ticket key, e.g. `PA-123: add Rule entity`.

## The per-ticket cycle

Each story/task follows: **architecture → tests (red) → implementation → tests (green) → integration → deployment**.

- The **red** step is pushed as its own real commit — CI actually runs and fails before the implementation commit follows. Tests written first and only pushed once green is not sufficient; the red state must be visible.
- **Architecture** step: if a non-trivial decision gets made, write or update an ADR in this repo's `docs/adr/` (Nygard format: Title/Status/Context/Decision/Consequences, numbered `0001-...`). Not every ticket produces an ADR — routine implementation following an already-decided architecture doesn't need one. Write this up as an ADR in `docs/adr/` before implementing if any of it turns out non-obvious in practice, **and update** `workspace.dsl` if this introduces a new container, external integration, or significant component.

## Autonomy and merging

Per-task autonomy tiers (from the portfolio architecture plan):

- **High/very-high autonomy** (CRUD-like work, dependency/package upgrades): the executing agent may **self-merge its own PR once CI is green** — no need to wait for explicit approval on these.
- **Low autonomy** (domain modeling, anything touching infrastructure/cost/destructive operations): **always requires explicit human approval** before merge, regardless of how confident the implementation looks.

## ADRs

- Any agent (or the user) may **propose** an ADR (status: Proposed).
- **Only the user can mark an ADR Accepted.** This applies uniformly — even ADRs about high-autonomy topics (e.g. a dependency-upgrade rationale) still need explicit sign-off before they're final. Proposing one doesn't block continued work if the decision itself isn't blocking; get to Accepted before treating the decision as settled precedent for later work.

## Ambiguity

If a ticket's acceptance criteria turn out ambiguous partway through implementation: **pause and ask**, don't guess and continue. This is a deliberate choice while the working patterns here are still new — revisit toward "documented assumption, keep going" once there's a track record to justify it.

## Scope discipline

If something broken or missing outside the current ticket's scope is noticed while working: **file a new ticket, stay in lane.** Don't bundle unrelated fixes into the current PR, no matter how small they seem — matches the existing preference for surgical, scoped changes.

## Jira hygiene

- **Auto-transition ticket status** (To Do → In Progress → Code Review → Ready to Deploy → Done; in Jira: `Do zrobienia` → `W toku` → `Code Review` → `Ready to Deploy` → `Gotowe`) as work actually progresses — don't leave the board stale for the user to update by hand. Pick transitions by their **target status**, not their name: in PA some transition names don't match where they lead (e.g. `In Review` → `Gotowe`).
- **Always post a completion comment** on the ticket when done: what was decided, a link to the PR, a link to any ADR produced. The ticket should be a self-contained record even for someone who never opens the PR.

## Usage budget

Claude Code usage is rationed across three competing needs: daily personal use, new portfolio development, and (once shipped) maintenance of live apps. Don't dispatch or run autonomous work at maximum achievable speed — pace it deliberately, prefer fewer well-scoped runs over many small ones, and treat this as an ongoing constraint, not just a build-phase concern.

## Secrets and infrastructure

- No secret **values** ever committed — Key Vault (or equivalent) holds actual credentials; repos hold only references/config.
- **Terraform runs only via CI** (GitHub Actions, OIDC-authenticated), never locally against real infrastructure — this was an explicit, repeated preference established while building the prototype this portfolio replaces, and applies to both `platform` and `email-triage`.
- Anything that provisions, modifies, or could destroy cloud resources is low-autonomy by definition (see above), regardless of how it's tagged elsewhere.

## Tempo

Everything above is copied verbatim from `platform`'s `CLAUDE.md` (see `platform` ADR-0001). This section is specific to the `tempo` repo.

- **Jira**: project `PA`, label `tempo`. Epics PA-160..PA-166; tickets `[T1]`..`[T30]` are PA-167..PA-196.
- **Branches**: `task/PA-<n>-<slug>`, `feature/PA-<n>-<slug>` or `fix/PA-<n>-<slug>`.
- **Design spec**: `docs/design-spec.md` is the source of truth for what Tempo does. It's exported from the Tempo design spec Claude Doc (https://claude.ai/artifact/MDx72A9d9dSvfm5hi4cEkT); re-export it in a PR whenever the doc changes.
- **Visibility**: this repo is **public**. Statements above that describe `platform` (e.g. "This repo is private" in § C4 diagrams) apply to `platform`, not here. Never commit secret values, tenant-internal config beyond public identifiers, or personal data.
