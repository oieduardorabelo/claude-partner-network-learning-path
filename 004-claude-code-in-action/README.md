# Claude Code in Action

https://anthropic-partners.skilljar.com/claude-code-in-action

This module explains what a coding assistant actually is under the hood, what makes Claude Code stand out, and how to get the most out of it: managing context, making changes, custom commands, MCP servers, GitHub integration, hooks, and the Agent SDK.

## Table of contents

- [What a coding assistant is](#what-a-coding-assistant-is)
- [What makes Claude Code stand out](#what-makes-claude-code-stand-out)
- [Claude Code in action](#claude-code-in-action)
- [Managing context with CLAUDE.md](#managing-context-with-claudemd)
- [Making changes](#making-changes)
- [Controlling the conversation](#controlling-the-conversation)
- [Custom commands](#custom-commands)
- [MCP servers](#mcp-servers)
- [GitHub integration](#github-integration)
- [Hooks](#hooks)
- [The Agent SDK](#the-agent-sdk)
- [Where to go next](#where-to-go-next)

## What a coding assistant is

A coding assistant is a tool that uses a language model to write code and complete development tasks. Under the hood it works much like a person would: given a task (say, fix a bug from an error message), it gathers context (reads the relevant files, understands what's throwing the error), forms a plan, and takes an action (edits a file, runs a test to confirm the fix).

The catch is that the first and last steps require *doing* something in the outside world, and a language model on its own can't do anything but take in text and return text. Ask a bare model what's in `main.go` and it will tell you it can't read files.

**Tool use** is how that gap gets closed. Behind the scenes, the coding assistant appends extra instructions to your request, something like: "if you want to read a file, respond with exactly `read file: <filename>`." The model, wanting to answer your question, replies with that carefully formatted message. The assistant recognizes it, actually reads the file, and feeds the contents back to the model, which then writes its final answer. Every language model works this way; tools are what let a text-only model read files, run commands, and everything else that isn't just generating text.

## What makes Claude Code stand out

Every model uses tools, but not equally well. The Claude models (Opus, Sonnet, Haiku) are particularly strong at understanding what a tool does, when to call it, and how to combine tools in interesting ways to complete complex tasks. That strength is the core of Claude Code, for three reasons:

- **Harder tasks.** Better tool use means Claude can handle more complex, multi-step work.
- **Extensibility.** Claude Code is easy to extend with new tools, and Claude will readily use them. This keeps it relevant as the development world changes; it's an assistant that grows with you.
- **Security.** Claude searches your codebase directly to find relevant code, rather than relying on indexing that ships your whole codebase to an outside server.

## Claude Code in action

Four demos show the range, all powered by that same tool-use ability:

- **Performance optimization.** Asked to optimize the `chalk` library (the 5th most downloaded JavaScript package, 429 million weekly downloads), Claude ran benchmarks, built a to-do list to track progress, wrote a file to isolate one case, used a CPU profiler to find the bottleneck, and implemented a fix, landing a 3.9x throughput improvement on one operation.
- **Data analysis.** Given a CSV of video-streaming user data and asked to analyze churn in a Jupyter notebook, Claude didn't just write code; it executed cells, viewed the results, and customized each successive cell to hone in on what it found.
- **Extending with new tools.** Given the Playwright MCP server (browser control), Claude opened a browser, navigated to a local app, screenshotted the current styling, updated it, and iterated on the design across a few screenshots.
- **GitHub PR review.** With AWS infrastructure defined in Terraform, Claude had a clear picture of how data flows through it. When a pull request added a user's email to a Lambda that writes to an S3 bucket shared with an external partner, Claude's automated review caught the exposure of personally identifiable information (PII), traced the full data flow, and flagged it, catching during development what's easy to miss months after the bucket was first set up.

The throughline: treat Claude Code as a flexible assistant that grows with your team through tool expansion, not a fixed-feature tool.

## Managing context with CLAUDE.md

Context management is the single most important skill for using Claude Code well. A project has dozens or hundreds of files; for any given task there's some ideal amount of information Claude needs. Add irrelevant information beyond that and Claude's effectiveness drops. The goal is to give it just enough, and to point it in the right direction.

**`/init`.** The first time you run Claude Code in a project, run `/init`. Claude takes a deep look at the whole codebase, works out its purpose, architecture, key commands, and critical files, and summarizes all of that into a `CLAUDE.md` file. The contents of that file are included in every request, so it serves two purposes: helping Claude find relevant code faster, and giving you a place to write standing guidance.

**Three levels of `CLAUDE.md`:**
- **Project** (`CLAUDE.md`): generated by `/init`, committed to source control, shared with the team, holds project-specific direction.
- **Local** (`CLAUDE.local.md`): not committed, your personal instructions for this project.
- **Machine/global**: applies to every project you run locally.

**Memory mode (`#`).** Instead of editing `CLAUDE.md` by hand, type a `#` in Claude Code to enter memory mode, then state your instruction in plain language (e.g. "don't write comments so often"). Claude merges it into the file you pick. A good use: telling Claude to always reference a critical file like your database schema, so that context is available on every request without Claude having to go search for it.

**`@` mentions.** Reference a specific file with `@` and its contents are pulled straight into your request. This is a precise way to point Claude at exactly the right files instead of letting it search. You can use the same `@` syntax inside `CLAUDE.md` to always include an important file's contents.

## Making changes

A few features make day-to-day edits smoother:

- **Screenshots.** Paste a screenshot into Claude Code with `Ctrl+V` (note: `Ctrl+V`, not `Cmd+V`, even on macOS) to show Claude exactly which UI element you mean.
- **Plan mode.** Press `Shift+Tab` twice (or once if you're already auto-accepting edits) to make Claude research more of your project and produce a detailed plan before doing anything. You then accept the plan or redirect it if it missed something.
- **Thinking mode.** Trigger phrases like "ultrathink" turn on extended thinking, giving Claude a larger reasoning budget. Different phrases grant progressively larger token budgets.
- **Git.** Claude Code is a capable Git assistant; ask it to stage and commit, and it writes a descriptive commit message.

**Planning vs. thinking:** planning handles *breadth* (tasks that span the codebase or take many steps), thinking handles *depth* (tricky logic or a stubborn bug). They combine for hard tasks. Both consume extra tokens, so there's a cost to leaving them on all the time.

## Controlling the conversation

A handful of shortcuts keep Claude focused, especially in long sessions and when switching tasks:

- **Escape**: interrupts Claude mid-response so you can redirect it.
- **Escape + memory (`#`)**: the fix for repeated mistakes. Stop Claude the moment it starts down a familiar wrong path, then use `#` to record the correction (e.g. the real name of a config file it keeps guessing wrong). Now it won't repeat that error.
- **Escape twice**: rewinds the conversation. It shows all prior messages so you can jump back to an earlier point, keeping the useful context (files Claude already read) while dropping irrelevant back-and-forth like a debugging detour.
- **`/compact`**: summarizes the whole conversation while preserving what Claude has learned about the current task. Use it when Claude has built up real expertise but the history has gotten cluttered.
- **`/clear`**: wipes the conversation entirely for a fresh start. Use it when moving to a completely unrelated task.

## Custom commands

Beyond the built-in slash commands, you can add your own for repetitive tasks. Create a `.claude/commands/` folder in your project and add a markdown file; the filename becomes the command name (`audit.md` becomes `/audit`). The file contents are the instructions Claude runs. Restart Claude Code after adding a command file.

Commands take arguments via a `$ARGUMENTS` placeholder in the command text. Run `/write-test src/auth.ts` and that path lands wherever `$ARGUMENTS` appears. The argument doesn't have to be a file path; it can be any string that gives Claude direction.

## MCP servers

MCP servers add new tools and capabilities to Claude Code, running locally or remotely. The Playwright MCP server, for example, lets Claude control a browser.

Install one from your terminal (not from inside Claude Code):

```
claude mcp add <name> <start-command>
```

The first time Claude uses a new tool, it asks permission. To stop the prompts, open `.claude/settings.local.json` and add an entry to the `allow` array in the form `mcp__<servername>` (note the two underscores). That lets Claude use every tool in that server freely. Restart Claude Code for it to take effect.

A concrete payoff: pointing Claude at a UI-generation app with Playwright, it opened the app in a browser, generated a component, judged the styling itself (it objected to an overused purple-to-blue gradient), rewrote the generation prompt, and produced a noticeably better component. MCP servers open the door to development workflows well beyond code editing.

## GitHub integration

Claude Code's official GitHub integration runs Claude inside a GitHub Action. Set it up by running `/install-github-app`, which walks you through installing the app on GitHub and adding an API key, then auto-generates a pull request adding two workflows:

- **Mentions**: write `@claude` in an issue or PR to hand Claude a task.
- **PR review**: Claude automatically reviews every new pull request.

Both are customizable via the config files in `.github/workflows`, and you can add more actions triggered by other events. Customization options include **custom instructions** (context passed directly to Claude) and wiring in **MCP servers** (for example, starting your dev server before Claude runs and giving it Playwright so it can test the app in a browser).

One important gotcha: when Claude Code runs inside an Action, you must explicitly list every permission you're granting. And if you're using an MCP server, you have to list each of its tools individually. There's no shortcut like the `mcp__<servername>` wildcard you can use locally, so a many-tool server means a long permissions list.

## Hooks

Hooks run commands before or after Claude runs a tool. Uses include auto-formatting a file after it's written, running tests after an edit, blocking access to certain files, or feeding quality checks back to Claude.

- **Pre-tool-use hooks** run *before* the tool. They can inspect what Claude wants to do and *block* it, sending an error message back.
- **Post-tool-use hooks** run *after* the tool. Too late to block, but you can do follow-up work and send feedback that Claude can act on.

Configure hooks in a settings file (global, project, or personal), either by hand or with the `/hooks` command. The config lives under a `hooks` key with a section per event type; each section is a list of matcher groups, and each group names the tools to watch and the commands to run:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "read|grep",
        "hooks": [
          { "type": "command", "command": "node ./hooks/read_hook.js" }
        ]
      }
    ]
  }
}
```

Each hook has a **matcher** naming the tools it targets and a **command** to run. When the command runs, Claude feeds it the tool-call details as JSON on stdin (fields like `session_id`, `hook_event_name`, `tool_name`, `tool_input`, and, on post-tool-use, `tool_response`). The command parses that, decides what to do, and signals its intent through the **exit code**: `0` allows the tool call, `2` blocks it (pre-tool-use only). On a blocking exit, anything the command wrote to stderr goes back to Claude as feedback, so you can deny the call and explain why at once. Restart Claude Code after changing hooks.

**Worked example: block reading `.env`.** This has to be a pre-tool-use hook (a post hook would fire after the file was already read). Two tools can read file contents, `read` and `grep`, so the matcher is `read|grep` (a pipe between tool names). If you're unsure of a tool's exact name, just ask Claude for a list of its available tools. The command (a small Node script) reads the JSON from stdin, checks whether the target path includes `.env`, and if so calls `console.error(...)` (writing to stderr for Claude's benefit) and `process.exit(2)` to block it.

**Two hooks worth stealing:**
- **Type-check on edit.** Claude will happily change a function's signature but often won't update the call sites, introducing a type error it doesn't notice. A post-tool-use hook that runs `tsc --noemit` after any TypeScript edit catches the error and feeds it back, prompting Claude to go fix the call sites. The same idea works for any typed language with a type checker, or for untyped languages by running tests instead.
- **Duplicate-query prevention.** On big tasks Claude loses focus and writes a new query (or a whole new file) instead of reusing an existing one. A hook watching the queries directory launches a *separate* Claude instance (via the SDK) to compare the change against existing code; if it's a duplicate, the hook exits `2` and tells the original Claude to reuse what's already there. The trade-off is extra time and cost per edit, so only watch your most important directories.

**Beyond pre/post-tool-use**, Claude Code has more hook events: `Notification` (Claude needs permission, or has been idle 60 seconds), `Stop` (Claude finished responding), `SubagentStop` (a subagent/Task finished), `PreCompact` (before a compact), `UserPromptSubmit` (you submit a prompt, before Claude processes it), `SessionStart`, and `SessionEnd`. The tricky part is that the stdin your command receives differs by hook type (and, for tool hooks, by which tool was called), so you don't always know the exact shape. A useful trick: add a temporary catch-all hook with matcher `*` and command `jq . > post-log.json` to dump the real input to a file, then inspect it to see exactly what your command will get.

## The Agent SDK

The SDK lets you run Claude Code programmatically, from the CLI or a TypeScript/Python library. It's the same Claude Code with the same tools; it's just driven by your code instead of the terminal. It shines as part of a larger pipeline, adding intelligence to a workflow (the duplicate-query hook above used it to launch a review). Running it streams the raw message-by-message conversation between your local Claude Code and the model, with Claude's final response as the last message.

The course video is out of date in two ways, so follow this instead. First, it uses an older package name that no longer works: the current package is **`@anthropic-ai/claude-agent-sdk`** (not `@anthropic-ai/claude-code`, which is the CLI itself and can't be imported). Second, the video says the SDK is read-only by default; in the current package it has the full tool set by default, and you narrow it by passing `allowedTools`.

Install it, then a minimal TypeScript example:

```
npm install @anthropic-ai/claude-agent-sdk
```

```javascript
import { query } from "@anthropic-ai/claude-agent-sdk";

const prompt = "List the files in the current directory";

for await (const message of query({ prompt })) {
  console.log(JSON.stringify(message, null, 2));
}
```

To restrict which tools it can use, pass `allowedTools` (the SDK equivalent of the CLI's `--allowedTools` flag):

```javascript
for await (const message of query({
  prompt,
  options: { allowedTools: ["Read", "Glob"] },
})) {
  // ...
}
```

The SDK supports everything the CLI does (custom system prompts, MCP servers, hooks, subagents, session resumption), and is best used inside helper commands, scripts, and hooks rather than on its own.

## Where to go next

Claude Code is under constant, active development, so watch its homepage for new features. Two habits to build:

- **Experiment.** Write custom commands, add instructions to your `CLAUDE.md`, and try MCP servers beyond the ones here.
- **Automate.** Use the GitHub integration to delegate common tasks to Claude automatically, triggered by events in your repository.
