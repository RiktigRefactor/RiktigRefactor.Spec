# Limitations

## Not part of "behavior"

Performance, memory and allocations, stack depth, and the contents of exception stack traces are **out of scope**. For example, Extract IIFE adds a stack frame and may allocate a delegate; code that inspects stack traces could observe that. We do not consider this a behavior change.

## Reflection and dynamic access

Code we cannot see statically is not accounted for: reflection, `eval`/`exec`, `getattr`/`setattr` with computed strings, serialization by member name, external callers of public APIs, and generated code. Where the tool can detect a likely dynamic access to an affected symbol (e.g. a string literal equal to the old name passed to a reflection API, or `getattr` in Python), it refuses or warns. It cannot detect all such cases.

## Rename: symbols only

Rename updates symbols and their references. It does **not** update comments, documentation, or string literals.

## Conditional compilation

`#if` (C#) and preprocessor conditionals/macros spanning a selection (C++) are **not supported** for now. Selections or renames that touch conditional regions are refused (`preprocessor-region`). If you need this, please open an issue; we'll prioritize based on demand.

## Language versions

For now, each tool targets the latest stable language version. Support for older versions, important for recovering legacy code, is planned.

## Trivia

Whitespace and comments are preserved where possible. Cases where they cannot be (e.g. comments inside a selection that moves into a generated lambda header) are reported as result warnings and listed in each language repo.
