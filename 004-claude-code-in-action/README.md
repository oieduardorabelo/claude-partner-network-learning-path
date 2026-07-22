# Claude Code in Action

https://anthropic-partners.skilljar.com/claude-code-in-action

This module is about running Claude Code on real, longer-running work and trusting the result. It covers steering long sessions, configuring Claude so it reliably follows your rules, automating repeat work, and verifying runs you did not watch. The throughline: decide up front what is safe to run without you, make the important rules impossible to skip, and check the output in proportion to how unsupervised the run was.

## Table of contents

- [Steer the work](#steer-the-work)
  - [Steering long sessions](#steering-long-sessions)
- [Configure Claude](#configure-claude)
  - [A CLAUDE.md that follows](#a-claudemd-that-follows)
  - [Verification skills](#verification-skills)
  - [Permission modes](#permission-modes)
  - [Hooks](#hooks)
- [Automate repeat work](#automate-repeat-work)
  - [Routines and headless](#routines-and-headless)
  - [GitHub Actions and code review](#github-actions-and-code-review)
- [Verify and share](#verify-and-share)
  - [Trust it: verifying unsupervised runs](#trust-it-verifying-unsupervised-runs)
  - [Plugins](#plugins)

## Steer the work

### Steering long sessions

A quick task is easy: you prompt Claude, watch it finish, and check the result. A long task is a different game. Refactoring across a dozen files or building a feature can take hours, and it comes down to two habits: scope the work before Claude starts, and steer it while it runs.

**Scope first with plan mode.** In plan mode Claude researches in read-only mode and hands you a plan. Actually read it. The more thorough the plan, the fewer hiccups when Claude executes. To revise, just ask Claude to change the part you want. Iterating over a plan is far faster than hoping for the best.

**Steer while it runs.** A few tools keep a long session on track:

- **Compact.** `/compact` summarises the conversation and uses that summary as the new context, freeing the context window. The risk is that something important gets dropped and Claude drifts. So direct the compaction: add instructions after the command to tell Claude what to keep. If you finished debugging a while ago but want to focus on the API changes, say so.
- **Rewind.** When Claude goes down the wrong path, you do not have to prompt your way out. Double-tap Escape on an empty prompt to open the rewind menu. Every user prompt is a checkpoint you can revert to, restoring the code, the conversation, or both. You can also summarise everything after a checkpoint (drop a side conversation) or everything before it (compress a long setup phase but keep the implementation).
- **Goal.** `/goal` sets a completion condition: you describe what done looks like and Claude keeps working across turns until a fast evaluator confirms the conditions are met, instead of stopping the first time it thinks it is finished. For example, `/goal all tests in src/billing pass and the type checker reports zero errors`. `/goal clear` cancels it. One constraint: the evaluator only reads the transcript, so the condition has to be checkable from Claude's output (such as test results it ran).
- **Loop.** Runs a prompt on an interval between turns, fixed or self-paced. Use it to pull something external, like a CI run or a deploy, and act when the state changes. Press Escape to stop it.

**Worktrees.** When several Claude sessions work on the same repository, you do not want two steering wheels in one car. Worktrees give each session an independent file tree, preventing sessions from fighting over the same files. A clean worktree is auto-removed on exit. A `.worktreeinclude` file at the repo root lists git-ignored files to copy into each worktree, useful for an environment file or local config you need but do not commit.

The summary: scope your work, then steer. Direct compaction so the summary keeps what matters, use rewind to course-correct, set a goal when you can describe done better than the steps, and run parallel work in worktrees.

## Configure Claude

### A CLAUDE.md that follows

`CLAUDE.md` is not enforced configuration. It is instructions Claude tries to follow, and the longer the file gets, the more its rules compete with each other, so the less reliably Claude follows any of them. The aim is a lean file.

**Is CLAUDE.md even the right tool?** A hard rule like "never push to main" belongs in a pre-tool-use hook, which stops the push even when Claude tries it. `CLAUDE.md` cannot guarantee that.

**Four memory locations.** Claude loads every memory file together and nothing gets dropped: managed policy (org, always in play), user, project, and local. Local is for what is only yours in this project, for example architectural decisions you want Claude to hold in mind during a refactor, rather than the shared project file.

**Imports.** Use the `@path/to/file` syntax to split instructions across files. Know what it buys you: Claude expands imported files inline at launch, right alongside the file that references them. Imports help you organise a large file, but everything still loads up front, so they do not reduce context.

**Phrasing decides obedience.** Most rules fail because they are vague:

- Be specific and checkable. "Follow best practices" means nothing. "Put new API routes in `src/api/handlers`, one per file" is explicit.
- Name the replacement. "Don't use default exports" leaves the alternative open. "Use named exports, not default exports" closes the gap.
- Emphasis is a budget. "IMPORTANT" and "you must" raise a rule's priority only relative to the quieter rules around it. Spend them on the two or three rules that hurt when broken.

**Keep it under revision.** When Claude does the wrong thing, treat it as a bug report against your `CLAUDE.md`. Tell Claude to add the rule and it will write it for you. Treat the file like production code: if you cannot justify a line, delete it. The leaner the file, the more of it Claude follows.

### Verification skills

As a project grows, automate the work that repeats. Skills are good for this, and the first skill worth building is one that verifies your work. Ask Claude to refactor something; when it finishes, the description matches a task that edited code and the skill fires. It runs the test suite, reads the diff, checks that no test was weakened just to pass, and reports pass or fail with the evidence attached. Checking the work no longer depends on you remembering to ask.

The same shape carries any procedure your team repeats, such as a release checklist or a migration recipe. If you have typed the same multi-step instruction twice, that is a skill.

A skill folder can hold more than `SKILL.md`. Drop a `reference.md` beside it for detailed material and link it from the skill; Claude reads it only when it needs the depth. Claude executes scripts in the folder rather than loading them, so a skill can carry its own tooling. Keep `SKILL.md` lean and push heavy material into side files.

A quick way to keep the three instruction surfaces straight:

- Conventions that apply all the time go in `CLAUDE.md`.
- Procedures or reference material tied to a kind of task go in a skill (only the description loads until the skill is needed).
- A rule Claude must not be able to skip goes in a hook, because `CLAUDE.md` and skills are instructions Claude follows, not code that runs.

### Permission modes

Permission mode lets you decide once what is safe to run without you. You cycle through several with Shift+Tab, and the status bar shows the current one:

- **Manual.** Reads without prompting; everything else asks.
- **Accept edits.** Reads, file edits, and common file-system bash commands. For iterating on code you review after the fact.
- **Plan.** Reads only. Researches and proposes without editing.
- **Auto.** Accepts everything, but a separate classifier model reviews each action before it runs and steps in on dangerous ones. The hands-off default.
- **Don't ask.** Only pre-approved tools are allowed; everything else is auto-denied with no prompt. Good for CI pipelines.
- **Bypass permissions.** Skips all checks. Equivalent to `--dangerously-skip-permissions`. Only run it inside an isolated container or virtual machine.

**How auto mode works.** Claude runs, but a classifier reviews each action first, guarding intent. It blocks moves that escalate beyond your request: production deploys and migrations, force-pushing, piping downloaded code into a shell, sending sensitive data to external endpoints, and destroying files. It allows everyday work: local edits in your project, installing declared dependencies, read-only requests, and pushing to your own branch.

The catch: the classifier guards intent, not correctness. Ask Claude to refactor authentication and it writes broken authentication, the classifier waves it through, because broken is not dangerous. So pair auto mode with a stop hook that runs your tests. Auto mode watches what Claude is trying to do; the hook confirms the code actually runs. The guardrails are still evolving, so check the docs for the current block and allow lists.

"Don't ask" is the right move whenever no human is there to approve prompts (CI, scheduled jobs, overnight batches). Anything off the allow list is auto-denied, so the pipeline keeps moving instead of hanging on an approval no one will give. Match the mode to the job.

### Hooks

A hook is deterministic code that runs at a fixed point in the loop, so it can guarantee things that instructions cannot, especially on a run you are not watching. It turns a rule from "Claude usually listens" into "Claude cannot skip it." Claude Code fires around 30 hook events. The ones you will reach for:

- **PreToolUse.** Fires before a tool call. The enforcement primitive.
- **PostToolUse.** Fires after a successful tool call. Where auto-formatting or linting goes; too late to block the call.
- **Stop.** Fires when Claude wants to end its turn. Refuse if conditions are not met.
- **SubagentStop.** Same signal, for a subagent finishing.
- **PreCompact / PostCompact.** Fire before and after compaction. To re-inject context after a compact, use SessionStart with the compact matcher, not PostCompact.
- **SessionStart.** Primes the environment. Use the startup matcher to run only on fresh starts.

**Controlling a PreToolUse hook.** Return JSON and exit 0 with a `permissionDecision` field: `allow`, `deny`, or `ask`. (There is a fourth value, `defer`, but it only applies to non-interactive `-p` runs where a calling process pauses the tool and resumes it later, so you will rarely reach for it.) You can also return updated input to modify the tool call without blocking, useful for redacting a secret out of a bash command. One catch: updated input replaces the whole input object, so echo back the fields you are not changing.

**Exit codes** work for hooks that do not return JSON. Three numbers matter:

- `0` is success. If stdout is JSON, Claude parses it. Plain text is ignored on most events, but on SessionStart, UserPromptSubmit, and prompt expansion it is added to context, which is what makes a state-preserver hook work.
- `2` is a blocking error. stderr is fed back to Claude as context. This is the blocking exit code almost everywhere. On Stop, exit 2 is how you say "you are not done."
- Anything else is non-blocking. Note that exit 1 feels like an error but does not block, so Claude runs the command anyway.

A few events ignore blocking (Notification, SessionStart), showing stderr and carrying on. PostToolUse fires after the tool already ran, so it cannot stop the call, though it can still feed text back to Claude.

Two patterns worth stealing:

- **Redact instead of block.** A PreToolUse guardrail with a matcher for `bash` and an optional condition narrowing it to a specific command. Return `deny` to stop a dangerous call, or return updated input to rewrite it, stripping a secret out of a command while still letting it run.
- **Survive a compact.** When Claude compacts a long conversation it drops detail. A SessionStart hook with the compact matcher runs right after, printing a short summary of the files you were working on. That summary goes back into context, so Claude picks up where it was instead of starting cold.

The setup pays back the first time it catches something on a run you were not watching.

## Automate repeat work

### Routines and headless

Once you trust Claude to do a task, stop doing it by hand. There are two paths.

**Routines (build nothing).** A routine is a saved prompt plus the repositories and connectors it needs, run in the cloud on Anthropic's managed infrastructure whenever a trigger fires: a cron schedule, an HTTP POST to its API endpoint, or a GitHub event like a new pull request. There is no machine of yours staying on overnight and no workflow file to maintain. Create one from the web at `claude.ai/code/routines` or from inside Claude Code with, for example, `/schedule daily dependency audit at 9am`. Anything that is the same prompt on a recurring trigger fits: a morning dependency audit, a PR triager.

Three things to know first: routines are a research preview, so behaviour and limits will keep moving; a recurring schedule runs at most hourly; and each run starts from a fresh clone of your default branch and can only push to `claude/`-prefixed branches unless you loosen that per repo. That last point is the guardrail that keeps an autonomous run from rewriting main.

**Headless mode (full control).** When the job needs your environment or logic around the run, drop to the `-p` (print) flag. It runs Claude Code as a one-shot command with no interactive UI, reading stdin and writing stdout so it pipes like any other shell tool. It skips auto-discovery of hooks, skills, plugins, MCP servers, and `CLAUDE.md`: you get Claude plus the tools you allow explicitly and nothing the local environment happens to load, so startup is much faster.

Pair a JSON schema with `--output-format json` and Claude constrains its structured output to match the schema. The result lands in the `structured_output` field of the JSON response, so you can pull it with `jq` and pipe it into a database or another script. For multi-step automation, capture the session ID from the JSON output and resume: one script kicks off the work, another resumes it later with full context.

**The Agent SDK** gives you a library that embeds Claude Code inside your own TypeScript or Python applications. Both expose a `query` function and the same primitives as the CLI: pass a prompt plus options like `allowedTools`, a system prompt, and a permission mode, then iterate the messages Claude streams back.

Routines are the default for repeat work. Drop to headless when the job needs your pipeline (`-p` to pipe data through a script), and the Agent SDK when the work belongs inside your own product.

### GitHub Actions and code review

The best place to hand off repetitive work is the pull request. Two ways to get there.

**Managed: Code Review.** An Anthropic-hosted service that reviews PRs through the Claude GitHub app and posts findings as inline comments, with nothing to build or host. An org admin turns it on from the Claude Code admin settings, installs the Claude GitHub app, and picks which repos it watches and when it runs: when a PR opens, on every push, or only when someone comments `@claude review`. A set of review agents analyses the diff against the full codebase and posts findings on specific lines, tagged by severity with a summary table in the check run. It deduplicates and ranks so you read a handful of real findings instead of a wall of nits. It never approves or blocks the PR; that judgment stays with a human. Code review is a research preview (currently team and enterprise plans), and there is no managed autofix, the service posts findings only. Apply them from your own terminal with `/code-review`, whose `--fix` flag applies findings to your working tree.

**Do-it-yourself: the GitHub Action.** For jobs beyond review, such as implementing changes from a comment or scheduled reports. Run `/install-github-app` from inside Claude Code (you need repo admin). It walks you through installing the GitHub app and setting the Anthropic API key secret on the repo. The action is `anthropics/claude-code-action@v1`. Inputs you will use:

- `anthropic_api_key` (required), or switch to Bedrock or Vertex if you use those providers.
- `github_token` (defaults to `secrets.GITHUB_TOKEN`).
- The trigger phrase the action listens for in comments (defaults to `@claude`).
- The prompt (the instruction for the run).
- `claude_args`, a string of CLI arguments passed through to Claude Code.

Drop a workflow into `.github/workflows/claude.yml` that listens for `@claude` on PR and issue comments. Someone writes "@claude, implement the spec in the linked issue" on a PR and the action picks it up, pushing commits and posting comments with what it did. A second workflow can be a daily rollup on a cron trigger (with `workflow_dispatch` so you can also run it manually from the Actions tab). The `claude_args` line is where the tuning happens: cap the agent loop with `--max-turns`, set the permission mode to "don't ask" for unattended runs, and set `--allowedTools` to exactly what the job needs (read-only for a report).

For PR reviews, take the managed path and apply fixes locally with `/code-review --fix`. Reach for the action when the job is more than review.

## Verify and share

### Trust it: verifying unsupervised runs

You handed Claude a task and let it run without watching every step. It says it is done. Before you ship, you need a way to check work you did not supervise, because no one saw it happen. Verify in proportion to how unsupervised the run was: a short session you watched needs a glance; an unattended run or a CI job with nobody in the loop needs a real check.

When a run goes unattended, keep it in auto mode rather than bypass permissions. The classifier still reviews each action for danger, but it never judges whether the code is right, so the verification bar stays where it was.

- **Read the diff, not the summary.** Run `/code-review` to walk the changes and flag issues, then put your eyes on `git diff`. The trap is a tidy summary that reads fine while the diff touched a file you did not expect. Read what changed, and read the files that were in the plan first.
- **Gate on tests with a hook.** The real gate is whether the tests passed and whether Claude ran them or only claimed it did. Do not leave that to trust: wire it as a hook so Claude cannot skip it. A stop hook that runs your tests and refuses to end the turn on a failure, or a post-tool-use hook that lints and type-checks after every edit. A hook that exits with code 2 feeds the failure straight back to Claude, which reads it and fixes it without you asking. The check fires on every run, whether or not you remember to ask.
- **Get a cold second opinion.** The subagent code review you would run before a PR works here too. Open a fresh session or subagent and have it review the changed code with no memory of how it was built. It has no stake in the approach and catches what the original run talked itself past.

Make the check as serious as the run was unsupervised. Read the diff, turn the test into a hook that gates the turn, verify headless runs by their JSON result and exit code, and get a cold second opinion on anything that matters. Do that, and "Claude did it while I was not looking" no longer takes faith.

### Plugins

A setup you trust is worth more when your whole team runs it. Plugins are how Claude Code packages a setup and moves it between people. There are two sides: using plugins others publish, and packaging your own once you have built something worth sharing.

**Using plugins.** A plugin is one installable unit, and where it lives sets how you install it. Inside a session, `/plugin install org-name@plugin-name`. For a team, add a private marketplace once with `/plugin marketplace add your-org/claude-plugins`; every install after that resolves through it, with centralised discovery, version tracking, and updates.

The important caveat: a plugin runs code on your machine with your privileges, and its hooks fire on every matching tool call. Install it for the skills and you also get its pre-tool-use and stop hooks whether you read them or not. A community plugin can ship a stop hook that calls a network endpoint, and nothing in your configuration will warn you. So see every hook, agent, and MCP server it adds, and install plugins or add marketplaces only from sources you truly trust. The in-app submission form posts to the community marketplace after Anthropic's automated review, and the official marketplace is curated on its own track, but reviewed is not the same as trusted, so check what it does.

A plugin does not overwrite your configuration; its components run alongside yours. Hooks stack: a plugin's pre-tool-use hook and your own both fire on every tool call, which is the reason to read the details first. Skills, agents, and commands are namespaced under the plugin name, so they never clash. A plugin can also ship a `settings.json`, but a narrow one: Claude Code honours only the `agent` and subagent status-line keys. Setting `agent` promotes one of the plugin's subagents to the main thread with its system prompt, tool restrictions, and model, so enabling that plugin changes how Claude Code behaves by default. One more reason to look before you turn it on.

**Building one.** Once you have a `.claude` directory you like, package it instead of having your team copy and paste between machines. A plugin bundles it all as one versioned installable unit: skills, subagents, hooks, and MCP server configs, plus the longer tail of language-server-protocol servers, background monitors, themes, and a `settings.json` slice. The manifest lives at `.claude-plugin/plugin.json` and holds name, version, description, and author. It is optional; leave it out and Claude Code discovers components by directory convention. The directory structure does the rest: the same `.claude` shape you already use, one folder per skill, one markdown file per subagent, `hooks/hooks.json` and `.mcp.json`, all at the plugin root. Name is the only required field, and it namespaces your skills as `company-name:skill-name`. Version it like any other dependency.

When you use plugins, read before you install. When you build one, package your `.claude` the moment it works. One manifest, one install, and the setup you trust reaches your entire team.
