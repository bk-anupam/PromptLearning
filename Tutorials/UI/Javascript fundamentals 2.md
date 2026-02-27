
# JavaScript Stage 2: Functions, Scope & Closures

This is the most important stage of the entire roadmap. Everything in React — components, hooks, event handlers, state updates — is built on these three concepts. Take your time here.

---

## Chapter 5: Functions Deep Dive

### Function Declarations

The most straightforward way to define a function. You use the `function` keyword, give it a name, and define its body.

```js
function greet(name) {
    return "Hello, " + name;
}

console.log(greet("Alice")); // "Hello, Alice"
```

The critical thing to understand about function declarations is **hoisting**. JavaScript, before executing any code, scans the file and "lifts" function declarations to the top of their scope. This means you can call a declared function _before_ the line where it's written.

```js
// This works, even though greet is defined below
console.log(greet("Alice")); // "Hello, Alice"

function greet(name) {
    return "Hello, " + name;
}
```

This feels magical but it's a concrete mechanical behavior: during the memory allocation phase of creating the execution context, JS engine stores function declarations in memory _before_ running any code line by line.

---

### Function Expressions

A function expression is when you assign a function as a value to a variable. The function itself can be named or anonymous.

```js
// Anonymous function expression
const greet = function(name) {
    return "Hello, " + name;
};

console.log(greet("Bob")); // "Hello, Bob"

// Named function expression (the name 'greetFn' is only visible inside the function itself)
const greet2 = function greetFn(name) {
    return "Hello, " + name;
};
```

The key difference from a declaration: **function expressions are NOT hoisted**. The variable is hoisted but its value (the function) is not yet assigned, so calling it before the definition throws an error.

```js
console.log(greet("Alice")); // ❌ TypeError: greet is not a function

const greet = function(name) {
    return "Hello, " + name;
};
```

This distinction matters in practice because it forces you to define your function expressions before using them, which actually makes code easier to read top-to-bottom.

---
### Hoisting in depth

In many languages, you cannot use a variable or a function until you have defined it. In JavaScript, "Hoisting" is the behavior where the engine moves declarations to the top of their containing scope during the **Compile/Parsing phase**, before the code even starts executing.

#### 1. How the Engine Sees Your Code

When the JS Engine reads your script, it takes two passes:

1. **Creation Phase (Parsing):** It scans the code for variable and function declarations and sets up memory space for them.    
2. **Execution Phase:** It runs your code line-by-line.
    
Because the memory was already allocated in the first pass, you can "call" a function before the line where it is written.

#### 2. Function Declarations vs. Expressions

Hoisting behaves differently depending on _how_ you write your function. This is a very common source of bugs.

#### A. Function Declarations (Fully Hoisted)

These are moved to the top in their entirety. You can call them anywhere.

```
greet(); // Output: "Hello!" (Works perfectly)

function greet() {
  console.log("Hello!");
}
```
#### B. Function Expressions (Not Hoisted)

If you assign a function to a variable (especially using `var`, `let`, or `const`), only the **variable name** is hoisted, not the function itself.

```
sayHi(); // ❌ TypeError: sayHi is not a function

var sayHi = function() {
  console.log("Hi!");
};
```

- **Why did it fail?** Because `var sayHi` was hoisted and initialized as `undefined`. When you tried to call `undefined()`, the engine crashed. If you used `let` or `const`, it would throw a **ReferenceError** because they stay in a "Temporal Dead Zone" until the code reaches them.

#### The Behavior Change: `var` vs. `let`/`const`

If you swap `var` for `let` or `const` in your example, the error message actually changes, which tells us a lot about what the engine is doing under the hood.
##### With `var` (The "Value" Trap)

```
sayHi(); // ❌ TypeError: sayHi is not a function

var sayHi = function() {
  console.log("Hi!");
};
```

- **What happens:** The engine hoists the variable name `sayHi` and initializes it with the value `undefined`.    
- **The Error:** You are effectively trying to run `undefined()`. Since `undefined` isn't a function, you get a `TypeError`.
    
##### With `let` or `const` (The "Dead Zone")

```
sayHi(); // ❌ ReferenceError: Cannot access 'sayHi' before initialization

let sayHi = function() {
  console.log("Hi!");
};
```

- **What happens:** The engine hoists the variable name `sayHi`, but it **does not initialize it**. It enters a state called the **Temporal Dead Zone (TDZ)**.
- **The Error:** You get a `ReferenceError`. The engine says, "I know `sayHi` exists somewhere, but I'm forbidden from letting you touch it until I reach the line where it is defined."

#### 3. The "Hoisting Hierarchy"

If you have a variable and a function with the same name, **Function Declarations take precedence.** They are hoisted "higher" than variable declarations.

```
console.log(typeof myItem); // Output: "function"

var myItem = 10;

function myItem() {
  return "I am a function";
}
```

#### 4. Why does Hoisting exist?

The creator of JavaScript, Brendan Eich, has noted that hoisting was an unintended consequence of how the original engine handled memory. However, it serves a practical purpose: **Mutual Recursion**.

If Function A calls Function B, and Function B calls Function A, one of them _must_ be defined after the other. Hoisting allows both to "know" about each other regardless of line order.

#### 💡 The Golden Rule of Modern JS

Even though hoisting exists, modern best practice is to:

1. **Always define functions/variables before using them** to make your code readable for humans.
2. **Use `const` and `let`** instead of `var` to avoid the "undefined" hoisting traps.
3. **Use Arrow Functions** (which are expressions) to prevent accidental hoisting.

---
### Why is `var` discouraged?

The use of `var` is generally considered "legacy" code or a "bad smell" in modern JavaScript for three main reasons:

#### A. Lack of Block Scoping

`var` does not care about curly braces `{}` (like `if` statements or `for` loops). It only cares about **functions**. This leads to accidental data leaks.

```
if (true) {
  var secret = "I am leaked!";
}
console.log(secret); // "I am leaked!" (Expected it to be private to the if-block)
```
#### B. Redeclaration is Allowed

With `var`, you can accidentally redefine a variable later in the same file without the engine warning you. This is a nightmare for debugging large files.

```
var user = "Alice";
// ... 500 lines of code later ...
var user = "Bob"; // No error, Alice is gone forever.
```

With `let`, the engine would throw an error: `SyntaxError: Identifier 'user' has already been declared`.
#### C. The "Undefined" Confusion

As seen in your example, `var` initializes as `undefined` during hoisting. This often leads to silent bugs where a variable "exists" but holds the wrong data, rather than crashing early so you can fix the logic.

### ⚖️ Summary Table

|**Feature**|**var**|**let / const**|
|---|---|---|
|**Scope**|Function Scoped|**Block Scoped** `{}`|
|**Hoisting**|Hoisted & Initialized (`undefined`)|Hoisted but **Uninitialized** (TDZ)|
|**Redeclaration**|Permitted (Dangerous)|**Forbidden** (Safe)|
|**Engine Strategy**|Soft/Forgiving|Strict/Predictable|

---
### Arrow Functions

Arrow functions are a shorter syntax introduced in ES6. They're not just cosmetic shorthand — they have a fundamentally different behavior around `this` (more on that in Stage 6), and that difference is why React heavily prefers them.

```js
// Traditional function expression
const double = function(n) {
    return n * 2;
};

// Arrow function — equivalent behavior here
const double = (n) => {
    return n * 2;
};

// Even shorter: if there's only one parameter, parens are optional
const double = n => {
    return n * 2;
};

// Even shorter: if the body is just a return statement, you can omit the braces and 'return'
const double = n => n * 2;

console.log(double(5)); // 10
```

When the arrow function body has multiple statements, you need the curly braces and explicit `return`:

```js
const processUser = (user) => {
    const name = user.name.toUpperCase();
    const greeting = `Hello, ${name}`;
    return greeting; // explicit return needed with curly braces
};
```

One important gotcha: when returning an **object literal** implicitly (without curly braces), you must wrap the object in parentheses, otherwise JS thinks the curly braces are the function body:

```js
// ❌ This is WRONG — JS thinks {} is the function body, not an object
const getUser = () => { name: "Alice" };
console.log(getUser()); // undefined

// ✅ Wrap the object in parentheses
const getUser = () => ({ name: "Alice" });
console.log(getUser()); // { name: "Alice" }
```

You'll write this pattern constantly in React when using `map()` to transform arrays into objects.

---

### Parameters, Default Parameters & Rest Parameters

JavaScript functions are very flexible about parameters — you can call a function with more or fewer arguments than it declares, and it won't throw an error.

```js
function add(a, b) {
    console.log(a, b);
}

add(1, 2);     // a=1, b=2
add(1);        // a=1, b=undefined  (no error!)
add(1, 2, 3); // a=1, b=2  (extra argument silently ignored)
```

**Default parameters** let you specify fallback values when an argument is not provided or is `undefined`:

```js
function greet(name = "stranger", greeting = "Hello") {
    return `${greeting}, ${name}!`;
}

console.log(greet());                   // "Hello, stranger!"
console.log(greet("Alice"));            // "Hello, Alice!"
console.log(greet("Alice", "Welcome")); // "Welcome, Alice!"
console.log(greet(undefined, "Hi"));    // "Hi, stranger!" — undefined triggers the default
console.log(greet(null, "Hi"));         // "Hi, null!" — null does NOT trigger the default
```

That `null` vs `undefined` distinction is subtle but real. Only `undefined` triggers a default parameter value. You'll see this matter in React when passing optional props.

**Rest parameters** let a function accept any number of arguments and collect them into an array:

```js
function sum(...numbers) {
    // 'numbers' is a real array containing all arguments passed
    return numbers.reduce((total, n) => total + n, 0);
}

console.log(sum(1, 2, 3));       // 6
console.log(sum(1, 2, 3, 4, 5)); // 15
```

---

### Return Values

A function returns `undefined` by default if you don't explicitly return a value. This is a common source of bugs.

```js
function add(a, b) {
    const result = a + b;
    // forgot to return! 
}

const sum = add(1, 2);
console.log(sum); // undefined — not 3!
```

A `return` statement also **immediately exits** the function, which is useful for early returns:

```js
function getDiscount(user) {
    // Early return for invalid input
    if (!user) return 0;
    if (!user.isPremium) return 0;
    
    // Only reach here if user exists and is premium
    return 0.20;
}
```

This "early return" pattern is very common in React components for handling edge cases like loading states or missing data.

---

### Functions as First-Class Citizens

This is the concept that makes JavaScript feel fundamentally different from languages like Java (where methods are tied to classes, not standalone values). In JavaScript, **functions are values** — just like numbers, strings, or objects. This means you can do anything with a function that you can do with any other value.

You can store a function in a variable (you've already seen this with function expressions). You can store functions in arrays or objects:

```js
const operations = {
    add: (a, b) => a + b,
    subtract: (a, b) => a - b,
    multiply: (a, b) => a * b,
};

console.log(operations.add(5, 3));      // 8
console.log(operations.multiply(4, 2)); // 8
```

You can **pass a function as an argument** to another function. A function that accepts another function as an argument is called a **higher-order function**:

```js
function runTwice(fn, value) {
    return fn(fn(value));
}

const double = n => n * 2;
console.log(runTwice(double, 3)); // 12 (3 → 6 → 12)
```

You can **return a function from another function**:

```js
function makeMultiplier(factor) {
    // This inner function is returned as a VALUE
    return (number) => number * factor;
}

const triple = makeMultiplier(3);
const quadruple = makeMultiplier(4);

console.log(triple(5));    // 15
console.log(quadruple(5)); // 20
```

Notice that `triple` and `quadruple` are themselves functions — `makeMultiplier` is a **function factory**. This pattern is the foundation of closures, which we'll cover shortly.

The practical relevance to React is enormous. Every event handler you write is a function passed as a value. Every React hook like `useEffect(() => { ... })` takes a function as an argument. Array methods like `map`, `filter`, and `reduce` — which React uses for rendering lists — all take functions as arguments. First-class functions are not just a curiosity; they're the mechanism that makes React work.

---

## Chapter 6: Scope

Scope defines where in your code a variable is **visible and accessible**. Think of scope as a set of rules that determines which variables a given line of code can "see." This is where JavaScript diverges significantly from languages with simpler scoping rules.

### Global Scope

Any variable declared outside of a function or block lives in the **global scope**. It's accessible everywhere in your program.

```js
const appName = "MyApp"; // global scope

function showApp() {
    console.log(appName); // ✅ accessible — functions can see global variables
}

showApp(); // "MyApp"
```

In a browser, global variables become properties on the `window` object. This is why globals are considered dangerous in large applications — any piece of code anywhere can accidentally overwrite them. React's component model solves this by encapsulating state inside components rather than globals.

---

### Function Scope

Variables declared inside a function are visible **only within that function**. They don't exist outside of it.

```js
function processOrder() {
    const orderId = 12345; // function scope — only visible inside processOrder
    const discount = 0.1;
    console.log(`Order ${orderId} with discount ${discount}`);
}

processOrder(); // works fine
console.log(orderId); // ❌ ReferenceError: orderId is not defined
```

Each time you call a function, a **new** function scope is created from scratch. Variables from previous calls don't persist (unless closures are involved — more on that soon).

```js
function counter() {
    let count = 0; // created fresh each time counter() is called
    count++;
    return count;
}

console.log(counter()); // 1
console.log(counter()); // 1 (not 2! A new 'count' is created each call)
console.log(counter()); // 1
```

This is why closures are needed to create persistent state — we'll see this directly.

---

### Block Scope

A block is any code inside curly braces `{}` — if statements, for loops, while loops, or even a standalone pair of curly braces. Variables declared with `let` or `const` are **block-scoped**, meaning they only exist within the block where they're defined.

```js
{
    let blockVar = "I'm in a block";
    const alsoBlock = "me too";
    console.log(blockVar); // ✅ works
}

console.log(blockVar); // ❌ ReferenceError — doesn't exist outside the block
```

This has a very important implication for loops:

```js
for (let i = 0; i < 3; i++) {
    // 'i' is block-scoped to this loop
    setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2 ✅

for (var i = 0; i < 3; i++) {
    // 'var' is function-scoped, NOT block-scoped
    // There's only ONE 'i' shared across all iterations
    setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3 ❌ — by the time setTimeout fires, i is already 3
```

This bug is one of the most famous JavaScript gotchas, and it's precisely why `var` was replaced by `let` and `const`. The `var` version has only one `i` variable that all three callbacks share — and by the time the callbacks run, the loop has finished and `i` is 3. With `let`, each iteration creates a fresh, separate `i`.

---

### Lexical Scoping

Lexical scoping is the rule that determines how JavaScript resolves variable names when there are nested functions. The key principle: **a function's scope is determined by where it is _written_ in the code, not where it is _called_ from.**

The word "lexical" comes from "lexis" (text) — it's about the textual, static structure of your code.

```js
const language = "JavaScript"; // outer scope

function outer() {
    const framework = "React"; // outer function scope

    function inner() {
        const tool = "Vite"; // inner function scope
        
        // 'inner' can see variables from ALL enclosing scopes:
        console.log(language);  // ✅ "JavaScript" — from global scope
        console.log(framework); // ✅ "React" — from outer function scope
        console.log(tool);      // ✅ "Vite" — from its own scope
    }

    inner();
    console.log(tool); // ❌ ReferenceError — outer cannot see inner's variables
}

outer();
```

JavaScript looks up variables by starting in the current scope and walking **outward** (toward the global scope) until it finds the variable. This chain of scopes is called the **scope chain**. It never walks inward — a parent cannot see a child's variables.

Think of it like nested rooms: you can see through the windows of the rooms outside you, but not into the rooms inside you.

The "lexical" part means this lookup is based on how the code is _written_, not how it's _executed_. Even if you take a function and call it from a completely different place in your code, it still remembers the scope chain from where it was _defined_. This is the foundation of closures.

---

## Chapter 7: Closures

Closures are the most important concept in JavaScript for understanding React. They feel abstract at first, but once the mental model clicks, you'll see them everywhere.

### The Core Idea

A **closure** is a function that **remembers the variables from the scope in which it was created**, even after that scope has finished executing. The inner function "closes over" the outer variables — hence the name.

Let's start with the simplest possible example:

```js
function makeCounter() {
    let count = 0; // this variable lives in makeCounter's scope

    // This inner function is a closure — it remembers 'count'
    function increment() {
        count++;
        return count;
    }

    return increment; // we return the function itself, not the result
}

const counter = makeCounter(); // makeCounter runs and finishes
// Normally, 'count' would be destroyed here...
// But 'increment' still has a reference to it!

console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

When `makeCounter()` returns, its execution context is popped off the call stack. In a world without closures, `count` would be garbage collected. But because `increment` was defined _inside_ `makeCounter` and references `count`, the JavaScript engine keeps `count` alive in memory for as long as `increment` exists.

You can create multiple independent counters — each closes over its own `count`:

```js
const counterA = makeCounter();
const counterB = makeCounter();

console.log(counterA()); // 1
console.log(counterA()); // 2
console.log(counterB()); // 1 — counterB has its OWN separate 'count'
console.log(counterA()); // 3 — counterA's 'count' is unaffected by counterB
```

This is exactly how React's `useState` works conceptually — each component instance has its own enclosed state that doesn't interfere with other instances of the same component.

---

### A More Realistic Pattern: Function Factories

Closures enable you to create specialized functions from a general template:

```js
function makeGreeter(greeting) {
    // 'greeting' is captured in the closure
    return (name) => `${greeting}, ${name}!`;
}

const sayHello = makeGreeter("Hello");
const sayHowdy = makeGreeter("Howdy");
const sayBonjour = makeGreeter("Bonjour");

console.log(sayHello("Alice"));   // "Hello, Alice!"
console.log(sayHowdy("Bob");      // "Howdy, Bob!"
console.log(sayBonjour("Claude")); // "Bonjour, Claude!"
```

Each returned function carries its own private `greeting` value. This is the function factory pattern — closures give each factory output its own enclosed state.

---

### Closures Enable Private State

Unlike Java, JavaScript doesn't have a native `private` keyword for standalone variables (classes have private fields, but that's different). Closures let you simulate privacy:

```js
function createBankAccount(initialBalance) {
    // 'balance' is private — nothing outside can directly access it
    let balance = initialBalance;

    return {
        deposit: (amount) => {
            balance += amount;
            console.log(`Deposited ${amount}. Balance: ${balance}`);
        },
        withdraw: (amount) => {
            if (amount > balance) {
                console.log("Insufficient funds");
                return;
            }
            balance -= amount;
            console.log(`Withdrew ${amount}. Balance: ${balance}`);
        },
        getBalance: () => balance,
    };
}

const account = createBankAccount(100);
account.deposit(50);    // "Deposited 50. Balance: 150"
account.withdraw(30);   // "Withdrew 30. Balance: 120"
console.log(account.getBalance()); // 120

// There is NO way to directly access or modify 'balance' from outside
console.log(account.balance); // undefined — it's truly private
```

This is the **module pattern** — a classic use of closures to create encapsulated, stateful objects.

---

### Stale Closures: The Gotcha That Bites React Developers

A stale closure is when a function "remembers" an _old version_ of a variable, because it closed over the value at a particular point in time but the variable has since been updated. This is probably the most common confusing bug in React.

Let's first understand it in plain JavaScript:

```js
function createTimer() {
    let count = 0;

    // This callback closes over 'count' at the time it's created
    const showCount = () => {
        console.log("Count is:", count);
    };

    setInterval(() => {
        count++;
    }, 1000);

    return showCount;
}

const show = createTimer();
setTimeout(show, 3500); // Shows "Count is: 3" ✅ — works fine here
```

This works because `showCount` closes over `count` _by reference_ — it looks up `count` in the outer scope each time it runs, so it sees the current value. Primitives in closures are read each time the function executes — the issue arises when closures capture a specific value at a specific moment in a more complex scenario.

Here's where it gets tricky, directly mirroring a real React bug:

```js
function setupSearch() {
    let query = "";

    document.getElementById("input").addEventListener("input", (e) => {
        query = e.target.value;
    });

    // Imagine this is set up once and called repeatedly
    const handleSearch = () => {
        // This function closes over 'query'
        console.log("Searching for:", query);
        fetch(`/api/search?q=${query}`);
    };

    document.getElementById("button").addEventListener("click", handleSearch);
}
```

This works fine because `handleSearch` always reads the _current_ `query`. The stale closure problem appears when closures are created in loops or when event listeners aren't properly cleaned up — the function holds a reference to an _old version_ of the context.

Here's a cleaner demonstration of the stale closure trap:

```js
function makeAdder(x) {
    return (y) => x + y; // closes over x
}

const add5 = makeAdder(5); // x is captured as 5

// Later, even if you wish x were different, this function always uses 5
console.log(add5(3));  // 8 — x is "stale" at 5, but here that's intentional

// The problem arises when staleness is UNINTENTIONAL
// This is exactly what happens in React with useEffect and useState
```

---

### How Closures Explain React Hooks

Now let's connect everything to React. You haven't written React yet, but understanding this now will save you hours of confusion later.

Here is a conceptually simplified version of what React's `useState` does under the hood:

```js
// Simplified mental model of how useState works
function createUseState(initialValue) {
    // React stores state OUTSIDE the component in its own memory
    let storedValue = initialValue;

    function useState() {
        // 'setValue' closes over 'storedValue'
        const setValue = (newValue) => {
            storedValue = newValue;
            // React also triggers a re-render here
        };

        return [storedValue, setValue];
    }

    return useState;
}
```

When you write a React component, the component function itself is re-called on every render. Each render creates a new closure over the current state values. This is why when you read state inside an event handler, you're reading the value from the render in which that handler was created.

The famous React stale closure bug looks like this (written in conceptual JS to illustrate the principle):

```js
// Imagine this represents a React component rendering
function renderComponent() {
    let count = 0; // imagine this is state

    // This event handler closes over 'count' at render time
    const handleClick = () => {
        // When this runs, it uses the 'count' from when it was CREATED
        console.log(count); // always sees the count from this render
        count + 1; // won't update the real state — this is a simplified model
    };

    return handleClick;
}

const handler1 = renderComponent(); // count is 0, handler closes over 0
const handler2 = renderComponent(); // count is 0, handler closes over 0
// Even if you updated count externally between these calls,
// each handler has its own closed-over version
```

In real React, the stale closure bug typically manifests in `useEffect`:

```js
// Real React code demonstrating the stale closure issue
function SearchComponent() {
    const [query, setQuery] = useState("");

    useEffect(() => {
        // This effect closes over 'query' from this render
        const timer = setInterval(() => {
            fetch(`/api/search?q=${query}`); // 'query' might be stale here!
        }, 1000);

        return () => clearInterval(timer);
    }, []); // ← empty dependency array means this effect only runs ONCE
    // So 'query' inside the interval will always be "" — the initial value
    // This is the stale closure bug
}
```

The fix is to include `query` in the dependency array, which tells React to re-create the effect (and thus a fresh closure over the new `query`) every time `query` changes. Understanding closures is _exactly_ what makes this fix intuitive rather than a cargo-culted rule.

---

## Connecting All Three Concepts

Scope, functions, and closures work together as one unified system. Here's an example that weaves all three together in a pattern you'd recognize in real frontend code:

```js
// A function factory that creates specialized data fetchers
function createFetcher(baseUrl) {
    // 'baseUrl' is in the closure
    
    let cache = {}; // private state via closure
    
    return {
        // Each method closes over both 'baseUrl' and 'cache'
        get: async (endpoint) => {
            const url = `${baseUrl}${endpoint}`;
            
            if (cache[url]) {
                console.log("Cache hit:", url);
                return cache[url];
            }
            
            const response = await fetch(url);
            const data = await response.json();
            cache[url] = data; // mutating the closed-over cache
            return data;
        },
        
        clearCache: () => {
            cache = {}; // reassigning the closed-over cache
        }
    };
}

const githubFetcher = createFetcher("https://api.github.com");
const openAiFetcher = createFetcher("https://api.openai.com");

// Each fetcher has its own private cache — they don't interfere
githubFetcher.get("/users/torvalds");
githubFetcher.get("/users/torvalds"); // cache hit!
openAiFetcher.get("/v1/models");       // separate cache, no interference
```

---

## ✅ Checkpoint Exercises

Before moving to Stage 3, make sure you can work through these confidently.

**Exercise 1 — Predict the output:**

```js
const x = "global";

function outer() {
    const x = "outer";
    function inner() {
        console.log(x); // What does this print?
    }
    inner();
}

function unrelated() {
    console.log(x); // What does this print?
}

outer();
unrelated();
```

**Exercise 2 — Create a closure:** Write a function `makeAccumulator()` that returns a function. Each time the returned function is called with a number, it should add that number to a running total and return the new total.

```js
const acc = makeAccumulator();
console.log(acc(5));  // 5
console.log(acc(10)); // 15
console.log(acc(3));  // 18

const acc2 = makeAccumulator();
console.log(acc2(1)); // 1 — separate accumulator, separate total
```

**Exercise 3 — Spot the bug:** Why does this not work as intended, and how would you fix it?

```js
function makeMultipliers() {
    const multipliers = [];
    
    for (var i = 1; i <= 5; i++) {
        multipliers.push(() => i * 10);
    }
    
    return multipliers;
}

const fns = makeMultipliers();
console.log(fns[0]()); // Expected 10, but what does it actually print?
console.log(fns[2]()); // Expected 30, but what does it actually print?
// How do you fix this with one character change?
```

**Exercise 4 — Mental model:** Without running the code, trace through what the call stack and closure scopes look like at each step:

```js
function makeLogger(prefix) {
    return function(message) {
        console.log(`[${prefix}] ${message}`);
    };
}

const infoLog = makeLogger("INFO");
const errorLog = makeLogger("ERROR");
infoLog("Server started");   // What prints?
errorLog("Connection lost"); // What prints?
```

Once you can reason through these exercises confidently — especially Exercise 3, which is a real-world bug — you have the mental foundation to understand everything in the React/Next.js ecosystem. Stage 3 (Objects, Arrays, and Immutability) will build directly on top of what you've learned here.