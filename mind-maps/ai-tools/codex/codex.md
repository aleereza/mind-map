## OpenAI Codex Desktop Mental Model {#codex-desktop}

> [!meta]
> type: category

A review-first operating model for OpenAI Codex, centered on the desktop app and the same agent workflow used in Codex web. Think "delegate a task, inspect evidence and diffs, then curate with Git" rather than "approve every edit before it lands."

### Mental Model Shift {#mental-model-shift}

> [!meta]
> type: category

The biggest change from traditional IDEs is where control lives.

#### Traditional IDE Loop {#traditional-ide-loop}

> [!meta]
> type: concept

```edges
contrasts: agent-review-loop
```

In an IDE or copilot flow, the editor is primary and you accept or reject suggestions before they become real changes.

#### Agent Review Loop {#agent-review-loop}

> [!meta]
> type: concept

```edges
contrasts: traditional-ide-loop
related: review-pane, terminal-evidence
```

In Codex, the unit of work is a task thread. You prompt, Codex works, then you inspect the result and decide whether to iterate, stage, revert, or commit.

#### Control Moves After Execution {#control-after-execution}

> [!meta]
> type: principle

```edges
prereqs: agent-review-loop
related: stage-means-next-commit, revert-means-discard
```

The app is not built around per-keystroke approval. Your main approval point is post-hoc review of the diff and evidence.

#### Codex As A Configured Teammate {#configured-teammate}

> [!meta]
> type: principle

```edges
prereqs: agent-review-loop
related: agents-md, four-part-prompt
```

OpenAI's best-practices docs recommend treating Codex less like a one-off assistant and more like a teammate you configure with context, rules, and verification steps.

### UI Surfaces {#ui-surfaces}

> [!meta]
> type: category

Each main surface answers a different question: what did I ask, what changed, and what evidence supports it?

#### Chat Is The Control Plane {#chat-control-plane}

> [!meta]
> type: concept

```edges
related: four-part-prompt, plan-first
```

The center chat is where you define the task, scope, and follow-up instructions. It is not the place where you approve each file edit.

#### Thread Means One Ongoing Task {#thread-means-task}

> [!meta]
> type: concept

```edges
related: chat-control-plane, worktree-background
```

A thread contains your prompts plus Codex's tool calls and outputs. You can continue the same thread with follow-up prompts instead of starting over.

#### Review Pane {#review-pane}

> [!meta]
> type: concept

```edges
prereqs: thread-means-task
related: review-pane-repo-state, inline-comments
```

The review pane is a Git diff surface. It only works inside a Git repo, and it is where you inspect diffs, open files in your editor, and shape what stays.

#### Terminal Evidence {#terminal-evidence}

> [!meta]
> type: concept

```edges
related: review-pane, verify-with-evidence
```

Codex can cite terminal logs and test outputs so you can trace what it ran and how it verified the task.

#### Inline Comments {#inline-comments}

> [!meta]
> type: concept

```edges
prereqs: review-pane
related: iterate-in-thread
```

Use inline comments on specific diff lines when the output is close but not quite right. This is often faster than rewriting the whole task.

#### Iterate In The Same Thread {#iterate-in-thread}

> [!meta]
> type: principle

```edges
prereqs: thread-means-task
related: inline-comments, plan-first
```

If Codex got part of the job right, keep the same thread and give precise follow-up instructions. Reusing the thread preserves context, comments, and discovered evidence.

### Git Review Flow {#git-review-flow}

> [!meta]
> type: category

Git is the safety system that turns Codex output into accepted work.

#### Review Pane Mirrors Repo State {#review-pane-repo-state}

> [!meta]
> type: concept

```edges
prereqs: review-pane
related: stage-means-next-commit
```

The review pane shows the Git state of the repo, not only Codex edits. It can show uncommitted changes, all branch changes, last-turn changes, and staged or unstaged views.

#### Stage Means Include In The Next Commit {#stage-means-next-commit}

> [!meta]
> type: principle

```edges
prereqs: review-pane-repo-state
related: should-you-stage-everything, partial-staging
```

Staging is not "approve before Codex edits." It means "keep this change for the next commit."

#### Revert Means Discard The Working Copy Change {#revert-means-discard}

> [!meta]
> type: principle

```edges
prereqs: review-pane-repo-state
related: stage-means-next-commit, should-you-stage-everything
```

Revert is the reject button. Use it for files or hunks you do not want to keep.

#### Partial Staging Is Normal {#partial-staging}

> [!meta]
> type: detail

```edges
prereqs: stage-means-next-commit
related: should-you-stage-everything
```

The same file can appear in both staged and unstaged views. OpenAI's review docs call out that this is normal Git behavior, not a Codex bug.

#### Should You Stage Every Change {#should-you-stage-everything}

> [!meta]
> type: detail

```edges
prereqs: stage-means-next-commit
related: revert-means-discard
```

No. Stage whatever you want in the next commit. If the whole diff is good, stage all once and commit. If the diff is mixed, stage only the good hunks or files and revert the rest.

#### Commit Is The Acceptance Boundary {#commit-acceptance-boundary}

> [!meta]
> type: principle

```edges
prereqs: should-you-stage-everything
related: clean-checkpoint
```

A commit is the durable checkpoint that says "this curated subset is accepted." Think of staging as selection and commit as acceptance.

#### Start From A Clean Checkpoint {#clean-checkpoint}

> [!meta]
> type: principle

```edges
related: commit-acceptance-boundary, worktree-background
```

Before bigger Codex tasks, commit or stash your current work so the review diff stays clean and easy to reason about.

### Local And Worktree {#local-and-worktree}

> [!meta]
> type: category

The app uses Git checkouts to separate foreground work from background work.

#### Local Is The Foreground {#local-foreground}

> [!meta]
> type: concept

```edges
related: worktree-background, handoff
```

Local is your main checkout. Use it when you want your usual IDE, one running app instance, or tight manual inspection.

#### Worktree Is The Background {#worktree-background}

> [!meta]
> type: concept

```edges
contrasts: local-foreground
related: app-for-delegation
```

A worktree is a second checkout for the same repo so Codex can run parallel tasks without disturbing your local checkout.

#### Handoff Moves The Thread Safely {#handoff}

> [!meta]
> type: principle

```edges
prereqs: local-foreground, worktree-background
related: review-pane
```

Handoff moves the thread and code between Local and Worktree. OpenAI's docs describe Local as the foreground and Worktree as the background.

#### Setup Scripts Prepare Worktrees {#setup-scripts}

> [!meta]
> type: detail

```edges
prereqs: worktree-background
related: agents-md
```

Because worktrees run in different directories, setup scripts can install dependencies or build the project automatically when a new worktree starts.

#### Actions And Integrated Terminal {#actions-and-terminal}

> [!meta]
> type: detail

```edges
related: setup-scripts, terminal-evidence
```

Local environments can define reusable actions like test or run commands, and the app can run them in its integrated terminal.

### Task Design And Context {#task-design-and-context}

> [!meta]
> type: category

Better prompts do not mean longer prompts. They mean clearer scope, constraints, and checks.

#### Four-Part Prompt {#four-part-prompt}

> [!meta]
> type: concept

```edges
related: chat-control-plane, plan-first
```

A strong default is Goal, Context, Constraints, and Done when. This gives Codex enough shape to work autonomously without guessing too much.

#### Keep Tasks Small Enough To Verify {#small-verifiable-tasks}

> [!meta]
> type: principle

```edges
prereqs: four-part-prompt
related: review-pane, verify-with-evidence
```

Focused tasks are easier for Codex to test and easier for you to review. Split big work into milestones when possible.

#### Plan First For Ambiguous Work {#plan-first}

> [!meta]
> type: principle

```edges
prereqs: four-part-prompt
related: iterate-in-thread
```

For complex tasks, use Plan mode or ask Codex to interview you first. Planning reduces wasted edits and makes later review simpler.

#### AGENTS.md Holds Durable Rules {#agents-md}

> [!meta]
> type: concept

```edges
related: configured-teammate, setup-scripts
```

Codex reads `AGENTS.md` before doing work, so repo norms, commands, do-not rules, and verification steps do not need to be repeated in every prompt.

#### Verify With Evidence {#verify-with-evidence}

> [!meta]
> type: principle

```edges
related: terminal-evidence, small-verifiable-tasks
```

Ask Codex to report the commands it ran, the tests it used, and the results. Evidence is the bridge between autonomy and trust.

### Pick The Right Surface {#pick-the-right-surface}

> [!meta]
> type: category

The best Codex interface depends on whether you want delegation, approval gating, or fast local exploration.

#### Codex App For Parallel Delegation {#app-for-delegation}

> [!meta]
> type: concept

```edges
related: worktree-background, chat-control-plane
```

The desktop app is a command center for multiple long-running tasks and background work. It is best when you want to supervise agents, not watch every edit live.

#### CLI For Approval Modes {#cli-approval-modes}

> [!meta]
> type: concept

```edges
contrasts: app-for-delegation
related: commit-acceptance-boundary
```

If you want stronger approval gates, the CLI is closer to that model. OpenAI's docs define Auto, Read-only, and Full Access modes, and Read-only will not edit or run commands until you approve a plan.

#### IDE Extension For Fast Local Exploration {#ide-fast-local}

> [!meta]
> type: concept

```edges
contrasts: app-for-delegation
related: review-pane
```

OpenAI's workflows docs describe the IDE extension as the fastest surface for local exploration, open-file context, and targeted prompts near the code.

#### Use The Surface That Matches Risk {#surface-matches-risk}

> [!meta]
> type: principle

```edges
prereqs: app-for-delegation, cli-approval-modes, ide-fast-local
```

Use the app when delegation speed matters, the CLI when approval control matters, and the IDE when close reading or surgical edits matter.

### Common Mistakes {#common-mistakes}

> [!meta]
> type: category

Most frustration comes from carrying the wrong expectations into an agent workflow.

#### Waiting To Approve Every Edit {#waiting-to-approve}

> [!meta]
> type: detail

```edges
related: traditional-ide-loop, control-after-execution
```

If you expect Codex to pause on each file edit, the app will feel wrong. The intended loop is delegate first, review second.

#### Treating Stage Like Pre-Edit Approval {#stage-is-not-pre-edit-approval}

> [!meta]
> type: detail

```edges
related: stage-means-next-commit, should-you-stage-everything
```

Stage is not the same as accepting an IDE suggestion. It only selects what goes into the next commit.

#### Running Overlapping Threads On The Same Files {#overlapping-threads}

> [!meta]
> type: detail

```edges
related: thread-means-task, worktree-background
```

Codex can run multiple threads at once, but avoid having two threads edit the same files. Parallelism works best with separated scope.

#### Using Vague Prompts {#vague-prompts}

> [!meta]
> type: detail

```edges
related: four-part-prompt, plan-first
```

"Improve this" or "clean up the codebase" creates noisy diffs. Name the files, the bug, the constraint, or the acceptance test.

#### Skipping Review Because Tests Passed {#skipping-review}

> [!meta]
> type: detail

```edges
related: verify-with-evidence, review-pane-repo-state
```

Passing tests help, but they do not replace reading the diff. Your job is still to inspect behavior, scope, and unintended side changes.
