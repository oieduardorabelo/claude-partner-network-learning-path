# Building with the Claude API

https://anthropic-partners.skilljar.com/claude-with-the-anthropic-api

This module covers the Claude API end to end: making requests, engineering and evaluating prompts, tool use, retrieval-augmented generation, advanced request features, the Model Context Protocol, Claude Code, and the difference between workflows and agents.

## Table of contents

- [Models and making requests](#models-and-making-requests)
- [Prompt engineering and evaluation](#prompt-engineering-and-evaluation)
- [Tool use](#tool-use)
- [Retrieval-augmented generation (RAG)](#retrieval-augmented-generation-rag)
- [Advanced request features](#advanced-request-features)
- [Model Context Protocol (MCP)](#model-context-protocol-mcp)
- [Claude Code and Computer Use](#claude-code-and-computer-use)
- [Agents vs. workflows](#agents-vs-workflows)
- [Where to go next](#where-to-go-next)

## Models and making requests

Claude ships in three families, each optimized for a different trade-off. **Opus** is for the deepest reasoning on complex, multi-step tasks, at higher cost and latency. **Sonnet** is the balanced default for most practical use. **Haiku** is for speed and cost on high-volume or real-time work. Only Opus and Sonnet support reasoning (spending extra time thinking through a hard problem before answering); Haiku doesn't. It's common to mix models within one application rather than pick a single one for everything.

A typical integration never lets the client talk to the Anthropic API directly, since that would leak the API key. Instead: client sends text to your server, your server calls the API with an SDK or plain HTTP, passing an API key, model name, a `messages` list, and a `max_tokens` limit. Claude tokenizes the input, embeds and contextualizes it, and generates output token by token until it hits `max_tokens` or an end-of-sequence token. The response, generated text plus token usage plus a `stop_reason`, flows back to the client.

In practice, that means: install the SDK (`pip install anthropic python-dotenv`), store the key in a `.env` file (never committed), load it, and call `client.messages.create(model=..., max_tokens=..., messages=[...])`. Messages are dictionaries with a `role` (`user` or `assistant`) and `content`. The generated text lives at `message.content[0].text`. Treat `max_tokens` as a safety cap, not a target length: Claude doesn't pad a response to reach it.

**Multi-turn conversations.** The API is stateless, it stores nothing between requests. Context has to be rebuilt by hand: append the assistant's reply to your own message list, add the next user message, and send the whole history again each time. Helper functions (`add_user_message`, `add_assistant_message`, `chat`) make this pattern manageable.

**System prompts** control *how* Claude responds, not what it responds with. Assigning it a role ("You are a patient math tutor") changes its behavior for the same question (hints instead of answers) without changing the underlying task. One implementation gotcha: passing `system=None` straight into `create()` throws an error, so build the request params conditionally and only include the `system` key when a prompt is actually set.

**Temperature** (0 to 1) controls how deterministic generation is by reshaping the probability distribution over the next token. Near 0 is best for factual/extraction tasks that need consistency; near 1 suits brainstorming and creative writing. It doesn't guarantee variation, just increases the odds of it.

**Streaming** displays the response chunk by chunk instead of making the user wait 10 to 30 seconds for the full thing. The event stream includes `message_start`, `content_block_start`, `content_block_delta` (the actual text chunks), and `content_block_stop`/`message_stop`. The SDK's `client.messages.stream()` with `.text_stream` gives you just the text; `.get_final_message()` reassembles everything for storage.

**Controlling output** without touching the prompt: pre-fill the assistant's turn with the start of the response you want (Claude continues from that exact point, not from a fresh sentence), and/or set a stop sequence so generation halts the instant that string appears. Combined, prefill an opening delimiter like `` ```json `` and stop on `` ``` ``, and this produces clean structured output (JSON, code, etc.) with none of Claude's usual explanatory wrapper text. Watch out for prefilling bare backticks alone: Claude still thinks it's opening a markdown code block and appends its own language tag right after, so include the language yourself in the prefill (`` ```json ``) to get truly clean output.

## Prompt engineering and evaluation

Writing a prompt and shipping it after eyeballing one or two outputs is a trap, and so is tweaking it against a handful of custom inputs. The reliable path is an evaluation pipeline that scores prompt performance objectively before you iterate.

**A typical eval workflow**, six steps: write a draft prompt, build a dataset of test inputs (a handful or thousands, hand-written or generated by Claude), interpolate each input into the prompt template, send each variation to Claude and collect outputs, grade each response (e.g. 1 to 10) and average the scores, then use the score to decide what to change and repeat. There's no single standard tool for this; a custom pipeline is a fine starting point.

A minimal pipeline needs three functions: `run_prompt` (merge a test case into the prompt template, call Claude), `run_test_case` (call `run_prompt`, grade the result), and `run_eval` (loop over the dataset, aggregate results).

**Grading** falls into three types:
- **Code graders**: programmatic checks (valid JSON/Python/regex, length, keyword presence). Fast and objective, but only catches what you thought to check.
- **Model graders**: a second Claude call scores the first call's output. Ask for a list of strengths and weaknesses plus a score, not just a bare number, since a score-only prompt tends to cluster its answers at a lazy middle (around 6 on a 1-to-10 scale).
- **Human graders**: the most flexible option, but also the most time-consuming and tedious to run at scale.

A combined score (e.g. `(model_score + syntax_score) / 2`) captures both semantic quality and technical validity when the output type is known in advance.

**Technique-by-technique, applied to a real prompt** (a one-day athlete meal plan), scores compounded roughly like this. Note the jumps below are larger than you'd typically see, since the demo deliberately runs a weak, fast model to make the effect of each technique obvious:
- **Being clear and direct**: start with an action verb and a plain statement of the task ("Generate a one-day meal plan that meets these dietary restrictions..."). This alone took an example prompt from 2.32 to 3.92.
- **Being specific**: add guidelines. Two kinds exist. *Attributes* describe desired qualities of the output (length, structure, format) and are useful in almost every prompt. *Steps* describe a reasoning procedure to follow and are useful for complex problems where you want Claude to consider angles it wouldn't on its own. Adding attribute-style guidelines took the same example from 3.92 to 7.86.
- **Structuring with XML tags**: wrap interpolated content in descriptive tags (`<athlete_information>`, `<sales_records>`) so Claude can tell where one kind of input ends and another begins. This matters more as more content gets interpolated, but helps even for short blocks.
- **Providing examples (one-shot / multi-shot)**: show sample input/ideal-output pairs, wrapped in tags, ideally with a short note on *why* the example output is good. Best used for corner cases (sarcasm, edge scenarios) and precise formatting requirements. Reuse your own highest-scoring eval outputs as examples: the eval pipeline's `output.html` report is where you'd go find one, since it lets you open the report and copy the input/output pair sitting at a 10.

## Tool use

By default Claude only knows what was in its training data: no live data, no side effects. Tool use closes that gap. Claude tells you it needs a piece of external information or an action performed, your code performs it, and you hand the result back for Claude to finish its answer.

The loop: send a request plus tool definitions, Claude decides it needs a tool and asks for it (rather than answering directly), your server executes the corresponding code, you send the result back to Claude in a follow-up request, and Claude produces the final answer using that result.

**Tool functions** are just well-named, input-validated Python functions. Raise a clear `ValueError` (or similar) on bad input; Claude sees that error message and can retry with corrected arguments.

**Tool schemas** describe those functions to Claude in JSON Schema: a `name`, a `description` (a few sentences on what it does, when to use it, what it returns), and an `input_schema` for the arguments. A practical shortcut: hand your function to Claude.ai along with Anthropic's tool-use docs and ask it to draft the schema. In the SDK, wrap the schema dict in `ToolParam`.

**Message shape changes once tools are involved.** An assistant turn can now contain multiple content blocks: a text block plus one or more `tool_use` blocks (each with a function name, arguments, and a unique ID). You must append the *entire* `content` list, not just the text, to your maintained history, and your `add_user_message`/`add_assistant_message` helpers need to handle multi-block content.

**Sending results back**: build a `tool_result` block with `tool_use_id` matching the original request's ID, `content` (the function's output, usually JSON-stringified), and `is_error` (true if the tool raised). This block goes in a *user* message, not an assistant one, and the follow-up request must still include the original tool schemas even if you don't expect another tool call.

**Chains of tool calls.** A single user query might need several tools in sequence (get the current date, then add a duration to it, then set a reminder). Since you can't know the chain length in advance, wrap the whole thing in a loop: call Claude, check `stop_reason`. If it's `"tool_use"`, run every `tool_use` block found in the response (via a small dispatcher function that matches tool name to implementation), append all the `tool_result` blocks as one user message, and loop again; otherwise the response is final and the loop ends. Adding a new tool to an existing setup is then just three steps: add its schema to the tools list, add a case to the dispatcher, implement the function.

A few extensions worth knowing:
- **Batch tool**: Claude technically *can* emit multiple `tool_use` blocks in one turn but rarely does unprompted. A "batch" tool that accepts a list of `{tool, arguments}` invocations, executed together server-side, forces genuinely parallel tool execution in one round trip instead of several sequential ones.
- **Tools for structured data extraction**: an alternative to prefill/stop-sequence extraction. Define a tool whose input schema *is* your desired JSON shape, force Claude to call it with `tool_choice: {"type": "tool", "name": "..."}`, and read the structured result straight off `response.content[0].input`. More setup than the prefill trick, but more reliable, so reach for it when reliability matters more than simplicity.
- **Fine-grained tool streaming**: by default the API buffers a tool call's JSON arguments until each key/value pair validates, so chunks arrive in bursts. Enabling `fine_grained: true` streams raw JSON chunks immediately (lower latency, more responsive UI) at the cost of the client having to handle occasionally invalid JSON itself.
- **Built-in tools**: Claude ships a couple of tools with schemas built in. The **text editor tool** gives Claude file read/write/replace/undo capabilities, but *you* still implement the actual filesystem operations; only a schema "stub" needs sending, and it auto-expands server-side. The **web search tool** needs no custom implementation at all, just a schema with a `max_uses` cap and optional `allowed_domains`; results come back as search-result blocks plus citation blocks you can render as source attributions (useful for constraining Claude to authoritative domains, e.g. NIH.gov for medical claims).

## Retrieval-augmented generation (RAG)

The problem RAG solves: how do you let Claude answer questions against a document that's too large to fit in a prompt (hundreds or thousands of pages)? Just stuffing the whole thing in doesn't scale: token limits, rising cost, slower responses, and degraded quality as prompts get longer. RAG instead chunks the document ahead of time and, per-query, retrieves only the chunks relevant to that question. The tradeoff: a retrieved chunk isn't guaranteed to contain everything Claude needs to answer well, since related context can live in a completely different section of the document.

**Chunking strategies:**
- **Size-based**: split into equal-length strings. Simple, most common in production, but risks cutting words/context mid-thought. An overlap between neighboring chunks (some duplicated text) mitigates that at the cost of redundancy.
- **Structure-based**: split on document structure (markdown headers, sections). Best results, but only works when the source is reliably structured.
- **Semantic-based**: group sentences by NLP-measured similarity. Most sophisticated, most complex to implement.

There's no universally best strategy, it depends on how reliably structured your source documents are. As a practical default, chunk-by-character is the old reliable that works most of the time; splitting on sentences is riskier (a stray period in code or an abbreviation breaks it), and splitting on sections gives the cleanest results but only when the document's structure is guaranteed.

**Embeddings** turn a chunk of text into a long list of numbers (roughly -1 to 1) representing its meaning; two chunks with similar meaning end up with similar vectors. Anthropic recommends Voyage AI for generating embeddings. Semantic search over a document set means embedding the query the same way and finding the closest stored vectors.

**The full pipeline, seven steps:** chunk the source documents, embed each chunk, normalize the vectors (scale each to a magnitude of 1.0 so they're directly comparable, which the embedding API handles automatically), and store them in a vector database. That's pre-processing, done once. Per query: embed the user's question with the same model, run a similarity search to find the closest stored chunks, then assemble those chunks plus the question into a prompt for Claude. The similarity search is typically **cosine similarity**, the cosine of the angle between two vectors (closer to 1 means more similar). **Cosine distance** is `1 - similarity` (closer to 0 means more similar).

**Beyond plain vector search:**
- **BM25** (short for "Best Match 25", a **lexical search** algorithm) complements semantic search by catching exact-term matches that embeddings can miss. It tokenizes the query, weights terms by rarity across the corpus (rare terms matter more than common ones), and ranks chunks by weighted term overlap. Running BM25 and vector search in parallel and merging the two (**hybrid search**) covers more ground than either alone: a query for a specific incident ID, for example, can get matched by semantic search to an unrelated but topically similar section, while BM25 catches the exact ID and pulls in the right one.
- **Reciprocal Rank Fusion** merges ranked result lists from different search methods: each document's score is the sum of `1/(rank + 1)` across every method it appeared in, so documents that rank well in *multiple* methods win out.
- **Reranking** adds an LLM call after retrieval: hand Claude the query plus the candidate chunks (by ID, not full text, for efficiency) and ask it to reorder them by relevance. Costs latency but catches nuance that pure lexical/semantic scoring misses, e.g. correctly matching "ENG team" against "engineering team" content over an unrelated but lexically similar match.
- **Contextual retrieval** fixes the problem where an isolated chunk loses the context of its parent document. Before embedding, send the chunk plus (if it fits) the surrounding document to Claude and ask it to generate a short situating blurb, then prepend that blurb to the chunk before indexing. For documents too large to include whole, use a selective strategy: the first few chunks (for a summary/abstract) plus the chunks immediately preceding the target chunk.

## Advanced request features

**Extended thinking** lets Claude reason internally before producing its final answer, visible to you as a separate thinking block. It costs extra tokens and latency, so turn it on only after prompt-engineering alone fails to hit the accuracy bar, measured via evals, not guesswork. Set a `thinking_budget` (minimum 1024 tokens) and make sure `max_tokens` exceeds it. Thinking blocks carry a cryptographic signature so they can't be tampered with when replayed in later turns; occasionally a thinking block comes back redacted (encrypted) by safety systems but is still passed along for conversation continuity. Anthropic also provides a special test string you can send to force a redacted thinking block on demand, useful for testing that your UI handles the redacted case without waiting to hit one naturally.

**Images** can be included as blocks in a user message (base64 data or a URL), up to 100 per request, and they consume tokens based on pixel dimensions. Getting good results depends heavily on prompt quality: step-by-step instructions, one/multi-shot examples, and explicit verification steps all matter.

**PDFs** work the same way as images, with `type: "document"` and `media_type: "application/pdf"` instead of the image equivalents. Claude reads text, embedded images, charts, and tables from the PDF directly.

**Citations** let Claude point back at exactly where in a source document (or plain text) a claim came from. Enable with `"citations": {"enabled": true}` and a `title` on the source; PDF sources get page-location citations, plain text sources get character-offset citations. Useful for building citation-popover UIs so users can verify Claude isn't speaking purely from memory.

**Prompt caching** avoids Claude reprocessing identical input across requests. Normally every request's input is processed from scratch and thrown away afterward; caching keeps that processing around (for up to an hour) so an identical prefix on a later request is retrieved instead of recomputed. The rules:

- **Set an explicit breakpoint.** The shorthand `content: "string"` form can't carry one, so use the longhand block form with a `cache_control: {"type": "ephemeral"}` field. Everything up to and including a breakpoint is cached; any change to that content invalidates it.
- **Order is tools, then system prompt, then messages.** You can set up to 4 breakpoints anywhere across that order (tool schemas, the system prompt, or individual message blocks, whether text, image, tool use, or tool result), so distinct parts of a request each get their own cache layer independently.
- **There's a 1024-token minimum** for content to be cacheable.
- **Watch the usage fields.** Responses report `cache_creation_input_tokens` (written on first use) and `cache_read_input_tokens` (served from cache) so you can confirm it's working.

**Code execution and the Files API.** The Files API lets you upload a file once and reference it by ID in later requests instead of re-sending raw bytes each time. Code execution is a built-in server-side tool: Claude writes and runs Python inside an isolated, network-less Docker container, can iterate on results, and can produce output files (plots, reports) that you download by ID. Combine the two by uploading data via the Files API and pointing the code-execution tool at the resulting file ID.

## Model Context Protocol (MCP)

**The problem MCP solves:** wiring Claude up to an external service (GitHub, Slack, a database) the manual way means writing and maintaining a tool schema and implementation for every operation you want exposed. For a service with many operations, that's a lot of ongoing work.

**MCP is a client/server protocol** that shifts that burden onto whoever maintains the MCP server. An **MCP server** wraps a service's functionality as ready-to-use tools (plus, optionally, resources and prompts); your application runs an **MCP client** that talks to it. Anyone can write an MCP server, and often the service provider ships an official one. MCP doesn't replace tool use, it changes *who* implements the tools: the server author, not you.

**Client/server communication** is transport-agnostic (stdio, HTTP, WebSockets, with stdio and client/server on the same machine being the common local setup) and follows a request/result pattern: the client asks the server to list its tools, the server responds with the list; later, the client asks the server to call a specific tool with arguments, the server executes it and returns a result. The full round trip for a user query: your server asks the MCP client for the tool list, forwards that list plus the user's query to Claude, Claude requests a tool call, your server asks the MCP client to run it, the MCP client calls the MCP server, and the result flows back through the same chain to Claude and finally to the user.

**Writing an MCP server** (with the official Python SDK) means decorating plain functions instead of hand-writing JSON schemas: `@mcp.tool(name=..., description=...)` over a function with typed parameters (using `Field()` from pydantic for per-argument descriptions) auto-generates the tool's schema. The **MCP Inspector** (`mcp dev server_file.py`) is an in-browser debugger for exercising a server's tools manually: connect, pick a tool, fill in parameters, run it, all without wiring up a real client first.

**Writing an MCP client** typically wraps the SDK's `ClientSession` in your own class for lifecycle/resource management, exposing `list_tools()` (calling `session.list_tools()`) and `call_tool()` (calling `session.call_tool(name, input)`) to the rest of your app.

**Resources** are the read-only counterpart to tools: instead of Claude deciding to call something, resources let a client proactively pull data by URI (a *direct* resource like `docs://documents`, or a *templated* one like `docs://documents/{doc_id}` whose parameters the SDK parses automatically). Defined via `@mcp.resource(uri, mime_type=...)`; the client reads one with `session.read_resource(AnyURL(uri))` and parses the result based on its declared MIME type (JSON vs. plain text). The distinction from tools: resources fetch data reactively, on reference (e.g. an `@`-mentioned document), while tools perform actions Claude decides to invoke.

**Prompts** let a server ship its own pre-tested prompt templates rather than leaving users to write ad-hoc ones. Defined with `@mcp.prompt`, returning a list of ready-to-send messages, optionally parameterized (e.g. a "reformat this document" prompt that takes a document ID). Clients typically surface these as slash-command-style autocomplete. `list_prompts()`/`get_prompt(name, arguments)` on the client side mirror the tool-listing pattern.

The course's own summary of when to reach for each primitive: **tools are model-controlled** (Claude alone decides when to call one), **resources are app-controlled** (your application code decides when to fetch one, e.g. to populate an autocomplete list or satisfy an `@`-mentioned document), and **prompts are user-controlled** (a person triggers one directly, via a button or slash command). Tools serve the model, resources serve the app, prompts serve the user.

## Claude Code and Computer Use

Claude Code and Computer Use are Anthropic's own applications, and this part of the course treats them as worked examples of agent design more than new concepts on top of what's above.

**Claude Code** can search/read/edit files, reach the web and terminal, and act as an MCP client to pull in more tools. After installing (`npm install`, then `claude` to log in), running `/init` scans the codebase and writes a `CLAUDE.md` capturing architecture and conventions, which then gets included automatically in future context. You can pass `/init` extra instructions (e.g. "include notes on defining MCP tools"), rerun it later as the project evolves, or edit the file by hand. A `#` prefix on a message is a shortcut for saving a quick note to memory without opening the file; it prompts you to pick which of the three memory levels (Project, Local, or User) the note goes to.

Effective use looks less like "generate this code" and more like collaborative engineering. Two patterns work well: (1) point Claude at relevant files, have it analyze them, have it *plan* before writing anything, then implement the plan, or (2) drive it test-first, getting it to propose tests, picking which ones matter, running `/clear` on the conversation so it can't "cheat" off its own prior solution, then having it write code until they pass. Claude can also stage, commit, and stash git changes on request. More detailed instructions consistently produce better results; treat it as an effort multiplier, not a code vending machine.

Claude Code's MCP-client support means `claude mcp add <name> <startup-command>` connects it to any MCP server (a document-processing server, Sentry, Jira, Slack, etc.) to expand what it can do without touching its core.

**Parallelizing Claude Code**: running several instances at once on the same repo causes conflicts if they touch the same files. Git worktrees solve this by giving each instance an isolated full copy of the project on its own branch; work happens in isolation, gets committed, then merged back (Claude can handle both the conflict resolution and worktree cleanup). Custom slash commands under `.claude/commands` (markdown files using an `$ARGUMENTS` placeholder) can automate the worktree lifecycle, letting one developer effectively run a small team of parallel engineering agents.

**Automated debugging**: a scheduled job (e.g. a nightly GitHub Action) pulls recent production logs (e.g. from CloudWatch), has Claude deduplicate and analyze the errors, generate fixes, and open a pull request. Useful for catching environment-specific issues (bad config, wrong model IDs) that never show up locally.

**Computer use** extends the same tool-call loop to actual screen interaction: Claude gets a special tool schema (that expands server-side into actions like mouse move, click, and screenshot), requests an action, your environment (typically an isolated Docker container) executes it and screenshots the result, and that screenshot comes back to Claude as the next tool result. Claude never touches a computer directly, the tool system plus a developer-provided execution environment is what makes it work. This is well suited to automated QA: describing a test scenario in chat and letting Claude navigate the UI, run through it, and report pass/fail.

## Agents vs. workflows

Both are ways to accomplish a task Claude can't do in a single request. The difference is whether you know the steps in advance.

**Workflows** are a fixed sequence of Claude calls (and other code) for a task whose steps are already known. Because the steps are fixed, workflows are easier to test, more reliable, and generally have higher success rates. The tradeoff is they can't adapt to a task they weren't built for. Common workflow shapes:
- **Evaluator-optimizer**: a producer generates output, an evaluator (the course calls it a "grader") checks it against the goal, and the loop repeats with feedback until it passes. Example: describe an uploaded image, model a 3D object from the description, render it, compare the render to the original, and redo with feedback if it doesn't match.
- **Parallelization**: split one overloaded prompt (weighing several candidate options against the same criteria at once) into several focused parallel prompts, one per candidate, plus a final aggregation step that compares the results. A material-selection prompt, for example, fans out into a separate "would metal work here / would polymer work here / ..." call per candidate material rather than one prompt juggling all of them. Easier to improve and evaluate each piece independently, and easy to extend with new subtasks.
- **Chaining**: break a large task into sequential single-purpose steps instead of one giant prompt. A social-media video pipeline is a clean example: search trending topics, Claude picks the most promising one, Claude researches it, Claude writes a script, generate the video, then post it, each step its own call rather than one prompt trying to do all of it at once. Chaining is also the fix when Claude keeps ignoring some of a long list of "don't do X" constraints in a single pass; splitting the work (or adding a dedicated rewrite step that targets the specific violations found) gets more consistent results than repeating the same instruction harder.
- **Routing**: an initial call classifies the input, then a specialized pipeline (its own prompt/tools tuned for that category) handles it, e.g. routing "educational" topics to a more explanatory tone and "entertainment" topics to punchier, trend-aware language.

**Agents** are the flexible alternative: instead of predetermined steps, Claude is given a small set of tools and plans its own path to the goal, combining tools in ways you didn't explicitly script. Unlike a workflow's fixed input shape, an agent can ask the user for missing information mid-task (e.g. "when does my 90-day warranty expire?" prompting Claude to ask when the item was purchased), or generate something for the user to approve before continuing, like drafting a sample cover image first. The tradeoff is real: agents have a lower task-success rate and are harder to test than workflows, since their execution path isn't fixed. Use agents when the task's steps genuinely can't be known ahead of time; use workflows whenever they can, since workflows are generally the more reliable choice. Two design principles help agents work well:
- **Prefer abstract, general-purpose tools over narrow, task-specific ones.** Claude Code itself is the example: `bash`, `web_fetch`, and file read/write, not `refactor_tool` or `install_dependencies`. A handful of generic tools (e.g. `get_current_datetime` + `add_duration` + `set_reminder`) can be recombined to solve a much wider range of requests than an equivalent set of hyper-specific ones.
- **Give the agent a way to inspect its environment after acting**, since it often can't predict the exact result of an action. Computer use takes a screenshot after every click; a coding agent reads a file before editing it; a video-generation agent might extract timestamped captions or sample frames to verify its own output before treating a step as done. Without this feedback loop, an agent operates blind to whether its actions actually worked.

The practical recommendation: default to workflows for reliability, and reach for agent-style flexibility only when the task truly can't be pinned down in advance. Users care far more about a task reliably working than about how sophisticated the underlying architecture is.

## Where to go next

The course explicitly didn't have time to cover everything, and the instructor calls out a specific list of follow-up topics worth researching on your own once you're through this material:

- **Agent orchestration**: getting multiple agents to work together on a task, rather than one agent operating alone.
- **Agent evaluation and monitoring**: the evals section of this course focused on prompts; scoring and tracking an *agent's* end-to-end performance in production is a related but distinct problem.
- **Agentic RAG**: a variation on the RAG pipeline covered here where retrieval itself becomes something an agent decides how and when to do, rather than a fixed step.
- **RAG evaluation**: systematically measuring retrieval quality, separate from evaluating the final generated answer.
- **Tool evaluation**: the same instinct as prompt evals, applied to tool descriptions, checking that a tool's schema and description actually steer Claude toward calling it correctly and at the right time.

None of these are covered in depth elsewhere in this README; they're flagged here as a starting point for further reading, not as material this course actually taught.
