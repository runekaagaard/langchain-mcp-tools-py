# MCP To LangChain Tools Conversion Utility / Python [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/hideya/mcp-langchain-client-ts/blob/main/LICENSE) [![pypi version](https://img.shields.io/pypi/v/langchain-mcp-tools.svg)](https://pypi.org/project/langchain-mcp-tools/)

This package is intended to simplify the use of
[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
server tools with LangChain / Python.

It contains a utility function `convertMcpToLangchainTools()`.  
This function handles parallel initialization of specified multiple MCP servers
and converts their available tools into an array of
[LangChain-compatible tools](https://js.langchain.com/docs/how_to/tool_calling/).

A typescript version of this utility library is available
[here](https://www.npmjs.com/package/@h1deya/langchain-mcp-tools)

## Requirements

- Python 3.11+

## Installation

```bash
pip install langchain-mcp-tools
```

## Quick Start

`convert_mcp_to_langchain_tools()` utility function accepts MCP server configurations
that follow the same structure as
[Claude for Desktop](https://modelcontextprotocol.io/quickstart/user),
but only the contents of the `mcpServers` property,
and is expressed as a `dict`, e.g.:

```python
    mcp_configs = {
        'filesystem': {
            'command': 'npx',
            'args': ['-y', '@modelcontextprotocol/server-filesystem', '.']
        },
        'fetch': {
            'command': 'uvx',
            'args': ['mcp-server-fetch']
        }
    }

    tools, mcp_cleanup = await convert_mcp_to_langchain_tools(
        mcp_configs
    )
```

The utility function initializes all specified MCP servers in parallel,
and returns LangChain Tools (`List[BaseTool]`)
by gathering all available MCP server tools,
and by wrapping them into [LangChain Tools](https://js.langchain.com/docs/how_to/tool_calling/).
It also returns a cleanup callback function (`McpServerCleanupFn`)
to be invoked to close all MCP server sessions when finished.

The returned tools can be used with LangChain, e.g.:

```python
    # from langchain.chat_models import init_chat_model
    llm = init_chat_model(
        model='claude-3-5-haiku-latest',
        model_provider='anthropic'
    )

    # from langgraph.prebuilt import create_react_agent
    agent = create_react_agent(
        llm,
        tools
    )
```
<!-- A simple and experimentable usage example can be found
[here](https://github.com/hideya/langchain-mcp-tools-ts-usage/blob/main/src/index.ts) -->

<!-- A more realistic usage example can be found
[here](https://github.com/hideya/langchain-mcp-client-ts) -->

An usage example can be found
[here](https://github.com/hideya/mcp-client-langchain-py)

## Limitations

Currently, only text results of tool calls are supported.
