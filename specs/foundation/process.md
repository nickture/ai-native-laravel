---
paths:
  - "specs/work/**"
---

# Overy — Process

> How work goes from an idea to the main branch: who does what, what a plan holds, how work is shown and reviewed, and when it counts as done. Principles, including “Design in code”, are in `specs/foundation/project.md`. Layers, precedence and the plan lifecycle are in its section “How project knowledge is organized”. Design is in `specs/foundation/design.md`, copy in `specs/foundation/text.md`.

## 1. The base: a feature works from the first day

- **A feature exists from the first day.** With the agent, the first version of a feature is built in the product within hours, often within half an hour. It runs on the product’s architecture, uses existing components and development data (section 3), and can be clicked through. This is the principle “Design in code” in `project.md`, and the whole process rests on it.
- **Versions differ only in how refined they are.** The first version is rough: a generic look, no illustrations, some states unfinished. Each pass refines the same feature, the way a progressive JPEG shows the whole picture at once and sharpens with every pass. Nothing is thrown away and redone from scratch.
- **Decisions are made on the working version.** The team judges a feature by clicking through it. Edge cases show up at the first demo, and a wrong decision costs one more pass, not a redesign.
- **A mockup first needs a strong reason.** The product itself is where a feature is designed, and a Figma mockup drawn before the code is an exception, not a step. A mockup shows one state of one screen, hides the cases a working version shows at the first click, and has to be built a second time in code. A reason that holds is a visual the code can’t produce yet, such as an illustration or a marketing image. The reason is written in the plan.

## 2. Roles

| Role | Brings | Reviews and owns |
|---|---|---|
| Product manager or designer | The idea, the first working version in a branch, the demo | What the feature does and how it looks |
| Agent | The plan, code, tests, records in the tracker and in `specs/` | Nothing on its own: a person accepts its work |
| Frontend developer | Review of the interface code | Components, wrappers, tokens, accessibility |
| Backend developer | Review of models, migrations and queries | Schema, invariants, performance budgets |
| Other specialist | Review where the change touches their area: security, infrastructure, data | Their area |

## 3. From idea to the main branch

- **Every idea starts in the tracker.** Ideas, tasks and questions to the owner go into the tracker (Jira, Linear and the like), not into a chat or into `specs/`. The agent reads and updates it through MCP. A chat gets lost, and the tracker keeps the history.
- **The agent works the tracker.** It takes the items it understands and sets their state. In an item it doesn’t understand, it asks a question and writes the answer back into the item. A decision never stays only in a conversation.
- **A feature gets its own branch.** The branch starts from the main line and goes back into it only after review. Which branch is the main line and how it reaches production is in `.claude/CLAUDE.md`. A mistake in a branch costs nothing, and on the main line it reaches users.
- **The product manager or designer builds the first version.** They describe the hypothesis to the agent in plain words, and the agent builds it in the product: on its architecture, with existing components and development data, without a mockup first (section 1).
- **Production data is never at risk.** The development database is always separate from production, and nothing in a branch can write to the production database.
- **Sensitive data stays in production.** If the data isn’t sensitive, development may run on a copy of production data, which brings real volume and real edge cases early. Personal, financial and customer data is never copied: then the branch runs on demo data, seeded or invented, without real brands or customers.
- **A draft is shown with a frame.** Before the demo the author says what to judge now (the flow, the structure, the copy) and what is not ready yet. Open doubts are listed next to the work. A hidden doubt is found later, when the fix costs more.
- **A demo is a link or a recording.** The branch runs on a preview environment, or the author records the screen. People watch it when it suits them and can come back to it.
- **After the demo the same branch grows.** States, polish, motion and assets are added to it, by the rule of section 1.
- **Specialists review before the merge.** The frontend developer reviews the interface code, the backend developer the data and queries, and each specialist whose area the change touches reviews that part. They push their fixes into the same branch. The feature reaches the main line when every reviewer accepts it and the checks are green.
- **People push and merge.** The agent commits. A person pushes, merges and promotes. Published history is never rewritten.

## 4. Plans

- **A non-trivial change starts with a plan.** The agent writes it in plan mode, and a person approves it. The first step of the work copies the plan to `specs/work/plans/<name>-execution.md`, because a plan outside the repository is lost with the session.
- **Questions come before the plan.** The agent reads the Foundation, the docs and the code it will touch, and confirms each premise in both the docs and the source. Open questions go to the owner before the plan is written, not halfway through the work.
- **A plan has fixed parts.** Context: why, what prompted it, the intended outcome. Decisions: only the chosen approach. Then concept, UX and implementation, in that order. Milestones, each of which leaves the product working. Corner cases, each of which becomes a test. A design review against SOLID, GRASP, GoF, KISS, DRY and YAGNI. Verification: how to prove the result end to end. A plan without a design review is not sent for approval.
- **A plan describes the current pass.** It holds no history of earlier versions. One plan covers one piece of work, and separate plans are not merged.
- **Each open question has a status.** Decided, hypothesis, being explored or parked. A disputed decision is made once for the whole product and written into the Foundation, so the argument doesn’t restart on every screen.

## 5. Work and commits

- **Backend work starts with a failing test.** The test describes the behavior, and then the code makes it pass.
- **One commit per milestone.** Each milestone is committed separately, and the message body says what changed and why. Formatting and automated refactoring (Pint, Rector) go into a commit of their own.
- **Only your own files.** Several sessions can work in one tree, so files are staged by path and a commit never takes someone else’s changes. A published commit is never amended.
- **A contract change sweeps its consumers.** When a shape, a field or a route changes, every writer and reader of it changes in the same piece of work.
- **A finding goes on record in the same pass.** Whatever is found and not fixed goes into `specs/work/observations.md`: where it is, what is wrong, what to do. A finding kept “for later” is lost.

## 6. Review

- **A review starts with what works.** Then what is missing, then the next step.
- **A quick pass comes first, then the full one.** First the impression and coherence, then the three to five most visible problems, then every rule of `nickture-interface` and `nickture-text-ru`.
- **A finding is concrete and proven.** It names the element, the discrepancy, the broken rule and one fix: “the card radius is 12 and its neighbor’s is 16, make both 16”, not “I don’t like it”. A claim that wasn’t checked is marked as an assumption.
- **A change is complete in every state.** A new component has its empty, loading, error, disabled and narrow states. A new control appears on every surface that needs it.
- **Deleted lines are read too.** The reviewer checks what the change removed: `aria-*`, `alt`, `focus-visible`, a token replaced with a literal.

## 7. Done

- **Done means seen working.** The feature is visible in the product and works through a real path: a click-through in the browser, a real API or MCP call. A green unit test alone is not done, and a verdict written earlier is not evidence.
- **The checks are green.** Tests for the change and its failure modes, static analysis (PHPStan at level 10 with zero errors) and the linters. The full suite runs before the merge.
- **The knowledge is updated.** The plan’s outcome goes into `specs/docs/` or `specs/foundation/`, and the plan moves to `specs/archive/plans/`. The tracker item gets its state and a note on what was done.
