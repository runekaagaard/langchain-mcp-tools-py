# MCP To LangChain Tools Conversion Utility [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/hideya/langchain-mcp-tools-py/blob/main/LICENSE) [![pypi version](https://img.shields.io/pypi/v/langchain-mcp-tools.svg)](https://pypi.org/project/langchain-mcp-tools/)

This package is intended to simplify the use of
[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
server tools with LangChain / Python.

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

## Technical Details

It was very tricky (for me) to get the parallel MCP server initialization
to work, including successful final resource cleanup...

I'm new to Python, so it is very possible that my ignorance is playing
a big role here...  
I'll summarize the difficulties I faced below.
The source code is available
[here](https://github.com/hideya/langchain-mcp-tools-py/blob/main/src/langchain_mcp_tools/langchain_mcp_tools.py).  
Any comments pointing out something I am missing would be greatly appreciated!
[(comment here)](https://github.com/hideya/langchain-mcp-tools-ts/issues)

1. Challenge:

   A key requirement for parallel initialization is that each server must be
   initialized in its own dedicated task - there's no way around this as far as
   I know. However, this poses a challenge when combined with
   `asynccontextmanager`.

   - Resources management for `stdio_client` and `ClientSession` seems
     to require relying exclusively on `asynccontextmanager` for cleanup,
     with no manual cleanup options
     (based on [the mcp python-sdk impl as of Jan 14, 2025](https://github.com/modelcontextprotocol/python-sdk/tree/99727a9/src/mcp/client))
   - Initializing multiple MCP servers in parallel requires a dedicated
     `asyncio.Task` per server
   - Server cleanup can be initiated later by a task other than the one
     that initialized the resources, whereas `AsyncExitStack.aclose()` must be
     called from the same task that created the context

2. Solution:

   The key insight is to keep the initialization tasks alive throughout the
   session lifetime, rather than letting them complete after initialization.

   By using `asyncio.Event`s for coordination, we can:
   - Allow parallel initialization while maintaining proper context management
   - Keep each initialization task running until explicit cleanup is requested
   - Ensure cleanup occurs in the same task that created the resources
   - Provide a clean interface for the caller to manage the lifecycle

   Alternative Considered:
   A generator/coroutine approach using `finally` block for cleanup was
   considered but rejected because:
   - It turned out that the `finally` block in a generator/coroutine can be
     executed by a different task than the one that ran the main body of
     the code
   - This breaks the requirement that `AsyncExitStack.aclose()` must be
     called from the same task that created the context

3. Task Lifecycle:

   The following task lifecyle diagram illustrates how the above strategy
   was impelemented:
   ```
   [Task starts]
     ↓
   Initialize server & convert tools
     ↓
   Set ready_event (signals tools are ready)
     ↓
   await cleanup_event.wait() (keeps task alive)
     ↓
   When cleanup_event is set:
   exit_stack.aclose() (cleanup in original task)
   ```
This approach indeed enables parallel initialization while maintaining proper
async resource lifecycle management through context managers.
However, I'm afraid I'm twisting things around too much.
It usually means I'm doing something very worng...

I think it is a natural assumption that MCP SDK is designed with consideration
for parallel server initialization.
I'm not sure what I'm missing...
(FYI, with the TypeScript MCP SDK, parallel initialization was
[pretty straightforward](https://github.com/hideya/langchain-mcp-tools-ts/blob/main/src/langchain-mcp-tools.ts))
