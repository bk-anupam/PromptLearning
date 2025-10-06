#langgraph #agents #rag_bot
[[Langgraph agent execution internals]]
## Prompt:
Can you explain the code in rag_agent.py step by step in detail. Basically I want to understand in detail how langgraph works under the hood to make the rag_agent functional. I understand that the code is building an agent that answers user's questions about Brahmakumaris murlis. It uses a murli_retriever_tool to retrieve murli extracts if it feels additional context is needed to answer the user's question. I basically want to understand in detail how the message flow happens, what are the different kinds of messages in langgraph, what is their use and how the code uses the messages in its workflow. Also I want to understand how langgraph performs state management of the agent, after execution of each node in the graph what exactly does langgraph do with the agent state.

## Response:

```python
from typing import List, TypedDict, Optional, Annotated
from operator import itemgetter

# Define the state structure for the graph
class AgentState(TypedDict):
    # messages is the primary list for interaction history and tool calls/results  
    messages: Annotated[List[BaseMessage], itemgetter("messages")]
    # Optional: Keep query for logging or specific needs, but flow relies on messages
    query: Optional[str]
```

Okay, let's break down that `AgentState` definition and the roles of `Annotated` and `itemgetter` within the context of LangGraph.

**Context: Agent State in LangGraph**

* **`AgentState(TypedDict)`**: In LangGraph, the `AgentState` is a dictionary (specifically a `TypedDict` for better type hinting) that holds all the information passed between the different steps (nodes) of your agent graph. As the agent processes information, makes decisions, or calls tools, the `AgentState` is updated.

* **State Management**: LangGraph needs to know how to update this state dictionary when a node finishes its execution and returns some data. Should it overwrite existing values? Should it append to lists? Should it merge dictionaries? This is where annotations come in.

**Explanation of the Code Line:**

```python
messages: Annotated[List[BaseMessage], itemgetter("messages")]
```

1.  **`messages: List[BaseMessage]`**:

    * This is standard Python type hinting. It declares that the `AgentState` dictionary has a key named `messages`.
    * The value associated with this `messages` key is expected to be a list (`List`) where each element is an instance of `BaseMessage` (or one of its subclasses like `HumanMessage`, `AIMessage`, `ToolMessage` from LangChain). This list typically holds the conversation history.

2.  **`Annotated[...]`**:

    * Imported from Python's `typing` module (`from typing import Annotated`).
    * `Annotated` allows you to add **metadata** to a type hint. It doesn't change the fundamental type (`List[BaseMessage]` here), but it attaches extra information that can be used by libraries or tools – in this case, LangGraph.
    * The first argument to `Annotated` is the original type hint (`List[BaseMessage]`).
    * Subsequent arguments are the metadata annotations.

3.  **`itemgetter("messages")`**:

    * Imported from Python's built-in `operator` module (`from operator import itemgetter`).

    * `itemgetter("messages")` creates a callable object (like a function). When this callable is given a dictionary (or any object that supports item access), it retrieves the item associated with the key `"messages"`. For example, if `d = {"messages": [1, 2], "other_key": 3}`, then `itemgetter("messages")(d)` would return `[1, 2]`.

**Purpose in LangGraph State Management:**

When you use `Annotated[SomeType, itemgetter("some_key")]` in your `AgentState` definition, you are providing instructions to LangGraph's state management system:

* **Selecting Update Data:** It tells LangGraph: "When a node in the graph finishes and returns a dictionary (representing a partial state update), look for the key named `"messages"` within that returned dictionary. The value associated with *that* key is what should be used to update the `messages` field in the main `AgentState`."

* **Implicit Accumulation (Common Pattern):** While `itemgetter` itself only *selects* the data, LangGraph's `StateGraph` often has default behaviors for merging state. When it sees a key named `messages` (a common convention for history) which is a list, and it receives new messages selected via `itemgetter("messages")`, the default mechanism is usually to **append** or **extend** the existing list in the state with the new messages returned by the node. This effectively accumulates the conversation history.

**Why not just use `messages: List[BaseMessage]`?**

Without the `Annotated` metadata, LangGraph might not know how to merge partial updates intelligently. It might default to simply overwriting the entire `messages` list with whatever a node returns for that key, which would lose the conversation history. The annotation provides the necessary hint for proper merging/accumulation.

**Alternative (More Explicit Accumulation):**

You might also see `operator.add` used in annotations, like:

```python
messages: Annotated[List[BaseMessage], operator.add]
```

This is a more explicit way to tell LangGraph to use addition (which for lists means concatenation/appending) to merge the state updates for the `messages` field. Both `itemgetter("messages")` and `operator.add` are common ways to achieve the accumulation of message history in LangGraph state, relying on LangGraph interpreting these annotations to manage the state updates correctly.

**In summary:** `Annotated` attaches metadata, and `itemgetter("messages")` is that metadata, instructing LangGraph to specifically look for the `"messages"` key in a node's output dictionary to find the data needed to update the main `AgentState`'s `messages` list, usually resulting in the accumulation of conversation messages.

### Execution Flow and State Management (Putting it all together)

Let's trace an invocation: app = build_agent(vectordb_instance) followed by app.invoke({"messages": [HumanMessage(content="What is effort according to the murlis?")]})

1. Initial State: {"messages": [HumanMessage(content="What is effort according to the murlis?")], "query": None}

2. Entry Point -> "agent": The graph starts at the agent node. agent_node_runnable is called with the initial state.

3. Inside agent_node (1st time):

    - messages_for_llm becomes [SystemMessage(...), HumanMessage(...)].

    - llm_with_tools is invoked. Let's assume the LLM decides it needs context.

    - response becomes AIMessage(content='', tool_calls=[ToolCall(name='retrieve_murli_context', args={'query': 'effort according to murlis'}, id='call_123')]).

    - The node returns {"messages": [response]}.

4. State Update 1: LangGraph receives the return dictionary. It sees the messages key is annotated with itemgetter. It appends the new AIMessage to the existing list.

    - New State: {"messages": [HumanMessage(...), AIMessage(tool_calls=[...])], "query": "What is effort according to the murlis?"} (query was also updated).

5. Conditional Edge from "agent": tools_condition is called. It looks at the last message added (AIMessage(tool_calls=[...])). It finds tool_calls. It returns "action".

6. Routing -> "action": The graph routes execution to the action node (ToolNode).

7. Inside action node (ToolNode):

    - It receives the current state.

    - It examines the last message (AIMessage(tool_calls=[...])).

    - It finds a tool_call matching the name of one of its tools (bound_murli_retriever).

    - It extracts the arguments ({'query': 'effort according to murlis'}).

    - It calls bound_murli_retriever(query='effort according to murlis'). Remember, vectordb was already bound using partial.

    - Let's say the tool returns the context string: "Murli excerpt about effort...".

    - ToolNode wraps this result in a ToolMessage: ToolMessage(content="Murli excerpt about effort...", tool_call_id='call_123').

    - The node returns {"messages": [ToolMessage(...)]}.

8. State Update 2: LangGraph receives the return dictionary. It appends the new ToolMessage.

    - New State: {"messages": [HumanMessage(...), AIMessage(tool_calls=[...]), ToolMessage(...)]}.

9. Edge from "action": The graph follows the unconditional edge action -> agent.

10. Routing -> "agent": Execution goes back to the agent node. agent_node_runnable is called again, but this time with the updated state.

11. Inside agent_node (2nd time):

    - messages_for_llm becomes [SystemMessage(...), HumanMessage(...), AIMessage(tool_calls=[...]), ToolMessage(...)].

    - llm_with_tools is invoked with this richer history, including the tool result.

    - The LLM now sees the original question and the retrieved context. It generates a final answer.

    - response becomes AIMessage(content="Based on the murli excerpt, effort means..."). It has no tool_calls this time.

    - The node returns {"messages": [response]}.

12. State Update 3: LangGraph appends the new final AIMessage.

    - New State: {"messages": [HumanMessage(...), AIMessage(tool_calls=[...]), ToolMessage(...), AIMessage(content="...")]}.

13. Conditional Edge from "agent": tools_condition is called again. It looks at the last message (AIMessage(content="...")). It finds no tool_calls. It returns END.

14. Routing -> END: The graph execution terminates.

15. Final Result: The invoke call returns the final state dictionary. The answer is typically extracted from the content of the last AIMessage in the messages list.

Summary of LangGraph Mechanics:

- State: A central dictionary (AgentState) holds all information.

- Nodes: Functions (agent_node) or pre-built objects (ToolNode) that receive the current state and return a dictionary representing updates to the state.

- State Updates: LangGraph merges the dictionary returned by a node into the main state. Special annotations (Annotated[..., itemgetter(...)]) control how updates happen (e.g., append vs. overwrite).

- Edges: Define the possible paths between nodes.

- Conditional Edges: Use functions (like tools_condition) that inspect the current state (usually the most recent update) to decide which path to take next.

- Messages: The primary way information (user input, LLM thoughts/calls, tool results, final answers) flows through the system, stored sequentially in the messages list within the state.

- Compilation: Turns the graph definition into an optimized, runnable object.