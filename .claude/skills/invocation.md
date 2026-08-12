# Model-invoked vs user-invoked

Every `SKILL.md` in this repo is a skill. The one axis that splits them is **invocation** -- who can reach it:

- **User-invoked** -- reachable **only by the human typing its name**. The `description` is **human-facing**: a one-line summary read by a person browsing slash-commands. Strip trigger lists ("Use when the user says...").
- **Model-invoked** -- reachable by **model or user**. The `description` is **model-facing** and keeps rich trigger phrasing ("Use when the user wants..., mentions..., asks for...") so auto-invocation fires. The test for whether a skill should stay model-invoked: _could the model usefully reach for this autonomously?_ (Reuse is the reason to extract a skill, not the test.)

Each harness excludes a user-invoked skill from the model's reach in its own way, so nothing but the human can fire it -- no other skill can. A user-invoked skill may invoke model-invoked skills, but it can never reach another user-invoked skill.

## Dependencies between them

Shared reference docs live inside the skill that owns them; other skills reach that material by invoking the skill, not by linking across folders.

## Passive vs active domain work

Merely _reading_ `CONTEXT.md` for vocabulary is a one-line prose pointer, not the `domain-modeling` skill. Only the active build/sharpen discipline (challenge terms, edge-case scenarios, write ADRs, update `CONTEXT.md` inline) is `domain-modeling`.

## Reference

Skills split into Engineering and Productivity disciplines mostly for user readability.

### Engineering

Skills I use daily for code work.

#### **User-invoked** Engineering

- **[ask-me](../skills/engineering/ask-me/SKILL.md)** -- Ask which skill or flow fits your situation. A router over the user-invoked skills in this repo.
- **[insight-with-docs](../skills/engineering/insight-with-docs/SKILL.md)** -- insight session that also builds your project's domain model, sharpening terminology and updating `CONTEXT.md` and ADRs inline.
- **[triage](../skills/engineering/triage/SKILL.md)** -- Move issues through a state machine of triage roles.
- **[improve-codebase-architecture](../skills/engineering/improve-codebase-architecture/SKILL.md)** -- Scan a codebase for deepening opportunities, present them as a visual HTML report, then insight through whichever one you pick.
- **[setup-skills](../skills/engineering/setup-skills/SKILL.md)** -- Configure this repo for the engineering skills (issue tracker, triage labels, domain doc layout). Run once per repo before using the other engineering skills.
- **[to-spec](../skills/engineering/to-spec/SKILL.md)** -- Turn the current conversation into a spec and publish it to the issue tracker. No interview -- just synthesizes what you've already discussed.
- **[to-tickets](../skills/engineering/to-tickets/SKILL.md)** -- Break any plan, spec, or conversation into a set of tracer-bullet tickets, each declaring its blocking edges -- written as text in a local file, or as a native blocking links on a real tracker.
- **[implement](../skills/engineering/implement/SKILL.md)** -- Build the work described by a spec or set of tickets, driving `/tdd` at pre-agreed seams and closing out with `/code-review` before committing.
- **[wayfinder](../skills/engineering/wayfinder/SKILL.md)** -- Plan a huge chunk of work, more than one agent session can hold, as a shared map of decision tickets on the issue tracker -- resolve them one at a time until the way to the destination is clear.

#### **Model-invoked** Engineering

- **[prototype](../skills/engineering/prototype/SKILL.md)** -- Build a throwaway prototype to answer a design question -- a single shareable HTML file for state/logic questions, or several radically different UI variations toggleable from one route.
- **[diagnosing-bugs](../skills/engineering/diagnosing-bugs/SKILL.md)** -- Disciplined diagnosis loop for hard bugs and performance regressions: build a feedback loop that goes red on this bug -> minimize -> hypothesize -> instrument -> fix -> regression-test.
- **[research](../skills/engineering/research/SKILL.md)** -- Investigate a question against high-trust primary sources and capture the findings as a cited Markdown file in the repo, run as a background agent.
- **[tdd](../skills/engineering/tdd/SKILL.md)** -- Test-driven development with a red-green-refactor loop. Builds features or fixes bugs one vertical slice at a time.
- **[domain-modeling](../skills/engineering/domain-modeling/SKILL.md)** -- Actively build and sharpen a project's domain model -- challenge terms against the glossary, stress-test with edge-case scenarios, and update `CONTEXT.md` and ADRs (Architectural Decision Records) inline.
- **[codebase-design](../skills/engineering/codebase-design/SKILL.md)** -- Shared discipline and vocabulary for designing deep modules: a lot of behaviour behind a small interface, placed at a clean seam, testable through that interface.
- **[code-review](../skills/engineering/code-review/SKILL.md)** -- Two-axis review of the diff since a fixed point: **Standards** (does it follow the repo's coding standards, plus a Fowler smell baseline?) and **Spec** (does it faithfully implement the originating issue/spec?), run as parallel sub-agents so neither pollutes the other.
- **[resolving-merge-conflicts](../skills/engineering/resolving-merge-conflicts/SKILL.md)** -- Work through an in-progress git merge or rebase conflict hunk by hunk, resolving by intent traced to each side's primary source, then finish the operation -- never `--abort`.
- **[wizard](../skills/engineering/wizard/SKILL.md)** -- Generate an interactive bash wizard that walks a human through steps only they can perform: provisioning infrastructure, setting up credentials or CI secrets, walking an unfamiliar third-party dashboard, or running a one-off migration or cutover.

### Productivity

General workflow tools, not code-specific.

#### **User-invoked** Productivity

- **[insight-me](../skills/productivity/insight-me/SKILL.md)** -- Get relentlessly interviewed about a plan or design until every branch of the design tree is resolved.
- **[handoff](../skills/productivity/handoff/SKILL.md)** -- Compact the current conversation into a handoff document so another agent can continue the work.
- **[teach](../skills/productivity/teach/SKILL.md)** -- Teach the user a new skill or concept over multiple sessions, using the current directory as a stateful teaching workspace.
- **[to-questionnaire](../skills/productivity/to-questionnaire/SKILL.md)** -- Turn a decision you can't answer alone into a Markdown questionnaire for the one person who can -- filled in async, or together over a meeting. It insights you about the send (who it's for, what you need back), not the subject.
- **[wait-what](../skills/productivity/wait-what/SKILL.md)** -- Fire this the moment a message doesn't land. The agent re-pitches it with the context you're missing, in plain English, using your `CONTEXT.md` vocabulary.

#### **Model-invoked** Productivity

- **[insighting](../skills/productivity/insighting/SKILL.md)** -- Interview the user relentlessly about a plan, decision, or idea until every branch of the design tree is resolved. The reusable interview primitive behind `insight-me`, `insight-with-docs`, `triage`, `wayfinder`, and `improve-codebase-architecture`.
- **[writing-for-agents](../skills/productivity/writing-for-agents/SKILL.md)** -- Writing documents for agents: skills, AGENTS.md/CLAUDE.md, and any doc an agent reaches by a pointer.
