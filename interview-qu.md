# Interview Questions & Answers — Phase 01: Programming Foundations & Object Thinking

Quick-reference Q&A for everything covered in [README.md](README.md).

| # | Question |
|---|----------|
| 1 | [Why does OOP exist?](#q1-why-does-oop-exist-what-problem-does-it-solve-that-procedural-code-doesnt) |
| 2 | [What is the "everything is an object" mindset?](#q2-what-is-the-everything-is-an-object-mindset-and-is-it-always-the-right-approach) |
| 3 | [State vs Behavior — what's the difference?](#q3-what-is-the-difference-between-state-and-behavior-in-an-object) |
| 4 | [What is "message passing"?](#q4-what-is-message-passing-in-oop) |
| 5 | [What is Responsibility-Driven Design?](#q5-what-is-responsibility-driven-design-briefly) |
| 6 | [Class vs Object — explain simply](#q6-class-vs-object--explain-the-difference-the-way-you-would-to-a-beginner) |

---

<a id="q1-why-does-oop-exist-what-problem-does-it-solve-that-procedural-code-doesnt"></a>

### Q1. Why does OOP exist? What problem does it solve that procedural code doesn't?

> Procedural code organizes a program around **functions operating on shared, often global data** — as a codebase grows, it becomes hard to know which function can change which data, and changes ripple unpredictably.
>
> OOP groups **state** and the **behavior** that operates on it into a single unit (an object), so each unit owns and protects its own data. This makes large systems easier to reason about, extend, and maintain because responsibility is **localized instead of scattered**.

---

<a id="q2-what-is-the-everything-is-an-object-mindset-and-is-it-always-the-right-approach"></a>

### Q2. What is the "everything is an object" mindset, and is it always the right approach?

> It means modeling real-world (or domain) entities as objects that **bundle their own state and behavior**, rather than treating data and logic as separate things floating around.
>
> It's **not always the right approach** — small scripts, pure data transformations, or math-heavy code often read more clearly in a functional/procedural style. OOP earns its cost when a domain has many interrelated entities with rules that need to be protected and reused.

---

<a id="q3-what-is-the-difference-between-state-and-behavior-in-an-object"></a>

### Q3. What is the difference between state and behavior in an object?

> **State** is the data an object holds (its attributes/fields) at any given moment — it answers *"what does this object know?"*
>
> **Behavior** is the set of methods an object exposes — it answers *"what can this object do?"*
>
> Good object design keeps behavior close to the state it operates on, instead of letting external code reach in and manipulate state directly.

---

<a id="q4-what-is-message-passing-in-oop"></a>

### Q4. What is "message passing" in OOP?

> Objects don't reach into each other's internals — they communicate by **calling each other's public methods**, i.e., sending a "message" and getting a response.
>
> This is the mechanism that keeps encapsulation meaningful: an object's internal state stays private, and all interaction happens through its public interface.

---

<a id="q5-what-is-responsibility-driven-design-briefly"></a>

### Q5. What is Responsibility-Driven Design, briefly?

> Instead of starting from *"what data do I need,"* you start from *"who is responsible for this piece of knowledge or this action?"*
>
> You assign responsibilities to objects based on what they naturally should own, which tends to produce classes with **clear, single purposes** rather than classes built around convenient data groupings.

---

<a id="q6-class-vs-object--explain-the-difference-the-way-you-would-to-a-beginner"></a>

### Q6. Class vs Object — explain the difference the way you would to a beginner.

> A **class** is a blueprint — it defines what fields and methods every object of that type will have, but it doesn't hold any real data itself.
>
> An **object** is a concrete instance created from that blueprint, with its own actual values in memory.
>
> You can create many objects from one class, each with independent state.
