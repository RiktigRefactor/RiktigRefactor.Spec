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

Some names are implicit: the language derives them from the spelling of another name, and the program can observe them.
When a rename would change such an implicit name, the tool writes the old name explicitly so behavior is kept, instead of changing it or refusing.
Before writing it, the tool checks that the implicit name is unchanged.
C# examples:

- An anonymous type member whose name is inferred: renaming `Count` in `new { box.Count }` gives `new { Count = box.Size }`, so the member name, `ToString()` output, and equality stay the same.
- A `[DllImport]` method with no `EntryPoint`: renaming `strlen` writes `[DllImport("c", EntryPoint = "strlen")]`, so the native function called stays the same.

This differs from names captured as strings (`nameof` and similar), which change on purpose (see [limitations.md](../limitations.md)).

Conflict repairs that insert casts are never accepted: the tool refuses instead.
For example, renaming `G(short)` to `F` next to an existing `F(int)` would make `G(1)` call `F(int)`; Roslyn's repair, `F((short)1)`, is not accepted.

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

## Language-specific rules

### Java

- Spelling: keywords and reserved names (`_`, `var`, `record`, `yield`, `sealed`, `permits` where they are restricted) are refused as new names; references written with Unicode escapes (`a`) are found and rewritten.
- Obscuring: a variable takes precedence over a type or package with the same simple name, so renaming a local to `Math` makes `Math.max(...)` refer to the local.
  The re-binding check covers qualifiers as well as simple names.
- A lambda body cannot redeclare an enclosing local, but an anonymous or local class can shadow one; renames that make a lambda parameter collide are refused, and shadowing in classes goes through the capture check.
- Field hiding across subclasses, static imports shadowed by members, and single-type imports shadowing on-demand imports.
- Overloads with varargs, boxing and generics; renaming a method must cover every override and implementation, including bridge methods the compiler generates, or refuse.
- Records: renaming a component renames its field, accessor, canonical and compact constructor parameter together; the component name is visible in `toString()` and reflection (see [limitations.md](../limitations.md)).
- Annotation elements: renaming `value()` breaks the `@A(5)` shorthand, so it is refused.
- Names the runtime looks up by convention are refused: `main`, serialization members (`serialVersionUID`, `writeObject`, `readObject`, `readResolve`, `writeReplace`), enum `values`/`valueOf`.
- Labels live in their own namespace.

### TypeScript

- Spelling: reserved words are refused as identifiers; as property names they are allowed.
- Structural typing: renaming a property must rename it in every type it is related to by assignability (interfaces, object literals, classes that implement or are passed as them), or refuse.
- Shorthand properties and destructuring: renaming the variable in `{ x }` writes `{ x: y }`, and in `const { x } = o` writes `const { x: y } = o`, so the property name stays (an implicit name, as described above).
- Exports: renaming a module export renames it in every importer in the workspace; renaming only a local binding of an import writes `import { x as y }`.
- Property names reach the runtime as text: bracket access with a string literal, `in`, `Object.keys`, `JSON.stringify`, `keyof` and string literal types.
  Literal uses of the old name in these positions are refused; computed keys are refused when their type includes the old name.
- Declaration merging (interfaces, namespaces, `declare global`), ambient declarations from outside the workspace, and `.d.ts` files: refuse if any declaration of the symbol is outside the workspace.
- Scoping: `var` is function-scoped and hoisted; `let`/`const` have a temporal dead zone, so a capture can turn a working read into a `ReferenceError`; class `#private` names have their own scope.
- Function and class names are visible as `.name`, and are inferred from the variable they are assigned to (`const f = () => 0` has `f.name === "f"`); see [limitations.md](../limitations.md).

### Rust

- Spelling: keywords are written as raw identifiers (`r#match`); `self`, `Self`, `super` and `crate` cannot be renamed to.
- `let` may shadow a binding in the same block, so captures and un-shadowing happen within one block, not only across scopes.
- Patterns: an identifier pattern binds a new variable unless a constant, unit struct, or unit enum variant with that name is in scope, in which case it matches against it.
  Renaming a binding to such a name, or a constant to a name used as a binding, is refused.
- Shorthand: `S { x }` in expressions and patterns becomes `S { x: y }` when only one side is renamed.
- Inline format arguments are references: renaming `x` rewrites `format!("{x}")` to `format!("{y}")`, even though the reference is inside a string literal.
- Macros: identifiers inside `macro_rules!` bodies and procedural macro input are renamed only when the expansion shows they refer to the symbol; anything the tool cannot trace through a macro is refused.
- Traits: renaming a trait method renames every implementation and every call through the trait, or refuses.
- `use` declarations, `as` renames, and glob imports (an explicit item shadows a glob-imported one, so a new name can hide a glob import).
- Lifetimes and loop labels have their own namespaces.
- Exported symbols: renaming a `#[no_mangle]` or `extern "C"` function writes `#[export_name = "old"]`, and renaming a foreign function in an `extern` block writes `#[link_name = "old"]`, so the linker-visible name stays (implicit names, as described above).
- `#[derive(Debug)]` and serialization derives use field and type names as text; see [limitations.md](../limitations.md).

### Go

- Exported-ness follows the first letter.
  Renaming `total` to `Total` exports it; renaming an exported name to an unexported one is refused if anything outside the package uses it.
- Short variable declarations: `x, err := f()` declares only the names that are new in the scope.
  Renaming `x` to a name already declared in the same scope would turn its declaration into an assignment; refuse.
- Interface satisfaction is implicit.
  Renaming a method changes which interfaces its type satisfies, which runtime type assertions and type switches observe; refuse unless every interface and implementation involved is renamed together, and refuse names that interfaces outside the workspace use (`String`, `Error`, `Read`).
- Embedded fields are named after their type, so renaming an embedded type renames the field (`s.OldType` becomes `s.NewType`), and promoted names can collide at a different depth.
- Package names: renaming a package updates every import's default name; an import with an explicit name keeps it.
- `init`, `main`, and the blank identifier `_` cannot be renamed.
- Struct field names are visible to `encoding/json`, `%+v`, and reflection; see [limitations.md](../limitations.md).

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
