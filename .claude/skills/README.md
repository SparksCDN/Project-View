# SKILLS

## Productivity

General workflow tools, not code-specific.

### User-invoked

Reachable only when you type them (Claude Code: `disable-model-invocation: true`).

- **[handoff](./handoff/SKILL.md)** -- Compact the current conversation into a handoff document so another agent can continue the work.
- **[idek](./idek/SKILL.md)** -- Stop. The user doesn't exactly know where to go or if they're doing the right thing with the current prompt / skill. Help them.
- **[insight-me](./insight-me/SKILL.md)** -- Get relentlessly interviewed about a plan or design until every branch of the design tree is resolved.
- **[teach](./teach/SKILL.md)** -- Teach the user a new skill or concept over multiple sessions, using the current directory as a stateful teaching workspace.
- **[to-questionnaire](./to-questionnaire/SKILL.md)** -- Turn a decision you can't answer alone into a Markdown questionnaire for the one person who can -- filled in async, or together over a meeting.
- **[wait-what](./wait-what/SKILL.md)** -- Fire this the moment a message doesn't land. The agent re-pitches it with the context you're missing, in plain English, using your `CONTEXT.md` vocabulary.

### Model-invoked

Model- or user-reachable (rich trigger phrasing so the model can reach for them).

- **[insighting](./insighting/SKILL.md)** -- Interview the user relentlessly about a plan, decision, or idea until every branch of the design tree is resolved.
- **[writing-for-agents](./writing-for-agents/SKILL.md)** -- Writing documents for agents: skills, AGENTS.md/CLAUDE.md, and any doc an agent reaches by a pointer.

## Engineering

Skills I use daily for code work.

### User-invoked

Reachable only when you type them (Claude Code: `disable-model-invocation: true`).

- **[ask-me](./ask-me/SKILL.md)** — Ask which skill or flow fits your situation. A router over the user-invoked skills in this repo.
- **[insight-with-docs](./insight-with-docs/SKILL.md)** — Insighting session that also builds your project's domain model, sharpening terminology and updating `CONTEXT.md` and ADRs inline.
- **[triage](./triage/SKILL.md)** — Move issues through a state machine of triage roles.
- **[improve-codebase-architecture](./improve-codebase-architecture/SKILL.md)** — Scan a codebase for deepening opportunities, present them as a visual HTML report, then insight through whichever one you pick.
- **[setup-skills](./setup-skills/SKILL.md)** — Configure this repo for the engineering skills (issue tracker, triage labels, domain doc layout). Run once per repo.
- **[to-spec](./to-spec/SKILL.md)** — Turn the current conversation into a spec and publish it to the issue tracker.
- **[to-tickets](./to-tickets/SKILL.md)** — Break any plan, spec, or conversation into a set of tracer-bullet tickets, each declaring its blocking edges — text in a local file, or native blocking links on a real tracker.
- **[implement](./implement/SKILL.md)** — Build the work described by a spec or set of tickets, driving `/tdd` at pre-agreed seams and closing out with `/code-review` before committing.
- **[wayfinder](./wayfinder/SKILL.md)** — Plan a huge chunk of work — more than one agent session can hold — as a shared map of decision tickets on the issue tracker, resolved one at a time until the way to the destination is clear.

### Model-invoked

Model- or user-reachable (rich trigger phrasing so the model can reach for them).

- **[prototype](./prototype/SKILL.md)** — Build a throwaway prototype to answer a design question: a single shareable HTML file for state/logic, or several toggleable UI variations.
- **[diagnosing-bugs](./diagnosing-bugs/SKILL.md)** — Disciplined diagnosis loop for hard bugs and performance regressions: build a feedback loop that goes red on this bug → minimise → hypothesise → instrument → fix → regression-test.
- **[research](./research/SKILL.md)** — Investigate a question against high-trust primary sources and capture the findings as a cited Markdown file in the repo, run as a background agent.
- **[tdd](./tdd/SKILL.md)** — Test-driven development with a red-green-refactor loop. Builds features or fixes bugs one vertical slice at a time.
- **[domain-modeling](./domain-modeling/SKILL.md)** — Actively build and sharpen a project's domain model — challenge terms, stress-test with scenarios, update `CONTEXT.md` and ADRs inline.
- **[codebase-design](./codebase-design/SKILL.md)** — Shared discipline and vocabulary for designing deep modules: small interfaces, clean seams, testable through the interface.
- **[code-review](./code-review/SKILL.md)** — Two-axis review of the diff since a fixed point: **Standards** (does it follow the repo's coding standards, plus a Fowler smell baseline?) and **Spec** (does it faithfully implement the originating issue/spec?), run as parallel sub-agents.
- **[resolving-merge-conflicts](./resolving-merge-conflicts/SKILL.md)** — Work through an in-progress git merge or rebase conflict hunk by hunk, resolving by intent traced to each side's primary source, then finish the operation — never `--abort`.
- **[wizard](./wizard/SKILL.md)** — Generate an interactive bash wizard that walks a human through steps only they can perform: provisioning infrastructure, setting up credentials or CI secrets, walking an unfamiliar third-party dashboard, or running a one-off migration or cutover.
