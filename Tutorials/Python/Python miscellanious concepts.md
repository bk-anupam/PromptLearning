
# Static duck typing using Protocols

Static duck typing in Python combines the flexibility of duck typing—where objects are defined by their behavior (methods/attributes) rather than class hierarchy—with static type checking at development time.
## Core Concept

Traditional duck typing is dynamic and runtime-focused: if an object has the needed methods (e.g., `quack()`), it "walks like a duck." Static duck typing, introduced via PEP 544, uses **Protocols** from the `typing` module to enable structural subtyping. Type checkers like mypy verify compatibility based on structure, not inheritance.viktor-bubanja.github+1
## Protocol Example

```python
from typing import Protocol

class Quackable(Protocol):
    def quack(self) -> str: ...

class Duck:
    def quack(self) -> str:
        return "Quack!"

class Human:
    def quack(self) -> str:  # No inheritance needed
        return "Quack?"

def make_quack(thing: Quackable) -> None:
    print(thing.quack())

make_quack(Duck())   # Works
make_quack(Human())  # Works, mypy approves

```
Here, `Human` matches `Quackable` structurally despite no `@runtime_checkable` or ABC inheritance.[[viktor-bubanja.github](https://viktor-bubanja.github.io/static-duck-typing-in-python/)]​

## Key Benefits

- **Static safety**: Catches missing methods before runtime.    
- **Flexibility**: No need for explicit `isinstance()` checks or base classes.    
- **Runtime**: Still dynamic—protocols don't enforce behavior at execution.[[xebia](https://xebia.com/blog/protocols-in-python-why-you-need-them/)]​
    

Use `@runtime_checkable` from `typing` for optional `isinstance()` support. Common in modern Python (3.8+) for APIs expecting iterable/file-like objects without rigid inheritance.[[peps.python](https://peps.python.org/pep-0544/)]​


