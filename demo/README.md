# Phase 01 — Programming Foundations & Object Thinking ✅

Detailed checklist for the concepts you need solid before touching a single class — the pre-OOP building blocks, and the mental shift from procedural to object thinking.

## Checklist

- [x] [Variables, data types, primitive vs reference types](#variables-data-types-primitive-vs-reference-types)
- [x] [Stack vs Heap, value vs reference, scope, memory allocation basics](#stack-vs-heap-value-vs-reference-scope-memory-allocation-basics)
- [x] [Control flow, collections, modules & packages (the pre-OOP building blocks)](#control-flow-collections-modules--packages)
- [x] [Procedural vs Object-Oriented programming — the difference, and _why_ OOP exists](#procedural-vs-object-oriented-programming)
- [x] [The "everything is an object" mindset — modeling real-world entities in code](#the-everything-is-an-object-mindset)
- [x] [The Four Pillars overview: Encapsulation, Abstraction, Inheritance, Polymorphism](#the-four-pillars-overview)
- [x] [Class vs Object — blueprint vs instance](#class-vs-object)
- [x] [State vs Behavior — the relationship between attributes (data) and methods (functions)](#state-vs-behavior)
- [x] [Message passing — how objects communicate with each other](#message-passing)
- [x] [Responsibility-Driven Design — thinking in terms of "who owns this responsibility," not just "what data exists"](#responsibility-driven-design)
- [x] [Where OOP fits and where it doesn't (cases where a functional/procedural approach is the better fit)](#where-oop-fits-and-where-it-doesnt)


<a id="variables-data-types-primitive-vs-reference-types"></a>

## Variables, data types, primitive vs reference types

A variable is a name that points to a location in memory where a value is stored. Every language splits data types into two groups:

- **Primitive types** — `number`, `string`, `boolean`, `null`, `undefined`, `symbol`, `bigint` (JS/TS). These are stored **by value** — the variable directly holds the value itself.
- **Reference types** — `object`, `array`, `function`, class instances. These live on the heap, and the variable actually holds a **reference (pointer)** to that heap location, not the value itself.

```js
// Primitive — copying creates an independent value
let a = 10;
let b = a;
b = 20;
console.log(a); // 10 — a is unchanged

// Reference — copying just copies the pointer; both variables point to the same object
let obj1 = { balance: 100 };
let obj2 = obj1;
obj2.balance = 500;
console.log(obj1.balance); // 500 — obj1 changed too, because both point to the same heap object
```

<a id="stack-vs-heap-value-vs-reference-scope-memory-allocation-basics"></a>

## Stack vs Heap, value vs reference, scope, memory allocation basics

- **Stack** — a small, fast memory region. Primitive values and the "pointer" part of reference variables live here. Every function call creates a new **stack frame**, which is automatically popped (destroyed) when the function returns. It's a LIFO (Last-In-First-Out) structure.
- **Heap** — a larger, dynamic memory region where the actual data for objects/arrays/functions lives. Heap allocation is comparatively slower, and it's managed by the garbage collector (GC) — once no reference to an object remains, the GC frees it.

```js
function createUser(name) {
  let id = 101;              // primitive — lives on the stack (within this frame)
  let user = { name, id };   // object is allocated on the heap; user variable only holds its reference on the stack
  return user;
}

let u1 = createUser("Rahim"); // the stack frame is destroyed when the function returns,
                               // but the heap object survives because u1 still references it
```

**Scope** determines where a variable can be accessed from — `global scope` (everywhere), `function scope` (`var`), `block scope` (`let`/`const`, inside `{}`). Misunderstanding scope is what causes closure-related bugs and memory leaks (e.g. holding on to an event listener's object so the GC can never clean it up).

<a id="control-flow-collections-modules--packages"></a>

## Control flow, collections, modules & packages (the pre-OOP building blocks)

These building blocks need to be solid before learning OOP, because the methods inside a class are written using exactly these tools:

- **Control flow** — `if/else`, `switch`, `for`/`while` loops, early `return`.
- **Collections** — `Array`, `Object`/`Map`, `Set` — the basic structures for grouping data, no class required.
- **Modules & packages** — splitting code across files with `export`/`import`, and pulling in reusable code via a package manager like npm.

```js
// control flow + collection
const books = [
  { title: "Clean Code", available: true },
  { title: "OOP Basics", available: false },
];

function listAvailableBooks(books) {
  const result = [];
  for (const book of books) {
    if (book.available) {
      result.push(book.title);
    }
  }
  return result;
}

console.log(listAvailableBooks(books)); // ["Clean Code"]
```

```js
// modules — utils.js
export function formatTitle(title) {
  return title.trim().toUpperCase();
}

// main.js
import { formatTitle } from "./utils.js";
console.log(formatTitle("clean code")); // "CLEAN CODE"
```

<a id="procedural-vs-object-oriented-programming"></a>

## Procedural vs Object-Oriented programming — the difference, and _why_ OOP exists

In procedural style, data and the functions that operate on it are kept separate — a function reaches in from outside, takes the data, and processes it.

```js
// Procedural approach
function withdraw(account, amount) {
  if (amount > account.balance) throw new Error("Insufficient funds");
  account.balance -= amount;
}

const account = { balance: 1000 };
withdraw(account, 300);
console.log(account.balance); // 700
```

The problem: `account.balance` is directly exposed from the outside — any code can just change it directly (`account.balance = -9999`), bypassing validation entirely. As the codebase grows, it becomes hard to track who is touching `balance` and from where.

In OOP, data and the logic that operates on it are bundled together and encapsulated inside a single unit:

```js
class Account {
  #balance; // private — no direct access from outside

  constructor(balance) {
    this.#balance = balance;
  }

  withdraw(amount) {
    if (amount > this.#balance) throw new Error("Insufficient funds");
    this.#balance -= amount;
  }

  getBalance() {
    return this.#balance;
  }
}

const acc = new Account(1000);
acc.withdraw(300);
console.log(acc.getBalance()); // 700
// acc.#balance = -9999; // ❌ SyntaxError — cannot be accessed directly from outside
```

**Why OOP exists:** the bigger a codebase gets, the harder it becomes to keep "who can change what data, and when" predictable. OOP localizes every responsibility inside an object, so even as the system grows, it stays relatively easy to maintain, extend, and reason about.

<a id="the-everything-is-an-object-mindset"></a>

## The "everything is an object" mindset — modeling real-world entities in code

This mindset says: when you look at a problem in a domain, first ask yourself, "What entities (nouns) exist here in the real world, and what state and behavior do they have?" — then model those entities as objects.

Example — reading a Library system's problem description to pull out nouns:

```text
"A Member can borrow a Book, and owes a fine if it isn't returned on time."

Nouns (candidate objects): Member, Book, Fine
Verbs (candidate behavior): borrow, return, payFine
```

```js
class Book {
  constructor(title, isbn) {
    this.title = title;
    this.isbn = isbn;
    this.isAvailable = true;
  }
}

class Member {
  constructor(name) {
    this.name = name;
    this.borrowedBooks = [];
  }

  borrow(book) {
    if (!book.isAvailable) throw new Error("Book not available");
    book.isAvailable = false;
    this.borrowedBooks.push(book);
  }
}

const book = new Book("Clean Code", "978-0132350884");
const member = new Member("Rahim");
member.borrow(book);
console.log(book.isAvailable); // false
```


<a id="the-four-pillars-overview"></a>

## The Four Pillars overview: Encapsulation, Abstraction, Inheritance, Polymorphism

- **Encapsulation** — bundling data and behavior together, not giving outside code direct access to internal state.
  ```js
  class Account { #balance = 0; deposit(amt) { this.#balance += amt; } }
  ```
- **Abstraction** — hiding complex implementation and exposing only the necessary ("what") part.
  ```js
  account.deposit(500); // the caller doesn't need to know how balance is updated internally
  ```
- **Inheritance** — a class can reuse another class's properties/methods (an "is-a" relationship).
  ```js
  class SavingsAccount extends Account { addInterest() { /* ... */ } }
  ```
- **Polymorphism** — the same method call can behave differently across different classes.
  ```js
  class Shape { area() { return 0; } }
  class Circle extends Shape { area() { return 3.14 * this.r * this.r; } }
  ```

<a id="class-vs-object"></a>

## Class vs Object — blueprint vs instance

A **class** is a blueprint/template — it defines what fields and methods an object will have, but doesn't hold any real data itself. An **object** is a concrete instance created from that blueprint, with its own actual values in memory.

```js
class Car {
  constructor(brand, model) {
    this.brand = brand;
    this.model = model;
  }

  drive() {
    console.log(`${this.brand} ${this.model} is driving`);
  }
}

const car1 = new Car("Toyota", "Corolla"); // object 1 — independent state
const car2 = new Car("Honda", "Civic");    // object 2 — independent state

car1.drive(); // "Toyota Corolla is driving"
car2.drive(); // "Honda Civic is driving"

console.log(car1 === car2); // false — two separate instances made from the same class
```

You can create countless objects from the same `Car` class, and each one's state (brand, model) is completely independent.

<a id="state-vs-behavior"></a>

## State vs Behavior — the relationship between attributes (data) and methods (functions)

**State** is what an object knows/holds (attributes/fields) — its data at a given moment. **Behavior** is what an object can do (methods).

```js
class BankAccount {
  // ---- State ----
  #balance;

  constructor(initialBalance) {
    this.#balance = initialBalance;
  }

  // ---- Behavior ----
  deposit(amount) {
    this.#balance += amount;
  }

  withdraw(amount) {
    if (amount > this.#balance) throw new Error("Insufficient funds");
    this.#balance -= amount;
  }

  getBalance() {
    return this.#balance;
  }
}
```

<a id="message-passing"></a>

## Message passing — how objects communicate with each other

Objects don't reach directly into each other's internal state — they communicate by **calling each other's public methods**, which is referred to as sending a "message" and getting a response.

```js
class Engine {
  #running = false;
  start() {
    this.#running = true;
    console.log("Engine started");
  }
}

class Car {
  #engine;
  constructor() {
    this.#engine = new Engine();
  }

  startCar() {
    this.#engine.start(); // Car sends a "start" message to the Engine object
  }
}

const car = new Car();
car.startCar(); // "Engine started"
// car never touched Engine's #running directly — it only called the start() method
```

This is exactly the pattern that makes encapsulation meaningful in practice — an object's internal state always stays private.

<a id="responsibility-driven-design"></a>

## Responsibility-Driven Design — thinking in terms of "who owns this responsibility," not just "what data exists"

A common mistake is starting by thinking "what data will I need," then grouping it together into a class. Responsibility-Driven Design (RDD) flips this around — "who should actually own this piece of work or this knowledge?"

```js
// ❌ Data-driven thinking — Order itself doesn't do the calculation, something outside does
function calculateTotal(order) {
  return order.items.reduce((sum, item) => sum + item.price * item.qty, 0);
}

// ✅ Responsibility-driven — Order takes on the responsibility of knowing its own total
class Order {
  #items = [];

  addItem(item) {
    this.#items.push(item);
  }

  getTotal() {
    return this.#items.reduce((sum, item) => sum + item.price * item.qty, 0);
  }
}

const order = new Order();
order.addItem({ price: 100, qty: 2 });
console.log(order.getTotal()); // 200 — Order itself owns the responsibility of knowing its total
```

Thinking this way naturally produces classes that are clear and single-purpose

<a id="where-oop-fits-and-where-it-doesnt"></a>

## Where OOP fits and where it doesn't (cases where a functional/procedural approach is the better fit)

OOP isn't the right answer for everything. In some cases, a functional/procedural style is far more readable and simple:

```js
// building a class here would be over-engineering — a plain function is enough
function sum(numbers) {
  return numbers.reduce((total, n) => total + n, 0);
}

function isEven(n) {
  return n % 2 === 0;
}

console.log(sum([1, 2, 3, 4])); // 10
```

**Where OOP is weak/unnecessary:**
- Small scripts or one-off data transformations.
- Pure, stateless utility functions (math, string formatting).
- Math-heavy code, where data doesn't really have an "identity" or "state" concept.

**Where OOP wins:**
- The domain has many interrelated entities whose state needs to be protected and reused (e.g. Library, Banking, E-commerce systems).
- Business rules are complex and at risk of being scattered across multiple places.

In short: **OOP adds value when a domain is complex and stateful; when the problem is essentially a pure data transformation. 

## Project: 🎯 **[Library Management →](./library-management/PROJECT_PLAN.md)**

## Interview Angle: 🎤 **[Full Question and Answers →](./interview-qu.md)**

