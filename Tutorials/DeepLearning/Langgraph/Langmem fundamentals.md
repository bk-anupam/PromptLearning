
### SummarizationNode
The primary goal of `SummarizationNode` is to prevent the conversation history from growing infinitely long. LLMs have a limited context window (the number of tokens they can process at once). If the history becomes too large, you'll either get an error or the model will lose track of early parts of the conversation.

The `SummarizationNode` solves this by watching the conversation length. When it gets too long, the node "compresses" the older messages into a concise summary, which is then fed back to the LLM in subsequent turns.

#### The Core Concept: A Smart Filter for Your LLM

At its heart, the SummarizationNode acts as an intelligent filter that sits between your full, unabridged conversation history and the LLM. Its primary job is to prevent the LLM's
context window from overflowing by condensing old messages into a summary, ensuring the conversation can continue indefinitely.

It achieves this by maintaining a "running summary" and deciding when to update it based on token counts.

#####  The Inner Workings: A Step-by-Step Flow

  Here’s what happens inside the node every time it's called in your LangGraph flow:

  Step 1: Receive the Full Conversation State

   * The node takes the current AgentState as input.
   * It looks for two key pieces of information:
       1. The full message history (from the input_messages_key, which defaults to "messages"). This is the complete, unaltered list of all messages in the conversation, loaded by the
          checkpointer.
       2. The previous summary information (from a context key in the state, which should contain a RunningSummary object). This object tells the node what has already been summarized.

  Step 2: Decide if Summarization is Necessary

  The node's main trigger is the token count.

   * It iterates through the messages that it hasn't seen before (i.e., messages that are not in the RunningSummary).
   * It counts the tokens of these new messages using the provided token_counter function.
   * If the total token count of new messages is *less than* `max_tokens_before_summary`, it does nothing. It simply passes the messages along without creating or updating a summary.
   * If the token count *exceeds* `max_tokens_before_summary`, the summarization process begins.

  Step 3: Generate or Update the Summary
  
  This is the core LLM-powered step.

   * First-time Summary: If there is no RunningSummary from a previous turn, the node takes the block of messages that exceeded the token threshold and sends them to the specified model
    using the initial_summary_prompt. This creates the very first summary.
   * Updating an Existing Summary: If a RunningSummary does exist, the node is more efficient. It takes the new block of messages (the ones that just went over the threshold) and the text 
     of the old summary and sends them to the model using the existing_summary_prompt. This prompt instructs the LLM to integrate the new information into the existing summary, creating an
     updated, consolidated summary.

  Step 4: Construct the New, Condensed Message List

  Once the summary is generated, the node creates the new, shorter list of messages that will be passed to the next node in the graph (e.g., your main agent).

   * It creates a new SystemMessage (or AIMessage depending on the prompt) containing the text of the new summary.
   * It takes all the messages that came after the summarized block (the most recent messages).
   * It combines them into a new list: [summary_message] + [recent_unsummarized_messages].

  Step 5: Update the State and Output

  The node's final job is to update the LangGraph state so the process can repeat correctly on the next turn.

   * It puts the new, condensed message list into the state under the output_messages_key (which defaults to "summarized_messages"). Your agent node will read from this key.
   * It creates or updates the RunningSummary object, storing the text of the new summary and the IDs of all the messages that have now been summarized. It places this object back into the
     state (under the context key).

  This ensures that on the next turn, the node knows exactly where it left off and doesn't re-summarize the same messages.

  **Key Parameters That Control the Behavior**

   * model: The LLM used to perform the summarization. You can use a smaller, faster model for this to save costs.
   * max_tokens_before_summary: This is the trigger. It defines how "long" the conversation can get before the node decides to summarize.
   * max_summary_tokens: This is a budgeting parameter. It tells the node how much space to reserve for the summary message when calculating the total token count. To enforce a strict
     output length, you must bind max_tokens to the model itself (e.g., model.bind(max_tokens=128)).
   * token_counter: The function used to count tokens. For accuracy, you should pass the LLM's own token counting method (e.g., llm.get_num_tokens_from_messages).
   * input_messages_key vs. output_messages_key: By default, the node reads from "messages" and writes to "summarized_messages". This is a crucial design choice: it preserves the full,
     unabridged history in your checkpointer (messages) while providing a separate, clean, summarized list for your agent to work with (summarized_messages).
   
#### Step 1: Initialization (`__init__`)

When you create an instance of the node, you configure its behavior. Let's look at the key parameters from your `graph_builder.py` file:

``` python
summarization_node = SummarizationNode(
    token_counter=llm.get_num_tokens_from_messages,
    model=summarization_model,
    max_tokens=config_instance.MAX_TOKENS_BEFORE_SUMMARY,
    max_tokens_before_summary=config_instance.MAX_TOKENS_BEFORE_SUMMARY,
    max_summary_tokens=config_instance.MAX_SUMMARY_TOKENS,
)
```

- **`model`**: This is the `LanguageModelLike` object (e.g., `ChatGoogleGenerativeAI`) that will be used to generate the actual summary text.
- **`max_tokens_before_summary`**: This is the **trigger**. The node counts the tokens in the message history. As soon as the count exceeds this value, the summarization process begins.
- **`max_tokens`**: This is the **budget** for the summarization payload. When summarization is triggered, the node gathers the most recent messages that fit within this token budget and sends _only that slice_ to the summarization model. As we discovered in our previous conversation, setting this value too low can cause the earliest messages (like the user's name) to be excluded from the summary.
- **`max_summary_tokens`**: This is used for internal token budgeting. The node subtracts this value from `max_tokens` to calculate how many tokens are left for the _non-summarized_ part of the conversation. **Crucially**, this parameter also caused the `TypeError` in our previous interaction because `langmem` tries to automatically bind `max_tokens` to the model, which is incompatible with `ChatGoogleGenerativeAI`.
- **`token_counter`**: A function to count message tokens. Using the model's own counter (`llm.get_num_tokens_from_messages`) is more accurate than the default approximation.

#### Step 2: Execution within the Graph

When the graph runs and hits the `summarization_node`, its `_func` (or `_afunc` for async) method is called. Here is the detailed breakdown of what happens inside.

##### 2.1. Parse Input (`_parse_input`)

The first thing the node does is read the current state of the graph.

- It gets the list of all messages from the state key `messages` (or whatever `input_messages_key` is set to).
- It gets the `context` dictionary from the state. This is where it looks for a `running_summary` object from the previous turn.        
##### 2.2. Preprocess Messages (`_preprocess_messages`)

This is the core logic where the node decides _if_ and _what_ to summarize.

1. **Isolate System Message**: It checks if the first message is a `SystemMessage` and temporarily removes it from consideration, as system prompts are usually not part of the summarized content.
2. **Check for Existing Summary**: It looks inside `state['context']` for a `RunningSummary` object. If one exists, it knows how many messages have already been summarized and what the previous summary was.
3. **Iterate and Count**: It starts iterating through the messages that have _not_ yet been summarized, counting tokens with the `token_counter`.
4. **Find the Cutoff**: It keeps counting until the token count exceeds the `max_tokens_before_summary` trigger. All messages from the start of the new batch up to this cutoff point are marked as `messages_to_summarize`.
5. **Handle Tool Calls**: It includes a smart check: if the very last message in the `messages_to_summarize` slice is an `AIMessage` with tool calls, it will also include the subsequent `ToolMessage`s that correspond to those calls. This is critical to avoid sending an incomplete thought to the summarizer.
#### 2.3. Prepare Input for the Summarizer LLM (`_prepare_input_to_summarization_model`)

Now that the node has a slice of messages to summarize, it prepares the final prompt for the `summarization_model`.

1. **Trim if Necessary**: It first checks if the `messages_to_summarize` slice is larger than the `max_tokens` budget. If so, it uses `trim_messages` to chop off the oldest messages from the slice to ensure it fits within the summarizer's context window.
2. **Select Prompt**:
    - If a `running_summary` exists, it uses `DEFAULT_EXISTING_SUMMARY_PROMPT`, which looks like this:
        > "This is summary of the conversation so far: {existing_summary}\n\nExtend this summary by taking into account the new messages above:"
        
    - If this is the first time summarizing, it uses `DEFAULT_INITIAL_SUMMARY_PROMPT`:        
        > "Create a summary of the conversation above:"
        
3. **Invoke the Model**: It calls `model.invoke()` with the chosen prompt and the `messages_to_summarize`. The LLM returns a new, updated summary string.

### How agent memory works in langgraph (for RAG BOT):

With the use of checkpointer while creating a StateGraph object in langgraph, the entire graph state (including all the messages ) is persisted to the checkpoint store (Firestore in case of RAG BOT). 

In a LangGraph StateGraph object initialized with a checkpointer, **the state is stored to the checkpoint storage after every step (super-step) in the graph execution**. This means a checkpoint is automatically created:

- Before any computation (for the initial input state)    
- After every node execution (when a node finishes and before the next starts)
- Upon graph completion (final state)[](https://langchain-ai.github.io/langgraph/concepts/persistence/)    
#### Precise Points Where State is Stored

1. **Before Graph Execution**  
    A checkpoint is created with the input (start) state and indicates which node will execute first.
    
2. **After Each Node Execution**  
    When a node completes and returns an updated state, a checkpoint is saved that captures:
    
    - The current values of all state channels        
    - The next node(s) to execute        
    - Associated metadata (writes, node name, step number)
        
3. **After the Final Node**  
    When the terminal node (END) is reached and execution finishes, a final checkpoint is written representing graph completion.

This checkpointing is **automatic**—if you compile your StateGraph with a checkpointer and provide a `thread_id` when invoking, you do not have to manually save the state.

#### The `AIMessage` and the Tool Call

First, it's important to clarify the relationship: A "tool call" is **not** a message type itself. It is a special component _inside_ an `AIMessage`.

When the agent needs to use a tool, the LLM generates an `AIMessage` that contains instructions to run one or more tools. This `AIMessage` is the LLM's "turn" in the conversation.

**Example `AIMessage`:**

Following a user query like "What did Baba say about soul consciousness on 1969-01-23?", the `agent_node` would receive an `AIMessage` from the LLM that looks like this:

```python
# This is an AIMessage object
AIMessage(
    content="",  # The text content is often empty when calling tools.
    tool_calls=[
        {
            "name": "retrieve_context",
            "args": {
                "query": "teachings on soul consciousness from murli on 1969-01-23",
                "date_filter": "1969-01-23",
                "language": "en"
            },
            "id": "tool_call_a1b2c3d4" # A unique ID for this specific call.
        }
    ]
)
```
**What this `AIMessage` contains:**

- **`content`**: This would hold a text response if the LLM were answering directly. Here, it's empty because the LLM's action is to call a tool, not to speak.
- **`tool_calls`**: This is a list of one or more tool call instructions. Each instruction is a dictionary with:
    - **`name`**: The name of the function to execute. This must match the name of a tool provided to the agent (in your case, `retrieve_context` from `[/home/bk_anupam/code/LLM_agents/RAG_BOT/src/context_retrieval/context_retriever_tool.py]
    - **`args`**: A dictionary of arguments that the LLM has extracted from the user's query, matching the parameters defined in the tool's signature (`query`, `date_filter`, `language`).
    - **`id`**: A unique identifier generated by the framework for this specific tool call. This is crucial for matching the result back to the request.
    
#### The `ToolMessage` (The Result)

After the framework executes the tool call from the `AIMessage`, the _output_ of that tool is wrapped in a `ToolMessage`. This `ToolMessage` represents the tool's "turn" in the conversation history.

Your `retrieve_context` tool is defined with `response_format="content_and_artifact"`. This is a special instruction that tells LangGraph how to structure the `ToolMessage` from the tool's return value, which is a tuple: `(status_string, list_of_documents)`.

**Example `ToolMessage`:**

After executing the `retrieve_context` tool, the following `ToolMessage` would be added to the agent's state:

```python
# This is a ToolMessage object
ToolMessage(
    # The first item of the tool's return tuple goes here.
    content="Retrieved 20 semantic + 10 BM25 chunks. Reconstructed context for 5 retrieved chunks.",

    # The second item of the tool's return tuple goes here.
    artifact=[
        (
            "Full text of the first reconstructed murli...",
            {"source": "Murli from 1969-01-23 (en)", "date": "1969-01-23", ...}
        ),
        (
            "Full text of the second reconstructed murli...",
            {"source": "Murli from 1969-01-23 (en)", "date": "1969-01-23", ...}
        )
        # ... more (content, metadata) tuples
    ],

    # This ID links the result back to the specific tool call in the AIMessage.
    tool_call_id="tool_call_a1b2c3d4"
)
```
**What this `ToolMessage` contains:**

- **`content`**: This holds the first part of your tool's return value—the human-readable status string.
- **`artifact`**: This is a special field that holds the second part of the return value—the complex Python object (the list of document tuples). This is incredibly useful because it avoids having to serialize and deserialize complex data into a string. Your `process_tool_output_node` then accesses this `artifact` directly to get the documents. It finds the `artifact` (the list of documents) and populates the `documents` field within the `AgentState`. After this node runs, the `AgentState` now contains the content and metadata of the documents retrieved by your tool.
- **`tool_call_id`**: This is the critical link. It contains the same ID (`tool_call_a1b2c3d4`) from the `AIMessage`'s tool call, telling the agent exactly which request this result corresponds to.

```python 
# /home/bk_anupam/code/LLM_agents/RAG_BOT/src/agent/process_tool_output_node.py

def _process_retrieve_context_success(doc_items_artifact: List[Tuple[str, Dict[str, Any]]]) -> Dict[str, Any]:
    # ... (validation logic) ...
    
    updated_state = _create_default_state()
    updated_state.update({
        "documents": _convert_to_documents(valid_doc_items), # <--- Here!
        "last_retrieval_source": "local",
        "web_search_attempted": False
    })
    
    logger.info(f"Processed {len(valid_doc_items)} hybrid local documents retrieved using retrieve_context and updated agent state.")
    return updated_state

```

**The lifecycle of retrieved documents in the agent's state:**

1. **Tool Call and `ToolMessage` Creation**: When the `retrieve_context` tool is executed, its return value (a status string and a list of documents) is packaged into a `ToolMessage`. The list of documents is placed in the `ToolMessage.artifact` field.
    
2. **Retention in Message History**: This `ToolMessage`, complete with its artifact containing the full document contents, is appended to the `messages` list in the `AgentState`.
    
3. **Persistence in Checkpoint**: When the agent saves a checkpoint to Firestore, the entire `AgentState` is serialized. This includes the `messages` list, so the `ToolMessage` and its embedded documents are permanently stored as part of that conversational step's history.
    
4. **Summarization**: Your final point is also correct. When the `LoggingSummarizationNode` is triggered, it takes a chunk of the oldest messages from the history—including any `HumanMessage`, `AIMessage`, and `ToolMessage` objects in that chunk—and replaces them with a single, new summary message. So, the verbose `ToolMessage` objects are indeed removed from the _active_ conversational window and condensed into the summary.

**The RAG BOT agent is intentionally designed _not_ to send the entire message history to the LLM for every new query.**

Instead, it strategically uses different parts of its memory for different tasks to ensure accuracy, prevent the LLM from getting confused, and manage costs. Here’s a breakdown of how it works for a new user question that requires information retrieval (a RAG query):

#### 1. For Deciding What to Do (Routing & Tool Calling)

When your new question arrives, the agent's first job is to understand your intent and decide which tool to use. For this, it deliberately uses a very narrow and focused context.

- **Router Node (`[router_node.py]`):** To decide if your query is a RAG question or just conversational chat, the router looks **only at the current user message**. It doesn't use the past conversation, ensuring a clean, unbiased classification of your immediate intent.
- **Agent Node (`[agent_node.py]`):** When deciding which tool to call (e.g., `retrieve_context`), this node also uses an "isolated prompt." It sends the LLM only the system instructions and your **current user message**.

**Why?** This prevents the LLM from being influenced by previous turns. If you asked about "soul consciousness" three messages ago and now you ask "what about the drama?", the agent needs to call the `retrieve_context` tool for "drama" without being distracted by the old topic. Using only the current query ensures the tool call is precise.

#### 2. For Generating the Final Answer (Grounding)

After the agent has called the `retrieve_context` tool and has the necessary information, its final step is to generate the answer.

- **Agent Node (`generate_final_response` in `[agent_node.py]`):** To craft the final answer, the agent sends the LLM a prompt containing:
    1. The `system_base` prompt (its persona and instructions).
    2. The `original_query` from the user.
    3. The `retrieved_context` from the tools.

Notice what's missing: the message history. This is a crucial RAG (Retrieval-Augmented Generation) best practice. By _not_ including the conversational history at this stage, the agent forces the LLM to base its answer **only on the facts provided in the retrieved context**. This dramatically reduces the chance of hallucination and ensures the answer is grounded and factual.

#### 3. When IS the Full History Used?

The message history is primarily used in two specific scenarios:

- **Conversational Node (`[conversational_node.py]`):** If the router determines your query is about the conversation itself (e.g., "summarize our chat"), this is the _only_ time the message history is sent to the LLM. Even then, it's a **cleaned version** of the history, with the noisy `ToolMessage` and tool-call `AIMessage` objects filtered out by the `get_conversational_history` utility.
- **Summarization Node (`[custom_nodes.py]`):** When the conversation gets too long, this node uses the message history to create a summary, which then replaces the older messages. This is how the agent maintains long-term memory without exceeding token limits.