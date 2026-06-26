# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

Copy `.env.example` to `.env` and fill in your `ANTHROPIC_API_KEY`. Python 3.14+ is not supported (dependencies lack pre-built wheels). Pin to 3.13 and install:

```bash
uv python pin 3.13
uv sync
```

FFmpeg must be installed separately for video conversion (`brew install ffmpeg` on macOS).

## Running the app

One or more root directory paths are required — the MCP server will only access files within these paths:

```bash
uv run main.py <root1> [root2] ...
uv run main.py /path/to/videos
uv run main.py /home/user/videos ~/Documents
```

## Architecture

This is a CLI chat app that connects Claude (via the Anthropic API) to an MCP server that exposes file system tools. The two processes communicate over stdio.

**Request flow:**
1. `main.py` starts both the MCP client and the CLI, passing root paths as arguments.
2. `CliApp` (`core/cli.py`) reads user input and streams responses using `prompt_toolkit`.
3. `Chat.run()` (`core/chat.py`) drives the agentic loop: send messages to Claude, handle `tool_use` stop reasons by executing tools, feed results back, repeat until a final text response.
4. `ToolManager` (`core/tools.py`) dispatches tool calls to the right `MCPClient` by scanning each client's tool list.
5. `MCPClient` (`mcp_client.py`) wraps the MCP `ClientSession`. It launches `mcp_server.py` as a subprocess and responds to the server's `list_roots` requests with the root paths provided at startup.
6. `mcp_server.py` implements the MCP server using FastMCP. Every tool call validates the requested path against the client-provided roots before acting.

**Key design: roots via MCP protocol.** Root directories are not passed as CLI args to the server — they are communicated through the MCP roots protocol. The client registers a `list_roots_callback` that returns its roots; the server calls `ctx.session.list_roots()` on each request to enforce path restrictions.

**Layer separation:**
- `core/claude.py` — thin async wrapper around the Anthropic SDK (`chat` and `chat_stream`)
- `core/chat.py` — base agentic loop (model ↔ tool execution cycle)
- `core/cli_chat.py` — extends `Chat` with MCP prompt support
- `core/cli.py` — streaming UI, renders tool calls as boxen-bordered panels
- `core/tools.py` — aggregates tools from all clients, routes calls, builds `ToolResultBlockParam` messages
- `mcp_server.py` — FastMCP server exposing `list_roots`, `read_dir`, `convert_video`
- `mcp_client.py` — async context manager wrapping MCP `ClientSession`
