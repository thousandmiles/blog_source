---
title: Introducing Semantic Code MCP Server
date: 2025-11-23 00:00:00
categories:
  - AI
tags:
  - MCP
  - LSP
  - TypeScript
  - AI
  - Tooling
---

## Project Overview

I'm excited to share my latest project, the **Semantic Code MCP Server**. This tool acts as a powerful bridge between the Model Context Protocol (MCP) and the Language Server Protocol (LSP). It enables AI models to perform semantic code analysis by leveraging the existing ecosystem of Language Servers, specifically starting with TypeScript.

<!-- more -->

### Tech Stack

- **TypeScript**: The primary language for the server implementation.
- **Model Context Protocol (MCP)**: The protocol for exposing tools to AI models.
- **Language Server Protocol (LSP)**: The standard for language intelligence (definitions, references, etc.).
- **Node.js**: The runtime environment.

## Features

The server currently supports several key capabilities that allow AI agents to "understand" the codebase structure rather than just reading text files:

- **Semantic Navigation**:
  - **Go to Definition**: Instantly find where symbols are defined.
  - **Find References**: Locate all usages of a variable or function across the project.
  - **Hover Info**: Retrieve type information and documentation.
- **Relationship Analysis**:
  - **Function Call Graph**: Analyze if function A calls function B using LSP reference data.

## Implementation

The core architecture involves spawning a `typescript-language-server` instance and translating MCP tool calls into LSP JSON-RPC requests. This allows the MCP client (like Claude Desktop) to query the language server as if it were an IDE.

### Tools Exposed

The server exposes the following tools to the MCP client:

- `get_definition`: Find where a symbol is defined.
- `get_references`: Find all usages of a symbol.
- `search_in_file`: Search for a string in a file to find its line and character position.
- `check_function_call`: Analyze if one function calls another.

### Configuration

To use this server, you simply configure your MCP client to run the built Node.js script. Here is an example configuration:

```json
{
  "mcpServers": {
    "semantic-code": {
      "command": "node",
      "args": ["/path/to/lsp-mcp-server/build/index.js"]
    }
  }
}
```

## Challenges & Solutions

One of the main challenges was correctly managing the lifecycle of the LSP process and mapping the asynchronous JSON-RPC responses back to the synchronous-like tool calls expected by MCP. Ensuring robust error handling when the language server is busy or initializing was also critical.

## Results

This project successfully enables AI assistants to navigate codebases with the same precision as a developer using VS Code. Instead of guessing file paths or grepping for strings, the AI can follow symbols and understand relationships, leading to much more accurate code assistance.

## Links

- **Source Code:** [https://github.com/thousandmiles/lsp-mcp-server](https://github.com/thousandmiles/lsp-mcp-server)
