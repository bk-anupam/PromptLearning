#langgraph #agents #rag_bot
## Prompt:
Here is the complete code from previous question can you explain what is the difference between add_node and add_edge methods. And how these are used for creation of an agent. Explain in detail how both add_node and add_edge methods work in the context of code provided, explain each of the parameters to these methods and what is their use.

How agent execution happens, basically all the details what happens when the agent is first invoked.

Also the retriever node output is a dictionary with key 'context'. How is this output from retriever node handled by langgraph and made available to the next node in the graph which is the generator node.

Also in generator node which is invoking the llm (gemini) we are passing in a list of messages to llm.invoke method. Is this correct. I have seen elsewhere that you pass in a dictionary with 'messages' key and list of messages as the value. Which invocation method is correct?

Code:
```python
from langchain_chroma import Chroma
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.prompts import PromptTemplate
from langchain.schema import HumanMessage, AIMessage
from langgraph.graph import START, StateGraph
from vector_store import load_existing_index  
from config import Config
import datetime
from logger import logger

def retriever_node(state: dict, vectordb: Chroma, k: int = 25, date_filter: str = None, 
                     search_type: str = "similarity", score_threshold: float = 0.5) -> dict:
    """Retrieves documents from the vector database with improved query normalization and filtering."""
    raw_query = state["query"]
    query = raw_query.strip().lower()  # Normalize query
    extra_kwargs = {"k": k, "score_threshold": score_threshold}

    if date_filter:
        try:
            filter_date = datetime.datetime.strptime(date_filter, '%Y-%m-%d')
            formatted_date = filter_date.strftime('%Y-%m-%d')
            logger.info(f"Applying date filter: {formatted_date}")
        except ValueError:
            raise ValueError("Invalid date format. Please use YYYY-MM-DD.")

        filter_criteria = {"date": {"$eq": formatted_date}}
        extra_kwargs["filter"] = filter_criteria
        # Optionally, if available, switch search type for filtered queries
        search_type = "similarity"  # or another type like "mmr" if appropriate

    # Create retriever with refined kwargs and search type option
    retriever = vectordb.as_retriever(search_type=search_type, search_kwargs=extra_kwargs)
    retrieved_docs = retriever.invoke(query)
    context = "\n\n".join([doc.page_content for doc in retrieved_docs])
    logger.info(f"Retrieved {len(retrieved_docs)} documents for query: {query}")
    return {"context": context}


def generator_node(state: dict, llm: ChatGoogleGenerativeAI) -> dict:
    """Generates a response using the LLM and retrieved context."""
    system_prompt_text = Config.SYSTEM_PROMPT
    custom_prompt = PromptTemplate(
        input_variables=["context", "question"],
        template=(
            system_prompt_text +
            "Context: {context}\n"
            "Question: {question}\n"
        ),
    )
    context = state["context"]
    query = state["query"]
    formatted_query = custom_prompt.format(context=context, question=query)
    messages = [HumanMessage(content=formatted_query)]
    response = llm.invoke(messages)
    if isinstance(response, AIMessage):
        return {"answer": response.content}
    else:
        return {"answer": str(response)}


def build_agent(vectordb: Chroma, date_filter: str = None, k: int = 25, 
                model_name: str = "gemini-2.0-flash") -> StateGraph:
    """Builds and returns a persistent LangGraph agent."""
    llm = ChatGoogleGenerativeAI(model=model_name, temperature=0.3)
    builder = StateGraph(dict)
    builder.add_node("retriever", lambda state: retriever_node(state, vectordb, k, date_filter))
    builder.add_node("generator", lambda state: generator_node(state, llm))
    builder.add_edge(START, "retriever", {"query": None})
    builder.add_edge("retriever", "generator")
    return builder.compile()
```
## Response:

Let's break down the LangGraph agent creation process using the provided code.

**Difference between `add_node` and `add_edge` methods:**

* **`add_node`:** This method is used to define the individual processing units within your LangGraph. Each node represents a specific function or operation that will be executed as part of the agent's workflow. In your code, `retriever_node` and `generator_node` are defined as functions, and `add_node` is used to incorporate them as nodes in the graph. Think of nodes as the *steps* or *stages* in your agent's process.

* **`add_edge`:** This method defines the connections and the flow of information between the nodes in your LangGraph. Edges specify the order in which the nodes will be executed and how the output of one node is passed to the next. In your code, `add_edge` is used to connect the `START` of the graph to the `retriever` node and then connect the `retriever` node to the `generator` node. Think of edges as the *pathways* or *transitions* that dictate the agent's execution flow.

**How these are used for the creation of an agent:**

You build an agent in LangGraph by first defining the individual processing steps as functions (like `retriever_node` and `generator_node`). Then, you use `add_node` to register these functions as nodes within a `StateGraph` builder. Finally, you use `add_edge` to define the sequence in which these nodes should be executed, essentially creating the workflow of your agent. The `builder.compile()` step then finalizes the graph into an executable agent.

**Detailed explanation of `add_node` and `add_edge` parameters in the context of the code:**

**`builder.add_node(key, func)`**

* **`key` (string):** This is the unique identifier or name you assign to the node within your graph. In your code:
    * `"retriever"`: This key identifies the node that executes the `retriever_node` function.
    * `"generator"`: This key identifies the node that executes the `generator_node` function.
    The `key` allows you to refer to this specific node when defining the edges in your graph.

* **`func` (callable):** This is the function that will be executed when this node is visited during the agent's execution. The function should take a `state` dictionary as its first argument. In your code:
    * `lambda state: retriever_node(state, vectordb, k, date_filter)`: This lambda function takes the current `state` and passes it, along with the `vectordb`, `k`, and `date_filter` variables (captured from the surrounding scope), to the `retriever_node` function.
    * `lambda state: generator_node(state, llm)`: This lambda function takes the current `state` and passes it, along with the initialized `llm`, to the `generator_node` function.
    The use of lambda functions here allows you to pass additional arguments (like `vectordb`, `llm`, `k`, `date_filter`) to your node functions.

**`builder.add_edge(start_key, end_key, config=None)`**

* **`start_key` (string or `langgraph.graph.START`):** This specifies the node from which the edge originates. It can be either the `key` of a previously added node or the special `START` object indicating the beginning of the graph execution. In your code:
    * `START`: This indicates that the execution of the graph begins at this edge, leading to the `"retriever"` node.
    * `"retriever"`: This indicates that after the `"retriever"` node finishes execution, the flow moves to the `"generator"` node.

* **`end_key` (string):** This specifies the node to which the edge leads. It should be the `key` of a previously added node. In your code:
    * `"retriever"`: This means the execution flows from the `START` to the node identified by the key `"retriever"`.
    * `"generator"`: This means the execution flows from the node identified by the key `"retriever"` to the node identified by the key `"generator"`.

* **`config` (dict, optional):** This argument allows you to specify conditions or data transformations for the transition between nodes. In your code:
    * `{"query": None}` in the edge from `START` to `"retriever"`: This `config` dictionary specifies a mapping. It indicates that the input provided when the agent is invoked (using `agent.invoke()`) should be placed under the key `"query"` in the `state` that is passed to the `"retriever"` node. This is how the initial query gets into the agent's workflow.

**How agent execution happens when first invoked:**

When the compiled agent's `.invoke()` method is called with an input dictionary (e.g., `agent.invoke({"query": "your question"})`), the following happens:

1.  **Initialization:** The execution starts from the implicit `START` of the `StateGraph`.
2.  **First Edge:** The graph follows the edge defined from `START` to the `"retriever"` node. The `config` associated with this edge, `{"query": None}`, indicates that the input provided to `.invoke()` will be made available in the agent's internal state under the key `"query"`.
3.  **`retriever_node` Execution:** The `retriever_node` function is executed. It receives the current state (which now contains your input query).
    * It extracts the query from the state: `raw_query = state["query"]`.
    * It normalizes the query.
    * It constructs the retrieval arguments (`k`, `search_type`, `score_threshold`, and potentially a `date_filter`).
    * It uses the provided `vectordb` (Chroma instance) to retrieve relevant documents based on the query and filter (if any).
    * It extracts the content from the retrieved documents and joins them into a single string stored in the `context` variable.
    * Finally, it returns a dictionary `{"context": context}`. This dictionary represents the output of the `"retriever"` node and will update the agent's state.
4.  **Second Edge:** The graph follows the edge from the `"retriever"` node to the `"generator"` node. By default, LangGraph merges the output of the previous node into the current state. So, the state passed to the `"generator"` node will now contain the original `"query"` and the newly added `"context"` from the `retriever_node` output.
5.  **`generator_node` Execution:** The `generator_node` function is executed. It receives the updated state.
    * It retrieves the `context` and `query` from the state: `context = state["context"]`, `query = state["query"]`.
    * It formats a prompt using the `system_prompt_text` from your `Config` and the retrieved `context` and `query`.
    * It creates a list of messages for the language model, in this case, a single `HumanMessage` with the formatted query.
    * It invokes the provided `llm` (Gemini) with this list of messages using `llm.invoke(messages)`.
    * It processes the `response` from the LLM. If it's an `AIMessage`, it extracts the content; otherwise, it converts the response to a string.
    * Finally, it returns a dictionary `{"answer": response.content}` or `{"answer": str(response)}`. This represents the final output of the agent.
6.  **Return Value:** The `.invoke()` method will return the final state of the graph, which in this case will likely contain the `"answer"` generated by the `"generator"` node.

**How the output from the retriever node is handled by LangGraph and made available to the generator node:**

LangGraph automatically manages the flow of data between nodes through the agent's state. When a node function (like `retriever_node`) returns a dictionary, the key-value pairs in this dictionary are merged into the agent's current state.

In your case, `retriever_node` returns `{"context": context}`. LangGraph takes this output and updates the state. When the execution moves to the next node, `"generator"`, the `generator_node` function receives this updated state as its `state` argument. Therefore, `generator_node` can access the retrieved context using `state["context"]`.

**LLM invocation in `generator_node`:**

Yes, in the provided code, you are correctly invoking the `ChatGoogleGenerativeAI` model using the `.invoke()` method with a list of messages:

```python
response = llm.invoke(messages)
```

The `messages` variable is a list containing a single `HumanMessage` in this case.

Regarding the alternative you've seen where a dictionary with a `'messages'` key is passed, this might be the format expected by a different LLM integration or a different method of invocation. For `langchain-google-genai.ChatGoogleGenerativeAI`, the `.invoke()` method typically accepts a list of messages (which can include `HumanMessage`, `AIMessage`, `SystemMessage`, etc.).

It's always best to refer to the documentation of the specific LLM integration you are using to confirm the correct format for invoking the model. However, based on your provided code and common LangChain patterns, the way you are invoking `llm.invoke(messages)` is correct for `ChatGoogleGenerativeAI`.

**How ```builder.add_edge(START, "retriever", {"query": None})``` Works**

Let's dissect this line:

```builder.add_edge(START, "retriever", {"query": None})```
- START: The starting point of the graph.
- "retriever": The name of the first node that will be executed.
- {"query": None}: This is not a third positional argument to add_edge in the way you might initially think. Instead, it's a way to specify the initial state that will be passed to the "retriever" node when the graph starts.

**What Happens Internally**

```1. StateGraph's add_edge (in state.py)```

- When add_edge is called with START as the start_key, the code in state.py does not directly handle the third argument.
- Instead, it creates an edge from START to "retriever" and it also does something else.
- It sets up the initial state for the graph.
- The {"query": None} part is used to define the initial state of the graph. It means that when the graph starts, the state will be a dictionary with a single key-value pair: {"query": None}.
- This initial state is then passed to the first node connected to START (in this case, the "retriever" node).
- The attach_edge method in CompiledStateGraph (in state.py) is where the magic happens. It creates a channel between START and "retriever". It also sets up the PregelNode for START to write to this channel.
- The _get_root function in CompiledStateGraph is used to extract the initial state from the input.

```2. retriever_node's Input:```

- The retriever_node function (in rag_agent.py) is defined as retriever_node(state: dict, ...)
- It expects a state dictionary as its first argument.
- When the graph starts, the state dictionary passed to retriever_node will initially be {"query": None}.
- Later, when the graph is invoked, the user will pass in a dictionary that will update the state. For example, agent.invoke({"query": "What is the meaning of life?"}) will update the state to {"query": "What is the meaning of life?"}.

**In Summary**

- The add_edge method in StateGraph is overloaded. When the start_key is START, it has special behavior.
- The third argument {"query": None} is not a direct argument to add_edge in the same way as start_key and end_key.
- It's a way to define the initial state of the graph, which is a dictionary.
- This initial state is passed to the first node connected to START.
- The attach_edge method in CompiledStateGraph is responsible for setting up the channel between START and the first node.
- The _get_root function in CompiledStateGraph is responsible for extracting the initial state from the input.

**Why This Design?**

This design allows for:

- Clear Initialization: You can explicitly define the initial state of your graph, ensuring that the first node has the necessary data to start processing.
- State Management: StateGraph can manage the state of the graph, passing it between nodes.
- Flexibility: You can have different initial states for different graphs.****