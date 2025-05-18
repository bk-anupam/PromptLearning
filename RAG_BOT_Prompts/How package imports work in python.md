### 1. When you `import module`, Python searches:

In **this order**:
1. **Current working directory** (i.e., `''`)
2. **Paths in `PYTHONPATH`** (if defined)
3. **Standard library paths** (from the Python install)
4. **Site-packages** (3rd-party installed modules)
5. `.pth` or `.egg-link` paths (if any)

This full search path is stored in: sys.path

`PYTHONPATH` is a **special environment variable** used by Python to determine **which directories to search** when **importing modules**. `PYTHONPATH` is essentially **prepended** to `sys.path` when Python starts.

**How to Set `PYTHONPATH`:**
export PYTHONPATH=/path/to/my/modules

#### What Are Standard Library Paths?
These are **built-in modules** that ship with Python — no installation required.
##### Examples:
- `os`, `sys`, `math`, `json`, `datetime`, etc.

Python finds `os` in the **standard library path**, like:
/usr/lib/python3.10/   # Linux
C:\Python310\Lib\      # Windows
These paths are **automatically included in `sys.path`** when Python starts.
#### What Are Site-Packages?

**`site-packages/` is where 3rd-party libraries live** — things you install via `pip`.
```bash
$ pip install requests
```

It lands in:
```bash
/usr/lib/python3.10/site-packages/requests/   # Linux
C:\Python310\Lib\site-packages\requests\      # Windows
```

So
```python
import requests
```
Works because the interpreter finds it in `site-packages`, which is also in `sys.path`.

### Example of how package import works in a project:
Assume a project structure like this:

The parent directory of LLM_agents is: code
```
LLM_agents/
├── __init__.py
├── main.py
├── RAG_BOT/
│   ├── __init__.py
│   ├── bot.py
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── agent_node.py
```

LLM_agents is the project root directory. The presence of `__init__.py` file in LLM_agents allows python to treat LLM_agents as a package. 

#### ~code> python -m llm_agents.main command:

When you use `-m`, Python treats the argument as a module name and runs it as if you had imported it. Here's a breakdown of how `python -m llm_agents.main` works when executed from parent directory of llm_agents:

- **Finding the package**: Python treats `llm_agents` as a package, and the parent directory of `project` is added to `sys.path`. Python looks for a package or module named `llm_agents` in the current working directory and in the directories listed in `sys.path`.

- **Importing the package**: Once Python finds the `llm_agents` package, it imports it as a module. This means that the `__init__.py` file in the `llm_agents` directory is executed, and the package's namespace is initialized.
    
- **Finding the module**: After importing the `project` package, Python looks for a module named `main` within the package. This module is expected to be a Python file named `main.py` in the `llm_agents` directory.

- **Running the module**: Finally, Python runs the `main.py` module as if it had been imported using `import llm_agents.main`. The `if __name__ == "__main__":` block in `main.py` will be executed, and the script will run as expected.

For importing other modules within the same package (or subpackages) you should use relative imports or absolute imports from the top-level package. 

**Relative imports:**

Now in main.py you can do relative imports like this:
```python main.py
# main.py
from .rag_bot.agent import agent_node
```
This will correctly resolve to llm_agents.rag_bot.agent.agent_node

And in bot.py similarly you can have a relative import like this
``` python
#bot.py
from .agent import agent_node
```

**Absolute imports:**

When you execute a Python script, the directory containing the script is added to `sys.path`. Absolute imports in Python, like `from rag_bot.agent import agent_node`, resolve paths starting from a directory in Python's `sys.path`. Thus if the directory llm_agents is in sys.path then absolute imports starting from project root directory would work.
```python main.py
# main.py
from RAG_BOT.bot import something
from RAG_BOT.agent.agent_node import AgentNode
```

This works _if you run from the project root_:
```bash
$ cd LLM_agents/
$ python main.py
```
Why?
Because `.` (current dir = `LLM_agents/`) is added to `sys.path`, so:
- Python sees `RAG_BOT/` as a top-level package
- It can resolve `RAG_BOT.bot` and deeper imports

#### ✅ Clean Setup for Package Imports

##### 🛠️ Option 1: Use editable install (`pip install -e .`)

Create a `setup.py` or `pyproject.toml`:

**setup.py**

```python
from setuptools import setup, find_packages

setup(
    name='LLM_agents',
    version='0.1',
    packages=find_packages(),
)
```

Then:

```bash
pip install -e .
```

Now Python knows `LLM_agents` is a package, and you can import it from anywhere.

---

##### 🔁 Imports Now Work Like This:

##### In `main.py`:

```python
from RAG_BOT.bot import start_bot
```

##### In `bot.py`:

```python
from RAG_BOT.agent.agent_node import AgentNode
```

No need for ugly `sys.path.append` hacks.

---

#### ⚠️ Site-Packages & `-e .`

When you install a package with `-e .`, Python places an **`.egg-link`** in `site-packages` that **points to your source directory**. So your code behaves like a proper installed package **without copying files**.

This is why `editable installs` are the best solution for **multi-file, multi-module projects**.

---

#### ✅ Summary

| Term                 | Purpose                                                          |
| -------------------- | ---------------------------------------------------------------- |
| **Standard Library** | Core modules that ship with Python (e.g. `os`, `sys`)            |
| **Site-Packages**    | 3rd-party installed modules (via pip)                            |
| **PYTHONPATH**       | Env var to add custom paths to Python’s search list              |
| **`sys.path`**       | List of all places Python looks for modules                      |
| **`-e .` install**   | Makes your project behave like a package, supports clean imports |

---
