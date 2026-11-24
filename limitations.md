# Limitations

## Not part of "behavior"

Performance, memory and allocations, stack depth, and the contents of exception stack traces are **out of scope**.
For example, Extract IIFE adds a stack frame and may allocate a delegate; code that inspects stack traces could observe that.
We do not consider this a behavior change.

## Reflection and dynamic access

Code we cannot see statically is not accounted for: reflection, `eval`/`exec`, `getattr`/`setattr` with computed strings, serialization by member name, external callers of public APIs, and generated code.
Where the tool can detect a likely dynamic access to an affected symbol (e.g. a string literal equal to the old name passed to a reflection API, or `getattr` in Python), it refuses or warns.
It cannot detect all such cases.

Reflection also observes names as text, and a rename changes that text.
The tool does not detect these cases:

- `typeof(T).Name`: it returns the new type name.
- Member names seen through expression trees: for `b => b.Count`, code that prints or inspects the member name sees the new name after the rename.
- Java: `getClass().getName()`, `Enum.name()`, a record's `toString()`, and Java serialization's field names.
- TypeScript and JavaScript: a function's or class's `.name`, including names inferred from the variable a function is assigned to.
- Rust: output of `#[derive(Debug)]`, `std::any::type_name`, and serialization derives such as serde's.
- Go: `%+v` and `%T` output, `reflect`, and `encoding/json` field names when a field has no tag.

## Rename: symbols only

Rename updates symbols and their references.
It does **not** update comments, documentation, or string literals.

## Rename: names captured as strings change

Some language features turn a name into a string at compile time: C#'s `nameof`, `[CallerMemberName]`, and `[CallerArgumentExpression]`.
A rename changes those strings, by design, so they keep matching the source.
This includes a rename that changes the value a `[CallerMemberName]` or `[CallerArgumentExpression]` parameter receives; that is acceptable.
For example, renaming `customer` to `client` in `ArgumentNullException.ThrowIfNull(customer)` changes the exception's parameter name from `"customer"` to `"client"`.
Renaming an enum member likewise changes its name as text: `ToString()`, `Enum.Parse`, and serializers that write enum names (such as JSON) use the new name.
Renaming a C# record or one of its properties changes what the record's synthesized `ToString()` prints.
Code that depends on the exact text, such as tests that compare error messages, tools that parse logs, or stored data and config files that contain these names, may see a difference.

## Moved members: the container is visible

Moving a member (Move static member, Convert static method to instance method) changes the type or module that reflection reports for it: C# `DeclaringType`, Java `getDeclaringClass()`, Python `__module__` and `__qualname__`, and stack traces.
Python's `pickle` stores functions and classes by module and qualified name, so pickles written before the move may no longer load.

## Changed signatures and external callers

Make method static, Convert static method to instance method, and Extract object from arguments change a method's signature, and Move static member changes its location.
The workspace is treated as the whole program: callers outside it (other repositories, compiled binaries that are not rebuilt) are not updated.
When the method is visible outside the workspace (public in a library, exported from a package), the tool reports a warning.

The number and names of a function's parameters are also visible at runtime: JavaScript `Function.length`, Python `inspect.signature`, and reflection over parameters.
The tool does not account for code that inspects them.

## Conditional compilation

`#if` (C#) and preprocessor conditionals/macros spanning a selection (C++) are **not supported** for now.
Selections or renames that touch conditional regions are refused.
If you need this, please open an issue; we'll prioritize based on demand.

## Language versions

For now, each tool targets the latest stable language version.
Support for older versions, important for recovering legacy code, is planned.

## Trivia

Whitespace and comments are preserved where possible.
Cases where they cannot be (e.g. comments inside a selection that moves into a generated lambda header) are reported as result warnings and listed in each language repo.
