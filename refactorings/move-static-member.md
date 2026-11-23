# Move static member

Move a static member (a member that needs no instance) from one type or module to another, and update every reference to it, across the whole workspace.

Together with [Make method static](make-method-static.md) and [Move static method to instance method](move-to-instance-method.md), this gives Fowler's *Move Method*.

## Input

- A cursor on the declaration of the member, or on any reference to it.
- `target`: the destination, as a qualified name in the language's own syntax (C# `Shop.Pricing`, Java `com.shop.Pricing`, C++ `shop::Pricing`, Python `shop.pricing.Pricing` or a module `shop.pricing`, TypeScript `src/pricing.ts#Pricing` or a module `src/pricing.ts`, Rust `crate::pricing::Pricing` or a module `crate::pricing`, Go an import path `example.com/shop/pricing`).

The target must be declared in the workspace, and must be a kind of container that can hold the member.

## What can move

| Language | Members | Targets |
|---|---|---|
| C# | static methods (including extension methods), fields, properties, events, constants | classes, structs, records; a static class for extension methods |
| Java | static methods and fields | classes, records, enums; interfaces for static methods and constants |
| C++ | static member functions and data members; namespace-scope functions and variables | classes, namespaces |
| Python | `@staticmethod` functions, class attributes, module-level functions and variables | classes, modules |
| TypeScript | static methods, fields and accessors; exported module-level functions and `const` declarations | classes, modules |
| Rust | associated functions and constants of an inherent `impl`; free functions, `const` and `static` items | types defined in the same crate (a new or existing inherent `impl`), modules |
| Go | package-level functions, variables and constants | packages |

Types, operators, constructors, and instance members do not move with this refactoring.

## Algorithm (normative outline)

1. Resolve the member and every declaration of it (a C++ declaration and its out-of-line definition move together).
   Refuse if it is not one of the kinds above, or is not declared in the workspace.
1. Resolve the target.
   Refuse if it is the current container, is not in the workspace, or cannot hold the member.
1. Record the binding of every identifier that could be affected:
   every reference to the member, including implicit ones (extension method calls, `using static`, static imports, `from m import f`, `use` paths, ADL);
   every name used in the member's signature and body, which today resolve from the source's scope;
   and every reference anywhere to a symbol with the member's name that is visible in the target, its subtypes, or files that import it.
1. Move the declaration.
   Rewrite names in the signature and body that resolved through the source's scope so they still resolve to the same symbols from the target: qualify the source's other members (`Source.Rate`), and replace self-type references (`Self` in Rust, the class name used unqualified) with the source type.
1. Rewrite every reference: `Source.M` becomes `Target.M`, an unqualified use inside the source becomes `Target.M`, and imports are added or updated.
   A reference through an expression (`order.M()` in Java or C++, where `M` is static) keeps the expression's evaluation, so it is allowed only when the expression is side-effect free.
1. **Re-binding check**: re-bind the result.
   Every recorded reference to the member binds to the moved member; every other recorded reference binds to the same symbol as before, including names in the moved body and calls elsewhere that now see the moved member in an overload set.
   Refuse if any binding changed.
1. Check the refactoring-specific invariants: initialization (below), accessibility, and the import graph.

Placement inside the target follows the language's conventions.
Where declaration order matters (C++ namespace scope, Python class and module bodies, TypeScript class static initializers and `const` declarations), the member goes before its first use in the target.

### Imports

Adding or removing an import can run code (Python, TypeScript, Go package initialization) or change other bindings (C# `using`, C++ `#include`).

- Imports are added only when the re-binding check passes over every file that gains one.
- C++: refuse when a reference site would need an `#include` that it does not already have, directly or transitively; adding a header can define macros that change other code.
- Python: never remove an import (it may be the only thing that runs the module); refuse when a file would need a new import of a module that it does not already import.
- TypeScript: when an import is left with no bindings, keep it as `import "./source";` if it was emitted at runtime under the project's compiler options; otherwise remove it.
- Go: unused imports must be removed.
  Refuse if adding or removing an import changes the program's package initialization order, unless every package whose position changes has no `init` functions and only constant package-level initializers.
- Refuse if the move adds an import cycle (Go and Rust forbid them; in Python and TypeScript a cycle changes evaluation order and can expose half-initialized modules).

### Initialization

When static state is initialized is observable, and it is tied to the container:

- Java: using a static method or a non-constant field initializes its class (running static initializers), and only that class.
- C#: a type with a static constructor runs it before first use; a type without one (`beforefieldinit`) runs its field initializers at some point before the first field access.
- C++: namespace-scope and static data member variables with dynamic initialization run in an order that depends on the translation unit.
- Python: class and module bodies run when the class statement or import runs; default argument values and decorators run when the `def` runs.
- TypeScript: static fields and `static {}` blocks run when the class definition is evaluated; module top-level code runs on first import.
- Go: package-level variables initialize in dependency order within a package, packages in import order.
- Rust: `const` and `static` items have no runtime initialization, so Rust has no hazard here (lazy statics initialize on first access wherever they live).

So:

- The member's own initializer (field, constant, variable, Python default arguments) must be a constant expression, or provably free of observable effects.
- The source's and the target's static initialization must both be free of observable effects.
  Otherwise, using the member could stop triggering the source's initialization or start triggering the target's.

## Preconditions (refuse otherwise)

- No member with the same name in the target, except overloads (C#, Java, C++) that pass the re-binding check.
- The moved member does not hide, shadow, or override anything it did not before (an inherited static with the same name in the target's base types, a module-level name in Python, a glob-imported name in Rust).
- Every reference to the member can still access it from the target, with its current accessibility; and the body does not use anything the target cannot access (private members of the source, C++ `friend` grants, Rust module-private items, Go unexported identifiers of the source package).
  Widening accessibility is a design change, so the tool refuses rather than doing it.
- The member does not use the source's type parameters (C#, Java, C++ templates, Rust generic `impl`s).
  C#: no field moves out of or into a generic type, since each constructed generic type has its own copy of its static fields.
- The member is not partial (C# partial methods), not an interface implementation or `abstract`/`virtual` static (C# static abstract interface members), not an `override`, and not required by a convention that names its container (C# `Main`, Java `main`, Python `__init_subclass__`).
- No lock is tied to the container: Java `static synchronized`, C# `[MethodImpl(MethodImplOptions.Synchronized)]` on a static method, and `lock (typeof(Source))` in the body all lock the class object, which would change.
- Thread-local storage moves only with its attribute (`[ThreadStatic]`, `thread_local`) intact.
- References through an expression keep that expression's evaluation: the expression must be side-effect free (a local, a parameter, `this`).
- No reference the tool cannot see statically: reflection, string-based lookup, Python `getattr`, TypeScript bracket access with a computed key (see [limitations.md](../limitations.md)).
- No conditional compilation spanning the member or a reference (see [limitations.md](../limitations.md)).

## Language-specific rules

### C#

- Extension methods move only to a top-level, non-generic static class.
  Every call site written as `x.M()` must still find `M` by extension lookup; adding a `using` for the target's namespace is allowed only if the re-binding check passes for the whole file, since it can change other extension and type lookups.
- Constants (`const`) move freely: their values are copied into callers at compile time.
- `using static Source;` directives that no longer import anything stay; the tool does not remove them, so other bindings are unchanged.

### Java

- Interfaces hold only `public static final` fields; a non-final or non-public field cannot move to an interface.
  Static interface methods cannot be called through an implementing class, so references must be written `Target.m()`.
- A reference `expr.m()` to a static method evaluates `expr` and discards it, and does not throw even when `expr` is `null`.
  The rewrite drops `expr`, so it must be side-effect free.
- Static imports (`import static Source.m;`) are updated; on-demand static imports (`import static Source.*;`) gain an explicit `import static Target.m;` only if no other binding in the file changes.

### C++

- A namespace-scope function found by argument-dependent lookup may lose that lookup when it moves to another namespace.
  Unqualified calls become qualified calls only if the re-binding check passes.
- Moving a static data member or variable with dynamic initialization is refused (initialization order, see above); `constexpr`, `constinit`, and constant-initialized variables move.
- `inline`, `static` (internal linkage), anonymous namespaces, and the one-definition rule: a function with internal linkage moves only within its translation unit.

### Python

- A module-level function that moves into a class gains `@staticmethod`; one that moves out of a class loses it.
- Only references through the class or module name itself are rewritten.
  Refuse any reference through an instance, `cls`, `self`, or `type(...)`, because a subclass or an instance attribute may supply the name instead.
- A variable moves only if it is assigned once, at its definition.
  `from source import x` copies the binding at import time, so a variable rebound later behaves differently depending on where it lives.
- A moved function's free names resolve in the target module's globals.
  Each one must already resolve there to the same object, or be importable without a new import (above).
- Refuse bodies that use `global`, and decorators other than `@staticmethod`.

### TypeScript

- A static method that uses `this` or `super` is refused: `this` is the class it was called through, which may be a subclass.
- References through a value of constructor type (`cls.m()` where `cls: typeof Source`) are refused.
- Function declarations are hoisted and `const` declarations are not; a `const` moves only before its first use in the target module.

### Rust

- An associated function moves into an inherent `impl` of the target in the same crate.
  `Self` in its signature and body is rewritten to the source type.
- An inherent associated function takes priority over trait methods with the same name; the re-binding check catches calls that would change.
- `use` declarations are updated; a glob import that would now bring in a second item with the same name is refused.
- Item visibility (`pub(crate)`, `pub(super)`, `pub(in path)`) is kept as written, which is relative to the module, so refuse if it would mean something different at the target.

### Go

- Moving between files of the same package changes nothing and is allowed.
- Moving to another package requires the identifier to be exported if anything outside the target package uses it.
  Refuse instead of changing its case; suggest [Rename](rename.md) first.
- Package-level variables move only with constant initializers.

## Edge cases that must be handled (tests required)

- A call in the target that bound to an existing overload and would bind to the moved member (`Target.M(long)` versus moved `M(int)`).
- A member in the target's base type with the same name, which the moved member would hide.
- The body uses another member of the source unqualified, and the target has a member with that name.
- References through subclasses (`Derived.M()`), through `using static` / static imports / `from m import f` / `use` paths, and through method groups or function references.
- Static initialization with observable effects in the source, the target, and the member's initializer; tests record the trace of initializers.
- Java `expr.m()` with a side-effecting `expr`, and with `expr` evaluating to `null`.
- C# extension method calls after the move, with and without the target namespace imported.
- C# static field of a generic type.
- C++ declaration in a header with an out-of-line definition; ADL-only calls.
- Python: a reference through `self`, a subclass that defines the same name, a module that would need a new import, a moved function whose body uses a global of the source module.
- TypeScript: a module that is only imported for the moved binding, an import cycle, a static method using `this`.
- Rust: `Self` in the body; a trait method with the same name; glob imports.
- Go: package initialization order; unexported identifiers; an import cycle.

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
Where another refactoring can remove the obstacle, the suggestion names it (for example [Make method static](make-method-static.md) for an instance method, or [Rename](rename.md) for a name collision in the target).
