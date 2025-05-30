#model_context_protocol #mcp #langgraph 

A "transport" in MCP defines how the client and server communicate with each other. It's the underlying mechanism for sending requests (like "call this tool") and receiving responses.

### STDIO Transport:

**What it is:** Standard Input/Output. This is the simplest form of inter-process communication.
    - The MCP server reads requests from its standard input (like what you type into a terminal).
    - It writes responses to its standard output (what normally gets printed to the terminal).
    
**Mechanism**:  
Communicates with a subprocess via standard input/output streams — like piping data to a Python script or CLI tool.

**How `FastMCP` handles it (`mcp.run(transport='stdio')`):**
    - The `FastMCP.run()` method, when `transport` is "stdio", will internally call `anyio.run(self.run_stdio_async)`.
    - `run_stdio_async` then uses `mcp.server.stdio.stdio_server()` to create an asynchronous context manager that provides `read_stream` and `write_stream` connected to the process's `stdin` and `stdout`.
    - The core `self._mcp_server.run()` method then uses these streams to listen for incoming MCP messages and send outgoing ones.

**Client-side (`multi_mcp_client.py`):**

- When your client configures a server with `"transport": "stdio"`, like your "math" server:

```python
"math": {
    "command": "python",
    "args": ["/home/bk_anupam/code/LLM_agents/LANGGRAPH_PG/demo_mcp_server.py"],
    "transport": "stdio",
}
```
- The `MultiServerMCPClient` (or the underlying `mcp` client library) will:
    1. Start the specified `command` with `args` as a subprocess.
    2. Establish pipes to the subprocess's `stdin` and `stdout`.
    3. Send MCP messages over the subprocess's `stdin` pipe.
    4. Read MCP responses from the subprocess's `stdout` pipe.

**When to use `stdio`:**
    - **Local, single-client tools:** Ideal for tools that run as separate local processes and are primarily intended to be used by a single client process on the same machine.
    - **Simplicity:** It's straightforward to set up for simple command-line tools.
    - **No network overhead:** Communication is direct via OS pipes, which is very fast for local communication.
    - **Security:** Inherently more secure for local-only tools as it doesn't open network ports.

**Limitations:**
    - Not suitable for remote access (across different machines).
    - Typically one server process per client connection (though the server itself might handle multiple requests from that one client sequentially or concurrently internally).
    - Doesn't naturally fit web-based architectures or microservices that expect HTTP.

### STREAMABLE-HTTP Transport:
- **What it is:** A more advanced HTTP-based transport. It uses standard HTTP requests and responses but is designed to handle the potentially long-lived, streaming nature of some MCP interactions (like tool calls that report progress or return large amounts of data in chunks).
    - The server listens on an HTTP port (e.g., `http://localhost:8000`).
    - Clients make HTTP POST requests to specific MCP endpoints (e.g., `/mcp` by default in `FastMCP`).
    - Responses can be standard JSON or streamed over HTTP.
    
- **How `FastMCP` handles it (`mcp.run(transport='streamable_http')`):**
    - The `FastMCP.run()` method calls `anyio.run(self.run_streamable_http_async)`.
    - `run_streamable_http_async` then:
        1. Calls `self.streamable_http_app()` to get a Starlette ASGI application.
        2. The `streamable_http_app()` method sets up a `StreamableHTTPSessionManager`. This manager is responsible for handling incoming HTTP requests, mapping them to MCP sessions, and managing the lifecycle of these sessions. It uses an `EventStore` if provided (for more persistent session management, though not strictly required for basic operation).
        3. It mounts an ASGI application (the `handle_streamable_http` function, which uses the session manager) at a specific path (default `/mcp`).
        4. Finally, it uses `uvicorn` (a standard ASGI server) to run this Starlette application, making it listen for HTTP requests on the configured host and port.
        
- **Client-side (`multi_mcp_client.py`):**
    - When your client configures a server with `"transport": "streamable_http"`, like your "murli" server:
        
    ```python
	"mcp_server": { 
		"url": "http://localhost:8000", 
		"transport": "streamable_http", 
	}
	```
        
    - The `MultiServerMCPClient` will:
        1. Make HTTP POST requests to the specified `url` (e.g., `http://localhost:8000/mcp`).
        2. Send MCP messages as the HTTP request body (typically JSON).
        3. Receive MCP responses from the HTTP response body. It's capable of handling chunked/streamed responses if the server sends them.
        
- **When to use `streamable_http`:**
    - **Network-accessible tools:** When your MCP server needs to be accessible over a network (localhost or remote).
    - **Web services / Microservices:** If your MCP tool is part of a larger web-based architecture.
    - **Multiple clients:** An HTTP server can naturally handle connections from many clients simultaneously.
    - **Standard infrastructure:** Leverages existing HTTP infrastructure (proxies, load balancers, etc.).
    - **Streaming capabilities:** Useful if your tools need to send progress updates or stream large data back to the client.
    
- **Considerations:**
    - Slightly more overhead than `stdio` due to HTTP.
    - Requires managing network ports and potentially security (HTTPS, authentication).

**What is Starlette?**

Starlette is a lightweight, async-friendly web framework for building high-performance web applications in Python. It's designed to be fast, flexible, and easy to use. Starlette is built on top of the ASGI (Asynchronous Server Gateway Interface) standard, which allows it to work seamlessly with async/await syntax and support asynchronous programming.

**ASGI Application**

An ASGI application is a type of web application that uses the ASGI standard to communicate with a web server. ASGI applications are designed to handle asynchronous requests and responses, making them well-suited for building high-performance web services.

**How Multiple Tools Work with a Single MCP Endpoint**

1. **Tool Registration:** When you define tools in your `murli_mcp_server.py` using the `@mcp.tool()` decorator, like your `retrieve_murli_extracts` function, you are registering them with the `FastMCP` instance (`mcp`). `FastMCP` keeps an internal registry of these tools by their names.
    
2. **Client Discovery:** When an MCP client (like your `MultiServerMCPClient`) provided by langchain_mcp_adapters library connects and initializes with the server, one of the first things it typically does is request a list of available tools. 
```python
from langchain_mcp_adapters.client import MultiServerMCPClient

async def main():
    # Initialize the MultiServerMCPClient with the server URL
    client = MultiServerMCPClient(
        {
            "murli": {
                "url": "http://localhost:8000/mcp",
                "transport": "streamable_http",
            },
            "math": {
                "command": "python",
                "args": ["/home/bk_anupam/code/LLM_agents/LANGGRAPH_PG/demo_mcp_server.py"],
                "transport": "stdio",
            }
        }
    )

    tools = await client.get_tools()
	# Create and run the agent
    llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash", temperature=0.1)
    agent = create_react_agent(llm, tools)    
```
The server responds with the names, descriptions, and input schemas of all registered tools (e.g., `retrieve_murli_extracts` and any others you add). The `load_mcp_tools` function then converts these MCP-native tools into `StructuredTool` objects that Langchain can understand and use. 
    
3. **Tool Invocation by Name:**
    
    - When the client (or an agent using the client) wants to execute a specific tool, it sends a "call tool" request to the main MCP endpoint (e.g., `http://localhost:8000/mcp`).
    - Crucially, this request message **includes the name of the tool** it wants to invoke (e.g., `"retrieve_murli_extracts"`) and the arguments for that tool.
    - The `FastMCP` server, upon receiving this request at its `/mcp` endpoint, looks up the tool name in its internal registry.
    - It then dispatches the call to the corresponding Python function you decorated (e.g., your `retrieve_murli_extracts` function), passing along the provided arguments.

So, you don't need separate HTTP endpoints for each MCP tool. The single `/mcp` endpoint acts as a gateway for all MCP protocol messages, including requests to list tools and requests to call specific tools by their registered name.

**Regarding "More Than One Endpoint" in a Broader Sense**

While MCP tools share a primary endpoint, `FastMCP` (being built on Starlette) _does_ allow you to define completely separate, custom HTTP endpoints if you need them for non-MCP purposes. This is done using the `@mcp.custom_route()` decorator.

For example, you might add a health check endpoint:

```python
from starlette.requests import Request
from starlette.responses import JSONResponse

@mcp.custom_route("/health", methods=["GET"])
async def health_check(request: Request) -> JSONResponse:
    logger.info("Health check endpoint called.")
    return JSONResponse({"status": "ok", "server_name": mcp.name})
```

This `/health` endpoint would be accessible directly via `http://localhost:8000/health` and operates outside the main `/mcp` tool invocation flow. This is useful for things like:

- Health checks for load balancers or monitoring systems.
- OAuth callback URLs.
- Simple administrative APIs.

But for your core MCP tools, stick to defining them with `@mcp.tool()` and let the MCP protocol handle routing through the main `/mcp` endpoint.