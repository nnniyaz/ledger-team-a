# Ledger: Development process (SDLC)

| | |
|---|---|
| Team | Team A, FIU Capstone 2, Fall 2026 |
| Version | 0.1 draft, 2026-10-02 |
| Status | Written by the Team Leader. Pending review by the team. |

## 1. Model

The team works iteratively: five two-week sprints, each ending with a review and a retrospective. Every sprint passes through the same phases on a small scale (requirements, design, implementation, verification, review), and each sprint also has a main emphasis as the project matures.

A plan-everything-first model was not chosen because the team started with no Ada or SPARK experience. What can be proved, and how long it takes, is only learned by doing it, so the plan has to be corrected every two weeks.

## 2. Phases and where they happen

| Phase | Main emphasis in | What it produces | Where the evidence is |
|---|---|---|---|
| Planning and elicitation | Sprint 1 | Scope agreed with the Product Owner; working toolchain | Sprint 1 review; board card "Meet with Product Owner" |
| Requirements analysis | Sprint 2 | Requirements with a verification method for each | `docs/requirements.md` |
| Design | Sprint 2 | Architecture, file formats, algorithms; decisions with rejected alternatives | `docs/design.md`, `docs/decisions/` |
| Implementation | Sprints 3 to 5 | SPARK units, merged through reviewed pull requests | Repository history |
| Verification | Sprints 3 to 5, continuous | GNATprove reports, crash-test results, unit tests | CI runs linked from board cards |
| Review and acceptance | End of every sprint | Demo of an end-to-end path; Product Owner accepts or asks for changes | Sprint review on the capstone site |
| Maintenance and improvement | End of every sprint | Retrospective actions; backlog and documents updated | Sprint retro; document history |

## 3. Where the project stands today

Stated plainly, because a process document that hides the starting point is not useful.

- **Sprint 1** went to scope and tooling. The team lost its first Team Leader mid-sprint and continued.
- **Product Owner sessions.** Across Sprints 1 and 2 there were three joint sessions with the Product Owner. They covered Alire (the Ada package manager) and SPARK.
- **Sprint 2.** The repository was created, the Alire project was initialized, and the requirements, design and decision records in this folder were written. Sprint planning was filed late and the board was not kept current during the sprint.
- **No engine code exists yet.** Implementation starts in Sprint 3 according to the build order in design section 12.

## 4. Sprint plan

| Sprint | Weeks | Emphasis | Increment |
|---|---|---|---|
| 1 | 3-4 | Planning, toolchain | Scope agreed, environment set up |
| 2 | 5-6 | Requirements and design | This document set; Alire project that builds |
| 3 | 7-8 | Implementation: durability core | WAL codec (Gold), WAL, I/O boundary, fault injection, first crash tests |
| 4 | 9-10 | Implementation: storage | Pager, B+tree, recovery (Gold), checkpoint, CLI |
| 5 | 11-12 | Hardening | Readers, recovery benchmark, remaining proofs, showcase material |

## 5. How a piece of work moves

1. **Backlog.** A card states one outcome and its acceptance criteria. One card, one outcome.
2. **Ready.** The card has exactly one owner and criteria the Product Owner can answer yes or no to. Set at sprint planning.
3. **In Progress.** The owner works on a branch.
4. **Review.** A pull request is open. One other team member reviews it. CI must pass.
5. **Done.** Merged to `main`, with evidence linked on the card.

### Definition of Done

A card is Done only when all of these hold:

- `alr build` succeeds on `main`.
- GNATprove reports no unproved checks for the units the card touched, at the level set in decision 0003.
- Tests for the change exist and pass.
- The pull request was reviewed by one other team member.
- The card links its evidence: the pull request and, where relevant, the proof or test output.
- If the change alters behaviour described in `docs/`, the document was updated in the same pull request.

## 6. Verification strategy

Each requirement names how it is verified (see the tables in `requirements.md`). Three methods are used, each for what it is suited to:

| Method | Used for | Tool |
|---|---|---|
| Proof | Properties that must hold for every input: no run-time errors, codec round trip, recovery idempotence | GNATprove |
| Crash tests | Behaviour that depends on the operating system and disk, which proof cannot reach | Fault-injecting I/O layer and a reference model |
| Unit tests and benchmark | Functional behaviour of the B+tree and API; recovery time | Test executables run in CI |

## 7. Ceremonies

| Ceremony | When | Purpose |
|---|---|---|
| Stand-up | Twice a week, posted on the capstone site by every member | What I did, what is next, what blocks me |
| Sprint planning | First days of the sprint | Sprint goal; committed cards, each with one owner |
| Product Owner meeting | Weekly, joint for all Ledger teams | Priorities, questions, technical guidance |
| Sprint review | Last days of the sprint | Show what is Done; Product Owner accepts or asks for changes |
| Sprint retro | After the review | Keep, change, one thing about the team |

## 8. Risks

| Risk | Effect | Response |
|---|---|---|
| Team is new to SPARK; proofs take longer than planned | Gold targets slip | Gold limited to two small units; fallback to Silver plus crash tests is written into decision 0003 |
| B+tree not finished in Sprint 4 | No end-to-end Put and Get demo | Fallback index described in decisions 0001 and 0002; durability work carries over |
| Work stalls between meetings, as it did in Sprint 2 | Late or empty review | Two stand-ups per member per week; every card has one owner; the Team Leader raises at mid-sprint anything that will not be ready |
| One card bundles several tasks | Progress is invisible on the board | Split cards so that each has one outcome and one owner |
| Roster changes again | Lost context | Decisions and design live in the repository, not in chat |

## 9. Traceability

Each requirement maps to a design section and a board card in `requirements.md` section 9. When a card is closed, its evidence link completes the chain: requirement, design, code, proof or test.
