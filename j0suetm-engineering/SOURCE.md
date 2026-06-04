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
- **⚡ Verify facts against the source; don't guess.** For a column name, an enum value, or an API field, check the real source of truth rather than assuming a plausible name. A guessed identifier is a latent bug.
- **Make tradeoffs explicit.** Every real decision exchanges something (flexibility/simplicity, speed/clarity, safety/velocity). State the tradeoff — hidden tradeoffs become future surprises.

## Naming

- **⚡ Business-logic names belong in the logic, not in constants.** A constant is for config and stable magic values — not for encoding a business rule someone has to go read somewhere else.
- **Plain public names** for things that are part of a module's surface; reserve the private prefix for what's actually internal.
- **No needless indirection.** Inline something used once or twice instead of wrapping it in a helper. Prefer the tighter version; remove ceremony. (When indirection *is* warranted is a design call — see `josue-architecture`.)

## Code organization

- **One responsibility per file; group as a package.** Contracts/types in their own file, one implementation per file, a thin entry module as the public surface. Consistent suffix conventions for module roles.
- **Local reasoning.** Code should be understandable without navigating the whole codebase. Prefer explicit dependencies over hidden ones; pass data over reaching into global state; minimize the files required to understand a feature. A reader should be able to explain a module after reading only that module.
- **Protocols/interfaces over base classes** for polymorphism. Implementations are **duck-typed** — they satisfy the contract without inheriting machinery. Sharing a *type* is fine; sharing a base class to get behavior usually isn't.
- **Circular imports are a design smell.** If avoiding one needs import-ordering tricks, the structure is wrong — extract shared contracts to a leaf module.
- **Follow existing patterns before inventing.** Reuse > Evolve > Create. But "what's already there" still loses to a clearly-better approach — convention is a default, not a cage.
- **Prefer the existing util/decorator in its canonical home.** Use the project's own mechanism (e.g. a memoize decorator) over hand-rolling cache get/set. When a small helper is shared, put it in the established shared module (e.g. the date utils), not a feature-local file — check there first.
- **Comment the *why* for non-obvious structure.** If you split or add something for a reason invisible from the code (e.g. a function that exists only to keep failures out of the cache), say why. The shape should explain its own intent.

## Error handling

- **Fail loud at the edge.** A misconfiguration or missing dependency should raise and be visible, not be silently swallowed into an empty/default result.
- **Graceful degradation is a deliberate, separate decision** — made one level up where it's a product choice, not baked into a low-level helper.
- **⚡ Don't cache failures.** When caching sits in front of a fallible call, structure it so only successes are cached and errors propagate to the edge that degrades — otherwise a transient failure gets served for the whole TTL. Cache the fetch; handle the error outside it.
- **Bound external calls** (timeouts) so a slow dependency can't hang the system.

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
