# Rename

Rename a symbol (variable, parameter, field, property, method, type, namespace/module member, etc.) and every reference to it, across the whole workspace.

## Input

- A cursor on any declaration or reference of the symbol.
- `newName`: the new identifier.

## Scope

Symbols only.
Comments, documentation, and string literals are **not** updated (see [limitations.md](../limitations.md)).

## Algorithm (normative outline)

1. Resolve the symbol at the cursor.
   Refuse if none, or if it is not renameable (e.g. defined outside the workspace, in metadata or a third-party package).
1. Record the binding of every identifier in the workspace that could be affected (at minimum: all references to the target symbol, and all references to any symbol named `newName`).
1. Rewrite the declaration and all references.
   Handle language-specific spelling (e.g.
   C# `@` escapes, keyword collisions).
1. **Re-binding check**: re-bind the result.
   Every previously-recorded reference must resolve to the same symbol as before (target references now spelled `newName`).
   Refuse if any binding changed.

## Edge cases that must be handled (tests required)

- **Capture**: renaming `x` to `y` where an inner reference to `x` would now bind to an existing inner `y`.
- **Un-shadowing**: renaming an inner `x` away so a reference that used to bind to it now binds to an outer `x`, or the reverse, where a reference that bound to an outer symbol is captured by the renamed one.
- Shadowing across every scope kind the language has (block, lambda, class, namespace/module, comprehension, etc.).
- Overload resolution changing (C#, C++): renaming a method so a call now binds to a different overload, or to an extension method.
- Overrides, interface implementations, virtual members: renaming must cover the whole hierarchy or refuse.
- Named arguments at call sites, `nameof` (C#), keyword arguments (Python).
- Partial types, multiple files, headers and translation units.
- Generic/template parameters; dependent names and ADL (C++).
- Python: `global`/`nonlocal`, class-body scope rules, comprehension scopes, `__all__`, keyword arguments, dynamic attribute access.
- Swaps and collisions in batch mode (see [protocol.md](../protocol.md)).

## Refusal reasons (initial)

`no-symbol`, `not-renameable`, `invalid-identifier`, `binding-would-change`, `conflicting-declaration`, `dynamic-access-possible` (Python), `preprocessor-region` (see limitations), `internal-verification-failed`.
