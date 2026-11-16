# Draft: Managing Coding Agents

This is a discussion artifact for organizing the future mind map. It is not yet parser-ready mind-map source.

## Recommended name

**Display title:** Managing Coding Agents

**Root question:** How do you organize, equip, constrain, and improve coding agents so their software work becomes trustworthy?

**Map location:** `mind-maps/ai-tools/agents/`

**Source file:** `agents.md`

This name is direct and broad enough to include team structure, orchestration, codebase design, verification, guardrails, evals, and increasing autonomy. “Trustworthy Coding-Agent Systems” is a stronger alternate if the map should emphasize system design more than day-to-day management.

## Proposed organizing idea

Treat the subject as an operating system for coding-agent work, not as a catalog of agent tools. The map should explain five system dimensions:

1. who is responsible
2. how work moves
3. how the codebase supplies context
4. how correctness and safety are enforced
5. how the whole system learns and earns trust

The “head chef and kitchen stations” metaphor can introduce this model, but the branch labels should remain literal so the map is useful without knowing the metaphor.

## Proposed first layer

### 1. Roles and Responsibilities

This branch answers: **Who does what, and who remains accountable?**

- **Human lead — the head chef**
  - sets outcomes, priorities, risk tolerance, and acceptance criteria
  - designs the working environment and decides what authority to delegate
  - remains accountable for integration and production consequences
- **Chief of staff agent**
  - turns an outcome into bounded work
  - selects agents or skills for each part
  - tracks dependencies, progress, and blocked work
  - synthesizes results and escalates decisions that require human judgment
- **Worker roles**
  - explorer or planner
  - implementer
  - verifier or reviewer
  - judge or integrator
- **Role composition**
  - one general agent can take several roles at different times
  - separate agents are useful when context isolation or independent judgment matters
  - a specialized agent is a configured worker, not merely a skill

#### Agent, role, skill, and tool are different

| Term | Practical meaning |
|---|---|
| Agent | An active worker with its own context, task, tools, permissions, and execution loop. |
| Role | A responsibility in the workflow, such as implementer, verifier, or coordinator. |
| Skill | Reusable instructions, knowledge, or a procedure that an agent loads to perform a class of work. |
| Tool | A capability the agent can invoke, such as editing files, running tests, or querying an issue tracker. |
| Specialized agent | An agent made specific through a stable role, selected skills, tools, permissions, model, and success criteria. |

This distinction directly answers “Are specific agents the same as agents with a specific skill?” A skill is one ingredient. Specialization may also require a different context, permission boundary, tool set, model, or independent execution.

### 2. Orchestration and Communication

This branch answers: **How is work divided, passed between agents, and integrated?**

- **Work patterns**
  - pipeline: explore → plan → implement → verify → integrate
  - supervisor and workers: a chief of staff delegates separate work units
  - parallel fan-out: several agents attempt or investigate the same problem
  - independent review: a fresh agent challenges an implementation or verdict
- **Communication contract**
  - task brief: goal, relevant context, constraints, and definition of done
  - artifact-based communication: diffs, files, tests, logs, and decision records
  - handoff: completed work, evidence, remaining risk, and next action
  - shared state: one explicit source of truth rather than assumed shared memory
- **Work granularity**
  - one concern per task or PR
  - small enough to verify and review
  - explicit ownership of shared files and integration points
- **Execution environment**
  - local agents use the developer’s machine and nearby feedback loop
  - cloud or remote agents run elsewhere, often asynchronously and in isolation
  - isolation enables parallel work but requires environment parity, access control, and observable results

Agents do not need free-form conversation with one another. In most software work, reliable communication comes from explicit contracts and durable artifacts coordinated by an orchestrator.

### 3. Agent-Friendly Codebase

This branch answers: **How can the repository make the locally obvious change more likely to be globally correct?**

- **Codebase map**
  - major areas and what each owns
  - entry points and common change paths
  - dependencies and data flow
  - invariants and forbidden boundaries
  - commands and tests that prove a change
  - links to deeper, maintained sources rather than duplicating everything
- **Locality and discoverability**
  - keep related feature code together
  - make ownership and the conventional path obvious
  - reduce the amount of searching and hidden context needed for a safe change
- **Architecture as guidance**
  - make the supported path easier than a shortcut
  - fail forbidden dependencies mechanically
  - give durable state one obvious writer
  - prefer isolated additions over branching through shared roots
  - keep exceptions narrow, explicit, and reviewed as architecture decisions
- **Design for bounded context**
  - assume an agent sees a narrow prompt and a few nearby files
  - refactor architecture when repeated local edits produce global mistakes
  - optimize the repository so an ordinary agent can succeed without relying on the strongest model

The five architecture rules above are the durable lessons from the Dune screenshots. Dune should appear as a named case study or source note, not as a primary pillar of the map.

### 4. Verification and Guardrails

This branch answers: **How do we prove the result and prevent unacceptable changes?**

- **Definition of done**
  - choose observable success criteria before delegation
  - make the verification method part of the task contract
- **Verification ladder**
  - inspect the actual diff or artifact
  - run static checks, types, linting, and focused tests
  - run integration, end-to-end, or runtime checks where behavior crosses boundaries
  - compare the result with the user-visible or production-like outcome when possible
- **Independent verification**
  - do not rely only on the implementer’s summary or self-certification
  - use a fresh verifier when independence materially reduces shared blind spots
  - use deterministic evidence before subjective judging
- **Verify the verifier**
  - seed known defects and confirm they are caught
  - include known-good cases and measure false alarms
  - use mutation or adversarial cases to test whether checks detect real failures
  - calibrate judge decisions against human-reviewed examples
- **Mechanical guardrails**
  - tests, type constraints, dependency checks, protected integration paths, sandboxing, and permissions
  - scope limits that keep changes reviewable
  - clear escape and escalation paths when a task exceeds authority
- **Human-compatible CI**
  - high-signal checks with low noise
  - failures name the invariant, evidence, and supported correction
  - automation should help both agents and humans contribute rather than punish every deviation with opaque friction

Written rules and skills can guide behavior, but important objective invariants should be enforced in code or CI when feasible.

### 5. Evals, Learning, and Trust

This branch answers: **How does the system measure capability, learn from failures, and earn more autonomy?**

- **Verification versus evaluation**
  - verification asks whether one work product meets its definition of done
  - evaluation asks whether an agent, skill, or workflow performs reliably across a class of tasks
- **Eval design**
  - representative tasks, regressions, difficult edge cases, and holdout cases
  - repeatable starting states and multiple trials where results vary
  - metrics for correctness, regressions, safety, cost, latency, and human review burden
- **Judging**
  - deterministic tests first
  - rubric-based human or model judgment for subjective qualities
  - blinded comparisons and calibrated examples
  - multiple independent judges when one judge could share the candidate’s blind spots
- **Hill-climbing loop**
  - establish a baseline
  - inspect thinking traces and tool calls to classify failures
  - change one lever: skill, context, architecture, guardrail, or orchestration
  - rerun the eval suite and keep only demonstrated improvements
  - check a holdout set so a nominal “10/10” does not merely mean the loop learned one judge or test set
- **Encode recurring lessons**
  - turn repeated objective PR feedback into tests, lint rules, types, or architecture constraints
  - use a skill when the lesson is procedural, contextual, or judgment-heavy
  - keep one-off subjective feedback in review rather than forcing every preference into CI
- **Trust and autonomy**
  - begin with narrow authority and close review
  - expand autonomy when evidence is repeatable across tasks
  - monitor drift and regressions after expansion
  - preserve human checkpoints for high-impact or irreversible actions

More agents do not automatically create more trust. Trust can rise when additional agents add genuinely independent evidence or redundancy; it can fall when coordination overhead and correlated mistakes increase. The proposed visual should therefore relate trust to **quality and independence of evidence**, not agent count alone.

## Compact operating loop inside the map

The final map can include one ordered procedure under “Orchestration and Communication”:

1. define the outcome and proof
2. map the relevant area
3. delegate bounded work
4. implement through supported paths
5. verify with direct and independent evidence
6. integrate a single concern
7. inspect failures and improve the system

This gives the learner a practical path through the conceptual branches without turning the entire map into a workflow diagram.

## What should remain examples or resources

These are useful supporting nodes, but they should not become first-layer pillars:

- **pstack:** a concrete plugin made of routing skills, playbooks, principles, subagent patterns, and eval workflows. It can illustrate how skills compose into an agent operating style. Primary source: [Cursor pstack plugin](https://github.com/cursor/plugins/tree/main/pstack).
- **Dune:** a case study for architecture that makes locally obvious edits safer; source material currently comes from the webinar screenshots.
- **Pretext library:** keep as an unresolved reference until the exact library and its relevance to this topic are confirmed.
- **Cloud and local agents:** execution environments within orchestration, not different fundamental species of agent.
- **Strong versus ordinary models:** an implementation choice and cost lever, not the organizing principle. The goal is a system in which solid outcomes do not depend exclusively on the strongest model.

## Source-note coverage

| Source note | Proposed location |
|---|---|
| Series of agents and agent communication | Orchestration and Communication |
| Chief of staff agent | Roles and Responsibilities |
| Human as head chef | Roles and Responsibilities |
| Specific agent versus agent with a skill | Agent, role, skill, and tool distinction |
| Local versus cloud agents | Execution environment |
| Single-concern PRs and bounded changes | Work granularity |
| Reading thinking blocks and tool calls | Hill-climbing loop |
| Creating a skill to address repeated failure | Encode recurring lessons |
| Evaluating a skill or agent | Eval design |
| Judge agents and iterative improvement | Judging and Hill-climbing loop |
| Importance of verification | Verification and Guardrails |
| Verifying the verification skill | Verify the verifier |
| Hard tests and CI guardrails over written rules | Mechanical guardrails |
| Guardrails against oversized or unsafe changes | Mechanical guardrails and Work granularity |
| CI becoming hostile to human contributors | Human-compatible CI |
| Convert repeated PR comments into hard rules | Encode recurring lessons, with an objectivity threshold |
| Codebase map for navigation | Agent-Friendly Codebase |
| Related features in one directory | Locality and discoverability |
| Refactor or rewrite for agent friendliness | Design for bounded context |
| Make ordinary agents effective | Design for bounded context |
| Dune architecture screenshots | Architecture as guidance and Dune case study |
| pstack plugin | Supporting resource |
| Pretext library | Unresolved reference |
| Trust versus number of agents | Trust and autonomy, reframed around independent evidence |

## Decisions to settle before creating the map

1. **Scope:** keep the map vendor-neutral, with Codex, Cursor, pstack, and Dune only as examples. Recommended: yes.
2. **Title:** use “Managing Coding Agents.” Recommended: yes.
3. **Dune:** include the five principles as general architecture lessons and attach the screenshots only if they improve learning. Recommended: principles first; one screenshot at most as supporting evidence.
4. **pstack:** include one link node as an example of composed skills and playbooks. Recommended: yes, after the conceptual structure is stable.
5. **Pretext:** confirm the exact name and why it was mentioned before adding it. Recommended: leave out of the first map draft until confirmed.
6. **Trust curve:** do not encode “more agents means more trust” as a general law. Recommended: show an autonomy ladder driven by accumulated verification evidence, with independent agents as one possible source of evidence.

## Likely editing cuts for the final visual

This discussion draft is intentionally more complete than the eventual diagram. The final map should use short labels, keep most explanation in node bodies, and remove details that do not improve orientation or action. In particular:

- keep the first layer at five pillars
- avoid making every worker role its own deep branch
- keep only a few orchestration patterns
- merge overlapping guardrail and CI nodes
- use very few cross-links
- preserve the verification/evaluation distinction even if other details are cut
