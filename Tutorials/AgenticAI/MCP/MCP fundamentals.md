
### Definition and MCP participants

MCP follows a client-server architecture where an MCP host — an AI application like [Claude Code](https://www.anthropic.com/claude-code) or [Claude Desktop](https://www.claude.ai/download) — establishes connections to one or more MCP servers. The MCP host accomplishes this by creating one MCP client for each MCP server. Each MCP client maintains a dedicated one-to-one connection with its corresponding MCP server.The key participants in the MCP architecture are:

- **MCP Host**: The AI application that coordinates and manages one or multiple MCP clients
- **MCP Client**: A component that maintains a connection to an MCP server and obtains context from an MCP server for the MCP host to use
- **MCP Server**: A program that provides context to MCP clients

**For example**: Visual Studio Code acts as an MCP host. When Visual Studio Code establishes a connection to an MCP server, such as the [Sentry MCP server](https://docs.sentry.io/product/sentry-mcp/), the Visual Studio Code runtime instantiates an MCP client object that maintains the connection to the Sentry MCP server. When Visual Studio Code subsequently connects to another MCP server, such as the [local filesystem server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), the Visual Studio Code runtime instantiates an additional MCP client object to maintain this connection, hence maintaining a one-to-one relationship of MCP clients to MCP servers.

![[Pasted image 20251002135553.png]]

#### Types of MCP servers (local and remote):

Note that **MCP server** refers to the program that serves context data, regardless of where it runs. MCP servers can execute locally or remotely. For example, when Claude Desktop launches the [filesystem server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), the server runs locally on the same machine because it uses the STDIO transport. This is commonly referred to as a “local” MCP server.

The official [Sentry MCP server](https://docs.sentry.io/product/sentry-mcp/) runs on the Sentry platform, and uses the Streamable HTTP transport. This is commonly referred to as a “remote” MCP server.


### JSON-RPC protocol:

#### 🔹 What is JSON-RPC?

- **JSON-RPC** is a **remote procedure call (RPC) protocol** that uses **JSON** to encode requests and responses.
    
- It’s a way for one process (the client) to ask another process (the server) to execute a function/method and return the result.
    
- It’s designed to be:
    
    - **Lightweight**: pure JSON, no extra metadata.        
    - **Transport-agnostic**: can work over HTTP, WebSockets, stdio, etc.        
    - **Simple**: no verbose schemas (like SOAP), no extra ceremony.

---
#### 🔹 JSON-RPC 2.0 Message Types

There are **three kinds of messages**:

##### 1. **Request**

A request asks the server to call a method.  
It contains:

```json
{
  "jsonrpc": "2.0",
  "method": "subtract",
  "params": [42, 23],
  "id": 1
}
```

- `jsonrpc`: always `"2.0"`.    
- `method`: the name of the method to call.    
- `params`: optional — can be array `[42, 23]` or object `{"x":42,"y":23}`.    
- `id`: unique identifier for this request → must be echoed in the response.

---
##### 2. **Response**

The reply from the server, matching a request `id`.

Successful call:

```json
{
  "jsonrpc": "2.0",
  "result": 19,
  "id": 1
}
```

Error case:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found"
  },
  "id": 1
}
```

- Either `result` OR `error` must be present (never both).    
- `error` has a numeric `code`, a human-readable `message`, and optional `data`.    

---
##### 3. **Notification**

A special request where **no response is expected** (fire-and-forget).  
It has no `id`:

```json
{
  "jsonrpc": "2.0",
  "method": "updateStatus",
  "params": {"progress": 80}
}
```

Used for events, logs, or updates where acknowledgments aren’t needed.

---
#### 🔹 Standard Error Codes

JSON-RPC 2.0 defines some common error codes:

- `-32600`: Invalid Request (bad JSON or structure)
- `-32601`: Method not found    
- `-32602`: Invalid params
- `-32603`: Internal error    
- `-32700`: Parse error (invalid JSON)

Servers can define custom codes too.

---

#### 🔹 Transport Independence

- JSON-RPC **doesn’t define how messages are transported**.
    
- It could be:
    
    - HTTP POST request/response        
    - WebSocket        
    - Unix sockets        
    - STDIN/STDOUT pipes  
        This makes it a great fit for MCP: the **data layer is JSON-RPC**, and the **transport layer** (stdio or HTTP/SSE) just carries those JSON blobs.
        
---
#### 🔹 Example Roundtrip

**Client → Server**

```json
{
  "jsonrpc": "2.0",
  "method": "sum",
  "params": [1, 2, 4],
  "id": "req-123"
}
```

**Server → Client**

```json
{
  "jsonrpc": "2.0",
  "result": 7,
  "id": "req-123"
}
```

If the method isn’t supported:

```json
{
  "jsonrpc": "2.0",
  "error": { "code": -32601, "message": "Method not found" },
  "id": "req-123"
}
```

---

## Data Layer

The data layer is the **inner layer** that defines the protocol semantics and message structure for client-server communication. It implements a JSON-RPC 2.0 based exchange protocol that governs how clients and servers interact.[](https://modelcontextprotocol.io/docs/concepts/architecture)

### Key Components of the Data Layer

### 1. Lifecycle Management

Before clients and servers can exchange “useful” messages (like tool calls), they have to **initialize** and negotiate capabilities:

- **initialize** request/response: The client sends an `initialize` with parameters such as:
    - `protocolVersion`: version string so both sides ensure compatibility. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    - `capabilities`: declares what features the client supports (e.g. elicitation, logging) [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture        
    - `clientInfo`: name, version, etc. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
        
    The server responds with its own capabilities (tools, resources, etc.) and serverInfo. This negotiation ensures neither side tries to use features the other doesn’t support. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    
- After negotiation, the client sends a notification `notifications/initialized` to signal it’s ready. [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)

- The protocol is **stateful** — meaning the connection goes through phases (initialization → ready → possibly termination) rather than being fully stateless.

### 2. Primitives: Tools, Resources, Prompts, etc.

These “primitives” are the main abstractions by which servers and clients exchange context and capabilities. The data layer defines how these are listed, retrieved, invoked, etc. [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)

- **Tools**: executable operations exposed by server (e.g. “call API”, “query DB”). The server provides metadata (name, description, input schema). Client can do `tools/list`, `tools/call`. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    
- **Resources**: read-only context/data the server offers (e.g. file contents, database records). The server offers methods like `resources/list` or `resources/read`. [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)
    
- **Prompts**: reusable templates or prompt fragments available for use. The server may expose prompt templates that client can retrieve and use. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    
Additionally, the data layer allows **client primitives** (things the client offers the server), e.g.:

- **sampling**: the server may request the client to call the host LLM (if the server doesn't embed an LLM) via a `sampling/complete` method. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    
- **elicitation**: server can ask the client to request user input (e.g. for clarifications) via `elicitation/request`. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    
- **logging**: server may send log messages to the client for debugging/monitoring via a `logging` primitive. [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)
    
Each primitive usually supports methods like `*/list`, `*/get` (or read), `tools/call` for tools, etc.

### Request / Response / Notifications

The data layer uses JSON-RPC 2.0 semantics:

- **Requests**: messages with `"method"`, an `"id"`, and `params`. The recipient must respond with a matching `id` (unless method is not supported). [modelcontextprotocol.io+2modelcontextprotocol.io+2](https://modelcontextprotocol.io/docs/learn/architecture)
    
- **Responses**: carry either a `result` or an `error` keyed to the `id`. Standard JSON-RPC pattern. [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)
    
- **Notifications**: messages that have `"method"` but no `"id"` — no response is expected. Used for events or updates, e.g. `notifications/tools/list_changed` when the server’s tool set changes. [modelcontextprotocol.io+2modelcontextprotocol.io+2](https://modelcontextprotocol.io/docs/learn/architecture)
    
Notifications are a crucial part: they allow a server to push updates instead of clients polling. For example, if the available tools change dynamically, the server can send a `tools/list_changed` notification. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture) The client, upon receiving that, can re-fetch the tool list.
### Example Flow

Here’s a typical sequence:

1. **Initialization**: Client → server: `initialize`, server responds with its capabilities. Then client sends `notifications/initialized`. [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)
    
2. **Tool discovery**: Client → server: `tools/list`, to know what tools are available. Server returns list of tool metadata. [modelcontextprotocol.io+1](https://modelcontextprotocol.io/docs/learn/architecture)
    
3. **Tool execution**: Client → server: `tools/call` with `name` and `arguments`. Server executes and replies with result: a `content` array of typed items (text, resources, etc.). [modelcontextprotocol.io](https://modelcontextprotocol.io/docs/learn/architecture)
    
4. **Notifications**: Later, server might send `notifications/tools/list_changed` to alert the client that the set of tools has changed. Client reacts by re-listing tools.

## Message Structure

All messages in the data layer use **JSON-RPC 2.0 format** and must be UTF-8 encoded. The protocol supports three message types: requests (requiring responses), notifications (no response expected), and responses (replies to requests). Each message includes standard JSON-RPC fields like `jsonrpc`, `id`, `method`, and `params` or `result`