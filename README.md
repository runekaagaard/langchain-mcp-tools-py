# MCP To LangChain Tools Conversion Utility [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/hideya/langchain-mcp-tools-py/blob/main/LICENSE) [![pypi version](https://img.shields.io/pypi/v/langchain-mcp-tools.svg)](https://pypi.org/project/langchain-mcp-tools/)

This package is intended to simplify the use of
[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
server tools with LangChain / Python.

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/),
introduced by
[Anthropic](https://www.anthropic.com/news/model-context-protocol),
extends the capabilities of LLMs by enabling interaction with external tools and resources,
such as web search and database access.
Thanks to its open-source nature, MCP has gained significant traction in the developer community,
with over 400 MCP servers already developed and shared:

- [Smithery: MCP Server Registry](https://smithery.ai/)
- [Glama’s list of Open-Source MCP servers](https://glama.ai/mcp/servers)

In the MCP framework, external features are encapsulated in an MCP server
that runs in a separate process.
This clear decoupling allows for easy adoption and reuse of
any of the significant collections of MCP servers listed above.

To make it easy for LangChain to take advantage of such a vast resource base,
this package offers quick and seamless access from LangChain to MCP servers.

It contains a utility function `convert_mcp_to_langchain_tools()`.  
This async function handles parallel initialization of specified multiple MCP servers
and converts their available tools into a list of LangChain-compatible tools.

A typescript equivalent of this utility library is available
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

tools, cleanup = await convert_mcp_to_langchain_tools(
    mcp_configs
)
```

This utility function initializes all specified MCP servers in parallel,
and returns LangChain Tools
([`tools: List[BaseTool]`](https://python.langchain.com/api_reference/core/tools/langchain_core.tools.base.BaseTool.html#langchain_core.tools.base.BaseTool))
by gathering available MCP tools from the servers,
and by wrapping them into LangChain tools.
It also returns an async callback function (`cleanup: McpServerCleanupFn`)
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
A simple and experimentable usage example can be found
[here](https://github.com/hideya/langchain-mcp-tools-py-usage/blob/main/src/example.py)

A more realistic usage example can be found
[here](https://github.com/hideya/mcp-client-langchain-py)


## Limitations

Currently, only text results of tool calls are supported.
