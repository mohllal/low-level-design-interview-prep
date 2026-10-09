# Low Level Design Interview Preparation

> [!NOTE]
> This is part of my personal interview prep. I also have separate repos for:
>
> - [Coding](https://github.com/mohllal/coding-interview-prep)
> - [High-level design](https://github.com/mohllal/high-level-design-interview-prep)

These are my personal notes and reference materials to help prepare for low-level, object-oriented design interviews.

## Contents

| Note                                                              | Description                                                                                                                                                                                             |
|-------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [01 — Interview playbook](01-interview-playbook.md)               | Phase-by-phase runbook with time boxes, what interviewers score, and an end-of-round checklist. The note to re-read the day of an interview.                                                            |
| [02 — Requirements and scoping](02-requirements-and-scoping.md)   | Clarifying question banks, actors and use cases, drawing the in-scope line, and which non-functional constraints belong at this level.                                                                  |
| [03 — Object modeling](03-object-modeling.md)                     | Noun and verb extraction, CRC cards, responsibility assignment, entities vs value objects vs services, god objects, anemic models, and naming.                                                          |
| [04 — OOP core concepts](04-oop-core-concepts.md)                 | Encapsulation, abstraction, inheritance, and polymorphism stated precisely enough to defend. Dispatch, interface vs abstract class, access modifiers.                                                   |
| [05 — Class relationships](05-class-relationships.md)             | Association, aggregation, composition, dependency, generalization, and realization — what each implies about lifetime and ownership. Multiplicity and composition over inheritance.                     |
| [06 — UML class diagrams](06-uml-class-diagrams.md)               | The notation to produce from memory, how much detail is enough under time pressure, and the mistakes that cost credibility.                                                                             |
| [07 — UML behavioral diagrams](07-uml-behavioral-diagrams.md)     | Sequence diagrams for flows, state diagrams for lifecycles, activity diagrams for branching processes, and when each is worth the time.                                                                 |
| [08 — Coupling and cohesion](08-coupling-and-cohesion.md)         | Cohesion and coupling taxonomies, the Law of Demeter, and connascence as a sharper vocabulary for what must be kept in sync.                                                                            |
| [09 — Domain modeling](09-domain-modeling.md)                     | Entities, value objects, aggregates and the invariants they protect, repositories, domain services, factories, and persistence boundaries.                                                              |
| [10 — General design heuristics](10-general-design-heuristics.md) | DRY, KISS, YAGNI, composition over inheritance, program to an interface, encapsulate what varies, Tell Don't Ask, and the over-engineering trap.                                                        |
| [11 — SOLID principles](11-solid-principles.md)                   | SRP, OCP, LSP, ISP, and DIP — each with a violation, the fix, the failure mode of over-application, and practical material on dependency injection and testability.                                     |
| [12 — GRASP principles](12-grasp-principles.md)                   | Information Expert, Creator, Controller, Low Coupling, High Cohesion, Polymorphism, Pure Fabrication, Indirection, and Protected Variations — the heuristics that answer "which class should own this?" |
| [13 — Concurrency](13-concurrency.md)                             | Shared mutable state and race conditions, Python's threading model and the GIL, locks, thread-safe class design, optimistic vs pessimistic locking, and idempotency.                                    |

## Books

*Not all of these have been read cover to cover — this list mixes books I have read, partially read, or heard consistently good things about.*

- *Domain-Driven Design: Tackling Complexity in the Heart of Software* — Eric Evans
- *Applying UML and Patterns: An Introduction to Object-Oriented Analysis and Design and Iterative Development* — Craig Larman
- *Patterns of Enterprise Application Architecture* — Martin Fowler
- *Refactoring: Improving the Design of Existing Code* — Martin Fowler
- *Design Patterns: Elements of Reusable Object-Oriented Software* — Gamma, Helm, Johnson & Vlissides
- *Clean Architecture: A Craftsman's Guide to Software Structure and Design* — Robert C. Martin
- *The Pragmatic Programmer: Your Journey to Mastery* — Andrew Hunt & David Thomas

## Contributing

This repository reflects personal learning and interview preparation. It is primarily for personal use, but suggestions and improvements are welcome through issues and pull requests.
