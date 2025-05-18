1. **Create a `setup.py` file in project root LLM_agents**
```python
from setuptools import setup, find_packages

setup(
    name="RAG_BOT",
    version="0.1.0",
    description="A RAG-based Telegram Bot for answering questions",
    author="Your Name",
    packages=find_packages(),
    python_requires=">=3.8",
    install_requires=[
        "langchain-google-genai",
        "langchain-chroma",
        "pyTelegramBotAPI",
        "Flask",
        "sentence-transformers",
        "chromadb",
        "langgraph",
        "pydantic",
        # Add all direct dependencies your code imports
    ],
    extras_require={
        'dev': [
            'pytest',
            'pytest-cov',
            'black',
            'flake8',
        ],
    }
)
```

2.  **Install the package in development mode:**
```shell
cd /home/bk_anupam/code/LLM_agents
pip install -e .
```
When you install a package with `-e .`, Python places an **`.egg-link`** in `site-packages` that **points to your source directory**. So your code behaves like a proper installed package **without copying files**. This is why `editable installs` are the best solution for **multi-file, multi-module projects**.

3. Make sure each directory has an [__init__.py] to mark it as a Python package.
4. **Update your imports:** Instead of manipulating [sys.path], use absolute imports:
```python
from RAG_BOT.agent.graph_builder import build_agent
from RAG_BOT.utils import detect_document_language
from RAG_BOT.config import Config
```

This approach:
- Makes your package properly importable
- Follows Python packaging best practices
- Avoids runtime manipulation of sys.path
- Makes dependencies explicit through `setup.py`
- Makes testing easier
- Makes deployment more reliable

The package can then be installed in any environment, and imports will work consistently without path manipulation.

### Purpose of setup.py:
- It's the build script for `setuptools`
- Defines metadata about your project (name, version, etc.)
- Specifies dependencies
- Helps make your project installable and importable

 **`install_requires`**:
- List all direct dependencies your code imports
- Should match the imports in your code
- Can specify version requirements (e.g., `"Flask>=2.0.0"`)
- Don't include development dependencies (put those in `extras_require['dev']`)

 **Development Mode Installation**: Yes, you need to install the package even for development. The `-e` flag makes it "editable":
```shell
cd /home/bk_anupam/code/LLM_agents
pip install -e .  # Install in editable mode
pip install -e ".[dev]"  # Also install development dependencies
```

This:
- Creates a special link in your Python environment to your project
- Allows Python to find your package
- Makes imports work correctly
- Reflects code changes immediately without reinstalling

#### **How `pip install -e .` Works**

When you run `pip install -e .` in your project directory:

1. **Editable Install**
    - The `-e` flag creates an "editable" or "development" install
    - Instead of copying files to your Python's site-packages, it creates a special `.egg-link` file in site-packages.
    - This file points to your project directory
    - Changes to your code are immediately reflected without reinstalling
2. **Dependency Handling**
    - `pip` reads your setup.py
    - Checks install_requires list against currently installed packages
    - Only installs dependencies that aren't already present in your environment
    - Won't reinstall or upgrade existing packages unless forced
#### In Your Case
Since you're using an existing virtual environment (`ml_env`) that already has the dependencies:

1. Running `pip install -e .` will:
    - Make your [RAG_BOT](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) package importable from anywhere
    - **Not** reinstall existing dependencies
    - Only install any dependencies listed in [setup.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) that might be missing

#### RAG BOT Project Structure:
```
LLM_agents/
├── setup.py
├── README.md
├── requirements.txt
├── RAG_BOT/
│   ├── __init__.py
│   ├── bot.py
│   ├── utils.py
│   ├── config.py
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── agent_node.py
│   │   ├── graph_builder.py
│   │   └── retrieval_nodes.py
│   └── tests/
│       ├── __init__.py
│       ├── unit/
│       │   └── test_bot.py
│       └── integration/
│           └── test_integration.py

```

### How Absolute Imports Work

1. **Package Root Determination**
- The presence of [__init__.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) files marks directories as Python packages
- Your project's root package is determined by the [packages](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) parameter in [setup.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html):

```python
packages=find_packages(include=['RAG_BOT', 'RAG_BOT.*'])
```
- This tells Python that [RAG_BOT](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) is your root package and any subpackages (like [RAG_BOT.agent](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)) should be included

2. **Import Path Resolution**
- After `pip install -e .`, Python can find your package from anywhere
- The absolute import path starts from your root package name ([RAG_BOT](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html))
- Submodules follow the directory structure, separated by dots
- Example:

```
LLM_agents/
└── RAG_BOT/              # Root package (has __init__.py)
    ├── agent/            # Subpackage (has __init__.py)
    │   ├── graph_builder.py
    │   └── state.py
    └── utils.py
```

- To import [graph_builder.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html): [from RAG_BOT.agent.graph_builder import build_agent](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)
- To import [utils.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html): [from RAG_BOT.utils import detect_document_language]

### Project Structure Best Practices

1. **Location of [setup.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) and [requirements.txt](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html)**

- These files should stay in the project root (`LLM_agents/`)
- Moving them to [RAG_BOT](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) would be incorrect because:
    - [setup.py](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) is meant to be at the top level of your project
    - It defines how to install your entire project, not just one package
    - [requirements.txt](vscode-file://vscode-app/d:/InstalledSoftware/Microsoft%20VS%20Code/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html) typically lives alongside [setup.py]

2. **Recommended Project Structure**

```
LLM_agents/                 # Project root
├── setup.py               # Package installation config
├── requirements.txt       # Development dependencies
├── README.md             # Project documentation
└── RAG_BOT/              # Main package
    ├── __init__.py       # Makes it a package
    ├── agent/            # Subpackage
    │   ├── __init__.py
    │   └── ...
    └── ...
```
