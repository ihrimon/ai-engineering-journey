# Interview Questions & Answers — Phase 01: Foundations and Engineering Mindset

Quick-reference Q&A for everything covered in [README.md](README.md).

| # | Question |
|---|----------|
| 1 | [What is the difference between procedural and object-oriented thinking?](#q1-what-is-the-difference-between-procedural-and-object-oriented-thinking) |
| 2 | [What is the difference between a class and an object?](#q2-what-is-the-difference-between-a-class-and-an-object) |
| 3 | [What happens during an object's lifecycle in JavaScript?](#q3-what-happens-during-an-objects-lifecycle-in-javascript) |
| 4 | [What is encapsulation, and why does it matter?](#q4-what-is-encapsulation-and-why-does-it-matter) |
| 5 | [What is abstraction, and how is it different from encapsulation?](#q5-what-is-abstraction-and-how-is-it-different-from-encapsulation) |
| 6 | [What is inheritance, and when should you avoid it?](#q6-what-is-inheritance-and-when-should-you-avoid-it) |
| 7 | [What is polymorphism, with a practical example?](#q7-what-is-polymorphism-with-a-practical-example) |
| 8 | [What is REST, and what makes an API RESTful?](#q8-what-is-rest-and-what-makes-an-api-restful) |
| 9 | [What's the difference between SQL and NoSQL databases?](#q9-whats-the-difference-between-sql-and-nosql-databases) |
| 10 | [How does token-based authentication (JWT) differ from session-based authentication?](#q10-how-does-token-based-authentication-jwt-differ-from-session-based-authentication) |
| 11 | [What is the difference between `git merge` and `git rebase`?](#q11-what-is-the-difference-between-git-merge-and-git-rebase) |
| 12 | [Why does AI-enhanced software need a different testing mindset than traditional software?](#q12-why-does-ai-enhanced-software-need-a-different-testing-mindset-than-traditional-software) |

---

<a id="q1-what-is-the-difference-between-procedural-and-object-oriented-thinking"></a>

### Q1. What is the difference between procedural and object-oriented thinking?

> Procedural code organizes a program as a sequence of functions that operate on data passed between them, so the data and the logic that changes it are kept separate.
>
> Object-oriented thinking bundles related data (state) and behavior (methods) together into objects, so the object itself is responsible for how its data is used and changed. This makes code easier to reason about and extend as an application grows.

---

<a id="q2-what-is-the-difference-between-a-class-and-an-object"></a>

### Q2. What is the difference between a class and an object?

> A **class** is a blueprint that defines the structure (fields) and behavior (methods) an object will have, but it doesn't hold any real data itself.
>
> An **object** is a concrete instance created from that class using `new`, with its own actual state in memory. You can create many independent objects from the same class.

---

<a id="q3-what-happens-during-an-objects-lifecycle-in-javascript"></a>

### Q3. What happens during an object's lifecycle in JavaScript?

> An object is **created** (memory is allocated, the constructor runs and sets initial state), **used** during the program's execution (methods called, state mutated), and eventually becomes eligible for **garbage collection** once no references to it remain — the JavaScript engine automatically reclaims that memory.

---

<a id="q4-what-is-encapsulation-and-why-does-it-matter"></a>

### Q4. What is encapsulation, and why does it matter?

> Encapsulation is restricting direct access to an object's internal state (using `private`/`#` fields) and only exposing controlled access through public methods or getters/setters.
>
> It matters because it prevents external code from putting an object into an invalid state, and it lets you change the internal implementation later without breaking code that depends on the public interface.

---

<a id="q5-what-is-abstraction-and-how-is-it-different-from-encapsulation"></a>

### Q5. What is abstraction, and how is it different from encapsulation?

> Abstraction is about hiding *complexity* — exposing a simple, high-level interface while the implementation details are hidden underneath (e.g., a `sendEmail()` method hides SMTP setup and retry logic).
>
> Encapsulation is about hiding *state* — controlling access to an object's data. In practice they work together: encapsulation is the mechanism, abstraction is the design goal.

---

<a id="q6-what-is-inheritance-and-when-should-you-avoid-it"></a>

### Q6. What is inheritance, and when should you avoid it?

> Inheritance lets a class (`Admin`) reuse and extend behavior from a base class (`User`) via `extends`/`super`.
>
> It should be avoided when the relationship isn't a true "is-a" relationship, or when it creates deep, rigid hierarchies that are hard to change — in those cases, composition (building behavior out of smaller, combinable objects) is usually more flexible.

---

<a id="q7-what-is-polymorphism-with-a-practical-example"></a>

### Q7. What is polymorphism, with a practical example?

> Polymorphism means different classes can be used interchangeably through a shared interface, each providing its own version of the same method.
>
> For example, `EmailNotifier` and `SlackNotifier` can both implement a `send(message)` method; calling code can loop over a list of notifiers and call `.send()` on each without knowing (or caring) which concrete type it is.

---

<a id="q8-what-is-rest-and-what-makes-an-api-restful"></a>

### Q8. What is REST, and what makes an API RESTful?

> REST (Representational State Transfer) is an architectural style for designing APIs around resources identified by URLs, manipulated using standard HTTP verbs (`GET`, `POST`, `PUT`/`PATCH`, `DELETE`) and communicating state through standard HTTP status codes.
>
> A RESTful API is stateless — each request contains all the information needed to process it.

---

<a id="q9-whats-the-difference-between-sql-and-nosql-databases"></a>

### Q9. What's the difference between SQL and NoSQL databases?

> SQL databases (e.g., PostgreSQL) use structured, relational tables with a fixed schema and support strong consistency and joins via transactions — good for structured, relational data.
>
> NoSQL databases (e.g., MongoDB) store flexible, document-based (JSON-like) data without a rigid schema, which suits rapidly evolving or loosely structured data, often trading off some consistency guarantees for flexibility and horizontal scalability.

---

<a id="q10-how-does-token-based-authentication-jwt-differ-from-session-based-authentication"></a>

### Q10. How does token-based authentication (JWT) differ from session-based authentication?

> Session-based auth stores session state on the server (e.g., in a database or memory store) and gives the client a session ID cookie to reference it.
>
> Token-based auth (JWT) encodes the user's identity and claims directly into a signed token held by the client, so the server can verify the token without storing session state — making it easier to scale across multiple servers, at the cost of harder immediate revocation.

---

<a id="q11-what-is-the-difference-between-git-merge-and-git-rebase"></a>

### Q11. What is the difference between `git merge` and `git rebase`?

> `git merge` combines two branches by creating a new merge commit that ties both histories together, preserving the full history.
>
> `git rebase` replays your branch's commits on top of another branch's latest commit, creating a linear history without a merge commit — cleaner history, but it rewrites commit hashes, so it should be avoided on shared/published branches.

---

<a id="q12-why-does-ai-enhanced-software-need-a-different-testing-mindset-than-traditional-software"></a>

### Q12. Why does AI-enhanced software need a different testing mindset than traditional software?

> Traditional code given the same input always produces the same output, so tests assert exact expected values.
>
> AI model calls are probabilistic — the same prompt can produce different (though usually similar) responses — so validation shifts toward checking properties of the output (does it contain required fields, stay within constraints, avoid unsafe content) rather than exact string matches, and toward designing fallback behavior for when the model gets it wrong.
