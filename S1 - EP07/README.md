# Episode 07 — Understanding Sync, Async, V8, libuv & `setTimeout(0)` in Node.js

> A detailed set of notes and code examples explaining synchronous vs asynchronous execution in Node.js, the role of V8 and libuv, file operations, cryptographic operations, and the behavior of `setTimeout(0)`.

---

## 📌 Table of Contents

- [Overview](#-overview)
- [What is Node.js?](#-what-is-nodejs)
- [V8 Engine](#-v8-engine)
- [libuv](#-libuv)
- [Synchronous Operations](#-synchronous-operations)
- [Asynchronous Operations](#-asynchronous-operations)
- [`fs.readFileSync()`](#-fsreadfilesync)
- [`fs.readFile()`](#-fsreadfile)
- [Synchronous vs Asynchronous File Reading](#-synchronous-vs-asynchronous-file-reading)
- [The `crypto` Module](#-the-crypto-module)
- [`pbkdf2Sync()` vs `pbkdf2()`](#-pbkdf2sync-vs-pbkdf2)
- [Why Synchronous Operations Can Be a Problem](#-why-synchronous-operations-can-be-a-problem)
- [`setTimeout(0)`](#-settimeout0)
- [Call Stack, Event Queue & libuv](#-call-stack-event-queue--libuv)
- [Output Example](#-output-example)
- [Important Interview Points](#-important-interview-points)
- [Common Mistakes](#-common-mistakes)
- [Best Practices](#-best-practices)
- [Quick Revision](#-quick-revision)
- [Conclusion](#-conclusion)

---

# 🚀 Overview

Node.js can execute JavaScript code synchronously while also providing mechanisms for handling asynchronous operations.

Two important components behind this behavior are:

- **V8** — responsible for executing JavaScript.
- **libuv** — handles asynchronous operations such as I/O, timers, and other tasks.

Understanding the interaction between these components is essential for understanding:

- Blocking vs non-blocking code
- Synchronous vs asynchronous execution
- File-system operations
- Cryptographic operations
- Timers
- The event loop
- Why `setTimeout(0)` does not execute immediately

The key idea is:

> **Synchronous code executes immediately and can block the main thread, while asynchronous operations allow the application to continue processing other work.**

---

# 🟢 What is Node.js?

Node.js provides a JavaScript runtime environment in which JavaScript can be executed outside the browser.

A simplified view of the execution environment is:

```text
JavaScript Code
      │
      ▼
     V8
      │
      ├── Executes JavaScript
      │
      ▼
   Node.js APIs
      │
      ▼
    libuv
      │
      ├── File I/O
      ├── Timers
      ├── Asynchronous operations
      └── Event Loop
```

Node.js therefore combines JavaScript execution with APIs that allow applications to perform I/O and other operations asynchronously.

---

# ⚙️ V8 Engine

## What is V8?

**V8** is the JavaScript engine responsible for executing JavaScript code.

In the context of these examples:

- V8 executes JavaScript instructions.
- Synchronous JavaScript executes one operation at a time.
- A synchronous operation can prevent the next JavaScript operation from executing until the current operation finishes.

### Simple example

```javascript
console.log("Hello");

function mulFn(x, y) {
  const result = x * y;
  return result;
}

var a = 1078698;
var b = 20986;

var c = mulFn(a, b);

console.log("Multiplication result is: ", c);
```

The execution is sequential:

```text
console.log("Hello")
        ↓
mulFn(a, b)
        ↓
store result in c
        ↓
console.log(result)
```

The multiplication function returns its result before the next statement executes.

---

# 🔄 libuv

## What is libuv?

**libuv** is a library used by Node.js to manage asynchronous operations.

It is responsible for managing operations such as:

- Asynchronous I/O
- Timers
- Event-loop related work
- Other operations that should not unnecessarily block JavaScript execution

A simplified model:

```text
                Node.js
                   │
          ┌────────┴────────┐
          │                 │
         V8               libuv
          │                 │
          │          ┌──────┼─────────┐
          │          │      │         │
          │         I/O   Timers   Other async work
          │
    Executes JS
```

This separation is important because JavaScript execution and asynchronous work are handled differently.

---

# ⛔ Synchronous Operations

A **synchronous operation** means that the next piece of code waits until the current operation finishes.

Example:

```javascript
console.log("Start");

function mulFn(x, y) {
  const result = x * y;
  return result;
}

var a = 1078698;
var b = 20986;

var c = mulFn(a, b);

console.log("Multiplication result is: ", c);

console.log("End");
```

The order is predictable:

```text
Start
Multiplication result is: ...
End
```

The next statement does not execute until the previous synchronous statement has completed.

---

# 📁 `fs.readFileSync()`

Node.js provides the `fs` (File System) module for working with files.

`fs.readFileSync()` reads a file **synchronously**.

```javascript
const fs = require("fs");

const dataSync = fs.readFileSync("./file.txt", "utf8");

console.log("File data fetched synchronously: ", dataSync);
```

## How it behaves

When Node.js reaches:

```javascript
fs.readFileSync("./file.txt", "utf8");
```

the execution waits for the file-reading operation to finish.

Conceptually:

```text
Start
  │
  ▼
readFileSync()
  │
  │  File is being read
  │
  ▼
File reading completed
  │
  ▼
Next JavaScript statement
```

The event loop is blocked while the synchronous operation is running.

---

## ❌ Incorrect Usage of `fs.readFileSync()`

A common mistake is trying to provide a callback to `readFileSync()`:

```javascript
fs.readFileSync("./file.txt", "utf8", (err, data) => {
  console.log("File data fetched synchronously: ", data);
});
```

This is incorrect because `readFileSync()` is a synchronous method and does not use a callback for the result.

The synchronous result should instead be captured:

```javascript
const dataSync = fs.readFileSync("./file.txt", "utf8");

console.log("File data fetched synchronously: ", dataSync);
```

For error handling:

```javascript
try {
  const dataSync = fs.readFileSync("./file.txt", "utf8");

  console.log("File data fetched synchronously: ", dataSync);
} catch (err) {
  console.error("Error reading file synchronously: ", err);
}
```

---

# 🔵 `fs.readFile()`

`fs.readFile()` provides an **asynchronous** way to read a file.

```javascript
const fs = require("fs");

fs.readFile("./file.txt", "utf8", (err, data) => {
  if (err) {
    console.error("Error reading file: ", err);
    return;
  }

  console.log("File data is: ", data);
});
```

Instead of waiting for the file operation to finish before continuing, Node.js can continue processing other JavaScript work.

Conceptually:

```text
JavaScript
   │
   ▼
fs.readFile()
   │
   ├──────────────► Asynchronous file operation
   │
   ▼
Continue executing JavaScript
   │
   ▼
Callback executes after operation completes
```

---

# ⚖️ Synchronous vs Asynchronous File Reading

| Feature                                       | `fs.readFileSync()` | `fs.readFile()`           |
| --------------------------------------------- | ------------------- | ------------------------- |
| Execution                                     | Synchronous         | Asynchronous              |
| Callback                                      | Not used for result | Used                      |
| Blocks execution                              | Yes                 | No                        |
| Main thread                                   | Can be blocked      | Can continue other work   |
| Result                                        | Returned directly   | Provided through callback |
| Suitable for performance-critical server code | Generally avoid     | Generally preferred       |

### Simple rule

```text
readFileSync()
     ↓
WAIT

readFile()
     ↓
CONTINUE
```

---

# 🔐 The `crypto` Module

Node.js includes a built-in **`crypto`** module for cryptographic operations.

Examples of Node.js core modules include:

```javascript
require("crypto");
require("fs");
require("https");
```

The `crypto` module can be imported using:

```javascript
const crypto = require("crypto");
```

or:

```javascript
const crypto = require("node:crypto");
```

The `node:` prefix explicitly indicates that the module is a Node.js core module.

---

# 🔑 `pbkdf2Sync()` vs `pbkdf2()`

One of the functions discussed in this episode is **PBKDF2**:

> **Password-Based Key Derivation Function 2**

It can be used to derive a cryptographic key from a password and salt.

Important parameters include:

- **Password** — input password
- **Salt** — additional value used during key derivation
- **Iterations** — number of times the derivation process is performed
- **Key length** — size of the resulting key
- **Digest algorithm** — for example, `sha512`
- **Callback** — used by the asynchronous version

---

## 🔴 `pbkdf2Sync()`

The synchronous version is:

```javascript
crypto.pbkdf2Sync("password", "salt", 50000, 50, "sha512");
```

Because it is synchronous, the JavaScript execution waits until the key has been generated.

Example:

```javascript
const crypto = require("crypto");

console.log("Hello world");

const firstKey = crypto.pbkdf2Sync("password", "salt", 50000, 50, "sha512");

console.log("First key is generated");

console.log("Some other synchronous code");
```

Conceptually:

```text
Start
  ↓
pbkdf2Sync()
  ↓
WAIT
  ↓
Key generated
  ↓
Continue JavaScript
```

---

# 🟢 `pbkdf2()`

Node.js also provides an asynchronous version:

```javascript
crypto.pbkdf2("password", "salt", 50000, 50, "sha512", (err, key) => {
  if (err) {
    console.error(err);
    return;
  }

  console.log("Second key is generated");
});
```

The callback runs when the key-generation operation has completed.

Example:

```javascript
const crypto = require("crypto");

console.log("Hello world");

crypto.pbkdf2("password", "salt", 50000, 50, "sha512", (err, key) => {
  if (err) {
    console.error(err);
    return;
  }

  console.log("Second key is generated");
});

console.log("Other code can continue");
```

The important distinction is:

```text
pbkdf2Sync()
     ↓
Synchronous
     ↓
Blocks JavaScript execution


pbkdf2()
     ↓
Asynchronous
     ↓
Allows JavaScript execution to continue
```

---

# ⚠️ What Does `Sync` at the End Mean?

When you see a Node.js API with `Sync` at the end of its name, it generally indicates that the API is synchronous.

Examples:

```javascript
fs.readFileSync();
crypto.pbkdf2Sync();
```

The key idea is:

> **The `Sync` suffix indicates that the operation is synchronous and can block execution while it is running.**

This should be considered carefully in performance-sensitive applications.

---

# 🧵 Why Synchronous Operations Can Be a Problem

Consider:

```javascript
crypto.pbkdf2Sync("password", "salt", 50000, 50, "sha512");

console.log("Hello");
```

If the synchronous cryptographic operation takes significant time, `"Hello"` cannot execute until that operation finishes.

Conceptually:

```text
JavaScript execution
──────────────────────────────────────────►

pbkdf2Sync()
████████████████████████
                      ↓
                 console.log()
```

During the blocking operation, other JavaScript work cannot proceed.

This becomes particularly important in server applications where many requests may need to be handled.

---

# 🌐 Other Asynchronous Examples

The episode also demonstrates several asynchronous Node.js APIs.

## 1. `https.get()`

```javascript
https.get("https://dummyjson.com/products/1", (res) => {
  console.log("Fetched data successfully!");
});
```

The HTTP request is asynchronous.

The callback is executed when the response is available.

---

## 2. `setTimeout()`

```javascript
setTimeout(() => {
  console.log("settimeout called after 5 sec");
}, 5000);
```

The timer schedules a callback to run after the specified delay.

The JavaScript execution does not simply stop for five seconds waiting for the timer.

---

## 3. `fs.readFile()`

```javascript
fs.readFile("./file.txt", "utf8", (err, data) => {
  console.log("File data is: ", data);
});
```

The file operation is asynchronous, and the callback handles the result when the operation completes.

---

## 4. `crypto.pbkdf2()`

```javascript
crypto.pbkdf2("myownpassword", "salt", 5000, 50, "sha512", (err, key) => {
  console.log("Key is generated: ", key);
  console.log("Key in hex format: ", key.toString("hex"));
  console.log("Key in base64 format: ", key.toString("base64"));
});
```

The key derivation is performed asynchronously, and the callback receives the generated key.

---

# ⏱️ `setTimeout(0)`

One of the most important concepts in this episode is:

```javascript
setTimeout(() => {
  console.log("call me ASAP");
}, 0);
```

A common misconception is:

> "`setTimeout(0)` means execute this function immediately."

That is **not** what it means.

The callback is scheduled for execution after the current synchronous execution has completed and when the timer/event-loop conditions allow it to run.

---

# 🧠 Call Stack, Event Queue & libuv

Consider:

```javascript
console.log("Hello world");

var a = 1078698;
var b = 20986;

setTimeout(() => {
  console.log("call me ASAP");
}, 0);

function mulFn(x, y) {
  const result = x * y;
  return result;
}

var c = mulFn(a, b);

console.log("multiplication result is: ", c);
```

Let's follow the execution.

### Step 1 — Synchronous code starts

```javascript
console.log("Hello world");
```

This executes immediately.

Output:

```text
Hello world
```

---

### Step 2 — `setTimeout(0)` is registered

```javascript
setTimeout(() => {
  console.log("call me ASAP");
}, 0);
```

The callback is scheduled as asynchronous work.

It does **not** interrupt the current synchronous execution.

---

### Step 3 — Synchronous multiplication executes

```javascript
var c = mulFn(a, b);
```

The function executes synchronously:

```javascript
function mulFn(x, y) {
  const result = x * y;
  return result;
}
```

The result is returned immediately to the current JavaScript execution.

---

### Step 4 — Result is printed

```javascript
console.log("multiplication result is: ", c);
```

This executes before the `setTimeout(0)` callback.

---

### Step 5 — Current call stack becomes empty

Only after the currently executing synchronous JavaScript has finished can the scheduled callback be processed.

Therefore:

```javascript
console.log("call me ASAP");
```

runs afterward.

---

# 🔍 Why Does `setTimeout(0)` Run Later?

The important sequence is:

```text
Synchronous JavaScript
        │
        ▼
     Call Stack
        │
        │
        ├───────────────► setTimeout(0)
        │                       │
        │                       ▼
        │                  Scheduled callback
        │
        ▼
All current synchronous code finishes
        │
        ▼
Call Stack becomes empty
        │
        ▼
Callback can execute
```

Therefore:

> **`setTimeout(0)` does not mean "execute now". It means "schedule this callback to run as soon as possible after the current synchronous work has completed and the event loop can process it."**

---

# 🧪 Complete `setTimeout(0)` Example

```javascript
console.log("Hello world");

var a = 1078698;
var b = 20986;

setTimeout(() => {
  console.log("call me ASAP");
}, 0);

function mulFn(x, y) {
  const result = x * y;
  return result;
}

var c = mulFn(a, b);

console.log("multiplication result is: ", c);
```

### Expected order

```text
Hello world
multiplication result is: 22637562228
call me ASAP
```

The exact numeric result follows from the multiplication:

```text
1078698 × 20986 = 22637562228
```

The important part of the example is the ordering:

```text
Hello world
      ↓
multiplication result
      ↓
call me ASAP
```

The timer callback does not execute before the synchronous multiplication and logging.

---

# 🧩 Another Timer Example

```javascript
setTimeout(() => {
  console.log("Call me right now");
}, 0);

setTimeout(() => {
  console.log("Call me after 3 seconds");
}, 3000);
```

The first callback is eligible to run after the current synchronous execution is complete.

The second callback is scheduled for a later time.

A simplified sequence is:

```text
Current synchronous code
        ↓
setTimeout(0) callback
        ↓
After approximately 3 seconds
        ↓
setTimeout(3000) callback
```

The exact execution time of a timer is dependent on the event loop and whether the JavaScript thread is currently available to process the callback.

---

# 📊 Synchronous vs Asynchronous

| Aspect                   | Synchronous             | Asynchronous                                             |
| ------------------------ | ----------------------- | -------------------------------------------------------- |
| Execution                | Sequential              | Can continue while async work is pending                 |
| Blocking                 | Can block execution     | Designed to avoid blocking the main JavaScript execution |
| Result handling          | Often returned directly | Often handled through callbacks/promises                 |
| Example                  | `fs.readFileSync()`     | `fs.readFile()`                                          |
| Crypto example           | `pbkdf2Sync()`          | `pbkdf2()`                                               |
| Timer                    | Not applicable          | `setTimeout()`                                           |
| Event loop               | Can be blocked          | Can continue processing other work                       |
| Production consideration | Use carefully           | Common approach for I/O                                  |

---

# ❌ Common Mistakes

## Mistake 1: Thinking `setTimeout(0)` means immediate execution

```javascript
setTimeout(() => {
  console.log("Hello");
}, 0);

console.log("World");
```

Output:

```text
World
Hello
```

### Why?

`console.log("World")` is synchronous and executes before the scheduled timer callback.

---

## Mistake 2: Passing a callback to `readFileSync()`

Incorrect:

```javascript
fs.readFileSync("./file.txt", "utf8", (err, data) => {
  console.log(data);
});
```

Correct:

```javascript
const data = fs.readFileSync("./file.txt", "utf8");

console.log(data);
```

---

## Mistake 3: Confusing `pbkdf2()` with `pbkdf2Sync()`

```javascript
pbkdf2Sync();
```

is synchronous.

```javascript
pbkdf2();
```

is asynchronous.

Remember:

```text
Sync suffix → synchronous
No Sync suffix → asynchronous version (for the APIs discussed here)
```

---

## Mistake 4: Assuming asynchronous means "runs immediately"

Asynchronous does not mean:

```text
Run immediately
```

It means that the operation can be scheduled/handled without forcing the current JavaScript execution to wait for the operation to finish.

---

# 🏆 Best Practices

## 1. Avoid unnecessary synchronous operations

The episode emphasizes avoiding synchronous methods in production code whenever possible, especially in performance-critical applications.

For example, prefer:

```javascript
fs.readFile("./file.txt", "utf8", callback);
```

over:

```javascript
fs.readFileSync("./file.txt", "utf8");
```

when blocking behavior is undesirable.

---

## 2. Understand the `Sync` suffix

Before using a Node.js API, check whether it has a synchronous variant.

For example:

```javascript
readFileSync();
pbkdf2Sync();
```

Synchronous APIs can block the JavaScript execution.

---

## 3. Do not treat `setTimeout(0)` as an immediate function call

Use the correct mental model:

```text
setTimeout(0)
      ↓
Schedule callback
      ↓
Current synchronous code completes
      ↓
Event loop can process callback
```

---

## 4. Understand blocking behavior

A single synchronous operation may appear harmless in a small script, but blocking operations can become important in performance-sensitive applications.

---

# 🎯 Important Interview Points

### Q1. What is V8?

**Answer:**

V8 is the JavaScript engine responsible for executing JavaScript code.

---

### Q2. What is libuv?

**Answer:**

libuv is the library used by Node.js to manage asynchronous operations such as I/O and timers and to support the event loop.

---

### Q3. What is the difference between `fs.readFileSync()` and `fs.readFile()`?

**Answer:**

`fs.readFileSync()` reads a file synchronously and blocks execution until the operation completes.

`fs.readFile()` reads a file asynchronously and provides the result through a callback after the operation completes.

---

### Q4. Why can `fs.readFileSync()` be problematic?

**Answer:**

Because it can block the main JavaScript execution/event loop while the file is being read, preventing other JavaScript work from progressing.

---

### Q5. What does `Sync` mean in `pbkdf2Sync()`?

**Answer:**

It indicates that the operation is synchronous and blocks execution while it is running.

---

### Q6. What is the asynchronous alternative to `pbkdf2Sync()`?

**Answer:**

```javascript
crypto.pbkdf2();
```

---

### Q7. What does `setTimeout(0)` mean?

**Answer:**

It schedules a callback to run as soon as possible after the current synchronous execution has completed and the event loop can process the callback. It does not mean that the callback executes immediately.

---

### Q8. Why does `setTimeout(0)` execute after synchronous code?

**Answer:**

Because the callback is scheduled as asynchronous work. The current call stack must finish before the callback can execute.

---

### Q9. Is `setTimeout(0)` guaranteed to execute exactly after 0 milliseconds?

**Answer:**

No. The `0` is not a guarantee of immediate execution. The callback depends on the state of the call stack and event loop.

---

# 🧠 Quick Revision

## V8

```text
V8
↓
Executes JavaScript
```

## libuv

```text
libuv
↓
Manages asynchronous operations
↓
I/O + timers + event-loop related work
```

## Synchronous

```text
Start
 ↓
Operation
 ↓
WAIT
 ↓
Finish
 ↓
Next code
```

## Asynchronous

```text
Start
 ↓
Start async operation
 ↓
Continue JavaScript
 ↓
Async operation completes
 ↓
Callback executes
```

## `readFileSync()`

```javascript
const data = fs.readFileSync("./file.txt", "utf8");
```

**Synchronous → blocks execution**

## `readFile()`

```javascript
fs.readFile("./file.txt", "utf8", callback);
```

**Asynchronous → callback handles result**

## `pbkdf2Sync()`

```javascript
crypto.pbkdf2Sync(...);
```

**Synchronous → blocks execution**

## `pbkdf2()`

```javascript
crypto.pbkdf2(..., callback);
```

**Asynchronous → callback handles result**

## `setTimeout(0)`

```javascript
setTimeout(callback, 0);
```

**Schedule callback → current synchronous code finishes → callback can execute**

---

# 📝 Mental Model

A useful way to remember the entire episode is:

```text
                    NODE.JS
                       │
             ┌─────────┴─────────┐
             │                   │
            V8                 libuv
             │                   │
       Executes JS        Handles async work
             │                   │
             │          ┌────────┼────────┐
             │          │        │        │
             │         I/O     Timers   Other
             │
             ▼
        Call Stack
             │
             ▼
   Synchronous execution
             │
             ▼
      Stack becomes empty
             │
             ▼
      Async callbacks
      can be processed
```

The central concept is:

> **JavaScript execution is sequential, but Node.js provides asynchronous mechanisms that allow I/O and other operations to be handled without unnecessarily blocking the main JavaScript execution.**

---
