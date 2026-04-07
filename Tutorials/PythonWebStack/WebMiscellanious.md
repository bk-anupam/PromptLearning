

# Multipart/form-data in api requests:

This is a fundamental concept in web development. To understand `multipart/form-data`, it helps to first look at how "normal" API requests work and why they fail when handling files.

Here is a detailed breakdown.

### 1. The "Normal" API Request: JSON

In modern web development, 90% of API requests use **JSON** (`application/json`).

- **Scenario:** You want to update a user's profile (name and email).
- **Encoding:** The client tells the server: "I am sending you text formatted as a Javascript Object."
- **Header:** `Content-Type: application/json`
- **Body:** The body is a **single, continuous text string**.

**What the raw HTTP request looks like:**

```http
POST /api/update-profile HTTP/1.1
Content-Type: application/json

{
    "user_id": "123",
    "email": "test@example.com"
}
```

**Why this is bad for files:** JSON is a **text** format. An image or PDF is **binary** data (raw bytes like `010110...`). To put an image inside a JSON string, you have to convert those bytes into text characters (usually using **Base64 encoding**).

- **Bloat:** Base64 increases the file size by ~33%. A 10MB image becomes 13.3MB of text.
- **CPU Cost:** The server has to burn CPU to decode that massive text string back into binary before it can save the file.

### 2. The Solution: `multipart/form-data`

This encoding was invented specifically to solve the problem of sending **mixed data** (simple text fields + massive binary files) in a single request without the overhead of Base64.

- **Scenario:** You are uploading a medical report (PDF) along with the user's ID.
- **Encoding:** The client tells the server: "I am sending multiple different pieces of data, separated by a specific boundary."
- **Header:** `Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW`

**What the raw HTTP request looks like:** Instead of one big JSON object, the body is split into "parts" using the boundary string.

```http
POST /api/ingestion/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="user_id"

user_123
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="files"; filename="report.pdf"
Content-Type: application/pdf

[... RAW BINARY DATA OF PDF GOES HERE ...]
------WebKitFormBoundary7MA4YWxkTrZu0gW--

```

**Why this is efficient:**

1. **No Encoding:** The PDF binary data is dumped directly into the request body. No conversion to text is needed.
2. **Streaming:** The server can read the `user_id` part, then start streaming the `files` part directly to disk without loading the whole thing into RAM.

#### a. The Boundary (`WebKitFormBoundary...`)

In a standard JSON request, the server knows the request is over when the JSON object closes with `}`. But in a file upload, you are sending multiple distinct things (e.g., a User ID string, a PDF file, and a JPEG image) all in one long stream.

The **Boundary** is a unique string that acts as a **delimiter** or separator between these parts.

- **Why is it needed?** The server reads the stream byte-by-byte. It needs a way to know: _"Okay, I'm done reading the User ID, now I'm starting to read the PDF."_
- **Why does it look so weird?** (`----WebKitFormBoundary7MA4YWxkTrZu0gW`) The boundary string **must be unique**. It cannot appear anywhere inside the actual data you are sending.
    - If you used a simple word like `END` as your boundary, and your PDF file happened to contain the word "END" inside its binary data, the server would cut the file in half and crash.
    - Browsers generate long, random strings to statistically guarantee that this sequence of characters will never accidentally appear inside your image or PDF data.
- **How it works:**
    1. **Header:** The browser tells the server in the main header: _"I am using `----WebKitFormBoundaryXYZ` as the separator."_
    2. **Body:** The browser puts that string before every new field.
    3. **End:** The browser puts the string with two extra dashes at the end (`----WebKitFormBoundaryXYZ--`) to signal the absolute end of the request.

#### b. Content-Disposition

Once the server finds a boundary and knows a new "part" has started, it needs to know what that part represents. This is the job of the `Content-Disposition` header.

It is a sub-header that appears at the top of _each_ part in the multipart body.

- **Syntax:** `Content-Disposition: form-data; name="field_name"; filename="file.txt"`
    
- **Why is it needed?** It maps the raw data to the specific variables in your code.
    
    - **`form-data`**: This is the standard type for multipart forms.
    - **`name="user_id"`**: This tells FastAPI: _"Take the data following this header and put it into the function argument named `user_id`."_
    - **`filename="report.pdf"`**: This is metadata. It tells the server the original name of the file on the user's computer. FastAPI uses this to populate the `.filename` attribute of the `UploadFile` object.

```python
async def upload_ingestion(
    user_id: str = Form(...),          # <--- Looks for part with name="user_id"
    files: List[UploadFile] = File(...) # <--- Looks for part with name="files"
):

```

When the request hits the server:

1. **Server:** _Reads boundary._ "New part starting."
2. **Server:** _Reads Content-Disposition._ "Okay, `name='user_id'`. I will save the next chunk of text into the `user_id` variable."
3. **Server:** _Reads boundary._ "Next part starting."
4. **Server:** _Reads Content-Disposition._ "Okay, `name='files'`. This is binary data. I will stream this into a temporary file and give it to the `files` variable."

### 3. What are "Form Fields"?

The term comes from HTML Forms (`<form>`).

- **In HTML:**
    
    ```html
    <form action="/upload" method="post" enctype="multipart/form-data">
	  <!-- This is a form field -->
	  <input type="text" name="user_id"> 
	
	  <!-- This is a file field -->
	  <input type="file" name="files">
	</form>

    ```
- **In an API Request:** A "form field" is simply a key-value pair sent in the body when the content type is `multipart/form-data` (or `application/x-www-form-urlencoded`).
    - In the raw HTTP example above, `user_id` is a **Form Field**.
    - `files` is a **File Field** (a specialized form field containing binary data).

### 4. Do all requests contain form fields?

**No.**

1. **GET Requests:** Usually have **no body** at all. They send data via the URL (Query Parameters), e.g., `?id=123`.
2. **JSON POST Requests:** Have a body, but it's a single JSON object. Conceptually, the keys inside the JSON act like fields, but technically they are just part of a text string, not distinct HTTP "parts."

### Summary: When to use what?

|Feature|JSON (`application/json`)|Multipart (`multipart/form-data`)|
|:--|:--|:--|
|**Primary Use**|Sending structured data (objects, arrays, numbers).|Uploading files (images, videos, docs).|
|**Data Format**|Text only.|Mixed (Text + Binary).|
|**Efficiency**|High for small data. Low for large files.|High for large files.|
|**Structure**|Nested (objects inside objects).|Flat (Key-Value pairs).|
|**In FastAPI**|Used with Pydantic models (`BaseModel`).|Used with `Form(...)` and `File(...)`.|

In your `(/home/bk_anupam/code/agents/swasthya_sahayak/src/api/main.py)`, you use `multipart/form-data` because the user is uploading medical records (PDFs/Images). If you tried to do this with JSON, your API would be slow and consume massive amounts of memory.

```python

@app.post("/api/ingestion/upload")
async def upload_ingestion(    
    user_id: str = Form(...),
    processing_mode: str = Form("auto"),
    auto_analyze: bool = Form(True),
    files: List[UploadFile] = File(...),    
    ingestion_service: IngestionService = Depends(get_ingestion_service)
):
    """
    Endpoint to upload files and start the ingestion process.
    Wraps IngestionService.start_ingestion_async.
    """
    try:                
        # Explicitly adapt UploadFile to the interface expected by IngestionService
        adapted_files = [FileAdapter(f) for f in files]            
        submission_id = await ingestion_service.start_ingestion_async(
            user_id=user_id,
            uploaded_files=adapted_files,
            processing_mode=processing_mode,
            auto_analyze=auto_analyze
        )        
        return {"submission_id": submission_id}
    except Exception as e:
        logger.error(f"Ingestion upload failed: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))
```

### `Form(...)`

- **Purpose:** Tells FastAPI to read the function argument from the **form fields** (text fields) of the request, rather than from JSON or query parameters.
- **In your code:**    
    `user_id: str = Form(...)`    
    This means: _"Look for a form field named `user_id` in the request body."_
- **Without it:** If you just wrote `user_id: str`, FastAPI would assume it is a **Query Parameter** (e.g., `?user_id=123`) because it's a simple type, which is not what you want for a POST upload.

### 3. `File(...)`

- **Purpose:** Tells FastAPI to read the function argument as a **file upload** stream.
- **In your code:**    
    `files: List[UploadFile] = File(...)`    
    This means: _"Look for a file part named `files` in the request body."_
- **Behavior:** It ensures the data is treated as bytes/binary. If you used `Form` here, FastAPI would try to read the file content as a text string, which would likely fail or corrupt the binary data.

### 4. What does the `...` (Ellipsis) mean?

The `...` is a special Python singleton called **Ellipsis**. In FastAPI (and Pydantic), using `...` as a default value means **"This field is Required"**.

- `user_id: str = Form(...)`: **Required**. If the client sends a request without `user_id`, FastAPI returns a `422 Validation Error`.
- `processing_mode: str = Form("auto")`: **Optional**. If the client doesn't send it, it defaults to `"auto"`.

---


# Server sent events in FastAPI


### Server-Sent Events (SSE)?

**Server-Sent Events (SSE)** is a browser-native streaming protocol over HTTP.
It allows:

* Client opens connection
* Server keeps it open
* Server continuously pushes events
* Client receives them in real time

It’s **one-way streaming**:

```
Server → Client
```

Not bidirectional like WebSockets.

#### What SSE Looks Like on the Wire

Raw SSE format:

```
event: message
data: {"text":"hello"}

event: message
data: {"text":"world"}

event: done
data: [DONE]
```

Each event has:

* `event:` → event name
* `data:` → payload
* blank line → end of event

---
### 2. What is an SSE Endpoint in the Context of a Chat Agent?

An SSE endpoint is simply:

> An HTTP endpoint that returns a streaming response with `Content-Type: text/event-stream`

In your case:

```python
return EventSourceResponse(event_generator())
```

This tells FastAPI:

* Don’t send JSON
* Send streaming events
* Keep connection alive

This is ideal for:

* LLM streaming
* Agent tool updates
* Long-running workflows
* Token-by-token responses

Exactly what you need in a chat agent.

---
### 3. What is `EventSourceResponse`?

`EventSourceResponse` comes from:

```python
from sse_starlette.sse import EventSourceResponse
```

It is a special FastAPI response class that:

* Accepts an async generator
* Converts yielded dicts into proper SSE format
* Handles headers:
  ```
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive
  ```
* Flushes events immediately

So when you do:
```python
EventSourceResponse(event_generator())
```
You are saying:

> "Take whatever this async generator yields and stream it as SSE events."

---
### Why Do We Need `event_generator()`?

This is the core of the design.

```python
async def event_generator():
```

This is an **async generator function**.

Key concept:

A normal async function:

```python
async def f():
    return result
```

An async generator:

```python
async def f():
    yield something
```

Generators:

* Don’t return everything at once
* Produce values over time
* Keep state between yields

Perfect for streaming.

---

### Why Not Just Return `events` Directly?

Because:

```python
events = await chat_service.process_user_message(...)
```

returns a full list.

But SSE requires:

* Yielding one event at a time
* Maintaining connection
* Flushing each message

So we wrap the list inside a generator and yield them individually:

```python
for event in events:
    yield {
        "event": "message" if event.get("role") == "assistant" else "specialist",
        "data": json.dumps(event)
    }
```

That converts your internal event format into proper SSE format.

---
### Execution Flow (Step-by-Step)

Here’s exactly what happens:

#### Step 1: Client calls endpoint

```
POST /api/chat/stream
```

#### Step 2: FastAPI executes `chat_stream`

It returns:

```python
EventSourceResponse(event_generator())
```

But `event_generator()` does NOT execute fully yet.
It returns an async generator object.

#### Step 3: EventSourceResponse starts consuming the generator

Internally it does something like:

```python
async for chunk in event_generator():
    send(chunk)
```

#### Step 4: Your generator runs

Inside:

```python
events = await chat_service.process_user_message(...)
```

It waits for your agent to finish.

Then:

```python
for event in events:
    yield {...}
```

Each `yield`:

* Is converted to SSE format
* Immediately flushed to client

#### Step 5: End of stream

```python
yield {"event": "done", "data": "[DONE]"}
```

Client receives:

```
event: done
data: [DONE]
```

Connection closes.

---

### Why Use This in a Chat Agent?

In agent systems (like the RAG + multi-step agent systems you're building), streaming allows:

Instead of:

```
User waits 8 seconds → gets full response
```

You can do:

```
Step 1: "Retrieving documents..."
Step 2: "Calling cardiology specialist..."
Step 3: "Generating final answer..."
Step 4: Answer tokens streaming...
```

Much better UX.

---

### 🧠 Conceptual Architecture

Here’s the mental model:

```
Client (EventSource)
        ↓
FastAPI endpoint
        ↓
EventSourceResponse
        ↓
Async generator
        ↓
ChatService (Agent)
        ↓
LLM + Tools + RAG
```

Streaming layer sits between agent and browser.

---

### Final Summary (Straight Talk)

* **SSE** = HTTP-based one-way streaming
* **SSE endpoint** = API returning `text/event-stream`
* **EventSourceResponse** = FastAPI wrapper for streaming async generators
* **event_generator()** = produces events gradually
* We use a generator because streaming requires incremental yielding
* It's defined inside for closure access
* This is the correct architectural pattern for LLM-based agents