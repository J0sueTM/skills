---
name: josue-architecture
description: How Josué thinks about system design and architecture — compressible contracts, boxes/lines, data flow, pushing side effects to the edges, the axis of variation, dependencies as decisions, business rules at the center, and what a healthy design looks like. Apply when designing a system, defining module boundaries, refactoring structure, reviewing architecture, choosing where side effects live, naming abstractions, or deciding whether to add a layer/dependency. For per-task execution rules (errors, tests, commits, reviews, communication), see josue-engineering.
---

# Josué — system design & architecture

> Source of truth = `SOURCE.md` (full prose). This file = caveman build. Edit SOURCE first, rebuild here.

How Josué think about *shape* of system: boundaries, contracts, data flow, where side effects live. Design-time companion to `josue-engineering` (per-task operational rules). Shape decision → here. Execute-a-task decision → there.

## Core: transformations & data flow

FP brain. System = **pipeline**. Each step **transform data**, no mutate state. Not everything fn — **state mutation only at edges** (req handlers, cache/DB writes, external calls). Middle = pure transform.

Model around **data flow, not object hierarchies**. Most business logic = data transform. Rich object graphs hide deps + mutation → prefer explicit in/out over implicit state. Read code → ask: *what data enters? what transform? what data leaves?* Hard answer → abstraction wrong.

Review design → ask: *where state mutated, pushed to edge?* Smeared thru logic → shape wrong.

**State = liability.** Mutable state → sync / ordering / invalid transitions / debug pain. Prefer immutable, derived state, recompute when cheap. Store only what can't be derived.

## Architecture = compressible contracts

Think **boxes + lines**. Box = responsibility. Line = contract. Impl live inside box. Goal not hide code — make system understandable thru **contracts alone**.

Box explainable as:

```text
Input -> Output
```

no need know internals. Need internals to understand box → boundary weak.

### Compressibility

Primary measure of arch quality = **compressibility**. Good system represented w/ progressively fewer concepts, no lose ability to reason correct.

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

compress to:

```text
Booking API
```

no lose understanding needed at that level.

**Purpose of abstraction not reuse — compression.** Good abstraction → big behavior treated as one concept. Failed abstraction → reader constantly expand internals in head.

### Zoom in/out

System understandable at multiple levels. Any box expand to smaller boxes:

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

and group collapse back to bigger box. Mental model consistent every level. Zoom in = reveal detail. Zoom out = remove detail. Neither rebuild understanding. Changing abstraction level cognitively expensive → boundaries wrong.

### Contracts > implementations

Impl inside box = local concern. Contract box expose = arch concern. Eval design → focus what enters boundary / what leaves / what guarantees. Impl change entirely, no affect consumers. Boundary success = consumers reason about it entirely thru contract.

### Independent reasoning

Boundaries exist to enable independent reasoning. Understand box without whole system. Understand whole system without every box. Good arch minimize info needed to reason correct at any level.

### Side effects = architectural events

Side effect fundamentally ≠ transform. Transform change information (`A -> B`). Side effect change **reality**: DB writes, cache invalidation, network req, publish events, send email, write files.

Transforms compress indefinitely. Side effects no. Moment reality change → ordering / failures / retries / timing / idempotency all matter. Info to reason correct jump up. So side effects = arch events, not impl details.

Arch root of "mutation at edges": most of system = pure transform. Data enters, transformed, decision produced — only then modify reality. Ideal shape:

```text
Reality
  ↓
Pure Transformations
  ↓
Reality
```

side effects only at boundaries. Preserve compressibility — middle easy to reason, only transform info.

### Hidden side effects = broken boundaries

Box no secretly modify reality. `Calculate Booking` no unexpectedly write DB / invalidate cache / send email. Those = arch relationships invisible in contract → visible graph + real graph diverge → abstraction leak. Boundary must accurately represent system interactions w/ reality.

### Side effect in middle → missing boundary

Side effect natural in middle of flow → often signal missing boundary. Instead of:

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

Side effect explicit, new contract appears, reasoning easier. Goal not purity for own sake — preserve clear contracts + compressible reasoning.

### Architecture reflect reality honestly

Visible arch match actual arch. Diagram no lie. Contracts in code correspond to real deps + interactions at runtime. Hidden coupling = arch debt. More accurate boundaries → easier understand + evolve.

### Good architecture creates nouns

Healthy subsystem eventually = noun. People stop talk internals, start talk capability — `Fraud Check / Provider Selection / Charge Creation / Persistence` → `Payment`. Details still exist, just no matter at that level. Ultimate goal of abstraction: moving parts → single concept reasoned independently.

### Names validate abstractions

Every box + every abstraction level = a name. Naming not cosmetic post-design — primary way to validate abstraction actually exist. Responsibility can't be named clear → boundaries probably unclear.

Boxes collapse → result needs meaningful name:

```text
Fraud Check
Provider Selection
Charge Creation
Persistence
```

compress to `Payment`, not `PaymentManagerService`. Strongest names describe **concepts**. Weak names describe impl. Prefer names reflecting business capability / domain concept / responsibility. Avoid names describing where code lives / technical mechanism / generic impl role. Zoom out → names align increasingly with business domain. Reader understand shape of system by reading box names + contracts.

### Smell: inability to compress

Design suspicious when:

- understand a box need understand neighbor boxes
- contracts unclear
- side effects hidden
- zoom out lose important info
- zoom in change meaning of system
- same concept appears in multiple places
- boundaries leak impl details

Signs system can't compress into stable concepts. → revisit boundaries. Problem usually architectural before implementation.

## Simplicity ≠ code golf

- Fewest **concepts**, not fewest lines.
- Complexity = count of things reader hold in head at once.
- Abstraction must remove complexity from caller. Only relocate it → failed.
- New layer/interface/helper = permanent maintenance cost. Justify.
- **Indirection useful only when hides volatility.** Good: interface separating stable business logic from changing providers. Bad: wrapper round single impl, no real alternative. Before add → identify what changes / how often / who benefits.

## Abstraction & axis of variation

- **Model real axis of variation.** Structure around thing that varies + grows, not today's instances. Next case = add unit, not edit everything.
- **Optimize for change, not prediction.** Design for likely changes. No extension points for hypothetical futures. Generalization *extracted from multiple concrete cases*, not imagined ahead — next impl validate abstraction, not justify it.
- **Reconcile the two** (sound opposed, aren't): model the axis *already varying in front of you* — concrete 2nd/3rd case appeared. No build axis for case you only *imagine*. Axis-of-variation = recognize real present variation, no hardcode around it. Optimize-for-change = no invent variation that doesn't exist yet. Known axis → structure for it. Imagined axis → wait for concrete case. Ask *"what change we expect?"* never *"what might happen someday?"* (Concrete: object-attribute path for inputs always dicts = imagined flexibility, delete; add when concrete caller passes objects.)
- **Push "which one?" choice into data, not evaluator.** Pure transform needs which field/key/threshold → input declares it, no hardcode inside. Engine assume `occurred_at` = coupled to one shape; policy declare `field: occurred_at` = engine stays general, choice explicit + auditable. No assume — contract say. Axis of variation applied to single decision: varying thing = data, not constant buried in logic.

## Architecture emerges from pressure

No add architecture because feels correct. Add on observed friction: duplication / coupling / change creates friction. Architecture solve observed pressure, not anticipated elegance.

## Business rules = center

Infra exist to support business logic. DB / framework / queue / API = impl details. Business rules survive rewrites. Organize so business logic isolated from infra, deps point inward (toward business rules), not outward.

**Persistence identity ≠ domain concern.** DB-assigned ids, row versions, storage timestamps → out of domain models (domain logic only). Attach identity at persistence boundary (e.g. `DBPolicy` wrapper in service layer pairing domain `Policy` + row id). Domain object constructible + meaningful before ever persisted; baking in `id` forces row to exist for value that doesn't need one, points domain outward at DB instead of inward.

## Explicit > magical

Avoid mechanisms that make behavior hard to trace. Prefer explicit config / wiring / data flow. Cautious with: reflection, hidden registration, implicit DI, metaprogramming, framework magic. Debugging begins where predictability ends.

## Dependencies = architectural decisions

Every dep → upgrade cost / op risk / security surface / cognitive load. Before add: real problem? reduce complexity overall? could own ourselves? Dep must buy more simplicity than it cost.

## Signs of healthy design

- business logic easy to locate
- data flow obvious
- dependencies point inward
- failures visible
- modules understood independently
- new cases added more often than existing code modified
- deleting code feels safe
- behavior easier to explain than impl

Design can't be explained simply → usually deeper misunderstanding hiding underneath.

## Tradeoff awareness

No universally correct decision. Every decision trade: flexibility↔simplicity, performance↔clarity, safety↔velocity, abstraction↔explicitness. **Make tradeoffs explicit** — hidden tradeoff = future surprise.
