# Principles

## Behavior preservation

A refactoring is correct if, for every input, the refactored program has the same **observable behavior** as the original: the same return values, outputs, side effects in the same order, and the same exceptions (by type and message) raised at the same points in the effect sequence.

Out of scope (see [limitations.md](limitations.md)): performance, memory/allocations, stack depth, and stack-trace contents.

## Soundness over completeness

- Each refactoring has explicit **preconditions**.
  If they hold, the result is guaranteed correct.
- If a precondition cannot be *proven*, the tool **refuses**.
  It never guesses.
- Every refusal carries a stable reason code, a human-readable explanation, the source location responsible, and where possible a suggested next step (e.g. "shrink the selection to exclude line 42").
- When future refactorings can get users out of a refusal (e.g. a transformation that removes a `return` from the selection), refusal messages should point to them.

## Semantic, not textual

Every implementation uses a real semantic model of its language (binding, types, scopes, data flow, control flow).
Regex or token-level transformations are not acceptable.

## Verification after transformation

Implementations do not trust their own transformation logic alone.
After producing edits, they re-analyze the result and check that:

1. it parses and type-checks (where the language has a type checker);
1. **re-binding check**: every reference that existed before resolves to the same symbol after (accounting for intended renames), and no reference changed which symbol it binds to;
1. any refactoring-specific invariants hold.

If any check fails, the refactoring is refused (reason code `internal-verification-failed`) and the failure is reported as a bug.

## Evaluation order

Refactorings that move expressions, or change how a value reaches a function, keep the order and the number of times every expression is evaluated.
Some refactorings can only do that when the expressions involved are independent.

Two expressions are **independent** when evaluating them in either order gives the same behavior: at most one of them has an observable effect (a write that the other or later code can see, I/O, or throwing), and neither reads anything the other writes.
Reading a local variable or parameter that neither expression writes, and a literal, are always independent of anything.
When the tool cannot prove two expressions independent, it treats them as dependent.

Where a language leaves an order unspecified (for example C++ function arguments, or Go operands that are not calls), the original program may already run in any of the allowed orders.
A refactoring may not narrow or widen the set of allowed orders unless the expressions whose relative order changes are independent.

## Whole-workspace

Refactorings operate on a workspace, not a single file.
Cross-file correctness (renames across files, partial classes, imports, headers) is required, not optional.

## Trivia

Whitespace and comments are preserved exactly outside the edited region and as faithfully as possible inside it.
Any case where trivia cannot be preserved is reported as a warning on the result, and documented as a limitation.

## Language versions

Target: the **latest** stable version of each language for now.
Supporting older versions (important for recovering legacy code) is a planned future goal; see [limitations.md](limitations.md).
