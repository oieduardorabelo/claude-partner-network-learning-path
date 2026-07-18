# Introduction to agent skills

https://anthropic-partners.skilljar.com/introduction-to-agent-skills

## Table of contents

- [What a skill is](#what-a-skill-is)
- [Creating a skill](#creating-a-skill)
- [Configuration and multi-file skills](#configuration-and-multi-file-skills)
- [How skills compare to other Claude Code features](#how-skills-compare-to-other-claude-code-features)
- [Sharing skills](#sharing-skills)
- [Troubleshooting skills](#troubleshooting-skills)

## What a skill is

A skill is a markdown file that teaches Claude how to do something once, instead of you re-explaining it every time (coding standards, PR review format, commit message style). Claude applies that knowledge automatically whenever it's relevant, without you having to ask.

Agent skills are folders of instructions, scripts, and resources that Claude can discover and use to work more accurately and efficiently. In Claude Code, this is the `SKILL.md` file. The `description` field is what Claude matches against: when you ask Claude to review a PR, it compares your request to the descriptions of all available skills and activates the one that matches.

Skills live in two main places, depending on who needs them:

- **Personal**: `~/.claude/skills`. Follows you across every project. This is where your own preferences live: commit message style, documentation format, how you like code explained.
- **Project**: `.claude/skills` at the repo root. Anyone who clones the repo gets these automatically. This is where team standards live: brand guidelines, review checklists, shared conventions.

Skills are unique among Claude Code's customization options because they're automatic and task-specific:

- **CLAUDE.md** loads into every conversation, always. Use it for things like "always use TypeScript strict mode."
- **Skills** load on demand, only when the request matches. At startup Claude only loads each skill's name and description, not its full content, so a PR review checklist doesn't sit in context while you're debugging something unrelated. It loads the moment you actually ask for a review.
- **Slash commands** require you to type them explicitly. Skills don't; Claude applies them when it recognizes the situation.

Skills work best for specialized knowledge tied to specific tasks: code review standards, commit message formats, brand guidelines. The rule of thumb: if you find yourself explaining the same thing to Claude repeatedly, that's a skill waiting to be written.

## Creating a skill

To create a skill, make a directory named after the skill inside a skills folder, containing a `SKILL.md` file. For a personal skill, that's under `~/.claude/skills/<skill-name>/SKILL.md`.

`SKILL.md` is a markdown file with a YAML frontmatter block (the section fenced between two `---` lines) followed by the instructions:

```markdown
---
name: pr-description
description: Write a PR description for the current changes. Use when the user asks to summarize a diff or draft a pull request.
---

Read the diff, then write a PR description with these sections:
- Summary: one paragraph on what changed and why.
- Changes: a bullet per notable change.
- Testing: how the change was verified.
```

That's the three parts:
- `name`: identifies the skill (in the frontmatter).
- `description`: tells Claude when to use it; this is the matching criteria (in the frontmatter).
- Everything after the closing `---`: the instructions Claude follows when the skill is active.

Claude Code loads skills at startup, so a new or edited skill requires a session restart before it's picked up. You can verify it's available by asking Claude what skills it has. To test it, make a change (say, edit some files on a branch) and ask Claude to "write a PR description for my changes." Claude announces that it's using the skill, then follows the template, producing the same format every time.

At startup, Claude Code scans four locations for skills, in this priority order when names collide:

1. **Enterprise**: managed settings, highest priority.
2. **Personal**: your home directory config.
3. **Project**: `.claude/skills` in the repo.
4. **Plugins**: installed plugin skills, lowest priority.

Only the name and description of each skill are loaded at startup, not the full content. When you send a request, Claude compares it against those descriptions ("explain what this function does" can match a skill described as "explain code with visual diagrams" because the intent overlaps), asks you to confirm loading it, and only then reads the complete file. The confirmation step keeps you aware of what context Claude is pulling in.

The priority order matters when you clone a repo with a project skill that shares a name with one you already have: enterprise beats personal beats project beats plugins. This lets organizations enforce mandatory standards while still leaving room for individual customization, as long as names don't collide. To avoid accidental collisions, use descriptive names, like `frontend-pr-review` or `security-review`, rather than just `review`.

To update a skill, edit its `SKILL.md`. To remove one, delete its directory. Either way, restart Claude Code for the change to take effect.

## Configuration and multi-file skills

A basic skill only needs `name` and `description`, but the [agentskills.io](https://agentskills.io) open standard supports more.

- `name`: lowercase letters, numbers, and hyphens only, max 64 characters, should match the directory name.
- `description`: required, max 1024 characters, and the single most important field, since it's what Claude matches on.
- `allowed-tools` (optional): restricts which tools Claude can use while the skill is active. Useful for security-sensitive or read-only workflows: if a skill should only ever read files, set `allowed-tools` so Claude can't write, edit, or run bash commands without asking. If omitted, the skill doesn't restrict anything and Claude falls back to its normal permission model.
- `model` (optional): pins which Claude model the skill should run with.

A good description answers two questions: what does this skill do, and when should Claude use it. Be as explicit as you'd be writing a job description for a person: vague ("help with dogs") gives no signal about when to act. If a skill isn't triggering, the fix is almost always to add more keywords that match how you'd actually phrase the request.

**Progressive disclosure.** Skills share Claude's context window with the rest of the conversation, and `SKILL.md` content only loads in when the skill activates. But some skills need references, examples, or utility scripts beyond what fits comfortably in one file, and cramming everything into one huge file is both wasteful of context and unpleasant to maintain. Instead:

- Put essential instructions in `SKILL.md`, keep it under 500 lines.
- Put detailed reference material in separate files that Claude reads only when needed (e.g. a `references/architecture.md` that only loads when someone asks about system design, and never loads otherwise).
- Use a `scripts/` folder for executable code, `references/` for documentation, and `assets/` for images, templates, or data files.
- Link to these supporting files from `SKILL.md`.

Think of `SKILL.md` as a table of contents rather than the whole document.

**Scripts** in a skill directory can execute without their contents ever loading into context, only their output consumes tokens. Tell Claude to *run* the script, not read it. This is well suited to environment validation, data transformations that need to be consistent, and anything more reliable as tested code than as freshly generated code.

## How skills compare to other Claude Code features

Claude Code has several customization options (CLAUDE.md, skills, subagents, hooks, MCP servers), and each solves a different problem. Using the wrong one for a given need means building the wrong thing.

- **CLAUDE.md** loads into every conversation, always. Use it for project-wide standards that should never be violated: "never modify the database schema," framework preferences, coding style.
- **Skills** add knowledge to the *current* conversation on demand. When a skill activates, its instructions join the existing context. Use skills for task-specific expertise that's only relevant sometimes, and for detailed procedures that would clutter every conversation if they were always loaded.
- **Subagents** run in a separate, isolated context. They receive a task, work on it independently, and return results, without polluting the main conversation. Use a subagent when you want to delegate to an isolated execution context, need different tool access than the main conversation has, or want isolation between delegated work and your main thread.
- **Hooks** are event-driven, not request-driven: a hook can run a linter every time Claude saves a file, or validate input before certain tool calls. Skills, by contrast, activate based on what you ask. Use hooks for things that should run on every save, validation before specific tool calls, or automated side effects of Claude's actions.
- **MCP** provides external tools.

These aren't competing options, they're complementary. A typical setup combines a CLAUDE.md for always-on standards, skills for task-specific expertise, and hooks for automated operations, each handling its own specialty rather than forcing everything into one mechanism.

## Sharing skills

A skill only you use is useful; the same skill shared across a team standardizes behavior and gives everyone a consistent experience. There are three ways to share skills, in increasing order of reach:

1. **Commit to the repo.** Place skills in `.claude/skills` at the repo root. Anyone who clones the repo gets them automatically, no extra installation, and pushing an update means everyone gets it on their next pull. This fits team coding standards, project-specific workflows, and skills that reference your codebase structure.
2. **Distribute via plugins.** Plugins extend Claude Code with functionality meant to be shared across teams and projects. Inside a plugin project, create a `skills` directory with the same `<name>/SKILL.md` structure as a project's `.claude/skills`. Once published to a marketplace, other users can install the plugin and get the skill. This fits functionality that isn't too project-specific and can be useful to the wider community.
3. **Deploy via enterprise managed settings.** Administrators can push skills organization-wide. Enterprise skills always take the highest priority, overriding personal, project, and plugin skills with the same name. Reserve this for mandatory standards, security requirements, compliance workflows, anything that genuinely *must* be consistent across the org.

**Subagents and skills.** Subagents don't automatically see your skills, since when you delegate to one, it starts from a fresh, clean context. Built-in agents (explorer, plan, verify) can't access skills at all. Only *custom* subagents you define can use skills, and only the skills you explicitly list.

To wire skills into a custom subagent, add a `skills` field to its `agent.md` under `.claude/agents`, listing which skills to load. Unlike the main conversation (where skills load on demand), a subagent's listed skills load when the subagent starts. Because of that, only list skills that are *always* relevant to that subagent's purpose, a front-end reviewer and a back-end reviewer subagent should carry different skill lists. This pattern is useful when you want isolated task delegation with baked-in expertise, enforced without relying on prompt instructions.

## Troubleshooting skills

When a skill doesn't work, the cause usually falls into one of four buckets: it doesn't trigger, doesn't load, conflicts with another skill, or fails at runtime.

**First step for any of these:** run the agent skills verifier (install via UV, then run it from the skill directory or point it anywhere). It validates structure and syntax before you go hunting manually.

**Doesn't trigger** (skill exists, passes the validator, but Claude never uses it): almost always a description problem. Claude uses semantic matching, so your request needs to overlap with the description's meaning; no overlap, no match. Check the description against how you actually phrase requests, try variations ("help me profile this," "why is this slow," "make this faster"), and add whichever trigger phrases fail as new keywords.

**Doesn't appear at all** (not even listed among available skills): check structure first. `SKILL.md` must live inside a directory named after the skill, not at the skills root, and the filename must be exactly `SKILL.md` (not `Skill.md` or `SKILL.MD`). Run `claude --debug` to see loading errors mentioning the skill by name, often that alone reveals the problem.

**Wrong skill triggers, or Claude seems confused**: two descriptions are too similar. Make them more distinct: specificity doesn't just help Claude decide *when* to use a skill, it also prevents it from stepping on similarly worded skills.

**Personal skill is being ignored**: check for a same-named enterprise (or otherwise higher-priority) skill shadowing it, since enterprise always wins over personal. Renaming your skill to something more distinct is the practical fix; escalating to an admin about the enterprise skill is the alternative, but renaming is more often the higher-odds fix.

**Installed a plugin but its skills don't show up**: clear the cache, restart Claude Code, and reinstall. If skills still don't appear, the plugin's internal structure is probably wrong: run it through the validator.

**Skill loads but fails during execution**: check that any external packages the skill depends on are actually installed (and document that dependency in the skill's description), that scripts have execute permission, and that paths use forward slashes everywhere, including on Windows.

Quick triage checklist:

| Symptom | Likely cause |
|---|---|
| Not triggering | Description doesn't overlap with how you phrase requests |
| Not loading / not listed | Wrong path, wrong filename, or bad YAML |
| Wrong skill used | Descriptions too similar |
| Being shadowed | Higher-priority skill (e.g. enterprise) has the same name |
| Plugin skills missing | Stale cache, clear, restart, reinstall |
| Runtime failure | Missing dependency, missing execute permission, or bad path |
