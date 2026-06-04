---
name: josue-engineering
description: How Josué works on a concrete task — his operational engineering rules: decision-making, naming, code organization, error handling, testing, code review, commits/PRs, and communication style. Apply when writing or reviewing code for him, handling errors or tests, writing commits or PRs, or communicating reasoning. For system-shape decisions (boundaries, contracts, data flow, where side effects live, adding a layer/dependency), see josue-architecture. Living doc, refined across sessions.
---

# Josué — engineering process & preferences

> Source of truth = `SOURCE.md` (full prose). This file = caveman build. Edit SOURCE first, rebuild here.

How Josué work on a concrete task: decisions, errors, tests, code org, reviews, commits/PRs, communication. Operational rules — change what you do on a given diff.

System *shape* decisions (boundaries, contracts, data flow, where side effects live, add layer/dep) → **`josue-architecture`** skill. Design-time companion.

**⚡ = non-default.** Rules most likely gotten wrong by default. Where this skill earns its keep — apply deliberate.

## Working shape (pointer)

Keep middle of any unit pure: data in → transform → decision out. Side effects (DB/cache writes, network, email, event publish) only at edges. Full arch reasoning (compressible contracts, side effects = arch events) → `josue-architecture`.

## Decision-making

- **Ask before conclude.** Real forks = explicit questions, he decide. No bury big arch choice in impl detail. No present finished assumption when real choice existed.
- **Understand intent before accept/reject.** Looks off → he ask *"what you trying to achieve / why this way?"* before deciding. Reciprocate: explain *why*, drop idea if why weak.
- **Reasoning > prior verdict — both sides.** He change mind on better arg, expect same. Track reasoning not earlier conclusion. "no" → "yes" moment justification clear.
- **Push back when real reason.** Asked remove needed thing → keep, *say why*. Correct objection > silent compliance.
- **⚡ Verify facts vs source. No guess.** Column name, enum value, API field → check real source of truth, no assume plausible name. Guessed identifier = latent bug.
- **Make tradeoffs explicit.** Every real decision trade something (flexibility/simplicity, speed/clarity, safety/velocity). State it — hidden tradeoff = future surprise.

## Naming

- **⚡ Business-logic names in logic, not constants.** Constant = config + stable magic value. No encode business rule someone go read elsewhere.
- **Plain public names** for module surface. Private prefix only for actual internal.
- **No needless indirection.** Used once/twice → inline, no helper wrap. Tighter version, remove ceremony. (When indirection *is* warranted = design call → `josue-architecture`.)

## Code organization

- **One responsibility per file, group as package.** Contracts/types own file, one impl per file, thin entry module = public surface. Consistent suffix conventions.
- **Local reasoning.** Code understandable without navigating whole codebase. Explicit deps > hidden. Pass data > global state. Minimize files to understand a feature. Reader explain module after reading only that module.
- **Protocols/interfaces > base classes** for polymorphism. Impls **duck-typed** — satisfy contract, no inherit machinery. Share *type* fine. Share base class for behavior usually not.
- **Circular import = design smell.** Avoid one needs import-order trick → structure wrong. Extract shared contracts to leaf module.
- **Follow existing patterns before invent.** Reuse > Evolve > Create. But "already there" lose to clearly-better. Convention = default not cage.
- **Prefer existing util/decorator in canonical home.** Use project mechanism (e.g. memoize decorator) > hand-roll cache get/set. Shared small helper → established shared module (e.g. date utils), not feature-local file. Check there first.
- **Comment *why* for non-obvious structure.** Split/add for reason invisible from code (e.g. fn exist only to keep failures out of cache) → say why. Shape explain own intent.

## Error handling

- **Fail loud at edge.** Misconfig / missing dep → raise + visible, no silent swallow into empty/default.
- **Graceful degradation = deliberate separate decision** — made one level up where product choice, not baked into low-level helper.
- **⚡ No cache failures.** Caching front of fallible call → only successes cached, errors propagate to edge that degrades. Else transient failure served whole TTL. Cache the fetch, handle error outside.
- **Bound external calls** (timeouts) → slow dep no hang system.

## Ownership of invariants

Every invariant = one obvious owner. Bad: validation duplicated across handlers / services / DB. Good: enforce at boundary where invariant introduced. Ask *"which module guarantees this property?"* — answer "multiple places" → ownership unclear.

## Performance

- Measure before optimize.
- Optimize bottlenecks, not aesthetics.
- Remove work > make work faster.
- Algorithmic win > micro-optimization.

## Operational thinking

Code = one phase of system life. Consider: observability / debugging / deploy / rollback / failure modes / maintenance burden. Best impl often = easiest to diagnose at 2 AM.

## Testing

- **One test file per unit, mirror code structure.** Not one giant file.
- **Mock at boundary**, exercise real transform logic.
- **Keep concerns in home.** Test about shared/infra helper live with helper, not smuggled into feature suite.
- **Explicit per-test setup** > module-level magic.
- Respect project testing rules: unit-on-logic > end-to-end-thru-edge, exact assertions, parametrize > copy-paste, no test the framework.

## Review philosophy

Reviewing code:

1. Understand the problem.
2. Understand the constraints.
3. Does solution address the problem?
4. Only then discuss impl details.

No optimize code that solve wrong problem. Most valuable review comments = about system shape, not syntax.

## Git & PRs

(Commit/PR text = write normal, not caveman.)

- **⚡ Commits authored by him**, signed (`-S`). **Never** add LLM co-authorship.
- Conventional Commits. Subject + a short body that explains the *why* and the one big design decision — not a changelog of files.
- **⚡ PRs focus on *why* over *how*** — the how is the code. Write for the reader, not the implementation:
  - State the problem and who it hurts.
  - Explain *why this approach*, and the **big** decisions that led there. Skip minor technical choices.
  - The reader's brain isn't yours — don't assume they'll make the same logical jumps. But don't condescend either.
  - Flag the one or two things a reviewer should actually weigh in on.
- Writes PRs in the team's working language.

## Communication

- Terse, direct, no fluff. Corrections short + precise — read as instructions not suggestions.
- He state mental model when matters. Capture + apply forward, beyond line in front of you.

### Communication = model transfer

Goal not state conclusions. Goal = transfer reasoning model that produced them.

- No jump to answer if reader can't reconstruct path.
- Explain system of forces, constraints, tradeoffs that produced decision.
- Expose reasoning chain > defend impl.
- Distinguish clear: observations / assumptions / constraints / conclusions.
- Conclusion depend on hidden context → surface context.

### Structure before detail

Complex thing:

1. Start top-level problem.
2. Introduce major abstractions/components.
3. Explain how they interact.
4. Then descend to impl detail.

No explain leaves before reader understand branches.

### Minimize cognitive load

Reader no share author context.

- No force reader reconstruct missing causal links.
- No require infer unstated assumptions.
- Small redundancy > ambiguity.
- Good explanation reduce questions about meaning, increase questions about tradeoffs.

Ideal response shift: from "What you mean?" → to "I disagree with this assumption." That = model transferred.

### Reveal decision boundaries

Significant proposal → make explicit: problem solved / constraints / alternatives considered / tradeoffs / final decision. Reader understand why decision exist without reading impl.

### Multiple levels of abstraction

Explain same idea at: one sentence / one paragraph / detailed. From: business / operational / architectural / impl. Underlying reasoning consistent across all levels.

### Communication quality signals

Good comm measured by: fewer clarification loops / more precise disagreements / faster convergence / compress without losing causal info / reread weeks later + still understand.

Text not clear because author understand it. Text clear when someone else reconstruct same mental model from it.
