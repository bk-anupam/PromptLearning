
# JavaScript Stage 1

## Chapter 1: How JavaScript Actually Runs

### The Big Picture First

Before a single line of syntax, you need to know _where_ JavaScript lives and _how_ it executes. This mental model will save you from hours of confusion later.

**JavaScript is single-threaded.** This means it has one call stack, executes one thing at a time, in order. No parallel execution (ignoring Web Workers for now).

Think of it like a chef who can only do one task at a time — but has assistants (the browser/Node runtime) who handle slow work like timers and network calls offscreen.

---
### The Runtime Environment

JavaScript doesn't run in isolation. It runs inside a **runtime** — either the **browser** (Chrome, Firefox, Safari) or **Node.js** on a server.

The runtime provides:

- The **JS Engine** (e.g., V8 in Chrome/Node) — actually executes your code
- **Web APIs** (in browser) or **Node APIs** — things like `setTimeout`, `fetch`, `fs.readFile`
- The **Event Loop** — the coordination mechanism (you'll study this deeply in Stage 5)

```
┌─────────────────────────────────────────┐
│           JavaScript Runtime            │
│                                         │
│  ┌──────────┐    ┌────────────────────┐ │
│  │ JS Engine│    │  Runtime APIs      │ │
│  │          │    │  (setTimeout,      │ │
│  │ Call     │    │   fetch, DOM, etc) │ │
│  │ Stack    │    │                    │ │
│  └──────────┘    └────────────────────┘ │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │          Event Loop              │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

### The JS Engine: The Pure Language Processor

The **JavaScript engine** is the component responsible for one thing only — taking JavaScript source code and executing it. It knows nothing about the web, nothing about files on disk, nothing about network requests. It just understands the JavaScript language specification.

The most famous JS engine is **V8**, built by Google. It powers both Chrome and Node.js. Firefox has **SpiderMonkey**, Safari has **JavaScriptCore**. These are all different implementations of the same ECMAScript specification, much like how different JVM vendors (Oracle, IBM, Amazon Corretto) all implement the same Java spec.

What does the engine actually do? At a high level:

```
Source Code ("let x = 1 + 2")
        ↓
   [Parser] — reads text, checks syntax, builds AST (Abstract Syntax Tree)
        ↓
   [Interpreter] — walks the AST and executes it quickly
        ↓
   [Profiler] — watches for "hot" code (frequently run loops/functions)
        ↓
   [JIT Compiler] — compiles hot code to optimized machine code
        ↓
   Machine Code running on your CPU
```

The engine is essentially a self-contained black box: JavaScript in, execution out. The JavaScript engine does not JIT compile the _entire_ code. It relies on the Profiler to identify only the "hot" paths (like a loop running thousands of times). Compiling everything to machine code takes time and memory; doing so for code that only runs once (like initial setup functions) would actually slow down the application's startup time.

---

### The JS Runtime: The Engine Plus Its Environment

The **runtime** is the engine *plus everything surrounding it* that makes it useful in a specific context. This is where the distinction becomes really clear.

The JavaScript language specification (ECMAScript) does not define `setTimeout`. It does not define `fetch`. It does not define `console.log`. These things don't exist in pure JavaScript — they are provided by the **runtime environment**.

When JavaScript runs in a **browser**, the runtime provides:
- The **DOM API** (`document.querySelector`, event listeners)
- **Web APIs** (`setTimeout`, `fetch`, `localStorage`, `WebSockets`)
- The **event loop** and **callback/task queues**
- `window` as the global object

When JavaScript runs in **Node.js**, the runtime provides:
- **Node APIs** (`fs.readFile`, `http.createServer`, `path`, `process`)
- The **event loop** (implemented using libuv under the hood)
- `global` as the global object
- No DOM, no `window`, no browser APIs

Both runtimes use the V8 engine at their core, but they wrap it in completely different environments. This is why the same JavaScript code can read files in Node but not in a browser, and can manipulate the DOM in a browser but not in Node. The *language* is the same — the *environment* is different.

A useful analogy: the JS engine is like a calculator chip. The runtime is like the full calculator device — it includes the chip, but also the buttons, the screen, the power supply, and all the extra functionality that makes the chip useful in context.

---
### Is the JavaScript Runtime Equivalent to the JVM?

This is a smart question, and the answer is: **partially yes, but the analogy breaks down in important ways.**

**Where the analogy holds:**

Both the JVM and the JS runtime act as an abstraction layer between your code and the underlying operating system. You write JavaScript and don't worry about whether you're on Windows, macOS, or Linux — the runtime handles that. Same with the JVM. In this sense, both are "run anywhere" platforms.

Both also manage memory for you. The JVM has a garbage collector, and so does the V8 engine inside every JS runtime. You don't manually call `malloc` and `free`.

**Where the analogy breaks down:**

The JVM is a single, standardized virtual machine with a well-defined bytecode format (`.class` files). When you compile Java, you get bytecode that *any* JVM can run. The JVM is the complete execution platform.

JavaScript runtimes are not standardized in the same way. The *language* is standardized (by ECMAScript), but the *runtime APIs* are not. The browser runtime and the Node runtime both run the same JavaScript, but they expose completely different APIs. There's no single unified "JS Runtime" the way there's a single JVM spec. You can think of it this way: ECMAScript is closer to the Java language spec, V8 is closer to the JVM's execution engine, and then the browser or Node is the broader platform (like how the JRE includes the JVM plus the standard library).

Another key difference: the JVM was *designed* as a bytecode virtual machine from the start — Java compiles to an intermediate format first. JavaScript was originally designed to be interpreted directly in the browser, and the compilation story evolved later (which brings us to the next question).

#### JRE vs. JVM

- **JVM (Java Virtual Machine):** The actual engine executing the program. It contains the interpreter, the JIT compiler, and the garbage collector.
    
- **JRE (Java Runtime Environment):** The complete package needed to run a Java app on a computer. It contains the JVM, plus all the core class libraries and supporting files the JVM needs to do its job.

####  Python (Interpreted) vs. Java (Compiled)

The traditional "compiled vs. interpreted" line is a bit blurry today, but here is the core difference in their standard implementations:

|**Feature**|**Python (Interpreted)**|**Java (Compiled to Bytecode)**|
|---|---|---|
|**Preparation**|Source code is distributed as-is.|Source code is pre-compiled by the developer into bytecode.|
|**Execution Phase**|The Python engine translates source code to bytecode on the fly at runtime, then evaluates it step-by-step.|The JVM skips source code parsing and directly executes the pre-compiled bytecode.|
|**Performance Profile**|Very fast to write and test, but generally slower peak execution speed.|Requires a compilation step before testing, but achieves faster execution due to heavy JIT optimization.|

---

### Javascript code execution process:

#### 1. The Baseline: Bytecode (The "Medium" Step)

Instead of going straight from source code to machine code, the **Interpreter** (called _Ignition_ in V8) converts the AST into **Bytecode**.

- **Why Bytecode?** Machine code is extremely verbose and architecture-specific (Intel vs. ARM). Bytecode is a compact, intermediate language that is easy for the engine to execute quickly without the massive memory overhead of full machine code.    
- **Execution:** The engine starts running this bytecode immediately. This is why JS starts up so fast—it doesn't wait for a long compilation process.
#### 2. The Promotion: JIT Optimization

While the Interpreter is running the bytecode, the **Profiler** is watching.

- **Warm Code:** If a function is called a few times, it's marked as "warm." The engine might perform some basic optimizations.
- **Hot Code:** If a function is called _constantly_ (like a loop or a high-traffic event handler), it's marked as "hot."
- **The JIT Flip:** The **JIT Compiler** (called _TurboFan_ in V8) takes that specific "hot" function and its bytecode, looks at the types of data passing through it (e.g., "this function always receives integers"), and compiles it into **Highly Optimized Machine Code**.
#### 3. The "De-optimization" Safety Net

Because JavaScript is **dynamically typed**, the JIT compiler has to make "assumptions." If you have a function `add(a, b)` and you've passed integers into it 1,000 times, the JIT creates machine code that _only_ handles integers to make it lightning-fast.

**But what if you suddenly pass a string?**

The machine code doesn't know what to do. The engine performs a **De-optimization**: it throws away the optimized machine code and falls back to the slower (but more flexible) Bytecode interpreter.
#### Summary of the Flow

| **Stage**        | **Format**                 | **Speed**    | **Goal**                                            |
| ---------------- | -------------------------- | ------------ | --------------------------------------------------- |
| **Parsing**      | AST (Abstract Syntax Tree) | N/A          | Understand the structure.                           |
| **Interpreter**  | **Bytecode**               | Moderate     | Get the app running instantly.                      |
| **JIT Compiler** | **Machine Code**           | Blazing Fast | Optimize the 10% of code that does 90% of the work. |

So, to answer your question: **Most of your JS code actually runs as Bytecode.** Only the high-performance "hot" parts ever make the jump to native Machine Code.

---

### The Java Code Execution Flow

Java uses a "Two-Step" compilation process. It doesn't go from Source to Machine code in one jump; it uses an intermediate step to ensure the code can run on any device (Windows, Mac, Linux) without being rewritten.

1. **Phase 1: Development (Compile Time)**
    
    - **Source Code:** You write `.java` files.        
    - **The Compiler (`javac`):** Before you run the app, you manually compile it. This checks for type errors (e.g., trying to add a String to an Integer) and converts the code into **Bytecode** (stored in `.class` files).        
    - _Result:_ This bytecode is a set of instructions for a "theoretical" computer, not your actual CPU.
        
2. **Phase 2: Runtime (The JVM)**
    
    - **Class Loader:** When you start the app, the JVM loads the `.class` files into memory.        
    - **The Interpreter:** The JVM begins interpreting the bytecode instructions one by one. This is slower than native code but gets the program moving.        
    - **The JIT Compiler:** Just like in JS, the JVM watches for "Hot" methods. When it finds them, it compiles that bytecode into **Native Machine Code** specifically tuned for your hardware (e.g., your specific Intel or ARM chip).
        
#### ⚔️ JavaScript vs. Java: The Showdown

Even though both use a JIT compiler, the way they get there is fundamentally different.

|**Feature**|**JavaScript (V8 Engine)**|**Java (JVM)**|
|---|---|---|
|**Input Format**|Receives **Plain Text** (Source Code).|Receives **Pre-compiled Bytecode**.|
|**Starting Line**|Must parse text and build an AST at runtime.|Skips parsing; starts executing bytecode immediately.|
|**Typing**|**Dynamic:** Types are discovered _while_ running.|**Static:** Types are known _before_ running.|
|**JIT Strategy**|Must "guess" types and **De-optimize** if the guess is wrong.|Highly stable optimizations because types never change.|
|**Memory**|Generally lighter/faster for small scripts.|Heavier "warm-up" but superior for massive, complex systems.|

### 📊 Performance Comparison at a Glance

|**Feature**|**JavaScript (V8 Engine)**|**Java (JVM)**|
|---|---|---|
|**Startup Speed**|🚀 **Winner:** Extremely fast; starts running code almost instantly.|🐢 **Slower:** Requires time to load the JVM and "warm up" bytecode.|
|**Peak Throughput**|📉 **Lower:** Dynamic types limit how much the compiler can optimize.|🏆 **Winner:** Static types allow for deeper, "permanent" machine code optimizations.|
|**CPU-Intensive Tasks**|❌ **Weak:** Single-threaded; struggles with heavy math/encryption.|✅ **Strong:** Multi-threaded; utilizes all CPU cores effectively.|
|**I/O-Intensive Tasks**|✅ **Strong:** Non-blocking event loop is great for handling 10k+ simple connections.|✅ **Strong:** Modern Java (Virtual Threads) is now highly competitive here too.|
|**Memory Usage**|🍃 **Lighter:** Better for small scripts and browser tabs.|🐘 **Heavier:** JVM has a large memory footprint "floor."|

---
### Is JavaScript Compiled or Interpreted?

The honest answer is: **it's both, and the line has blurred significantly.** This question has a surprisingly nuanced history.

**Originally, it was interpreted.** In the early days of the web, a browser would receive JavaScript source code, read it line by line, and execute it directly. There was no separate compilation step. This made it simple to deploy (just ship the `.js` file) but slow to execute.

**Today, it's JIT-compiled.** Modern engines like V8 use **Just-In-Time (JIT) compilation**, which means compilation happens at runtime, right before or during execution. Here's how V8 does it specifically:

When your code first runs, V8 uses a fast interpreter called **Ignition** that starts executing immediately without waiting for full compilation. This gives you fast startup. As your code runs, V8's **profiler** watches for functions that are called repeatedly — called "hot" code. It then passes those hot functions to its optimizing JIT compiler called **TurboFan**, which compiles them to highly optimized machine code. If the assumptions the optimizer made turn out to be wrong (say, a variable that was always a number suddenly receives a string), V8 **de-optimizes** back to interpreted mode and tries again. This cycle repeats constantly during execution.

So JavaScript today is neither purely interpreted nor compiled ahead-of-time — it sits in a middle ground where interpretation and compilation happen *together, dynamically*. The technical term for this is **"interpreted language with JIT compilation."**

Compare this to Java, which is **compiled ahead-of-time to bytecode, then JIT-compiled at runtime by the JVM**. Java has two compilation stages; JavaScript effectively has one that happens at runtime.

### JavaScript vs Java: Similarities and Differences

People often joke that "Java and JavaScript are as related as car and carpet," but there are actually more structural similarities than that joke suggests — and the differences are deep and architectural.

**Surface similarities (and why they exist):**

The syntax was deliberately influenced by Java. When Brendan Eich created JavaScript in 1995, Netscape wanted it to look familiar to Java developers, so curly braces, `if/else`, `for` loops, and similar constructs look nearly identical. This was a marketing decision as much as a technical one.

Both are garbage-collected, so you don't manage memory manually. Both have a concept of classes (though JavaScript's are fundamentally different underneath, as you'll see in Stage 6 when you study prototypes). Both are used heavily in networked, multi-tier applications today.

**Deep differences:**

The most fundamental difference is their **type systems**. Java is **statically typed** — every variable's type is declared at compile time and enforced by the compiler. If you declare `int x = 5`, you can never put a string in `x`. The type checker catches errors before the program runs.

JavaScript is **dynamically typed** — types are associated with *values*, not *variables*, and are checked at *runtime*. The same variable can hold a number, then a string, then an object. This is why TypeScript (which you'll learn in Stage 9) exists — it adds static typing on top of JavaScript.

```java
// Java — type is part of the variable declaration
int count = 0;
count = "hello"; // ❌ COMPILE ERROR — caught before running
```

```js
// JavaScript — variable has no fixed type
let count = 0;
count = "hello"; // ✅ completely valid — no error until runtime if it causes a problem
```

The second major architectural difference is the **object model**. Java uses **classical inheritance** — you define classes, which are blueprints, and create instances from them. The inheritance hierarchy is rigid and class-based.

JavaScript uses **prototypal inheritance** — every object has a prototype (another object) that it delegates to when a property lookup fails. The `class` syntax you see in modern JavaScript is syntactic sugar built *on top of* this prototype system, which is why JavaScript classes behave differently from Java classes in subtle but important ways. You'll explore this in Stage 6.

The third difference is **concurrency model**. Java is multi-threaded — you can run code in parallel using threads, and you have to worry about thread synchronization, race conditions, and locks. JavaScript is single-threaded with an event loop, which sidesteps those concurrency bugs entirely but introduces a different mental model for handling async work.

Java threads are like multiple chefs working simultaneously in the kitchen, requiring coordination so they don't grab the same knife at the same time. JavaScript's event loop is one chef who handles prep work while a timer (the runtime) is handling the oven — they never work at the same time, which avoids collisions but means the chef can never just stand there waiting.

Finally, **compilation and deployment** are fundamentally different. Java requires a compile step (`javac`) before you can run anything. JavaScript is shipped as source code and compiled by the engine on the user's machine. This is why you can open any webpage, view source, and read the original JavaScript — there's no separate bytecode artifact.


---
### Execution Context

Every time JavaScript runs code, it creates an **Execution Context** — a container that holds:

1. **The code to run**
2. **Variable bindings** (what variables exist and their values)
3. **A reference to the outer scope** (more on this in Stage 2)

There are two kinds:

**Global Execution Context** — created once when your script loads. In the browser, `this` here is `window`. In Node, it's `global`.

**Function Execution Context** — created _each time_ a function is called.

```js
// Global Execution Context is created
const name = "Alice";       // stored in global memory

function greet() {
    // New Function Execution Context created HERE
    const message = "Hello"; // stored in this function's memory
    console.log(message + " " + name);
    // Function Execution Context destroyed when function returns
}

greet(); // "Hello Alice"
```

When `greet()` is called, a new execution context is created, pushed onto the call stack, runs, then gets popped off and destroyed.

---

### The Call Stack

The **call stack** is how JavaScript tracks _where it is_ in execution. It's a stack (LIFO — last in, first out).

```js
function multiply(a, b) {
    return a * b;         // 3rd: runs, returns, popped off
}

function square(n) {
    return multiply(n, n); // 2nd: calls multiply, waits
}

function printSquare(n) {
    const result = square(n); // 1st: calls square, waits
    console.log(result);
}

printSquare(4); // → 16
```

The call stack at peak depth looks like:

```
| multiply(4, 4)   |  ← top (currently running)
| square(4)        |
| printSquare(4)   |
| global           |  ← bottom (always here)
```

Each function finishes, gets popped, and the one below resumes.

**Stack overflow** is when you exceed the stack's limit — classic example: infinite recursion.

```js
function infinite() {
    return infinite(); // calls itself forever → stack overflow
}
```

---

### Memory Allocation Basics

JavaScript allocates memory in two places:

**Stack memory** — for primitive values and function call frames. Fixed size, fast, automatically managed.

**Heap memory** — for objects (arrays, functions, objects). Dynamic size, managed by the garbage collector.

```js
// Stack: 'x' stores the value 42 directly
let x = 42;

// Heap: 'user' stores a REFERENCE (memory address) to the object
let user = { name: "Alice", age: 30 };
```

This distinction — **value on the stack** vs **reference to the heap** — is why "pass by value vs pass by reference" exists. You'll see it become critical when we hit React state.

---

## Chapter 2: Variables and Data Types

### `var`, `let`, `const`

These three all declare variables, but they differ in **scope** and **reassignability**.

```js
var oldWay = "function-scoped, hoisted, avoid this";
let changeable = "block-scoped, can be reassigned";
const fixed = "block-scoped, cannot be reassigned";
```

**The rule for modern JS:** Use `const` by default. Use `let` only when you know the variable will be reassigned. Treat `var` as legacy — you'll see it in old codebases but shouldn't write it.

```js
const PI = 3.14159;
PI = 3; // ❌ TypeError: Assignment to constant variable

let count = 0;
count = 1; // ✅ fine

const user = { name: "Alice" };
user.name = "Bob"; // ✅ FINE — the object mutated, but 'user' still points to same object
user = {};         // ❌ TypeError — you're reassigning the variable itself
```

That last pair is subtle and important. `const` prevents _reassignment_, not _mutation_. This is directly relevant to React state management.

---

### Primitive Types

Primitives are stored **by value** on the stack. There are 7:

```js
// string
const greeting = "Hello";
const template = `Hello ${"world"}`; // template literal, covered later

// number (no int/float distinction — it's all IEEE 754 float)
const age = 30;
const price = 9.99;
const result = 0.1 + 0.2; // 0.30000000000000004 — classic float gotcha

// boolean
const isActive = true;
const isLoggedIn = false;

// null — intentional absence of value (you set this deliberately)
const selectedItem = null;

// undefined — value not assigned (JS sets this automatically)
let username; // undefined
const obj = {};
console.log(obj.missingKey); // undefined

// symbol — unique identifier, rarely used unless building libraries
const id1 = Symbol("id");
const id2 = Symbol("id");
console.log(id1 === id2); // false — always unique

// bigint — integers beyond Number's safe range
const huge = 9007199254740991n; // note the 'n' suffix
```

**`null` vs `undefined`** — a common point of confusion:

```js
typeof null;      // "object" ← this is a famous JS bug, not meaningful
typeof undefined; // "undefined"

null == undefined;  // true  (loose equality)
null === undefined; // false (strict equality — different types)
```

In practice: use `null` when you want to explicitly say "no value here." `undefined` shows up automatically when something hasn't been set.

---

### Reference Types (Objects)

Everything that isn't a primitive is an **object** — including arrays and functions. Objects live on the **heap** and variables hold a _reference_ (pointer) to them.

```js
const a = { x: 1 };
const b = a; // b holds the SAME reference, not a copy

b.x = 99;
console.log(a.x); // 99 — a and b point to the same object!
```

This is **pass by reference** behavior. Extremely important for React — mutating objects directly can cause bugs because React can't detect the change.

---

### Pass by Value vs Pass by Reference

```js
// Primitives: pass by VALUE
function addOne(n) {
    n = n + 1;
    return n;
}
let num = 5;
addOne(num);
console.log(num); // 5 — unchanged. The function got a COPY.

// Objects: pass by REFERENCE
function rename(person) {
    person.name = "Bob";
}
const user = { name: "Alice" };
rename(user);
console.log(user.name); // "Bob" — the original object was mutated!
```

---

### Mutation vs Reassignment

```js
const arr = [1, 2, 3];

// Mutation — modifying the existing object in memory
arr.push(4);           // ✅ allowed even with const
arr[0] = 99;           // ✅ allowed even with const
console.log(arr);      // [99, 2, 3, 4]

// Reassignment — pointing the variable to a NEW object
arr = [1, 2, 3];       // ❌ TypeError — const prevents this
```

In React, you'll always create _new_ objects/arrays instead of mutating existing ones. The reason: React compares references to decide whether to re-render. If you mutate in place, the reference stays the same, and React thinks nothing changed.

---

### ✅ Checkpoint 2

Try these in a browser console (open DevTools → Console):

```js
// Predict the output before running
const x = 10;
let y = x;
y = 20;
console.log(x); // ?

const obj1 = { val: 10 };
const obj2 = obj1;
obj2.val = 20;
console.log(obj1.val); // ?

const obj3 = { val: 10 };
const obj4 = { ...obj3 }; // spread operator — creates a shallow copy
obj4.val = 20;
console.log(obj3.val); // ?
```

---

## Chapter 3: Operators & Expressions

### Arithmetic

```js
5 + 3    // 8
10 - 4   // 6
3 * 4    // 12
10 / 3   // 3.333...
10 % 3   // 1  (remainder/modulo — useful for even/odd checks)
2 ** 3   // 8  (exponentiation)
```

One gotcha — `+` with strings does concatenation:

```js
"5" + 3  // "53" — number gets coerced to string!
"5" - 3  // 2   — subtraction coerces string to number
```

---

### Comparison Operators

```js
5 == "5"   // true  — loose equality, coerces types
5 === "5"  // false — strict equality, no coercion

5 != "5"   // false
5 !== "5"  // true

10 > 5     // true
10 >= 10   // true
3 < 5      // true
```

**Rule: Always use `===` and `!==`.** Loose equality (`==`) has surprising behavior due to type coercion and is a common source of bugs.

---

### Logical Operators

```js
true && false  // false — AND: both must be true
true || false  // true  — OR: at least one must be true
!true          // false — NOT: inverts the value
```

**Short-circuit evaluation** — this is used extensively in React:

```js
// && returns first falsy value, or last value if all truthy
false && "hello"   // false (stops at false)
true && "hello"    // "hello"
true && true       // true

// || returns first truthy value, or last value if all falsy
null || "default"  // "default"
"Alice" || "Bob"   // "Alice"

// ?? (nullish coalescing) — returns right side only if left is null/undefined
null ?? "default"  // "default"
0 ?? "default"     // 0    ← different from ||, which would return "default"
```

In React, you'll see patterns like:

```jsx
{isLoggedIn && <Dashboard />}  // render Dashboard only if logged in
{username || "Guest"}           // show username or fallback
```

---

### Ternary Operator

```js
condition ? valueIfTrue : valueIfFalse
```

```js
const age = 20;
const status = age >= 18 ? "adult" : "minor";
// status = "adult"

// In React JSX:
// <span>{isActive ? "Online" : "Offline"}</span>
```

---

### Truthy and Falsy Values

In JavaScript, every value has an inherent boolean quality.

**Falsy values** (only 6 of them):

```js
false
0
""        // empty string
null
undefined
NaN
```

**Everything else is truthy**, including:

```js
"0"       // non-empty string — truthy!
[]        // empty array — truthy!
{}        // empty object — truthy!
-1        // any non-zero number — truthy!
```

```js
if ("") {
    console.log("runs"); // never runs — empty string is falsy
}
if ("hello") {
    console.log("runs"); // runs — non-empty string is truthy
}
if ([]) {
    console.log("runs"); // runs — empty array is truthy!
}
```

That last one surprises people. An empty array is truthy in JS, unlike in some other languages.

---

## Chapter 4: Control Flow

### if / else

```js
const score = 75;

if (score >= 90) {
    console.log("A");
} else if (score >= 75) {
    console.log("B");
} else if (score >= 60) {
    console.log("C");
} else {
    console.log("F");
}
// → "B"
```

---

### switch

Useful when comparing one value against many options:

```js
const day = "Monday";

switch (day) {
    case "Saturday":
    case "Sunday":
        console.log("Weekend");
        break;
    case "Monday":
        console.log("Start of week");
        break;
    default:
        console.log("Weekday");
}
// → "Start of week"
```

Note the `break` — without it, execution "falls through" to the next case. Usually unintentional.

---

### for loop

```js
for (let i = 0; i < 5; i++) {
    console.log(i); // 0, 1, 2, 3, 4
}
```

Three parts: `initialization; condition; increment`

---

### while loop

```js
let count = 0;
while (count < 3) {
    console.log(count); // 0, 1, 2
    count++;
}
```

Use `while` when you don't know upfront how many iterations you need.

---

### for...of

Iterates over _values_ in an iterable (arrays, strings, etc.):

```js
const fruits = ["apple", "banana", "cherry"];

for (const fruit of fruits) {
    console.log(fruit); // apple, banana, cherry
}

// Works on strings too
for (const char of "hello") {
    console.log(char); // h, e, l, l, o
}
```

This is the modern, clean way to loop over arrays. You'll use this constantly.

---

### break and continue

```js
// break — exit the loop entirely
for (let i = 0; i < 10; i++) {
    if (i === 5) break;
    console.log(i); // 0, 1, 2, 3, 4
}

// continue — skip current iteration, continue loop
for (let i = 0; i < 5; i++) {
    if (i === 2) continue;
    console.log(i); // 0, 1, 3, 4
}
```

---

## 🏋️ Practice Exercises

Work through these before moving to Stage 2. They test the concepts in ways that mirror real React thinking.

**Exercise 1 — Mental model check:**

```js
let a = 5;
let b = a;
b = 10;
console.log(a); // What is a? Why?

let obj1 = { count: 5 };
let obj2 = obj1;
obj2.count = 10;
console.log(obj1.count); // What is obj1.count? Why is this different from above?
```

**Exercise 2 — Truthy/falsy:**

```js
// Predict: which of these log "yes"?
if (0) console.log("yes");
if ("") console.log("yes");
if ([]) console.log("yes");
if (null) console.log("yes");
if ("false") console.log("yes");
if (-1) console.log("yes");
```

**Exercise 3 — Write a function** that takes an array of numbers and returns only the even ones. Use a `for...of` loop and an `if` statement with the `%` operator.

**Exercise 4 — Execution context trace:** Write out, step by step, what the call stack looks like when this code runs:

```js
function add(a, b) { return a + b; }
function sum3(x, y, z) { return add(add(x, y), z); }
console.log(sum3(1, 2, 3));
```

---

## What's Coming in Stage 2

Stage 2 (Functions & Scope) is where JavaScript stops feeling like any other language. Closures in particular are the key to understanding why React hooks like `useState` and `useEffect` behave the way they do.

Once you've worked through these exercises, let me know what felt clear and what felt shaky. We can dig deeper into anything before moving on, or go straight into Stage 2.