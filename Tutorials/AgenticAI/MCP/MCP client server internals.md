```python
# Create server parameters for stdio connection
import asyncio
from langchain_google_genai import ChatGoogleGenerativeAI
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent
import os
from dotenv import load_dotenv, find_dotenv

dotenv_path = find_dotenv()
load_dotenv(dotenv_path)  # This loads the variables from .env

# StdioServerParameters from mcp is used to define how to connect to an MCP server that communicates over standard 
# input/output (stdio).
# command="python": Specifies that the server is a Python script.
# args=["/path/to/demo_mcp_server.py"]: This is a crucial placeholder. It tells the client to execute the Python interpreter 
# with the demo_mcp_server.py script. 
server_params = StdioServerParameters(
    command="python",
    # Make sure to update to the full absolute path to your math_server.py file
    args=["/home/bk_anupam/code/LLM_agents/LANGGRAPH_PG/demo_mcp_server.py"],
)

async def main():
    # This async context manager establishes a connection to the MCP server using the stdio_client. It yields read and write 
    # streams for communication. The stdio_client will start the demo_mcp_server.py process.
    async with stdio_client(server_params) as (read, write):
        #  Inside the stdio_client context, an MCP ClientSession is created using the read/write streams. 
        # This session object will be used for all further interactions with the server.
        async with ClientSession(read, write) as session:
            # Initialize the connection
            await session.initialize()

            # load_mcp_tools queries the connected MCP server (our demo_mcp_server.py) for its available tools. 
            # The load_mcp_tools function then converts these MCP-native tools into BaseTool objects that Langchain 
            # can understand and use.
            tools = await load_mcp_tools(session)
            # Print the loaded tools to verify
            for tool in tools:
                print(f"Loaded tool: {tool.name} - {tool.description}")

            # Create and run the agent
            llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0.1)
            agent = create_react_agent(llm, tools)
            messages = await agent.ainvoke({"messages": "what's 3 + 5 ?"})
            for m in messages['messages']:
                m.pretty_print()

asyncio.run(main())
```

---


# 🚀 Tutorial: How MCP Client & Server Work Internally (demo_mcp_client.py)

MCP (Model Context Protocol) is just JSON-RPC 2.0 over some transport (here: **stdio**).
Your client flow is:

```
demo_mcp_client.py  →  stdio_client()  →  ClientSession()  →  initialize() handshake
                      ↕
            (spawns subprocess)
                      ↕
             demo_mcp_server.py (FastMCP server)
```

We’ll trace from the **client code** down to the **library internals**, showing what actually happens.

---

## 1. Spawning the MCP Server (stdio transport)

### Code:

```python
server_params = StdioServerParameters(
    command="python",
    args=["/home/bk_anupam/code/LLM_agents/LANGGRAPH_PG/demo_mcp_server.py"],
)
```

### What’s happening:

* `StdioServerParameters` is just a **data model** (pydantic class) that holds:

  * `command`: executable (`python`)
  * `args`: what script to run (`demo_mcp_server.py`)
  * Optional: `env`, `cwd`, `encoding`…

Think of it as a recipe for: “How do I start the server process?”

---

## 2. Opening a connection with `stdio_client`

### Code:

```python
async with stdio_client(server_params) as (read, write):
```

### What’s happening internally:

* `stdio_client` is an **async context manager** that:

  1. Spawns the MCP server process via `anyio.open_process([command, *args])`.
  2. Opens pipes to its `stdin` (for writing requests) and `stdout` (for reading responses).
  3. Wraps those pipes in async tasks:

     * **`stdout_reader`**:

       * Reads lines from the server’s `stdout`.
       * Each line is JSON → parsed into a `JSONRPCMessage` → wrapped as `SessionMessage` → pushed into a memory stream (`read_stream`).

        ```python
        async def stdout_reader():
        # 1. Sanity Check: Ensures the subprocess was created with a readable stdout pipe.
        assert process.stdout, "Opened process is missing stdout"

        try:
            # 2. Context Manager: Ensures the 'read_stream_writer' is properly closed on exit.
            #    This stream is the "write end" of a memory queue that the client reads from.
            async with read_stream_writer:
                # 3. Buffer for Incomplete Lines: Initializes a buffer to handle data chunks
                #    that don't end with a newline.
                buffer = ""
                
                # 4. Main Loop: Continuously reads text chunks from the server's stdout.
                #    TextReceiveStream is an anyio helper that decodes the raw byte
                #    stream into text using the configured encoding.
                async for chunk in TextReceiveStream(
                    process.stdout,
                    encoding=server.encoding,
                    errors=server.encoding_error_handler,
                ):
                    # 5. Line Splitting: Prepends the buffer (from the previous chunk) to the
                    #    new chunk and splits the result into lines.
                    lines = (buffer + chunk).split("\n")
                    
                    # 6. Re-buffering: The last item in 'lines' will either be an empty string
                    #    (if the chunk ended perfectly on a newline) or a partial line.
                    #    It's saved back to the buffer for the next iteration.
                    buffer = lines.pop()

                    # 7. Process Complete Lines: Iterates over the fully formed lines.
                    for line in lines:
                        try:
                            # 8. JSON Parsing: Attempts to validate and parse the line as a
                            #    JSON-RPC message using a Pydantic model.
                            message = types.JSONRPCMessage.model_validate_json(line)
                        except Exception as exc:
                            # 9. Error Handling: If parsing fails (e.g., malformed JSON),
                            #    it sends the exception object itself into the stream for the
                            #    client to handle, then continues to the next line.
                            await read_stream_writer.send(exc)
                            continue

                        # 10. Message Wrapping: Wraps the valid JSON-RPC message in a
                        #     higher-level SessionMessage.
                        session_message = SessionMessage(message)
                        
                        # 11. Forwarding: Sends the complete, parsed message into the memory
                        #      stream for the client application to consume.
                        await read_stream_writer.send(session_message)
                        
        # 12. Graceful Shutdown: This handles the expected error when the stream is
        #     closed from the other end, which is part of a clean shutdown sequence.
        except anyio.ClosedResourceError:
            # 13. Yield Control: Allows the async event loop to process other tasks,
            #     preventing blocking during cleanup.
            await anyio.lowlevel.checkpoint()
        ```

     * **`stdin_writer`**:

       * Waits for messages from the client code via `write_stream`.
       * Serializes them as JSON → writes to server’s `stdin`.
  4. Yields **two memory streams**:

     * `read_stream`: where client receives parsed JSON-RPC messages.
     * `write_stream`: where client sends messages to be forwarded to server’s stdin.

In short:
📤 Client writes → `write_stream` → stdin_writer → server stdin
📥 Server writes → stdout_reader → `read_stream` → Client reads

```python
async with (
    anyio.create_task_group() as tg,
    process,
):
    tg.start_soon(stdout_reader)
    tg.start_soon(stdin_writer)
    try:
        yield read_stream, write_stream
    finally:
        ...

```

1. async with (...)

This is the async version of Python’s with.
Instead of requiring __enter__/__exit__, the objects inside must implement __aenter__/__aexit__.

anyio.create_task_group() returns such an object (a task group is async-manageable).

process (the result of anyio.open_process(...) or Windows equivalent) also implements async context manager protocol: entering sets up the process handle, exiting ensures cleanup (closing pipes, terminating process if needed).

2. Multiple context managers

Notice the parentheses and the comma:
```python
async with (anyio.create_task_group() as tg, process):
```
This is just syntactic sugar for:
```python
async with anyio.create_task_group() as tg:
    async with process:
        ...
```
So it’s actually entering two async context managers in one line.

**What is anyio.create_task_group()?**

anyio.create_task_group() gives you a task group, which is a structured way to run multiple async tasks concurrently.

Think of it as a “container” for background tasks that:

* Start together
* Are cancelled if one fails
* Are cleaned up automatically when the async with block exits

Equivalent concept in asyncio is asyncio.TaskGroup (Python 3.11+).

**What is tg.start_soon(...)?**

Inside the task group:
```python
tg.start_soon(stdout_reader)
tg.start_soon(stdin_writer)
```
* stdout_reader and stdin_writer are async functions defined earlier.
* tg.start_soon(fn) schedules them to run concurrently in the task group.
* The key benefit: you don’t need to await them immediately. They run in the background, but are supervised by the task group.

If either task fails:
* The task group cancels the others.
* The exception propagates out of the async with.

So the task group ensures these two “forever running” loops (reading/writing pipes) live for as long as the async with block is alive.

**Role of process in this async with**

* The process object (from anyio.open_process) is also an async context manager.
* Entering the context sets up the process resources.
* Exiting ensures stdin/stdout/stderr pipes are closed and the process is terminated if still alive.
* So by including process in the same async with, you tie its lifecycle to the task group:
* If you exit the block, both the task group and the process are guaranteed to be cleaned up.

```vbnet
async with stdio_client(...) as (read, write):

┌─────────────── Setup (before yield) ────────────────┐
│ spawn process                                       │
│ create task group                                   │
│ start stdout_reader + stdin_writer                  │
└─────────────────────────────────────────────────────┘

           yield (read_stream, write_stream)  → back to caller

┌─────────────── With-block executes ────────────────┐
│ caller uses (read, write) to talk to server        │
└────────────────────────────────────────────────────┘

┌─────────────── Cleanup (after yield) ──────────────┐
│ task group exits → cancels reader/writer tasks      │
│ process context exits → closes process resources    │
│ finally block → ensures process termination         │
└────────────────────────────────────────────────────┘
```

---

## 3. Creating a `ClientSession`

### Code:

```python
async with ClientSession(read, write) as session:
```

### What’s happening internally:

* `ClientSession` (from `session.py`) is a subclass of `BaseSession`.
* It glues the raw `read_stream` / `write_stream` into:

  * JSON-RPC **request sending**
  * JSON-RPC **response handling**
  * **notifications** (fire-and-forget messages)

Key methods inside:

* `send_request(...)`: sends a JSON-RPC request, waits for a matching response.
* `send_notification(...)`: sends a JSON-RPC notification (no response expected).
* `initialize()`: performs the MCP **handshake**.

---

## 4. The Initialization Handshake

### Code:

```python
await session.initialize()
```

### Internals (from `session.py`):

1. Client sends an `initialize` request:

   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "method": "initialize",
     "params": {
       "protocolVersion": "2024-xx",
       "capabilities": { "sampling": {}, "roots": { "listChanged": true } },
       "clientInfo": { "name": "mcp", "version": "0.1.0" }
     }
   }
   ```
2. Server responds with its own capabilities + info:

   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "result": {
       "protocolVersion": "2024-xx",
       "capabilities": { "tools": {}, "resources": {} },
       "serverInfo": { "name": "Demo MCP Server", "version": "0.1.0" }
     }
   }
   ```
3. Client validates protocol version (must be supported).
4. Client sends a **notification**:

   ```json
   {
     "jsonrpc": "2.0",
     "method": "notifications/initialized"
   }
   ```

This signals: “I’m ready, let’s exchange tools/resources.”

At this point, both sides know:

* What protocol version they’re speaking.
* What features each supports.

---

## 5. Discovering Tools

### Code:

```python
tools = await load_mcp_tools(session)
```

### Internals:

* Calls `session.list_tools()` (JSON-RPC `tools/list`).
* Server replies with metadata for each tool (name, description, input schema).
* `load_mcp_tools` adapts them into LangChain `BaseTool` wrappers.

Now the LLM agent can call MCP tools as if they were native LangChain tools.

* load_mcp_tools(session) takes the MCP session (the actual JSON-RPC client connection to the server).
* It then queries the server (tools/list) to discover the available MCP tools (e.g., add, get_greeting).
* For each tool, it wraps it into a LangChain Tool object (like a BaseTool).

What is inside these Tools?

The wrapping is the trick. A LangChain Tool has:

* a name
* a description
* a schema for its arguments
* and, most importantly, a function (Tool.func) that is executed when the agent decides to call it.

In this case, load_mcp_tools(session) makes the function something like:
```python
async def call_tool_via_mcp(args):
    return await session.call_tool(name, args)
```

So under the hood, the Tool is just a proxy that delegates to the MCP ClientSession.

---

## 6. Agent workflow

When you run:

```python
messages = await agent.ainvoke({"messages": "what's 3 + 5 ?"})
```

Here’s the flow:

1. The agent’s reasoning loop runs inside LangChain.
2. The LLM decides: *“I need to call the tool `add` with arguments `{a: 3, b: 5}`.”*
3. LangChain looks up the `Tool` object named `add`.
4. It calls that tool’s `func`, which was bound to `session.call_tool("add", {"a":3,"b":5})`.
5. The `ClientSession` sends a JSON-RPC request to the MCP server:

   ```json
   {
     "jsonrpc": "2.0",
     "id": 42,
     "method": "tools/call",
     "params": {
       "name": "add",
       "arguments": { "a": 3, "b": 5 }
     }
   }
   ```

6. The MCP server executes its local function `add(3, 5) → 8`.
7. The server sends back a JSON-RPC response with the result:

   ```json
   {
     "jsonrpc": "2.0",
     "id": 42,
     "result": { "content": 8 }
   }
   ```

8. `ClientSession` receives it from the `read_stream`, resolves the awaiting call, and returns `8`.
9. The LangChain agent takes the result and continues its reasoning → produces final answer.

---

# 🔹 Big Picture Flow

1. **Client starts server** via `stdio_client`.
2. **stdin_writer / stdout_reader** connect both processes with JSON lines.
3. **ClientSession.initialize()** performs MCP handshake.
4. **list_tools** discovers server tools.
5. **Agent** uses LLM reasoning + MCP tools via JSON-RPC calls.
6. **Results** flow back through stdio → parsed → delivered to agent.

---

# ✅ Summary

* **`StdioServerParameters`**: recipe for spawning the server.
* **`stdio_client`**: spawns process, sets up pipes, bridges JSON messages to async streams.
* **`ClientSession`**: JSON-RPC session manager (sends requests, processes responses/notifications).
* **Handshake**: `initialize` request/response + `notifications/initialized`.
* **Tool usage**: JSON-RPC `tools/list` and `tools/call`.

Effectively, your client/server are just two processes talking **JSON-RPC over stdio**, wrapped in nice async Python abstractions.

---