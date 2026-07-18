# Introduction to Model Context Protocol

https://anthropic-partners.skilljar.com/introduction-to-model-context-protocol

MCP (Model Context Protocol) is a communication layer that gives Claude context and tools without you having to write and maintain a pile of integration code yourself. This module explains the architecture (clients, servers, and the messages they exchange), then builds a CLI document chatbot that implements both sides, covering the three server primitives: tools, resources, and prompts.

The course assumes basic Python and uses the `uv` CLI to manage the project.

## Table of contents

- [The problem MCP solves](#the-problem-mcp-solves)
- [Clients, servers, and the message flow](#clients-servers-and-the-message-flow)
- [The project: a CLI document chatbot](#the-project-a-cli-document-chatbot)
- [Tools](#tools)
- [Testing with the MCP Inspector](#testing-with-the-mcp-inspector)
- [Implementing the client](#implementing-the-client)
- [Resources](#resources)
- [Prompts](#prompts)
- [Choosing between tools, resources, and prompts](#choosing-between-tools-resources-and-prompts)

## The problem MCP solves

Imagine building a chat app that lets a user ask questions about their GitHub data, like "what open pull requests do I have across all my repositories?" You'd answer that with tool use: Claude calls a tool that reaches out to GitHub, reads the account, and returns the data.

The catch is that GitHub has an enormous surface: repositories, pull requests, issues, projects, and much more. A complete GitHub chatbot would need a large number of tools, and every one of those tool schemas and functions is code you have to write, test, and maintain. That maintenance burden, spread across every integration you want, is the core problem MCP addresses.

MCP shifts the work of defining and running tools off your server and onto a separate **MCP server**. Think of an MCP server as an interface to some outside service: a GitHub MCP server wraps up a ton of GitHub functionality as a ready-made set of tools. You consume those tools instead of authoring them.

Three questions come up constantly when people first meet MCP:

- **Who writes MCP servers?** Anyone can, but often the service provider ships an official one (AWS releasing an AWS MCP server, for example) with a wide range of tools built in.
- **How is this different from just calling the service's API directly?** If you called GitHub's API directly, you'd be back to authoring the tool schema and function yourself. Adding the MCP server is what saves you that work.
- **Isn't MCP just the same as tool use?** This is a common criticism, usually from people who haven't quite grasped MCP. Tool use and MCP are complementary, not competing. Both involve tools, but MCP is about *who* authors and runs them: someone else wrote the tool and wrapped it in the server, so you don't have to.

## Clients, servers, and the message flow

The **MCP client** is the communication channel between your server and an MCP server. It's your access point to every tool the MCP server implements.

MCP is **transport agnostic**: the client and server can talk over different protocols. A very common setup right now runs both on the same machine and communicates over standard input/output (stdio), which is what the project uses. They can also connect over HTTP, WebSockets, or others.

Once connected, the client and server exchange messages defined by the MCP spec. The four you'll see most:

- **list tools request** (client asks the server what tools it has)
- **list tools result** (server responds with the list)
- **call tool request** (client asks the server to run a specific tool with arguments)
- **call tool result** (server responds with the tool's output)

Here's the full round trip, using the GitHub example. It's involved, but every piece shows up again when you build your own client and server:

1. A user submits a query to your server ("what repositories do I have?").
2. Before calling Claude, your server needs the tool list. It asks the MCP client, which sends a **list tools request** to the MCP server; the server replies with a **list tools result**, and the client hands the tools back to your server.
3. Your server calls Claude with the user's query plus the tool list.
4. Claude decides it needs a tool and responds with a tool use request.
5. Your server no longer runs tools itself. It asks the MCP client to run the tool; the client sends a **call tool request** to the MCP server.
6. The MCP server calls GitHub, gets the repositories back, wraps them in a **call tool result**, and returns that through the client to your server.
7. Your server sends a follow-up request to Claude with the tool result inside a user message.
8. Claude now has everything it needs and writes the final answer, which flows back to the user.

## The project: a CLI document chatbot

The rest of the module builds a CLI chatbot to make all of this concrete. Users work with a small collection of fake documents that live only in memory. You build an MCP client that connects to your own custom MCP server, and the server starts with two tools: one to read a document's contents and one to update them.

One important caveat: on a real project you'd normally build *either* a client *or* a server, not both. You'd publish a server for others to consume, or write a client that connects to servers other people built. This project implements both sides only so you can see the whole picture in one place.

Setup: download the `CLI_project.zip` starter code, extract it, open it in your editor, and follow the `readme.md`. That covers putting your API key in the `.env` file and installing dependencies with or without `uv`. Run it with `uv run main.py` (or `python main.py`), and you should get a working chat prompt before you've added any MCP features.

## Tools

Tools are the primitive meant to be consumed by the language model. The server implementation lives in `mcp_server.py`, and the win here is the official **MCP Python SDK**: it creates the server in one line and generates the JSON tool schema for you, so you skip the verbose hand-written schemas from earlier tool-use work.

Defining a tool is just decorating a typed function:

- `@mcp.tool(name=..., description=...)` marks the function as a tool.
- Type-annotate each argument, and use `Field(description=...)` from pydantic to describe it. The SDK folds all of this (the decorator, the types, the field descriptions) into a generated JSON schema that gets passed to Claude.
- Validate inputs and raise a `ValueError` with a clear message when something's wrong, so Claude can see the error and correct itself.

The two tools in the project (note that the tool `name` in the decorator is separate from the Python function name, and it's the `name` that Claude and the Inspector see):

- `read_contents` (function `read_document(doc_id)`): looks `doc_id` up in the in-memory `docs` dictionary, raises a `ValueError` if it's missing, otherwise returns the contents.
- `edit_document` (function `edit_document(doc_id, old_string, new_string)`): a simple find-and-replace on the document's contents, with the same existence check first.

A good description matters in practice (it's how Claude knows exactly when to reach for a tool), even though the course keeps them short to save time.

## Testing with the MCP Inspector

Because you're using the Python SDK, you get an in-browser debugger for free, so you can exercise a server without wiring it into a real app. Run `mcp dev mcp_server.py` and it starts the server (listening on port 6277, with the inspector UI on 6274 by default) and gives you a URL to open.

In the inspector: click **Connect** to start your server, then use the top menu (resources, prompts, tools, and more) to find the Tools section. **List Tools** shows what you defined; clicking a tool opens a panel on the right where you fill in arguments and **Run Tool** to see the result. You can chain calls to verify behavior, for example run `edit_document` to replace a string, then run `read_contents` on the same ID to confirm the change stuck. (When you test resources, note the Inspector lists direct resources and templated resources separately, so the templated `fetch_doc` shows up under "resource templates," not in the main resource list.)

The inspector is under active development, so the exact UI will drift, but the core flow (connect, pick a primitive, invoke it manually) stays the same. Expect to lean on it a lot while building your own servers.

## Implementing the client

The client lives in `mcp_client.py` as a single `MCPClient` class. It wraps a **client session**, which is the actual connection to the MCP server provided by the SDK. The session needs resource cleanup when the program shuts down, and that cleanup (the `connect`, async enter, and async exit logic) is the reason the class looks bulkier than the server code. Wrapping the raw session in a management class like this is common practice rather than using the session directly.

The client's job is to expose the server's functionality to the rest of your codebase. For tools that's two small async methods:

- `list_tools()`: `await self.session.list_tools()`, then return `result.tools`.
- `call_tool(name, input)`: `await self.session.call_tool(name, input)`, passing the tool name and the arguments Claude provided.

The file has a small testing harness at the bottom so you can run `mcp_client.py` directly, connect to the server, and print the tool list to confirm it works. Once these two methods exist, the rest of the project (already written for you in the `core` directory) can list tools to send to Claude and call a tool when Claude asks, so you can run the CLI and get Claude to actually read and edit documents.

## Resources

Resources let the server expose data to the client for read operations. The rule of thumb is one resource per distinct read. The project adds an `@`-mention feature to show why they exist: when a user types `@`, an autocomplete lists the available documents, and when they submit a message mentioning one, the document's contents are fetched and injected straight into the prompt sent to Claude. That means Claude never has to call a tool to read the file, since the context is already there.

That feature needs two reads, so two resources: one returning the list of document IDs (for the autocomplete), and one returning a single document's contents by ID.

Resources are addressed by a **URI**, sent in a read resource request. There are two kinds:

- **Direct** (also called static): a fixed URI like `docs://documents`.
- **Templated**: a URI with one or more parameters, like `docs://documents/{doc_id}`. The SDK parses the parameter out of the URI and passes it to your function as a keyword argument with the matching name. Add more parameters and they show up as more keyword arguments.

Defining them mirrors tools:

- `@mcp.resource("docs://documents", mime_type="application/json")` over `list_docs()`, returning the dictionary keys as a list.
- `@mcp.resource("docs://documents/{doc_id}", mime_type="text/plain")` over `fetch_doc(doc_id)`, returning the document text after the existence check.

The **MIME type** is a hint to the client about what it's getting back: `application/json` means "this is a JSON string, deserialize it," while `text/plain` means "use it as-is." The SDK automatically serializes whatever you return into a string, and it's the client's job to deserialize when needed. (In a real app you'd often return a full record, like a dictionary with the document's id, content, and author, rather than just the raw text.)

On the client side, one method handles reads:

- `read_resource(uri)`: `await self.session.read_resource(AnyUrl(uri))` (`AnyUrl` imported from pydantic), take `result.contents[0]`, confirm it's a `TextResourceContents`, then branch on its MIME type. If it's `application/json`, return `json.loads(resource.text)`; otherwise return `resource.text` unchanged.

## Prompts

Prompts are pre-defined, tested prompt templates the server exposes to clients. The project demonstrates them with slash commands: typing `/` lists available commands, and a `/format` command reformats a chosen document into Markdown.

The interesting part is *why* this is worth building. A user could already type "reformat report.pdf in Markdown syntax" by hand and Claude would do a reasonable job. The value of a prompt is that you, the server author, can write, evaluate, and thoroughly test a strong prompt tailored to this exact task, then expose it so every client gets that quality without reinventing it. You could hardcode such a prompt into the client instead, but putting it on the server lets a specialized server ship its best prompts for anyone to use.

Defining one, again mirroring tools and resources:

- `@mcp.prompt(name="format", description=...)` over `format_document(doc_id)`, with `doc_id` type-annotated and optionally given a `Field` description.
- The function returns a **list of messages** (using `base.UserMessage` from `mcp.server.fastmcp.prompts`), which are real user/assistant messages you can send straight to Claude. The `doc_id` argument gets interpolated into the prompt text. The project's prompt tells Claude to read the document with a tool, rewrite it in Markdown, and save it back with the edit tool.

On the client side, two methods:

- `list_prompts()`: `await self.session.list_prompts()`, return `result.prompts`.
- `get_prompt(name, args)`: `await self.session.get_prompt(name, args)`, return `result.messages`. The `args` dictionary (e.g. `{"doc_id": ...}`) is passed to the prompt function as keyword arguments and interpolated into the returned messages.

## Choosing between tools, resources, and prompts

The three server primitives differ in *who controls when each one runs*, and that's the clearest guide to which to reach for:

- **Tools are model-controlled.** Claude alone decides when to call a tool. Use a tool when you want to add a capability Claude can invoke on its own.
- **Resources are app-controlled.** Your application code decides when to fetch a resource, then does something with the data, like populating a UI list or augmenting a prompt. Use a resource when you need data in your app, typically for the UI.
- **Prompts are user-controlled.** A user triggers a prompt directly, by clicking a button or menu item or running a slash command. Use a prompt to package a predefined workflow.

You can see all three in the Claude.ai interface: the suggested-prompt buttons under the chat input are **prompts** (user-triggered, pre-written), "add from Google Drive" is a **resource** (the app decides which documents to list and injects the chosen one's contents), and asking Claude to "use JavaScript to calculate" a value is a **tool** (Claude decides to run it).

The shorthand: tools serve the model, resources serve the app, prompts serve the user. These are high-level guidelines, not hard rules, but they're a reliable starting point for deciding what to build.
