# Josué — system design & architecture

How Josué thinks about the *shape* of a system: boundaries, contracts, data flow, where side effects live. This is the design-time companion to `josue-engineering` (the per-task operational rules). When a decision is about system shape, it lives here; when it's about how to execute a concrete task, it lives there.

## Core: transformations & data flow

Heavily influenced by functional programming. A system should behave like a **pipeline**: each step **transforms data** rather than mutating state. This doesn't mean everything is a function — it means **state mutation lives only at the edges** (entrypoints and outpoints: request handlers, cache/DB writes, external calls). The middle is pure transformation.

Model systems around **data flow, not object hierarchies**. Most business logic is data transformation. Rich object graphs tend to hide dependencies and mutation; prefer explicit inputs and outputs over implicit state. When reading code, ask: *what data enters? what transformation occurs? what data leaves?* If that's hard to answer, the abstraction is probably wrong.

When reviewing a design, ask: *where does state get mutated, and is it pushed to the edge?* If mutation is smeared through the logic, the shape is wrong.

**State is a liability.** Every piece of mutable state introduces synchronization problems, ordering concerns, invalid transitions, and debugging difficulty. Prefer immutable data, derived state, and recomputation when it's cheap. Store only what cannot be derived.

## Architecture as compressible contracts

Think in terms of **boxes and lines**. Boxes are responsibilities. Lines are contracts. Implementations live inside boxes. The goal of architecture is not to hide code — it's to create a system that can be understood through its **contracts alone**.

A box should be explainable as:

```text
Input -> Output
```

without requiring knowledge of its internals. If understanding a box requires understanding its implementation, the boundary is weak.

### Compressibility

The primary measure of architectural quality is **compressibility**. A good system can be represented using progressively fewer concepts without losing the ability to reason correctly about it.

```text
Request
  ↓
Identity
  ↓
Booking
  ↓
Payment
  ↓
Response
```

compresses into:

```text
Booking API
```

without losing the understanding required at that level.

**The purpose of abstraction is not reuse — it's compression.** A successful abstraction lets a large amount of behavior be treated as a single concept. A failed abstraction forces the reader to constantly expand its internals in their head.

### Zooming in and out

A system should be understandable at multiple levels. Any box should expand into smaller boxes:

```text
Payment
```

becomes:

```text
Fraud Check
  ↓
Provider Selection
  ↓
Charge Creation
  ↓
Persistence
```

and a group of boxes should collapse back into the larger box. The mental model stays consistent at every level. Zooming in reveals detail; zooming out removes detail. Neither should require rebuilding the understanding of the system. If changing abstraction levels is cognitively expensive, the boundaries are wrong.

### Contracts are more important than implementations

The implementation inside a box is a local concern. The contract a box exposes is an architectural concern. When evaluating a design, focus on what enters a boundary, what leaves it, and what guarantees it provides. The implementation can change entirely without affecting consumers. A boundary is successful when consumers can reason about it entirely through its contract.

### Independent reasoning

The purpose of boundaries is to enable independent reasoning. A developer should be able to understand a box without understanding the whole system, and understand the overall system without understanding every box. Good architecture minimizes the information required to reason correctly at any given level. This is one of the primary goals of architecture.

### Side effects are architectural events

Side effects are fundamentally different from transformations. A transformation changes information (`A -> B`). A side effect changes **reality**: database writes, cache invalidation, network requests, publishing events, sending emails, writing files.

Transformations can be compressed indefinitely. Side effects cannot. The moment reality changes, ordering matters, failures matter, retries matter, timing matters, idempotency matters — the information required to reason correctly increases dramatically. So side effects are architectural events, not implementation details.

This is the architectural root of the operational rule "mutation lives at the edges": most of a system should be pure transformation — data enters, is transformed, a decision is produced — and only then is reality modified. The ideal shape:

```text
Reality
  ↓
Pure Transformations
  ↓
Reality
```

with side effects only at the boundaries. This preserves compressibility: the middle stays easy to reason about because it only transforms information.

### Hidden side effects are broken boundaries

A box should not secretly modify reality. `Calculate Booking` should not unexpectedly write to a database, invalidate a cache, or send an email. Those are architectural relationships invisible in the contract — the visible graph and the real graph diverge, and the abstraction leaks. A boundary should accurately represent the system's interactions with reality.

### When a side effect appears, a boundary may be missing

If a side effect naturally appears in the middle of a flow, that often signals a missing boundary. Instead of:

```text
Request
  ↓
Calculate + Persist
  ↓
Response
```

prefer:

```text
Request
  ↓
Calculate
  ↓
Booking Plan
  ↓
Persist
  ↓
Stored Booking
  ↓
Response
```

The side effect becomes explicit, a new contract appears, and reasoning gets easier. The goal isn't purity for its own sake — it's preserving clear contracts and compressible reasoning.

### Architecture should reflect reality honestly

The visible architecture should match the actual architecture. A diagram of the system should not lie. The contracts visible in the code should correspond to the real dependencies and interactions at runtime. Hidden coupling is architectural debt. The more accurately boundaries represent reality, the easier the system is to understand and evolve.

### Good architecture creates nouns

A healthy subsystem eventually becomes a noun. People stop talking about its internals and start talking about the capability it provides — `Fraud Check / Provider Selection / Charge Creation / Persistence` become `Payment`. The details still exist; they just no longer matter at that level. This is the ultimate goal of abstraction: turning a collection of moving parts into a single concept reasoned about independently.

### Names validate abstractions

Every box, and every level of abstraction, should have a name. Naming is not cosmetic post-design work — it's one of the primary ways to validate whether an abstraction actually exists. If a responsibility cannot be named clearly, its boundaries are probably unclear.

When boxes collapse into a larger box, the result should have a meaningful name:

```text
Fraud Check
Provider Selection
Charge Creation
Persistence
```

should compress into `Payment`, not `PaymentManagerService`. The strongest names describe **concepts**; weak names describe implementations. Prefer names that reflect business capabilities, domain concepts, and responsibilities. Avoid names that merely describe where code lives, technical mechanisms, or generic implementation roles. As you zoom out, names should align increasingly with the business domain. A reader should be able to understand the shape of the system by reading the names of its boxes and the contracts between them.

### Architectural smell: inability to compress

A design is suspicious when:

- understanding a box requires understanding neighboring boxes
- contracts are unclear
- side effects are hidden
- zooming out loses important information
- zooming in changes the meaning of the system
- the same concept appears in multiple places
- boundaries leak implementation details

These are signs the system cannot be compressed into stable concepts. When this happens, revisit the boundaries — the problem is usually architectural before it is implementation-related.

## Simplicity is not code golf

- Prefer the solution with the fewest **concepts**, not the fewest lines.
- Complexity comes from the number of things a reader must hold in their head simultaneously.
- Every abstraction should remove complexity from the caller. If it merely relocates complexity, it failed.
- A new layer, interface, or helper carries a permanent maintenance cost. Justify it.
- **Indirection is only useful when it hides volatility.** Good: an interface separating stable business logic from changing providers. Bad: a wrapper around a single implementation with no realistic alternative. Before introducing indirection, identify what changes, how often it changes, and who benefits.

## Abstraction & the axis of variation

- **Model the real axis of variation.** Structure code around the thing that actually varies and will grow, not around the specific instances that happen to exist today. Adding the next case should mean adding a unit, not editing everything.
- **Optimize for change, not prediction.** Design for the changes you know are likely; avoid building extension points for hypothetical futures. Generalization should be *extracted from multiple concrete cases*, not imagined in advance — the next implementation should validate the abstraction, not be required to justify it.
- **Reconciling the two** (they sound opposed, they aren't): model the axis that is *already varying in front of you* — the concrete second or third case has appeared. Don't build the axis for a case you only *imagine* might come. "Axis of variation" is about recognizing real, present variation and not hardcoding around it; "optimize for change, not prediction" is about not inventing variation that doesn't exist yet. Known axis → structure for it. Imagined axis → wait for the concrete case. Ask *"what change do we expect?"*, never *"what might happen someday?"* (Concretely: supporting an object-attribute path for inputs that are always dicts is imagined flexibility — delete it; add it when a concrete caller passes objects.)
- **Push a "which one?" choice into the data, not the evaluator.** When a pure transform needs to know which field, key, or threshold to use, let the input declare it rather than hardcoding it inside. An engine that assumes `occurred_at` is coupled to one shape; a policy that declares `field: occurred_at` keeps the engine general and makes the choice explicit and auditable. Don't assume — let the contract say. This is the axis of variation applied to a single decision: the varying thing becomes data, not a constant buried in the logic.

## Architecture emerges from pressure

Don't introduce architecture because architecture feels correct. Introduce it when there's observed friction: duplication creates friction, coupling creates friction, change creates friction. Architecture should solve observed pressure, not anticipated elegance.

## Business rules are the center of the system

Infrastructure exists to support business logic. Databases, frameworks, queues, and APIs are all implementation details. Business rules are what survive rewrites. Organize systems so business logic stays isolated from infrastructure concerns, and so dependencies point inward (toward the business rules), not outward.

**Persistence identity is not a domain concern.** Keep DB-assigned ids, row versions, and storage timestamps out of domain models — those carry domain logic only. Attach identity at the persistence boundary (e.g. a `DBPolicy` wrapper in the service layer that pairs a domain `Policy` with its row id). A domain object should be constructible and meaningful before it has ever been persisted; baking in an `id` forces a row to exist for a value that doesn't need one, and points the domain outward at the database instead of inward.

## Explicit over magical

Avoid mechanisms that make behavior difficult to trace. Prefer explicit configuration, explicit wiring, explicit data flow. Be cautious with reflection, hidden registration, implicit dependency injection, metaprogramming, and framework magic. Debugging begins where predictability ends.

## Dependencies are architectural decisions

Every dependency introduces upgrade cost, operational risk, security surface, and cognitive load. Before adding one, ask: is it solving a real problem? is it reducing complexity overall? could we reasonably own this ourselves? A dependency should buy more simplicity than it costs.

## Signs of a healthy design

A healthy design tends to exhibit these properties:

- business logic is easy to locate
- data flow is obvious
- dependencies point inward
- failures are visible
- modules can be understood independently
- new cases are added more often than existing code is modified
- deleting code feels safe
- behavior is easier to explain than the implementation

If a design cannot be explained simply, there is usually a deeper misunderstanding hiding underneath.

## Tradeoff awareness

There are no universally correct engineering decisions. Every decision exchanges one thing for another: flexibility for simplicity, performance for clarity, safety for velocity, abstraction for explicitness. **Make tradeoffs explicit** — hidden tradeoffs become future surprises.
