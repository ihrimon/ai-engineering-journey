# Phase 01 — Foundations and Engineering Mindset ✅

Detailed checklist for the engineering base you need solid before touching AI features — programming and object thinking, backend/API/database fundamentals, Git discipline, and the mental shift from deterministic to AI-enhanced software.

## Checklist

- [x] [Programming & Object-Oriented Foundations](#programming--object-oriented-foundations)
- [x] [Backend, APIs & Databases](#backend-apis--databases)
- [x] [Git, GitHub & Professional Workflow](#git-github--professional-workflow)
- [x] [AI Awareness & Engineering Mindset](#ai-awareness--engineering-mindset)

<a id="programming--object-oriented-foundations"></a>

## Programming & Object-Oriented Foundations

Before writing AI-powered features, you need to think in terms of **objects, state, and behavior** instead of loose scripts.

### Programming Foundations & Object Thinking

- **Variables, types, and functions** — the basic building blocks in JavaScript/TypeScript (`let`/`const`, primitive vs reference types, function declarations vs arrow functions).
- **Object thinking** — modeling real-world entities (a `User`, an `Order`, a `Message`) as objects that bundle related data and behavior together, instead of scattering related logic across many independent functions.
- **TypeScript basics** — static typing, interfaces, and type inference that make object shapes explicit and catch bugs before runtime.

```ts
// Procedural style — data and logic are disconnected
function getFullName(user: { first: string; last: string }) {
  return `${user.first} ${user.last}`;
}

// Object-oriented style — data and behavior live together
class User {
  constructor(
    private first: string,
    private last: string,
  ) {}

  getFullName() {
    return `${this.first} ${this.last}`;
  }
}
```

**Why it matters:** every backend service, API layer, and AI integration you build later is composed of objects passing data and calling behavior on each other.

### Classes, Objects & Object Lifecycle

- **Class vs instance** — a `class` is a blueprint; an object (instance) is created from it with `new`.
- **Constructors** — initialize an object's starting state when it is created.
- **Object lifecycle** — creation (instantiation) → active use (method calls, state changes) → garbage collection (when no references remain, JavaScript's garbage collector reclaims the memory).
- **`this` binding** — understanding how `this` refers to the current instance, and common pitfalls (losing `this` inside callbacks, arrow functions vs regular functions).

```ts
class Connection {
  private isOpen = false;

  constructor(private url: string) {
    this.isOpen = true; // lifecycle: created and opened
  }

  close() {
    this.isOpen = false; // lifecycle: closed, ready for cleanup
  }
}

const db = new Connection('postgres://localhost'); // instantiation
db.close(); // end of active use
```

### Encapsulation & Abstraction

- **Encapsulation** — hiding internal state behind `private`/`protected` fields and exposing only what's needed through public methods, so external code can't corrupt an object's state directly.
- **Abstraction** — exposing a simple interface while hiding complex implementation details (e.g., a `sendEmail()` method hides SMTP configuration, retries, and templating).
- **Getters/setters** — controlled access to internal state, useful for validation or computed values.

```ts
class Account {
  #balance: number; // private field — encapsulated

  constructor(initialBalance: number) {
    this.#balance = initialBalance;
  }

  deposit(amount: number) {
    if (amount <= 0) throw new Error('Invalid amount');
    this.#balance += amount;
  }

  get balance() {
    return this.#balance; // controlled read access
  }
}
```

**Why it matters:** when you later wrap AI providers (OpenAI, Anthropic) behind your own service classes, encapsulation is what lets you swap providers without breaking the rest of the app.

### Inheritance & Polymorphism

- **Inheritance** — a class (`Admin`) can extend a base class (`User`) to reuse and specialize behavior with `extends` and `super`.
- **Polymorphism** — different classes can be used interchangeably through a shared interface, each providing its own behavior for the same method call.
- **Composition over inheritance** — knowing when to favor composing small objects together instead of deep inheritance chains, which is the more common pattern in modern backend/AI codebases (e.g., strategy pattern for swapping AI providers).

```ts
class Notifier {
  send(message: string) {
    throw new Error('Not implemented');
  }
}

class EmailNotifier extends Notifier {
  send(message: string) {
    console.log(`Emailing: ${message}`);
  }
}

class SlackNotifier extends Notifier {
  send(message: string) {
    console.log(`Posting to Slack: ${message}`);
  }
}

function notifyAll(notifiers: Notifier[], message: string) {
  notifiers.forEach((n) => n.send(message)); // polymorphism: same call, different behavior
}
```

<a id="backend-apis--databases"></a>

## Backend, APIs & Databases

### Backend Basics with Node.js & Express

- **Request/response cycle** — a client sends an HTTP request, Node's event loop picks it up, your route handler runs, and a response is sent back. This is the foundation for everything you'll build later, including streaming AI responses.
- **Routing** — mapping a URL + HTTP method to a handler function.
- **Middleware** — functions that run _between_ the request arriving and the final handler, used for logging, auth checks, parsing request bodies, etc. Middleware chains are the same mental model you'll later use for AI request pipelines (validate → rate-limit → call model → log).

```ts
import express from 'express';

const app = express();
app.use(express.json()); // middleware: parse JSON bodies

// logging middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // pass control to the next middleware/handler
});

app.get('/users/:id', (req, res) => {
  res.json({ id: req.params.id, name: 'Ada' });
});

app.listen(3000);
```

### REST API Design

- **Resource-based URLs** — model nouns, not verbs: `/users`, `/orders/42`, not `/getUser`.
- **HTTP verbs map to actions**: `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove).
- **Status codes**: `200` OK, `201` Created, `400` Bad Request, `401` Unauthorized, `404` Not Found, `500` Server Error.
- **Consistent payload shape** — a predictable JSON response structure (`{ data, error }`) makes an API easy for any client — including an AI agent calling it as a tool — to consume reliably.

| Verb   | URL         | Action                |
| ------ | ----------- | --------------------- |
| GET    | `/users`    | List users            |
| GET    | `/users/42` | Get one user          |
| POST   | `/users`    | Create a user         |
| PATCH  | `/users/42` | Update part of a user |
| DELETE | `/users/42` | Delete a user         |

### Databases: PostgreSQL vs MongoDB

- **PostgreSQL (relational)** — data lives in tables with a fixed schema; relationships between tables are defined with foreign keys; transactions guarantee multiple writes succeed or fail together (ACID). Best when data is structured and relationships matter.
- **MongoDB (document)** — data lives as flexible JSON-like documents in collections. Best when data shape varies or evolves quickly (logs, chat/message history, unstructured content — common in AI apps that store prompts/responses).

```sql
-- PostgreSQL: relational, with a foreign key relationship
CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT NOT NULL, email UNIQUE NOT NULL);
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  total NUMERIC NOT NULL
);
```

```js
// MongoDB: flexible document
db.messages.insertOne({
  userId: '42',
  role: 'user',
  content: 'Hello!',
  createdAt: new Date(),
});
```

### Authentication & Basic System Design

- **Password hashing** — never store plain-text passwords; hash with something like `bcrypt` so even a database leak doesn't expose real passwords.
- **Sessions vs JWT** — sessions store login state on the server and reference it via a cookie; JWTs encode identity into a signed token the client holds, so the server can verify a request without a database lookup.
- **Basic system design** — thinking about how pieces fit together: client → API server → database, where to add caching, and how a single server differs from one that needs to scale horizontally behind a load balancer.

```ts
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

const hashed = await bcrypt.hash(plainPassword, 10); // store this, never the raw password

const isValid = await bcrypt.compare(plainPassword, hashed);
if (isValid) {
  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET!, {
    expiresIn: '1h',
  });
}
```

### Error Handling & Debugging

- **`try/catch` for expected failure points** — network calls, database queries, parsing untrusted input.
- **Centralized error handling** — instead of repeating error-formatting logic in every route, funnel errors through one Express error-handling middleware.
- **Debugging mindset** — reproduce the bug with the smallest possible input, read the actual stack trace instead of guessing, and use breakpoints/logging to inspect state at the point of failure. This same discipline is critical later for debugging why an AI response was wrong or a tool call failed.

```ts
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await db.findUser(req.params.id);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json({ data: user });
  } catch (err) {
    next(err); // hand off to centralized error handler
  }
});

// centralized error-handling middleware (must have 4 args)
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
});
```

<a id="git-github--professional-workflow"></a>

## Git, GitHub & Professional Workflow

- **Core commands** — `git commit` (save a snapshot), `git branch` (isolate work), `git merge` (combine histories), `git rebase` (replay commits on top of another branch for a linear history).
- **Branching workflow** — create a feature branch off `main`, commit small logical changes, open a pull request, address review comments, merge.
- **Commit message conventions** — short, imperative summary (`fix: handle null user in auth middleware`) so history stays readable and tools like changelogs can parse it.

```bash
git checkout -b feature/user-auth
git add src/auth.ts
git commit -m "feat: add JWT-based login endpoint"
git push -u origin feature/user-auth
```

**Why it matters for AI work:** AI-assisted codebases change fast; disciplined version control (small commits, clear PRs) is what keeps changes reviewable and reversible when something goes wrong.

<a id="ai-awareness--engineering-mindset"></a>

## AI Awareness & Engineering Mindset

### Understanding AI, ML, Deep Learning, and Generative AI

These terms nest inside each other, not stand apart:

- **AI (Artificial Intelligence)** — the broad field of building systems that perform tasks normally requiring human intelligence.
- **ML (Machine Learning)** — a subset of AI where systems learn patterns from data instead of being explicitly programmed with rules.
- **Deep Learning** — a subset of ML using multi-layered neural networks, well suited to unstructured data like text, images, and audio.
- **Generative AI** — a subset of deep learning (typically large language models and diffusion models) that generates new content — text, code, images — rather than just classifying or predicting a number.

### From Traditional Software to AI-Enhanced Software Thinking

- **Deterministic vs probabilistic** — traditional code given the same input always produces the same output; AI model calls are probabilistic — the same prompt can produce different (though usually similar) responses. This changes how you test, validate, and design fallback behavior.
- **Framing business problems for AI** — before reaching for an LLM, ask: is this task actually ambiguous/language-based (a good fit for AI), or is it a deterministic rule that a normal function or database query solves more reliably and cheaply?

### Building the Mindset to Connect Technology with Business Problems

- Start from the business problem, not the technology — "what outcome does this need to produce" before "which model/framework should I use."
- Weigh AI against simpler alternatives — a well-indexed database query or a plain `if` statement is often faster, cheaper, and more predictable than an LLM call for a task.
- Treat reliability, cost, and latency as first-class product requirements, not afterthoughts, when deciding whether and how to add an AI feature.

**Why it matters:** this mindset shift is the bridge from Phase 01's engineering fundamentals into Phase 02's LLM concepts — you'll keep applying the same object-oriented, API, and error-handling foundations, just with a probabilistic component in the mix.

---

⬅️ Back to [Table of Contents](../README.md)

## Interview Angle: 🧠 **[Full Question and Answers →](interview-qa.md)**
