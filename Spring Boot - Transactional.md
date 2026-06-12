## What is `@Transactional`?

At its core, `@Transactional` is a Spring annotation used to declaratively manage database transactions. Instead of writing boilerplate code to manually open, commit, or roll back a transaction using JDBC or Hibernate API, Spring allows us to use this annotation to handle transaction boundaries automatically using **AOP (Aspect-Oriented Programming)** proxies.

## What Does It Actually Do?

When you apply `@Transactional` to a class or a method, Spring wraps that bean in a proxy. This proxy intercepts the method call and executes the following sequence:

- **Opens a Transaction:** Before the method code even starts executing, the proxy opens a database connection and begins a transaction.
- **Maintains the Session:** It keeps the persistence context (like a Hibernate session) alive for the entire duration of that method.
- **Commits on Success:** If the method completes successfully without throwing an unchecked exception, the proxy automatically commits the transaction.
- **Rolls Back on Failure:** If a runtime exception occurs, it triggers an automatic rollback, ensuring that all database operations within that method fail or succeed as a **single atomic unit**.

## Architectural Best Practices

A key part of understanding `@Transactional` is knowing *where* it belongs in a multi-layered application:

1. The Service Layer is the "Sweet Spot"

Transactions should almost always live in the **Service Layer**. This is where business logic is orchestrated. A single business transaction often requires multiple database operations (e.g., saving a user, assigning a role, and creating an audit log). Placing the annotation at the service level ensures that if any of those steps fail, the entire operation rolls back, preventing inconsistent database states.

2. Avoid Putting It on the Repository Layer

Putting `@Transactional` on repository interfaces is a common anti-pattern. Repositories are low-level data access blocks that should only care about CRUD operations.

- **It's redundant:** Spring Data JPA already automatically wraps standard CRUD methods (like `save()` or `delete()`) in transactions under the hood using optimized `readOnly = true` defaults.
- **It breaks flexibility:** Hardcoding transactions at the repository level makes it incredibly difficult to orchestrate multiple repository calls inside a single service transaction or manage complex transaction propagation (like `REQUIRED` vs `REQUIRES_NEW`).

3. Avoid Putting It on the Controller Layer

Controllers deal with HTTP request/response handling. Putting `@Transactional` here keeps database connections open far too long, drastically reducing application throughput and performance.

## Summary of Placement

| Layer | Should have `@Transactional`? | Why? |
| --- | --- | --- |
| **Controller** | ❌ No | Keeps HTTP connections open too long; business logic shouldn't live here. |
| **Service** | Yes | This is where business transactions are defined and orchestrated. |
| **Repository** | ❌ No | Too low-level; Spring Data handles basic CRUD transactions automatically. |

Ultimately, `@Transactional` is not just a tool to open database connections, but as an architectural boundary marker that defines a strict business unit of work.

---

