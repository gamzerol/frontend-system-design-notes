# What is Frontend System Design?

Frontend system design is the practice of making **intentional, scalable architectural decisions** for client-side applications — covering how components are structured, how data flows, how state is managed, and how the system evolves over time. It's distinct from UI implementation: you're not writing components, you're designing the system that components live in.

---

## Why It's Different from Backend System Design

Backend system design focuses on servers, databases, APIs, and distributed infrastructure. Frontend system design shares some vocabulary but the concerns are fundamentally different.

| Concern         | Backend                      | Frontend                                       |
| --------------- | ---------------------------- | ---------------------------------------------- |
| Scalability     | Requests per second, DB load | Bundle size, render performance, team scale    |
| State           | Database, cache              | URL, memory, server cache, UI state            |
| Failure modes   | Service downtime, data loss  | Blank screens, stale data, broken interactions |
| Deployment unit | Service / container          | JavaScript bundle, HTML, assets                |
| "User"          | Other services / APIs        | Human beings with browsers                     |

Frontend scale is often **organizational** — the hardest problems appear when 10+ engineers work on the same codebase, not necessarily when you have millions of users.

---

## What Gets Designed in a Frontend System

A complete frontend system design covers six layers:

```
┌─────────────────────────────────────┐
│           Rendering Layer           │  CSR / SSR / SSG / ISR
├─────────────────────────────────────┤
│          Component Layer            │  Design system, composition
├─────────────────────────────────────┤
│          State Layer                │  Local, global, server state
├─────────────────────────────────────┤
│          Data Layer                 │  API contracts, caching, sync
├─────────────────────────────────────┤
│          Infrastructure Layer       │  Build, CI/CD, hosting, CDN
├─────────────────────────────────────┤
│          Cross-cutting Concerns     │  Auth, i18n, a11y, observability
└─────────────────────────────────────┘
```

Each layer has its own design decisions. A good system design explicitly states _what_ decisions were made at each layer and _why_.

---

## Frontend System Design in Interviews

In an interview, you will typically be asked one of two question types:

**Component Design** — "Design a reusable data table component."

- Focus: API design, props interface, internal state, edge cases.
- Output: Component structure, usage examples, trade-offs.

**System Design** — "Design the frontend architecture for an e-commerce checkout flow."

- Focus: Routing, state management, data fetching, performance, error handling.
- Output: Architecture diagram, technology choices, data flow, trade-offs.

### The RADIO Framework (for interviews)

A useful structure for answering system design questions:

| Step              | What you do                                         |
| ----------------- | --------------------------------------------------- |
| **R**equirements  | Clarify functional and non-functional requirements  |
| **A**rchitecture  | High-level components and how they connect          |
| **D**ata model    | What data exists, where it lives, how it flows      |
| **I**nterface     | API contracts between components / with the backend |
| **O**ptimizations | Performance, scalability, edge cases                |

> **Interview tip:** Always start by asking clarifying questions. "Is this a mobile-first experience?" or "How many concurrent users?" changes everything.

---

## The Trade-off Triangle

Every frontend architecture decision lives somewhere in this triangle. Moving toward one corner pulls you away from the others.

```
         Performance
             △
            / \
           /   \
          /     \
         /       \
Simplicity ——————— Maintainability
```

**Example:** Aggressive caching improves performance but adds complexity and makes the codebase harder to maintain. Choosing a simple prop-drilling pattern is easy to maintain but doesn't scale as the app grows.

A good system design doesn't aim for a perfect balance — it **consciously prioritizes** based on the product's needs, and documents why.

---

## How to Think About Scale

"Scale" in frontend means different things depending on context:

**User scale** — How many concurrent users? This affects CDN strategy, bundle size budgets, and rendering strategy (SSR vs CSR).

**Feature scale** — How many features will this app have in 2 years? This affects folder structure, code splitting strategy, and state management choice.

**Team scale** — How many engineers? This is often the most important one. A modular architecture with clear boundaries matters more at team-of-20 than team-of-2.

**Data scale** — How much data does the UI render? 100 rows vs 100,000 rows requires completely different component strategies (virtualization, pagination, etc.).

> Scaling a frontend is mostly about **keeping complexity manageable** as the codebase grows — not about handling traffic spikes (that's the backend's job).

---

## Architecture Decision Records (ADRs)

A practical habit: document significant decisions as ADRs. Each ADR captures:

- **Context** — What situation forced a decision?
- **Decision** — What was chosen?
- **Rationale** — Why?
- **Consequences** — What trade-offs does this introduce?

```markdown
# ADR-001: Use Zustand instead of Redux for global state

## Context

We need global state for auth and cart. The team is 3 people and
the app is small-to-medium scale.

## Decision

Use Zustand.

## Rationale

Redux adds significant boilerplate (actions, reducers, selectors)
that isn't justified at this scale. Zustand provides the same
capabilities with a much simpler API.

## Consequences

- Less boilerplate, faster iteration
- Fewer conventions — team must agree on store structure manually
- If the app grows significantly, migration to Redux may be needed
```

ADRs are valuable because decisions that seem obvious today are mysterious 6 months later.

---

## Key Mental Models

**1. Separate "what" from "where"**
What data exists vs where it lives are different questions. A shopping cart item is the same entity whether it's in server state, local state, or a URL param — but each location has different trade-offs.

**2. Push complexity down, not up**
Generic infrastructure (API clients, error boundaries, auth guards) should live low in the system. Business logic should live close to where it's used.

**3. Design for deletion**
Good architecture makes it easy to remove a feature. If deleting one feature requires touching 20 files across the codebase, the coupling is too tight.

**4. Boring is good**
Choose well-understood patterns over clever ones. The most interesting problems in frontend are hard enough — your architecture shouldn't add more.
