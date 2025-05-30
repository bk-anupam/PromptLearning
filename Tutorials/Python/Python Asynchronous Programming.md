#python #asynchronous_programming

Let us walk through the event loop mechanism by first establishing how Python normally executes code, then showing you how asynchronous programming fundamentally changes this execution model.

## How Synchronous Python Code Executes

When you run regular Python code, the interpreter follows a straightforward, linear execution model. Think of it like reading a book from top to bottom - each line gets executed one after another, and the program waits for each operation to complete before moving to the next one.

```python
# Synchronous execution example
print("Starting task 1")
result1 = expensive_operation()  # Program waits here until complete
print("Task 1 finished")
print("Starting task 2") 
result2 = another_operation()    # Program waits here too
print("Task 2 finished")
```

In this model, if `expensive_operation()` takes 5 seconds to complete, your entire program is essentially frozen during those 5 seconds. The Python interpreter has a single thread of execution, and it's completely occupied waiting for that operation to finish.

## The Problem This Creates

Imagine you're building a web server that needs to handle multiple requests simultaneously. With synchronous code, if one request involves reading a large file from disk or making a network call, all other incoming requests have to wait in line. This is incredibly inefficient because during I/O operations (file reading, network calls, database queries), the CPU is mostly idle - it's just waiting for external systems to respond.

## How Asynchronous Code Changes Everything

Asynchronous programming introduces a completely different execution model. Instead of waiting for each operation to complete, the program can start multiple operations and then efficiently switch between them as they make progress.

```python
# Asynchronous execution example
import asyncio

async def main():
    print("Starting both tasks")
    # Both operations start almost simultaneously
    task1 = asyncio.create_task(async_operation_1())
    task2 = asyncio.create_task(async_operation_2())
    
    # Program can do other work while waiting
    print("Both tasks are running in the background")
    
    # Wait for both to complete
    result1 = await task1
    result2 = await task2
    print("Both tasks finished")
```

The key insight here is that when an async function encounters an operation that would normally cause waiting (like a network request), it can voluntarily give up control and let other operations run instead of blocking the entire program.

## The Event Loop

The event loop is the heart of this asynchronous execution model. Think of it as a sophisticated traffic controller that manages all the concurrent operations in your program. Here's how it fundamentally works:

The event loop maintains several key data structures:

- A queue of tasks ready to run
- A registry of I/O operations that are currently in progress
- Timers for scheduled operations

When you run asynchronous code, here's what happens step by step:

1. **Event Loop Creation**: When you use `asyncio.run()` or create an event loop manually with `asyncio.new_event_loop()`, you're creating an instance of the event loop.
2. **Task Scheduling**: When you create a task using `asyncio.create_task()` or `loop.create_task()`, you're scheduling a coroutine to run on the event loop.
3. **Execution Cycle**: The event loop starts an infinite loop where it repeatedly:
    - Picks a task from the ready queue and runs it
    - When the task hits an `await` statement, it checks if the awaited operation is complete
    - If not complete, the task gets suspended and control returns to the event loop
    - The loop then picks another ready task and repeats, executing tasks, and managing IO operations until there are no more tasks to run or it's explicitly stopped.

```python
# Here's what's happening conceptually inside the event loop
async def simulate_event_loop():
    ready_tasks = [task1, task2, task3]
    waiting_tasks = {}
    
    while ready_tasks or waiting_tasks:
        # Run ready tasks
        for task in ready_tasks:
            try:
                task.run_until_await()  # Run until it hits 'await'
                if task.is_waiting_for_io():
                    waiting_tasks[task] = task.io_operation
                    ready_tasks.remove(task)
            except TaskComplete:
                ready_tasks.remove(task)
        
        # Check if any I/O operations completed
        for task, io_op in waiting_tasks.items():
            if io_op.is_complete():
                ready_tasks.append(task)
                del waiting_tasks[task]
```

## The Critical Difference in Execution Flow

In synchronous code, the execution flow is:

```
Start → Operation 1 → Wait → Complete → Operation 2 → Wait → Complete → End
```

In asynchronous code with an event loop, the execution flow becomes:

```
Start → Begin Op1 → Switch to Op2 → Switch to Op3 → Check Op1 → Continue Op1 → Check Op2 → Continue Op2 → End
```

The event loop enables this switching by maintaining the state of each operation and knowing exactly where to resume when an operation becomes ready to continue.

## A Concrete Example

Let me show you a practical example that demonstrates this difference:

```python
import asyncio
import time

# Synchronous version - takes about 6 seconds total
def sync_example():
    start = time.time()
    
    def slow_operation(name, duration):
        print(f"Starting {name}")
        time.sleep(duration)  # Simulates I/O operation
        print(f"Finished {name}")
        return f"Result from {name}"
    
    result1 = slow_operation("Task 1", 2)
    result2 = slow_operation("Task 2", 2) 
    result3 = slow_operation("Task 3", 2)
    
    print(f"Total time: {time.time() - start:.2f} seconds")

# Asynchronous version - takes about 2 seconds total
async def async_example():
    start = time.time()
    
    async def slow_operation(name, duration):
        print(f"Starting {name}")
        await asyncio.sleep(duration)  # Simulates async I/O operation
        print(f"Finished {name}")
        return f"Result from {name}"
    
    # All three operations run concurrently
    results = await asyncio.gather(
        slow_operation("Task 1", 2),
        slow_operation("Task 2", 2),
        slow_operation("Task 3", 2)
    )
    
    print(f"Total time: {time.time() - start:.2f} seconds")

# Run the async version
asyncio.run(async_example())
```

In the synchronous version, each task blocks the entire program. In the asynchronous version, the event loop allows all three tasks to run concurrently, dramatically reducing the total execution time.

The event loop makes this possible by continuously cycling through all active tasks, giving each one a chance to make progress whenever they're not waiting for I/O operations to complete. This cooperative multitasking approach is what makes asynchronous Python so powerful for I/O-intensive applications.

Does this help clarify how the event loop fundamentally changes Python's execution model? Would you like me to dive deeper into any particular aspect of this mechanism?

## Asynchronous Programming in Python with `async` and `await`

Asynchronous programming is a concurrency model that allows a single thread to handle multiple I/O-bound operations concurrently. Instead of waiting for one operation to complete before starting the next (as in synchronous programming), an asynchronous program can start an I/O operation and then yield control to the event loop, which can then proceed with other tasks. Once the I/O operation completes, the program is notified and can resume processing the result.

Python's `async` and `await` keywords, introduced in Python 3.5, provide a clean and readable way to write asynchronous code. They are built on top of the event loop mechanism.

**Fundamental Concepts:**

1. **`async`:**
    
    - The `async` keyword is used to define a coroutine function. A coroutine is a function that can be paused and resumed.
    - When you call an `async` function, it doesn't execute immediately. Instead, it returns a _coroutine object_.
    - Think of a coroutine object as a promise of work to be done in the future.
    
    ``` python
    import asyncio
    
    async def my_coroutine():
        print("Coroutine started")
        await asyncio.sleep(1)  # Simulate an I/O-bound operation
        print("Coroutine finished")
        return "Result from coroutine"
    
    # Calling the async function returns a coroutine object
    coro = my_coroutine()
    print(f"Type of coro: {type(coro)}")
    
    # To actually run the coroutine, you need to use an event loop
    async def main():
        result = await coro
        print(f"Result: {result}")
    
    asyncio.run(main())
    ```

#### **What is a Coroutine?**

In Python asynchronous programming, a coroutine is a special type of function that can suspend and resume its execution at specific points, allowing other coroutines to run in between. Coroutines are defined using the `async def` syntax.

**How is it different from a normal Python function?**

Here are the key differences:

- **Definition**: A coroutine is defined using `async def`, whereas a normal function is defined using `def`.
- **Execution**: A coroutine doesn't execute immediately when called. Instead, it returns a coroutine object, which can be scheduled to run on an event loop. A normal function executes immediately when called.
- **Suspension and Resumption**: A coroutine can suspend its execution at specific points using the `await` keyword, allowing other coroutines to run. A normal function executes from start to finish without suspension.    
- **Return Value**: A coroutine returns a coroutine object when called, whereas a normal function returns its result directly.
- **Async/Await Syntax**: Coroutines use the `async` and `await` keywords to define asynchronous operations and suspend execution.

2.  **`await`:**
    
    - The `await` keyword can only be used inside an `async` function.
    - It is used to pause the execution of the current coroutine until an _awaitable_ object completes.
    - Common awaitable objects include other coroutines, tasks (which wrap coroutines), and futures.
    - When `await` is encountered, the coroutine yields control back to the event loop, allowing other tasks to run. Once the awaitable completes, the coroutine resumes from where it left off.
    
    In the example above, `await asyncio.sleep(1)` pauses `my_coroutine` for 1 second. During this pause, the event loop can do other things (if there were any).
    
3. **Event Loop:**
    
    - The event loop is the heart of asynchronous programming. It monitors awaitable objects and runs the coroutines that are ready to make progress.
    - When a coroutine encounters an `await`, it tells the event loop, "I'm waiting for this to finish. You can do other things in the meantime."
    - When the awaited operation completes, the event loop schedules the coroutine to resume its execution.
    - `asyncio.run(main())` in the example above creates and runs the event loop, and then runs the `main` coroutine on it.
    
4. **Tasks:**
    
    - A `Task` is used to schedule a coroutine to run concurrently within the event loop.
    - `asyncio.create_task()` wraps a coroutine in a `Task` object.
    - You can `await` a `Task` to wait for its completion and get its result.
        
    ```python
    import asyncio
    
    async def fetch_data(url):
        print(f"Fetching data from {url}...")
        await asyncio.sleep(2)  # Simulate network request
        return f"Data from {url}"
    
    async def main():
        task1 = asyncio.create_task(fetch_data("http://example.com/data1"))
        task2 = asyncio.create_task(fetch_data("http://example.com/data2"))
    
        results = await asyncio.gather(task1, task2)
        print(f"Results: {results}")
    
    asyncio.run(main())
    ```
    
    In this example, `fetch_data` for two different URLs runs concurrently because they are wrapped in tasks and awaited together using `asyncio.gather()`.
    

**Why Asynchronous Programming?**

Asynchronous programming is particularly useful for I/O-bound operations, such as:

- Network requests (fetching data from APIs, web scraping).
- File I/O.
- Database interactions.

In these scenarios, the CPU often spends a lot of time waiting for the I/O operation to complete. Asynchronous programming allows the CPU to do other work during these waiting periods, leading to more efficient use of resources and potentially faster overall execution.

**Contrast with Synchronous Programming:**

In synchronous programming, if you have two I/O-bound operations, the program will wait for the first one to finish completely before starting the second.

```python
import time

def fetch_data_sync(url, delay):
    print(f"Fetching data from {url}...")
    time.sleep(delay)
    return f"Data from {url}"

def main_sync():
    start = time.time()
    result1 = fetch_data_sync("http://example.com/data1", 2)
    result2 = fetch_data_sync("http://example.com/data2", 2)
    end = time.time()
    print(f"Result 1: {result1}")
    print(f"Result 2: {result2}")
    print(f"Total time: {end - start:.2f} seconds")

main_sync()
```

Output:

```
Fetching data from http://example.com/data1...
Fetching data from http://example.com/data2...
Result 1: Data from http://example.com/data1
Result 2: Data from http://example.com/data2
Total time: 4.01 seconds
```

Now compare this with the asynchronous example:

Output (order might vary slightly due to scheduling):

```
Fetching data from http://example.com/data1...
Fetching data from http://example.com/data2...
Results: ['Data from http://example.com/data1', 'Data from http://example.com/data2']
Total time: around 2 seconds
```

You can see that the asynchronous version completes much faster because the two "fetching" operations happen concurrently.

In summary, `async` and `await` in Python provide a powerful and elegant way to write concurrent code for I/O-bound tasks, making your applications more responsive and efficient. The key is that `await` allows a coroutine to pause and yield control, enabling the event loop to manage multiple operations simultaneously.

#### **Asynchronous Tasks and Threads**

This is a common point of confusion. **By default, `asyncio` tasks run concurrently on a single thread, not on separate threads.**

- **Single-Threaded Concurrency**: `asyncio` uses an **event loop** running in a single thread. This event loop manages multiple tasks by switching between them whenever a task `await`s an operation (typically I/O-bound).
- **Cooperative Multitasking**: The tasks "cooperate" by explicitly yielding control via `await`. When a task `await`s, it's saying, "I'm going to be busy waiting for this I/O operation, so, Mr. Event Loop, you can run something else in the meantime."
- **Concurrency vs. Parallelism**:
    - `asyncio` provides **concurrency**: multiple tasks can make progress in overlapping time periods.
    - It does _not_ provide **parallelism** (simultaneous execution on multiple CPU cores) by itself for Python code. Python's Global Interpreter Lock (GIL) generally restricts true parallelism for threads running Python bytecode.
- **When to use threads/processes with `asyncio`**: If you have CPU-bound code (e.g., complex calculations) that would block the single-threaded event loop for too long, you should run that code in a separate thread pool (using `loop.run_in_executor()`) or a separate process pool. This offloads the blocking work, allowing the event loop to remain responsive for other I/O-bound tasks.
#### **Async context managers**

```python
# Create server parameters for stdio connection
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent

server_params = StdioServerParameters(
    command="python",
    # Make sure to update to the full absolute path to your math_server.py file
    args=["/home/bk_anupam/code/LLM_agents/LANGGRAPH_PG/demo_mcp_server.py"],
)

async def main():
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize the connection
            await session.initialize()

            # Get tools
            tools = await load_mcp_tools(session)

            # Create and run the agent
            agent = create_react_agent("openai:gpt-4.1", tools)
            agent_response = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})

asyncio.run(main())
```

1. **`async with stdio_client(server_params) as (read, write):`**
    
    - The `async with` statement is the asynchronous version of the regular `with` statement. It's used for **asynchronous context managers**.
    - An asynchronous context manager is an object that defines two special methods: `async def __aenter__(self)` and `async def __aexit__(self, exc_type, exc_val, exc_tb)`.
    - The `async` in `async with` signals that the `__aenter__` method (when entering the block) and the `__aexit__` method (when exiting the block) are themselves coroutines and must be `await`ed by the event loop.
    - So, `stdio_client(...)` returns an asynchronous context manager. When the `async with` block is entered, its `__aenter__` coroutine is run (and awaited). When the block is exited, its `__aexit__` coroutine is run (and awaited).
    
2. **`async with ClientSession(read, write) as session:`**
    
    - This is identical in principle to the `async with stdio_client(...)`. `ClientSession(...)` also returns an asynchronous context manager, and its `__aenter__` and `__aexit__` methods are coroutines that will be awaited.
    

