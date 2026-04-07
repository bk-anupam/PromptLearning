
# 🚀 Stage 3 — Objects & Arrays (React Core Thinking)

---

# 8️⃣ Objects

## 🔹 Mental Model

An **object = a bag of key → value pairs stored in heap memory**.

```js
const user = {
  name: "Alice",
  age: 25,
};
```

👉 Variable `user` doesn’t hold the object — it holds a **reference (pointer)**.

---

## 🔹 Accessing Properties

```js
user.name       // "Alice"
user["age"]     // 25
```

### When to use what?

- Dot → static key
    
- Bracket → dynamic key
    

```js
const key = "name";
user[key]; // "Alice"
```

---

## 🔹 Dynamic Keys (VERY important in React)

```js
const field = "email";

const formData = {
  [field]: "alice@example.com"
};

console.log(formData);
// { email: "alice@example.com" }
```

👉 This is used in forms, reducers, state updates.

---

## 🔹 Nested Objects

```js
const user = {
  name: "Alice",
  address: {
    city: "Delhi",
    pincode: 110001
  }
};

console.log(user.address.city); // Delhi
```

---

## 🔹 Object Reference Behavior (CRITICAL)

```js
const a = { value: 10 };
const b = a;

b.value = 20;

console.log(a.value); // 20 😱
```

👉 Both point to same memory.

---

## 🔥 Why this matters (React)

React checks **reference equality**, not deep equality.

```js
const obj1 = { x: 1 };
const obj2 = { x: 1 };

obj1 === obj2 // false
```

👉 Even identical objects ≠ same reference

---

## 🔹 Common Pitfall

```js
const state = { count: 0 };

function increment() {
  state.count++; // ❌ mutation
}
```

👉 React won’t detect this change reliably.

---

## ✅ Practice

```js
const user = { name: "Alice" };
const copy = user;

copy.name = "Bob";

console.log(user.name); // ?
```

---

# 9️⃣ Arrays

---

## 🔹 Mental Model

Array = **ordered list (also an object under the hood)**

```js
const arr = [1, 2, 3];
```

---

## 🔹 map() — Transformation (MOST IMPORTANT)

```js
const nums = [1, 2, 3];

const doubled = nums.map(n => n * 2);
// [2, 4, 6]
```

👉 returns NEW array (does NOT mutate)

---

### React Usage

```jsx
{users.map(user => (
  <div key={user.id}>{user.name}</div>
))}
```

---

## 🔹 filter() — Selection

```js
const nums = [1, 2, 3, 4];

const even = nums.filter(n => n % 2 === 0);
// [2, 4]
```

---

## 🔹 reduce() — Aggregation

```js
const nums = [1, 2, 3];

const sum = nums.reduce((acc, curr) => acc + curr, 0);
// 6
```

👉 Think: accumulator pattern

---

## 🔹 find()

```js
const users = [{ id: 1 }, { id: 2 }];

const user = users.find(u => u.id === 2);
// { id: 2 }
```

---

## 🔹 some()

```js
[1, 2, 3].some(n => n > 2); // true
```

---

## 🔹 every()

```js
[1, 2, 3].every(n => n > 0); // true
```

---

## 🔥 Key Insight

|Method|Purpose|Returns|
|---|---|---|
|map|transform|array|
|filter|select|array|
|reduce|combine|any|
|find|first match|value|
|some|any true?|boolean|
|every|all true?|boolean|

---

# 🔟 Immutability (THIS IS THE REACT BREAKTHROUGH)

---

## 🔹 The Core Idea

👉 **Never modify existing data — create new data**

---

## 🔥 Why Mutation Breaks React

React does:

```js
if (oldState === newState) {
  // skip re-render
}
```

So:

```js
state.count = 1; // same reference ❌
```

React thinks nothing changed.

---

## 🔹 Correct Way — Immutable Update

```js
const newState = {
  ...state,
  count: 1
};
```

👉 New object → new reference → React re-renders

---

## 🔹 Spread Operator

### Objects

```js
const user = { name: "Alice" };

const updated = { ...user, name: "Bob" };
```

---

### Arrays

```js
const arr = [1, 2];

const newArr = [...arr, 3];
```

---

## 🔹 Object.assign()

```js
const updated = Object.assign({}, user, { name: "Bob" });
```

👉 older syntax, same idea

---

## 🔹 Nested Update (IMPORTANT EDGE CASE)

```js
const state = {
  user: {
    name: "Alice"
  }
};

const newState = {
  ...state,
  user: {
    ...state.user,
    name: "Bob"
  }
};
```

👉 shallow copy → must copy each level

---

## 🔥 React Pattern

```js
setState(prev => ({
  ...prev,
  loading: true
}));
```

👉 You now understand this fully.

---

## ❌ Common Mistake

```js
const newState = state;
newState.count = 1;
```

👉 still mutation!

---

## ✅ Practice

```js
const arr = [1, 2, 3];

const newArr = arr;
newArr.push(4);

console.log(arr); // ?
```

---

# ⚙️ Stage 4 — Modern JavaScript (ES6+)

---

# 1️⃣1️⃣ Destructuring

---

## 🔹 Objects

```js
const user = { name: "Alice", age: 25 };

const { name, age } = user;
```

---

### Rename variables

```js
const { name: username } = user;
```

---

### Default values

```js
const { city = "Unknown" } = user;
```

---

## 🔹 Arrays

```js
const arr = [10, 20];

const [first, second] = arr;
```

---

## 🔥 React Usage

```js
function Profile({ name, age }) {
  return <div>{name}</div>;
}
```

---

# 1️⃣2️⃣ Spread & Rest

---

## 🔹 Spread (expanding)

```js
const arr = [1, 2];

const newArr = [...arr, 3];
```

---

## 🔹 Rest (collecting)

```js
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
```

---

## 🔥 Key Difference

|Operator|Use|
|---|---|
|spread|expand|
|rest|collect|

---

# 1️⃣3️⃣ Template Literals

---

```js
const name = "Alice";

const msg = `Hello ${name}`;
```

---

### Multi-line

```js
const text = `
Line 1
Line 2
`;
```

---

### Expressions

```js
`2 + 2 = ${2 + 2}`
```

---

## 🔥 React Usage

```jsx
<div className={`btn ${isActive ? "active" : ""}`} />
```

---

# 1️⃣4️⃣ Modules

---

## 🔹 Named Export

```js
// math.js
export const add = (a, b) => a + b;
```

```js
import { add } from "./math";
```

---

## 🔹 Default Export

```js
export default function greet() {}
```

```js
import greet from "./greet";
```

---

## 🔥 Key Difference

|Type|Import Style|
|---|---|
|named|`{ add }`|
|default|`greet`|

---

## 🔥 React Project Structure

```js
import React from "react";
import Header from "./Header";
import { fetchData } from "./api";
```

---

# 🧠 Final Mental Model (VERY IMPORTANT)

Everything you learned boils down to this:

### React = Functions + Objects + Arrays + Immutability

---

### If you understand:

- Objects → state shape
    
- Arrays → rendering lists
    
- Immutability → state updates
    
- Destructuring → clean props/state
    
- Modules → file structure
    

👉 You can read 80% of React code.

---

# 🏋️ Practice Set (Real React Thinking)

---

### 1. Transform data

```js
const users = [
  { name: "A", age: 20 },
  { name: "B", age: 30 }
];

// return names only
```

---

### 2. Immutable update

```js
const state = { count: 0 };

// update to count = 1 (immutably)
```

---

### 3. Nested update

```js
const state = {
  user: { name: "Alice", age: 25 }
};

// update age only
```

---

### 4. Array update (React style)

```js
const todos = ["a", "b"];

// add "c" without mutation
```

---
