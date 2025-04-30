#model_context_protocol #agents #langgraph #langchain #mcp
## Prompt: 
**What is model context protocol and how can I use it while building an agent using langgraph, langchain and google gemini 2.5 pro experimental.**

## Response:
### Gemini 2.5 pro answer:
Okay, let's break down your questions regarding LangGraph, LangChain, Gemini 2.5 Pro Experimental, and tools.

**1. Model Context Protocol with Gemini 2.5 Pro Experimental and LangGraph**

* **Native Tool/Function Calling:** Gemini models, especially the newer ones like 2.5 Pro, support native function calling (also referred to as tool use). This is the core concept behind what some frameworks might call a "Model Context Protocol". The idea is that you describe the available tools (functions) to the model, and the model itself can decide when to call a specific tool and with what arguments, outputting a structured request for the tool call.
* **LangChain Integration:** LangChain provides wrappers for models like Gemini (`ChatGoogleGenerativeAI`) that support this native function calling. You can define your tools using LangChain's standard methods (see next point) and then bind them to the model instance (often using the `.bind_tools()` or similar methods). When you invoke the model within your LangGraph agent, LangChain handles formatting the tool descriptions for Gemini, sending them with the prompt, and parsing Gemini's response if it requests a tool call.
* **LangGraph Orchestration:** LangGraph is designed to orchestrate these interactions. A node in your graph might involve calling the Gemini model. If Gemini decides to use a tool, LangGraph routes the execution flow to another node responsible for running that specific tool, gets the result, and then typically feeds that result back to the Gemini model node for the next step.

**Conclusion:** Yes, you can effectively use the *principles* of model-driven tool use (native function calling) with Gemini 2.5 Pro Experimental within a LangGraph agent. LangChain provides the necessary abstractions to define tools and integrate them with the Gemini model's native capabilities, and LangGraph orchestrates the resulting multi-step process.

**2. Options for Creating New Tools for Your Agent**

LangChain offers several flexible ways to create custom tools:

1.  **The `@tool` Decorator (Recommended for Simplicity):**
    * This is the easiest and most common method.
    * Define a standard Python function that performs the desired action.
    * Decorate it with `@tool` from `langchain_core.tools`.
    * **Crucially, write a clear and descriptive docstring for the function.** This docstring is automatically used as the tool's description, which the LLM (Gemini) uses to understand *what the tool does*, *when to use it*, and *what arguments it expects*.
    * LangChain automatically infers the tool's name (from the function name) and its arguments (from the function signature and type hints).
    * Example:
        ```python
        from langchain_core.tools import tool

        @tool
        def get_weather(location: str) -> str:
          """Returns the current weather for a given location."""
          # ... (your code to fetch weather) ...
          return f"The weather in {location} is sunny."

        @tool
        def complex_calculation(x: float, y: float, operation: str = "add") -> float:
          """Performs a complex calculation (add, subtract, multiply) on two numbers x and y."""
          if operation == "add":
            return x + y
          elif operation == "subtract":
            return x - y
          # ... etc ...
          return 0.0
        ```

2.  **Subclassing `BaseTool` (More Control):**
    * For more complex scenarios where you need finer control over the tool's name, description, argument schema (using Pydantic), or how asynchronous execution is handled.
    * You inherit from `langchain_core.tools.BaseTool` and must implement:
        * `name`: A string identifier for the tool.
        * `description`: A detailed string explaining the tool's purpose and usage.
        * `args_schema`: A Pydantic `BaseModel` defining the expected input arguments and their types.
        * `_run()`: The synchronous execution logic.
        * `_arun()`: (Optional) The asynchronous execution logic.
    * This gives you maximum flexibility but requires more boilerplate code.

3.  **Using `StructuredTool.from_function()` (Schema Flexibility):**
    * Similar to `@tool` but allows you to explicitly define the input schema using a Pydantic model if needed, separate from the function's signature.

**Key Considerations When Creating Tools:**

* **Descriptions are Vital:** The quality of your tool's description (usually from the docstring) is paramount. The LLM relies solely on this description to decide if and how to use the tool. Be clear, specific, and provide examples if necessary.
* **Argument Schema:** Use type hints in your functions (`@tool`) or define a Pydantic `args_schema` (`BaseTool`) so the LLM knows exactly what arguments are needed and their types.
* **Error Handling:** Implement robust error handling within your tool's logic.

**3. Prebuilt Tools Available in Your Tech Stack (LangChain)**

LangChain has a rich ecosystem of prebuilt tools and integrations, saving you from reinventing the wheel. Since LangGraph uses LangChain's tools, you have access to all of them. Here are some common categories and examples:

* **Web Search:**
    * `GoogleSearchRun` (via `GoogleSearchAPIWrapper`)
    * `DuckDuckGoSearchRun`
    * `TavilySearchResults` (often recommended for agent search)
    * `BingSearchRun`
    * `ArxivQueryRun`
    * `PubMedQueryRun`
* **Information Retrieval (RAG - Retrieval-Augmented Generation):**
    * Tools built on top of Vector Stores (like Chroma, FAISS, Pinecone, etc.) using `create_retriever_tool`. These allow your agent to search through your own documents.
* **Code Execution:**
    * `PythonREPLTool` (executes Python code - use with caution!)
    * Bash Process tools.
* **Database Interaction:**
    * `SQLDatabaseToolkit`: Allows querying SQL databases.
* **API Interaction:**
    * `RequestsTools`: Generic tools for making HTTP GET, POST, etc., requests.
    * Specific API tools: Zapier, OpenWeatherMap, Wolfram Alpha, Wikipedia, various Google services (Drive, Calendar - often via `GoogleApiResourceTools`), GitHub, etc.
* **File System:**
    * Tools for reading, writing, and listing files.
* **Toolkits:** Collections of related tools (e.g., `SQLDatabaseToolkit`, `CSVToolkit`, `JsonToolkit`).

**Where to Find Them:**

* **LangChain Documentation:** The best place to explore is the official LangChain documentation, specifically the "Integrations" -> "Tools" section.
* **LangChain Community:** The ecosystem is constantly growing, with community contributions adding new tools frequently.

When building your agent, start by checking if a prebuilt LangChain tool already exists for the functionality you need before creating a custom one.

### Claude's Sonnet 3.7 Answer:
Model context protocol refers to a standardized way of managing, passing, and accessing context information between different components in an AI system, especially in agent implementations. It's particularly relevant when building agents with frameworks like LangGraph and LangChain that interact with models like Google Gemini 2.5 Pro.

Here's how you can use model context protocol when building an agent:

1. **Context Management**
   - Define what context your agent needs to track (conversation history, user preferences, tool outputs, etc.)
   - Structure this context in a consistent format that can be efficiently passed between components

2. **Implementation with LangGraph and LangChain**
   - Use LangGraph's state management to maintain context between nodes in your agent graph
   - Leverage LangChain's memory components to persist relevant information

3. **Integration with Gemini 2.5 Pro**
   - Format your context appropriately for Gemini's input requirements
   - Use structured prompts that clearly delineate context information
   - Consider context window limitations (Gemini 2.5 Pro Experimental has an expanded context window)

A basic implementation might look like:

```python
from langchain.memory import ConversationBufferMemory
from langgraph.graph import StateGraph
from langchain_google_genai import ChatGoogleGenerativeAI

# Initialize the model
model = ChatGoogleGenerativeAI(model="models/gemini-2.5-pro-experimental")

# Create memory component for context
memory = ConversationBufferMemory(return_messages=True)

# Define state schema
class AgentState(TypedDict):
    context: dict  # Context protocol container
    messages: list
    next: str

# Create state graph with context protocol
graph = StateGraph(AgentState)

# Define nodes that maintain context
def process_input(state):
    # Extract context
    current_context = state["context"]
    # Update context with new information
    updated_context = {**current_context, "last_input": state["messages"][-1].content}
    # Return updated state
    return {"context": updated_context, "messages": state["messages"]}

# Define reasoning node that uses context
def reasoning(state):
    context = state["context"]
    formatted_context = format_context_for_model(context)  # Convert to model-friendly format
    
    # Pass formatted context to model
    response = model.invoke([
        {"role": "system", "content": "You are a helpful agent with the following context."},
        {"role": "user", "content": f"Context: {formatted_context}\n\nUser query: {state['messages'][-1].content}"}
    ])
    
    return {"messages": state["messages"] + [response], "next": "tool_selection"}
```

When working specifically with Gemini 2.5 Pro Experimental, you might want to:

1. Structure your context as JSON or other structured format that Gemini can easily parse
2. Take advantage of Gemini's multimodal capabilities by including images or other data types in your context
3. Utilize the expanded context window to maintain more comprehensive history

Would you like more detailed information about any specific aspect of implementing model context protocol in your agent?

## Question 2: 
What do we mean when we say that we can build MCP servers for almost anything. How does an agent interact with a MCP server. How can I build an MCP server with using langgraph, langchain and google gemini 2.5 pro experimental.

## Gemini 2.5 pro answer:

Okay, let's clarify the term "MCP Server" in this context and how it relates to your agent architecture.

As mentioned before, "Model Context Protocol" isn't a single standard protocol but a *pattern* for how models interact with tools. It seems you're using "MCP Server" to refer to an **external service, API, or application that exposes specific functionalities (tools) designed to be consumed by an LLM agent following this pattern.**

It's not a standard industry term, so let's think of it as a **"Tool Server"** or **"Agent-Action Server"** – a backend service providing capabilities for your agent.

**1. What does "building MCP [Tool] servers for almost anything" mean?**

This means you can take virtually *any* piece of functionality, software capability, data source, or even hardware interaction and make it available to your LLM agent. You do this by:

1.  **Encapsulating the Functionality:** Write code that performs the desired action (e.g., query a database, call a specific internal API, check inventory, control a smart device, perform a complex calculation, search a specific document repository).
2.  **Exposing it via an API:** Create a web server (using frameworks like FastAPI, Flask, Node.js/Express, etc.) with specific API endpoints (URLs) that trigger this functionality. Define clear inputs (request parameters or body) and outputs (response format, often JSON).
3.  **Describing it as a Tool:** Create a corresponding "tool definition" for your LLM agent (using LangChain's `@tool` or `BaseTool`). This definition tells the LLM:
    * The tool's name (which might map to the API endpoint or a conceptual action).
    * A clear description of *what the tool does* and *when to use it*.
    * The required input parameters (which correspond to the API inputs).

Essentially, you can build a bridge (the Tool Server/API) between the agent's "brain" (the LLM) and almost any capability you can programmatically access. The "server" is just the backend application hosting this capability and making it available over a network (usually HTTP).

**2. How does an agent interact with an MCP [Tool] Server?**

The interaction is orchestrated by your LangChain/LangGraph agent code, using the LLM's native function calling/tool use capability:

1.  **User Request:** The user gives a prompt to the agent (e.g., "What's the current stock level for product SKU 12345?").
2.  **LLM Decision:** The Gemini model, configured with a tool description like `get_stock_level(sku: str)` (which you defined in LangChain), analyzes the prompt. It recognizes that fulfilling the request requires using this tool and determines the `sku` is "12345".
3.  **LLM Output (Tool Call):** Gemini outputs a structured message indicating a call to the `get_stock_level` tool with the argument `sku="12345"`.
4.  **LangGraph/LangChain Interception:** Your LangGraph agent receives this structured message. The `ToolExecutor` or your custom tool-handling node identifies the requested tool (`get_stock_level`).
5.  **Tool Execution (API Call):** The *implementation* of your LangChain `get_stock_level` tool is executed. This implementation contains code (e.g., using the `requests` library) to:
    * Construct an HTTP request (e.g., `GET http://your-tool-server.com/api/stock/12345`).
    * Send this request to your running "Tool Server".
6.  **Tool Server Processing:** Your external "Tool Server" (e.g., a FastAPI app) receives the request at its `/api/stock/{sku}` endpoint. It runs its internal logic (e.g., queries the inventory database for SKU 12345).
7.  **Tool Server Response:** The Tool Server sends back an HTTP response containing the result (e.g., `{"sku": "12345", "level": 57}`).
8.  **Result Processing:** The LangChain tool implementation receives the HTTP response, parses the stock level (57), and formats it as a `ToolMessage`.
9.  **Feedback to LLM:** LangGraph routes this `ToolMessage` (containing "Stock level is 57") back to the Gemini model.
10. **Final Response:** Gemini processes the tool's result and generates the final natural language response for the user (e.g., "The current stock level for product SKU 12345 is 57 units.").

**Key Point:** The LLM *decides* to use the tool. The LangChain/LangGraph framework *orchestrates* the process. The LangChain *tool definition* contains the code to *actually call* the external Tool Server's API.

**3. How can I build an MCP [Tool] Server with LangGraph, LangChain, and Gemini 2.5 Pro Experimental?**

You use this stack to build the **agent** (the consumer), but you need a separate process/application for the **server** (the provider).

**A. Building the Tool Server (Example using FastAPI):**

This is a separate Python application. You *don't* typically run this inside your LangGraph agent process.

```python
# tool_server.py (Run this as a separate application: uvicorn tool_server:app --reload)
from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn # pip install fastapi uvicorn

app = FastAPI(title="My Agent Tool Server")

# --- Example 1: Simple Data Lookup ---
class ProductStock(BaseModel):
    sku: str
    level: int
    status: str

@app.get("/api/stock/{sku}", response_model=ProductStock)
async def get_stock_level_endpoint(sku: str):
    """Retrieves the stock level for a given product SKU."""
    print(f"--- Tool Server: Received request for SKU: {sku} ---")
    # --- Replace with your actual database query/logic ---
    await asyncio.sleep(0.5) # Simulate DB call
    stock_level = 57 # Dummy data
    # --- End Replace ---
    return ProductStock(sku=sku, level=stock_level, status="in_stock")

# --- Example 2: Action ---
class OrderDetails(BaseModel):
    sku: str
    quantity: int

class OrderResult(BaseModel):
    order_id: str
    status: str
    message: str

@app.post("/api/orders", response_model=OrderResult)
async def place_order_endpoint(order: OrderDetails):
    """Places an order for a given product SKU and quantity."""
    print(f"--- Tool Server: Received order for {order.quantity} of {order.sku} ---")
    # --- Replace with your actual order processing logic ---
    await asyncio.sleep(1) # Simulate order processing
    new_order_id = f"ORD-{abs(hash(order.sku)) % 10000}"
    # --- End Replace ---
    return OrderResult(order_id=new_order_id, status="accepted", message="Order placed successfully.")

if __name__ == "__main__":
     uvicorn.run(app, host="0.0.0.0", port=8000) # Make accessible on local network if needed
```

**B. Building the Agent (LangGraph/LangChain/Gemini) to USE the Server:**

This is where your main agent code lives.

```python
# agent.py (Your main LangGraph application)
import requests # Or httpx for async
from langchain_core.tools import tool
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.prebuilt import ToolExecutor
from langgraph.graph import StateGraph, END
# ... (import AgentState, BaseMessage, HumanMessage, ToolMessage etc. as before) ...

# Define LangChain tools that CALL your Tool Server API endpoints
TOOL_SERVER_BASE_URL = "http://localhost:8000" # Or the deployed URL of your server

@tool
def get_stock_level(sku: str) -> str:
    """Looks up the current stock level for a specific product SKU."""
    try:
        response = requests.get(f"{TOOL_SERVER_BASE_URL}/api/stock/{sku}")
        response.raise_for_status() # Raise an exception for bad status codes (4xx or 5xx)
        data = response.json()
        return f"Stock level for SKU {data['sku']} is {data['level']} ({data['status']})."
    except requests.exceptions.RequestException as e:
        return f"Error contacting stock server: {e}"
    except Exception as e:
        return f"Error processing stock data: {e}"

@tool
def place_order(sku: str, quantity: int) -> str:
    """Places an order for a given quantity of a product SKU."""
    try:
        payload = {"sku": sku, "quantity": quantity}
        response = requests.post(f"{TOOL_SERVER_BASE_URL}/api/orders", json=payload)
        response.raise_for_status()
        data = response.json()
        return f"Order status: {data['status']}. Order ID: {data['order_id']}. Message: {data['message']}"
    except requests.exceptions.RequestException as e:
        return f"Error contacting order server: {e}"
    except Exception as e:
        return f"Error processing order response: {e}"

# --- Agent Setup ---
my_tools = [get_stock_level, place_order]
tool_executor = ToolExecutor(my_tools)

llm = ChatGoogleGenerativeAI(model="gemini-2.5-pro-experimental-latest") # Use your specific model ID
llm_with_tools = llm.bind_tools(my_tools)

# --- Graph Definition (Similar to previous examples) ---
# ... (Define AgentState, call_model node using llm_with_tools, call_tool node using tool_executor)
# ... (Define should_continue conditional edge logic)
# ... (Build and compile the StateGraph) ...

# --- Running the Agent ---
# ... (Use app.stream or app.invoke with user input like "How much stock of SKU XYZ is there?")
```

**In Summary:**

1.  You build a standard web server (like FastAPI) to host your custom logic/actions – this is your "Tool Server".
2.  You build your agent using LangChain/LangGraph and Gemini.
3.  Within the agent, you define LangChain tools where the implementation logic makes HTTP calls to your Tool Server's API endpoints.
4.  Gemini decides *when* to use these tools based on their descriptions, and LangGraph orchestrates the process of calling the tool (which calls the server) and feeding the results back.

## Claude Sonnet 3.7 answer:

When we talk about building MCP (Model Context Protocol) servers for "almost anything," we're referring to creating specialized services that manage context for different types of information or capabilities that an agent might need to access.

### What is an MCP Server?

An MCP server is essentially a dedicated service that:
1. Stores, manages, and provides context relevant to a specific domain or function
2. Presents this context in a standardized format that models can effectively use
3. Can be queried or updated by agents as needed

### How Agents Interact with MCP Servers

The interaction flow typically works like this:

1. **Context Request**: The agent sends a request to the MCP server when it needs specific contextual information
2. **Context Retrieval**: The MCP server retrieves and formats the relevant context
3. **Context Injection**: The formatted context is injected into the agent's prompt or reasoning process
4. **Context Update**: After processing, the agent may send updates back to the MCP server

This happens through API calls or direct integration depending on your architecture.

### Building an MCP Server with LangGraph, LangChain, and Gemini 2.5 Pro

Yes, you can definitely build an MCP server using these tools. Here's a conceptual approach:

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import Dict, List, Any, Optional
from langchain.vectorstores import Chroma
from langchain.embeddings import GoogleGenerativeAIEmbeddings
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.graph import StateGraph
from langgraph.prebuilt import ToolExecutor
import uuid
import json

# Initialize FastAPI app
app = FastAPI(title="Model Context Protocol Server")

# Initialize the embedding model for vector storage
embeddings = GoogleGenerativeAIEmbeddings(model="models/embedding-001")

# Initialize LLM
llm = ChatGoogleGenerativeAI(model="models/gemini-2.5-pro-experimental")

# Database to store different context types
context_stores = {}

class ContextRequest(BaseModel):
    query: str
    context_type: str
    additional_params: Optional[Dict[str, Any]] = None

class ContextUpdate(BaseModel):
    context_type: str
    data: Dict[str, Any]
    metadata: Optional[Dict[str, Any]] = None

class ContextResponse(BaseModel):
    context_id: str
    formatted_context: str
    metadata: Dict[str, Any]

# Function to initialize a context store for a specific type
def initialize_context_store(context_type: str):
    if context_type not in context_stores:
        # Create a vector store for this context type
        context_stores[context_type] = {
            "vector_store": Chroma(
                collection_name=f"mcp_{context_type}",
                embedding_function=embeddings,
                persist_directory=f"./data/{context_type}"
            ),
            "raw_data": {}
        }
    return context_stores[context_type]

# Define context processing for different context types
def process_context(context_type: str, query: str, additional_params: Optional[Dict[str, Any]] = None):
    # Get or initialize the store
    store = initialize_context_store(context_type)
    
    # Define a context graph for processing
    class ContextState(dict):
        context_data: Dict[str, Any]
        query: str
        formatted_result: str
    
    # Create processing graph
    context_graph = StateGraph(ContextState)
    
    # Define retrieval node
    def retrieve_context(state):
        # Retrieve relevant documents from vector store
        docs = store["vector_store"].similarity_search(
            state["query"], 
            k=5,
            fetch_k=20
        )
        
        # Get raw data for additional context
        raw_data = {}
        for doc_id in [doc.metadata.get("id") for doc in docs]:
            if doc_id in store["raw_data"]:
                raw_data[doc_id] = store["raw_data"][doc_id]
        
        return {"context_data": {"retrieved_docs": [doc.page_content for doc in docs], "raw_data": raw_data}}
    
    # Define formatting node
    def format_context(state):
        # Use the LLM to format the context appropriately
        prompt = f"""
        You are formatting context for an AI agent. The agent needs context about: {context_type}
        
        The query is: {state["query"]}
        
        Retrieved context information:
        {json.dumps(state["context_data"], indent=2)}
        
        Format this context in a clear, structured way that would be most useful for an AI agent.
        Return ONLY the formatted context, nothing else.
        """
        
        response = llm.invoke(prompt)
        formatted_context = response.content
        
        return {"formatted_result": formatted_context}
    
    # Add nodes to the graph
    context_graph.add_node("retrieve", retrieve_context)
    context_graph.add_node("format", format_context)
    
    # Add edges
    context_graph.add_edge("retrieve", "format")
    context_graph.set_entry_point("retrieve")
    context_graph.set_finish_point("format")
    
    # Compile and run the graph
    app = context_graph.compile()
    result = app.invoke({
        "query": query,
        "context_data": {},
        "formatted_result": ""
    })
    
    return result["formatted_result"]

# API endpoints
@app.post("/context", response_model=ContextResponse)
async def get_context(request: ContextRequest):
    try:
        formatted_context = process_context(
            request.context_type,
            request.query,
            request.additional_params
        )
        
        context_id = str(uuid.uuid4())
        
        return ContextResponse(
            context_id=context_id,
            formatted_context=formatted_context,
            metadata={
                "context_type": request.context_type,
                "query": request.query,
                "timestamp": "current_timestamp"
            }
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/update")
async def update_context(update: ContextUpdate):
    try:
        store = initialize_context_store(update.context_type)
        
        # Generate an ID for this context
        doc_id = str(uuid.uuid4())
        
        # Store the raw data
        store["raw_data"][doc_id] = update.data
        
        # Create a text representation for the vector store
        text_representation = json.dumps(update.data)
        
        # Add to vector store
        store["vector_store"].add_texts(
            texts=[text_representation],
            metadatas=[{"id": doc_id, **update.metadata}] if update.metadata else [{"id": doc_id}]
        )
        
        return {"message": "Context updated successfully", "id": doc_id}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

```

### Types of MCP Servers You Can Build

You can build MCP servers for virtually any context type, such as:

1. **Knowledge MCPs**: Provide domain-specific knowledge (medicine, law, finance)
2. **Tool MCPs**: Manage context about available tools and how to use them
3. **User MCPs**: Store user preferences, history, and personalization data
4. **Reasoning MCPs**: Provide templates and examples for specific reasoning patterns
5. **Planning MCPs**: Maintain context about plans, goals, and progress
6. **Multimodal MCPs**: Handle contextual information across different modalities (text, image, audio)

### Key Implementation Considerations

When building your MCP server:

1. **Structured Context Format**: Design a consistent format for your context
2. **Context Relevance**: Implement effective retrieval to provide only relevant context
3. **Stateful vs. Stateless**: Decide if your MCP server maintains state between requests
4. **Context Window Management**: Format context efficiently to work within model token limits
5. **Context Prioritization**: Develop mechanisms to prioritize the most important context
6. **Security**: Implement appropriate access controls for sensitive context

### Invoking MCP Server Tools with LangGraph Agents

When building a LangGraph agent that invokes tools exposed via MCP servers, you need to establish a structured interaction pattern. Here's the best approach:

#### Architecture Overview

1. **Tool Registration**
   - Register MCP server tools with your LangGraph agent
   - Define tool schemas that match the MCP server's API

2. **Invocation Flow**
   - Create specialized nodes in your graph for MCP tool invocation
   - Handle authentication, request formatting, and response parsing

3. **Context Management**
   - Maintain tool execution context across turns
   - Update the agent state with results from MCP tools

##### Best Implementation Approach

Here's a code example showing how to implement this:

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.graph import StateGraph
from langgraph.prebuilt import ToolExecutor, ToolInvocation
from typing import TypedDict, List, Dict, Any
import requests
import json

#### Define state schema
class AgentState(TypedDict):
    messages: List[Dict[str, Any]]
    tools: List[Dict[str, Any]]
    tool_results: Dict[str, Any]
    next_step: str

#### Initialize LLM
llm = ChatGoogleGenerativeAI(model="models/gemini-2.5-pro-experimental")

#### Define MCP tool interfaces
mcp_tools = [
    {
        "name": "add",
        "description": "Add two numbers together",
        "parameters": {
            "type": "object",
            "properties": {
                "a": {"type": "number", "description": "First number"},
                "b": {"type": "number", "description": "Second number"}
            },
            "required": ["a", "b"]
        },
        "server": "http://localhost:8000"  # MCP server endpoint
    },
    # More tools...
]

#### MCP Tool Executor
class MCPToolExecutor:
    def __init__(self, tools):
        self.tools = {t["name"]: t for t in tools}
    
    def execute(self, tool_invocation: ToolInvocation) -> Any:
        """Execute a tool by calling the appropriate MCP server"""
        tool_name = tool_invocation.name
        arguments = tool_invocation.arguments
        
        if tool_name not in self.tools:
            return {"error": f"Tool {tool_name} not found"}
        
        tool = self.tools[tool_name]
        server_url = tool["server"]
        
        # Construct the MCP server request
        response = requests.post(
            f"{server_url}/tools/{tool_name}",
            json=arguments,
            headers={"Content-Type": "application/json"}
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            return {"error": f"Failed to call MCP tool: {response.text}"}

#### Initialize tool executors
mcp_executor = MCPToolExecutor(mcp_tools)

# Create LangGraph nodes
def agent_node(state):
    """Agent reasoning node"""
    messages = state["messages"]
    available_tools = state["tools"]
    
    # Format the tools for the model
    tool_descriptions = "\n".join([
        f"- {tool['name']}: {tool['description']}" for tool in available_tools
    ])
    
    # Create prompt
    prompt = f"""
    You have access to the following tools from MCP servers:
    {tool_descriptions}
    
    To use a tool, respond with:
    ```json
    {{
      "tool": "tool_name",
      "arguments": {{
        "arg1": "value1",
        "arg2": "value2"
      }}
    }}
    ```
    
    User query: {messages[-1]['content']}
    """
    
    # Get response from LLM
    response = llm.invoke(prompt)
    
    # Parse the tool call from response
    try:
        # Extract tool call from response (this will vary based on your model output format)
        tool_call_text = response.content
        # Extract JSON from response if needed
        import re
        json_match = re.search(r'```json\s*(.*?)\s*```', tool_call_text, re.DOTALL)
        if json_match:
            tool_call_json = json.loads(json_match.group(1))
        else:
            tool_call_json = json.loads(tool_call_text)
        
        return {
            "messages": messages + [{"role": "assistant", "content": response.content}],
            "next_step": "execute_tool",
            "tool_call": ToolInvocation(
                name=tool_call_json["tool"],
                arguments=tool_call_json["arguments"]
            )
        }
    except Exception as e:
        # If no tool call is detected
        return {
            "messages": messages + [{"role": "assistant", "content": response.content}],
            "next_step": "end"
        }

def tool_execution_node(state):
    """Node for executing MCP tools"""
    tool_call = state["tool_call"]
    result = mcp_executor.execute(tool_call)
    
    return {
        "messages": state["messages"],
        "tool_results": {**state.get("tool_results", {}), tool_call.name: result},
        "next_step": "response_synthesis"
    }

def response_synthesis_node(state):
    """Node for synthesizing responses after tool execution"""
    messages = state["messages"]
    tool_results = state["tool_results"]
    
    # Create prompt for response synthesis
    tools_output = "\n".join([
        f"- {tool}: {json.dumps(result)}" for tool, result in tool_results.items()
    ])
    
    prompt = f"""
    You used tools and got the following results:
    {tools_output}
    
    Please synthesize a helpful response based on these results.
    """
    
    # Get response from LLM
    response = llm.invoke(prompt)
    
    return {
        "messages": messages + [{"role": "assistant", "content": response.content}],
        "next_step": "end"
    }

#### Create state graph
graph = StateGraph(AgentState)

#### Add nodes
graph.add_node("agent", agent_node)
graph.add_node("execute_tool", tool_execution_node)
graph.add_node("response_synthesis", response_synthesis_node)

#### Add edges
graph.add_edge("agent", "execute_tool")
graph.add_edge("execute_tool", "response_synthesis")
graph.add_edge("response_synthesis", "end")
graph.add_edge("agent", "end")

#### Set entry point
graph.set_entry_point("agent")

#### Compile the graph
agent_executor = graph.compile()

#### Example usage
result = agent_executor.invoke({
    "messages": [{"role": "user", "content": "What is 25 plus 17?"}],
    "tools": mcp_tools,
    "tool_results": {}
})
```

#### Best Practices

1. **Tool Schema Alignment**
   - Ensure your tool definitions match exactly what the MCP server expects
   - Include proper schema definitions for validation

2. **Error Handling**
   - Implement robust error handling for MCP server connectivity issues
   - Have fallback strategies when servers are unavailable

3. **Context Passing**
   - Pass relevant context with each tool invocation
   - Include conversation history when needed

4. **Caching**
   - Cache results from MCP servers when appropriate
   - Implement efficient retries with exponential backoff

5. **Parallel Execution**
   - Consider parallel tool execution for complex workflows
   - Use LangGraph's parallel nodes for multiple MCP tool calls

6. **Dynamic Discovery**
   - Implement dynamic discovery of MCP server capabilities
   - Update available tools based on context

This approach gives you a flexible, maintainable way to integrate MCP server tools with LangGraph agents while preserving the agent's reasoning capabilities.

