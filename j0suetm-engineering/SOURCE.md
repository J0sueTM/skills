# Josué — engineering process & preferences

How Josué works on a concrete task: decisions, errors, tests, code organization, reviews, commits/PRs, communication. These are operational rules — they change what you do on a given diff.

For decisions about *system shape* — boundaries, contracts, data flow, where side effects live, whether to add a layer or dependency — see the **`josue-architecture`** skill. That's the design-time companion to this one.

**⚡ = non-default.** Marks the rules most likely to be gotten wrong by default. They're where this skill earns its keep — apply them deliberately.

## Working shape (pointer)

Keep the middle of any unit pure: data in → transform → decision out. Put side effects (DB/cache writes, network calls, emails, event publishing) only at the edges. The full architectural reasoning behind this — compressible contracts, side effects as architectural events — lives in `josue-architecture`.

## Decision-making

- **Ask before concluding.** Surface the real forks as explicit questions and let him decide. Don't bury a big architectural choice inside an implementation detail, and don't present a finished assumption when there was a genuine choice to make.
- **Understand intent before accepting or rejecting.** When something looks off he asks *"what are you trying to achieve / why this way?"* before deciding. Reciprocate: explain the *why*, and drop the idea if the why turns out weak.
- **Reasoning beats prior verdicts — for both sides.** He changes his mind when a better argument appears, and expects you to as well. Track the reasoning, not the earlier conclusion. A "no" can become a "yes" the moment the justification is clear.
- **Push back when you have a real reason.** If asked to remove something that's actually needed, keep it and *say why*. He prefers a correct objection over silent compliance.
- **⚡ Let the evidence locate the fault before changing anything.** When a fix is proposed that the error contradicts, push back with the specific line that rules it out instead of trying it. A TCP `Connection timed out` is a network drop, not an auth problem — so switching the SSH user can't help, and the log proves it before any credential is sent. Quote the symptom that disqualifies the guess; don't swap components on a hunch when the error already says where the fault is.
- **⚡ Verify facts against the source; don't guess.** For a column name, an enum value, or an API field, check the real source of truth rather than assuming a plausible name. A guessed identifier is a latent bug. (This session: the aiosql returning operator is `<!`, not the `<!>` I assumed — confirmed by reading the adapter source, not by trying it.)
- **⚡ Don't depend on serialization-format ordering for meaning.** JSON objects are unordered, and `jsonb` actively reorders keys (by length, then bytes), so anything loaded from storage cannot rely on declaration order. When order encodes meaning — a dependency chain between steps — derive it from the data's relationships (resolve each step from its declared input), not from the order the keys happen to appear in. A value round-tripped through storage is the moment a hidden order-dependency surfaces as a bug.
- **Make tradeoffs explicit.** Every real decision exchanges something (flexibility/simplicity, speed/clarity, safety/velocity). State the tradeoff — hidden tradeoffs become future surprises.

## Naming

- **⚡ Business-logic names belong in the logic, not in constants.** A constant is for config and stable magic values — not for encoding a business rule someone has to go read somewhere else.
- **Plain public names** for things that are part of a module's surface; reserve the private prefix for what's actually internal. He pushes back hard on a wall of `_`-prefixed helpers — most of them want to be inlined, not hidden.
- **No needless indirection.** Inline something used once or twice instead of wrapping it in a helper. Prefer the tighter version; remove ceremony. (When indirection *is* warranted is a design call — see `josue-architecture`.)
- **Lookup helpers read `by_<field>`, not `for_`.** `fetch_alarms_by_event`, not `fetch_alarms_for_event` — the name reads as the query it runs.

## Code organization

- **One responsibility per file; group as a package.** Contracts/types in their own file, one implementation per file, a thin entry module as the public surface. Consistent suffix conventions for module roles.
- **⚡ Name a file for what it produces, not the mechanism that produces it.** A service that creates speed-alarm decisions is `speed_alarm_decision` — not `speed_policy` (a thing it uses) and not `policy_run` (how it runs). Anything that doesn't feed that produced result moves to its own box: the decision and alarm DB writes became their own `decision` and `alarm` services, named for what *they* produce. Naming by the end result keeps cohesion honest and anticipates the axis of variation — when the next kind of alarm appears, it gets its own file instead of swelling this one. (Separate the real *stages* — find / run / save / an orchestrator on top — but don't shred them into sub-helpers below that; the unit of extraction is a responsibility, not a few lines.)
- **Local reasoning.** Code should be understandable without navigating the whole codebase. Prefer explicit dependencies over hidden ones; pass data over reaching into global state; minimize the files required to understand a feature. A reader should be able to explain a module after reading only that module.
- **⚡ Avoid module-level globals when they can be inlined or localized.** Operator-lookup tables, single-convention constants, frozen fixtures — they add names to scan before reaching the logic without buying anything. A six-entry op table plus its imports collapses into a `match`; a one-field timestamp convention becomes a literal at its use site, or better, moves into the data the function operates on. Reach for a module global only when the value is genuinely shared config or an expensive thing built once.
- **⚡ Flatten: small functions and guard clauses over deep nesting.** Watch the indentation depth. A body that nests `for → if → if/try` should split the inner step into a named helper, fold a miss into a sentinel/guard, or become a comprehension. But the cut must *remove* what the reader holds, not relocate it: extract a helper when it kills nesting or names a real step; inline it when it's pure ceremony (a one-use wrapper, a trivial pass-through). The test is cognitive load, not line or name count.
- **⚡ Build by transformation, not accumulation.** Prefer a comprehension or a recursion that *returns* the value over a loop that mutates an accumulator (`append`, then `del` from a pending set). Resolving an order of dependent steps is a recursion on each step's declared input, not a mutate-until-stable loop over a worklist. When he says "simpler, but no mutation," the pure version is usually also the shorter one — once you stop reaching for the loop.
- **Protocols/interfaces over base classes** for polymorphism. Implementations are **duck-typed** — they satisfy the contract without inheriting machinery. Sharing a *type* is fine; sharing a base class to get behavior usually isn't.
- **Circular imports are a design smell.** If avoiding one needs import-ordering tricks, the structure is wrong — extract shared contracts to a leaf module. A primitive shared by two stages goes in a shared leaf so the stages don't import each other. But a cross-module dependency that reflects a *real* relationship — a filter that evaluates conditions over items genuinely depends on the conditions stage — is meaningful, not accidental coupling; don't extract it just to flatten the graph.
- **Follow existing patterns before inventing.** Reuse > Evolve > Create. But "what's already there" still loses to a clearly-better approach — convention is a default, not a cage.
- **Prefer the existing util/decorator in its canonical home.** Use the project's own mechanism (e.g. a memoize decorator) over hand-rolling cache get/set. When a small helper is shared, put it in the established shared module (e.g. the date utils), not a feature-local file — check there first.
- **Make the value serializable; don't hand-roll a serializer.** If a result must be persisted or sent, push the capability into the type — a pydantic model serializes itself via `model_dump_json` — instead of writing a `_serialize_result` that walks it field by field. A hand-rolled walker drifts from the shape it serializes; the model can't. Same instinct as reusing the canonical util: don't reimplement what the framework or the type system already gives you.
- **Comment the *why* for non-obvious structure.** If you split or add something for a reason invisible from the code (e.g. a function that exists only to keep failures out of the cache), say why. The shape should explain its own intent.
- **Document non-obvious helpers, not just the public surface.** What's obvious to you — because you hold the whole flow in your head — can be opaque to someone new to the codebase. Give intent-stating docstrings to helpers that carry domain meaning (e.g. what *anchor* means for a time window: the instant the window is measured back from). State the intent, not a restatement of the code.
- **⚡ A docstring or comment earns its place only by saying something the code can't — apply the test subtractively.** The companion to the rule above: if a line restates what the signature and body already show (`save_alarm` → "grava o alarme e devolve o id"), delete it, even on the public surface. Keep only the lines carrying the invisible *why*: why a fallback exists (`ON CONFLICT DO NOTHING` returns no id on a redelivery), why a failure is isolated (one bad policy must not sink the others), what a window is anchored to. A restated docstring is noise that ages into a lie when the code moves on without it.
- **⚡ Code is the primary source of understanding.** When code is hard to follow, fix its structure and expressiveness *first* — reordering functions or adding docs is treating the symptom. (The architectural side — making stage boundaries structural — is in `josue-architecture`.)
- **No `__all__` / re-export curation.** Import each symbol from the module that defines it. A package's `__init__` holds the composition and the surface it actually builds (e.g. `apply_policy`, `Decision`) — not a re-export list of its submodules' names.

## Error handling

- **Fail loud at the edge.** A misconfiguration or missing dependency should raise and be visible, not be silently swallowed into an empty/default result.
- **Graceful degradation is a deliberate, separate decision** — made one level up where it's a product choice, not baked into a low-level helper.
- **⚡ Don't cache failures.** When caching sits in front of a fallible call, structure it so only successes are cached and errors propagate to the edge that degrades — otherwise a transient failure gets served for the whole TTL. Cache the fetch; handle the error outside it.
- **Bound external calls** (timeouts) so a slow dependency can't hang the system.
- **Context-specific exception subclasses under a shared base.** Prefer `EngineOutputError(EngineError)` over one catch-all, so callers catch broadly (the base) or narrowly (the step). Introduce the base first; specialize per step as the steps appear.

## Ownership of invariants

Every invariant should have one obvious owner. Bad: validation duplicated across handlers, services, and the database layer. Good: validation enforced at the boundary where the invariant is introduced. Ask *"which module is responsible for guaranteeing this property?"* — if the answer is "multiple places", ownership is unclear.

## Performance

- Measure before optimizing.
- Optimize bottlenecks, not aesthetics.
- Prefer removing work over making work faster.
- Algorithmic improvements beat micro-optimizations.

## Operational thinking

Code is only one phase of a system's life. Consider observability, debugging, deployment, rollback, failure modes, and maintenance burden. The best implementation is often the one easiest to diagnose at 2 AM.

## Testing

- **One test file per unit, mirroring the code structure.** Not one giant file.
- **Mock at the boundary**, exercise the real transformation logic.
- **Keep concerns in their home.** A test about a shared/infra helper lives with that helper, not smuggled into a feature's test suite.
- **Explicit per-test setup** over module-level magic.
- **⚡ Don't pin tests to a fixed point in time when the logic is relative.** Use real `datetime.now()` — relative windows (last 15 minutes, since-event) stay deterministic no matter when the test runs. A frozen `NOW = datetime(2026, …)` constant is a fake reference and a module global that buys nothing here. Pin the clock only when an *absolute* instant is genuinely under test.
- Respects the project's testing rules: prefer unit-on-the-logic over end-to-end-through-the-edge, exact assertions, parametrize over copy-paste, don't test the framework.

## Review philosophy

When reviewing code:

1. Understand the problem.
2. Understand the constraints.
3. Evaluate whether the solution addresses the problem.
4. Only then discuss implementation details.

Don't optimize code that solves the wrong problem. The most valuable review comments are often about system shape, not syntax.

## Git & PRs

- **⚡ Commits authored by him**, signed (`-S`). **Never** add LLM co-authorship.
- Conventional Commits. Subject + a short body that explains the *why* and the one big design decision — not a changelog of files.
- **⚡ PRs focus on *why* over *how*** — the how is the code. Write for the reader, not the implementation:
  - State the problem and who it hurts.
  - Explain *why this approach*, and the **big** decisions that led there. Skip minor technical choices.
  - The reader's brain isn't yours — don't assume they'll make the same logical jumps. But don't condescend either.
  - Flag the one or two things a reviewer should actually weigh in on.
- Writes PRs in the team's working language.

## Communication

- Terse and direct, no fluff. Corrections are short and precise — read them as instructions, not suggestions.
- He'll state his mental model when it matters. Capture it and apply it forward, beyond the line in front of you.

### Communication as model transfer

The goal of communication is not to state conclusions. The goal is to transfer the reasoning model that produced them.

- Do not jump directly to the answer if the reader cannot reconstruct the path that led there.
- Explain the system of forces, constraints, and tradeoffs that produced a decision.
- Prefer exposing the reasoning chain over defending the implementation.
- Distinguish clearly between observations, assumptions, constraints, and conclusions.
- If a conclusion depends on hidden context, surface the context.

### Structure before detail

When communicating something complex:

1. Start with the top-level problem.
2. Introduce the major abstractions or components involved.
3. Explain how they interact.
4. Only then descend into implementation details.

Avoid explaining the leaves of the tree before the reader understands the branches.

### Minimize cognitive load

The reader does not share the author's context.

- Do not force readers to reconstruct missing causal links.
- Do not require them to infer unstated assumptions.
- Prefer a small amount of redundancy over ambiguity.
- A good explanation reduces questions about meaning and increases questions about tradeoffs.

The ideal response changes from "What do you mean?" to "I disagree with this assumption." That means the model was transferred successfully.

### Reveal decision boundaries

For any significant proposal, make explicit: the problem being solved, the constraints, alternatives considered, tradeoffs, and the final decision. Readers should understand why a decision exists without reading the implementation.

### Multiple levels of abstraction

Be able to explain the same idea at different levels (one sentence / one paragraph / detailed) and from different perspectives (business / operational / architectural / implementation). The underlying reasoning should remain consistent across all levels.

### Communication quality signals

Good communication is measured by: fewer clarification loops, more precise disagreements, faster convergence, the ability to compress without losing causal information, and the ability to reread the text weeks later and still understand it.

A text is not clear because the author understands it. A text is clear when someone else can reconstruct the same mental model from it.
