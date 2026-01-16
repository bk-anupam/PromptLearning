
---

# 🔍 `urllib.parse.urlparse()` and `urllib.parse.unquote()` in Python

Both come from Python’s built-in **`urllib.parse`** module, which helps you work with URLs safely and correctly.

---

## 1️⃣ `urllib.parse.urlparse()`

📌 **What it does:**  
Takes a URL string → splits it into its components.

Example:

```python
from urllib.parse import urlparse

url = "https://example.com:8080/path/to/page?query=123#section1"
parsed = urlparse(url)

print(parsed)
```

Output (a named tuple):

```
ParseResult(
  scheme='https',
  netloc='example.com:8080',
  path='/path/to/page',
  params='',
  query='query=123',
  fragment='section1'
)
```

You can access fields directly:

```python
print(parsed.scheme)   # https
print(parsed.path)      # /path/to/page
print(parsed.query)     # query=123
```

📌 **Why/When used:**

- Extracting pieces of a URL safely (scheme, domain, route, etc.)
    
- Routing logic in APIs and web crawlers
    
- Parsing query parameters without string hacks
    
- Validating or transforming URLs
    

💡 Much safer than manually splitting strings by `/` and `?`.

---

## 2️⃣ `urllib.parse.unquote()`

📌 **What it does:**  
Decodes **URL-encoded** characters back into normal text.

URLs use **percent-encoding**, e.g.,

- `" "` → `%20`
    
- `":"` → `%3A`
    
- Unicode encoded using UTF-8 → `%E2%9C%85` (✔)
    

Example:

```python
from urllib.parse import unquote

encoded = "Hello%20World%21"
decoded = unquote(encoded)

print(decoded)  # Hello World!
```

📌 **Why/When used:**

- Handling URLs received from browsers or APIs
    
- Cleaning up encoded query parameters
    
- Decoding filenames in download links
    
- Extracting readable text from encoded paths
    

---

## 🧠 They’re often used _together_

Say you receive a URL like:

```python
url = "https://example.com/search?q=Deep%20Learning%20Tutorial"
```

You want the actual query text:

```python
from urllib.parse import urlparse, unquote, parse_qs

parsed = urlparse(url)
query_params = parse_qs(parsed.query)

search_term = unquote(query_params["q"][0])
print(search_term)  # Deep Learning Tutorial
```

---

## 🔑 Key Takeaways

|Function|Purpose|Typical Use Case|
|---|---|---|
|`urlparse()`|Split URL into components|Inspect routing, domain, query values|
|`unquote()`|Decode percent-encoded text|Make URLs human-readable or process strings|

---

## Quick Rule of Thumb

|You want to…|Use…|
|---|---|
|Break a URL into pieces|`urlparse()`|
|Turn `%XX` encoded characters back into real text|`unquote()`|
|Understand/parse query parameters|`parse_qs()` (related helper)|

---

