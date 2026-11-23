# Extract IIFE

(In C#, also called "extract static lambda".)

Wrap a selected expression or sequence of statements in an immediately-invoked function.
Variables the code would capture from the surrounding scope are passed in as parameters instead, so the function captures nothing (except `this` in C++, see below).

## Input

- A range selection covering exactly one expression, or a contiguous sequence of whole statements in the same block.

## Generated form

### C#

Uses `Func`/`Action` with a `static` lambda.

```csharp
// Before
var total = /*[*/price * qty/*]*/;

// After
var total = ((Func<decimal, int, decimal>)(static (price, qty) => price * qty))(price, qty);
```

Statements use `Action<...>` (or `Func<...>` when values flow out).

**Writes to outer variables**: since `Func`/`Action` cannot have `ref`/`out` parameters, any outer variable assigned in the selection and read afterwards is returned in a tuple and reassigned at the call site:

```csharp
(a, b) = ((Func<int, int, (int, int)>)(static (a, b) => { a++; b = a * 2; return (a, b); }))(a, b);
```

`this` and instance members used in the selection are passed explicitly as a parameter.
Refuse if `this` is a struct and the selection mutates it, or if the selection uses `base.`.

### C++

Uses `[]` lambdas, or `[this]` when the selection uses members.
Captured locals are passed as parameters (by reference where the selection writes to them or where copying would change behavior).

```cpp
auto total = [](double price, int qty) { return price * qty; }(price, qty);
```

### Python

A `lambda` can only hold an expression, so expressions use `lambda` and statements use a nested `def` named `_extractedN` (N chosen to be unique in scope), defined immediately before its single call:

```python
def _extracted1(a, b):
    a += 1
    b = a * 2
    return a, b
a, b = _extracted1(a, b)
```

Writes are returned and reassigned at the call site, as in C#.

## Preconditions (refuse otherwise)

- The selection is a complete expression, or complete statements in a single block.
- No non-local control flow leaves the selection: `return`, `break`, `continue`, `goto`, `yield`, labels.
  (Future refactorings will help users eliminate these first.)
- No `await` in the selection.
- The type of the expression and every parameter can be written out explicitly (no anonymous types in C#, etc.).
- Wrapping does not change the type, conversion, or overload resolution of the expression (target typing: `null`, `default`, lambdas, `new()`, collection expressions, implicit conversions).
- Evaluation order and evaluation count of every subexpression are unchanged.
- C#: no `ref` locals, `ref struct` values, `stackalloc`, pointers or `fixed` across the boundary; no `base.` access; definite-assignment state is preserved.
- C++: object lifetimes, temporary lifetime extension, copy elision, destructor timing, value category and `decltype` of the expression are unchanged.
- Python: no zero-argument `super()`, `__class__`, `locals()`, `globals()` mutation, `nonlocal`/`global` targets, or walrus targets crossing the boundary.

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
