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

### Java

A lambda cannot be called directly, so it is cast to a functional interface and its method is called.
Lambdas can only capture effectively final locals, so every local the selection uses is passed in, as in C#.
`this` and instance members are passed explicitly, as in C#.

```java
int total = ((IntBinaryOperator) (price, qty) -> price * qty).applyAsInt(price, qty);
```

The interface is a `java.util.function` type only when its parameter and return types match exactly, without boxing (`Function`, `Supplier` and `BiFunction` for reference types; `IntUnaryOperator`, `DoubleSupplier`, `ToIntFunction` and the like for primitives) and the selection throws no checked exception.
Boxing would change `==` on the parameters and could add a `NullPointerException` on unboxing.
Otherwise the tool declares a local interface immediately before the enclosing statement, with exact types and a `throws` clause listing the checked exceptions the selection can throw:

```java
interface Extracted1 { double apply(double price, int qty) throws IOException; }
double total = ((Extracted1) (price, qty) -> price * read(qty)).apply(price, qty);
```

Writes to more than one outer variable are returned in a local record and assigned back:

```java
record Extracted1Result(int a, int b) {}
var r = ((Extracted1) (a, b) -> { a++; b = a * 2; return new Extracted1Result(a, b); }).apply(a, b);
a = r.a();
b = r.b();
```

Local interfaces and records are implicitly static, so when a type mentions a type variable of the enclosing method, the local interface declares it as its own type parameter.
Where no statement can hold the declaration (a field initializer, an explicit constructor call's arguments), refuse.

### TypeScript

An arrow function called immediately.
Arrow functions take `this` from the enclosing scope, so `this` is not passed in (like C++ `[this]`).

```typescript
const total = ((price: number, qty: number): number => price * qty)(price, qty);
```

Writes are returned in an object and assigned back with object destructuring, which, unlike array destructuring, does not use the iterator protocol:

```typescript
({ a, b } = ((a: number, b: number): { a: number; b: number } => { a++; b = a * 2; return { a, b }; })(a, b));
```

Types are written out (`typeof x` is allowed where a type has no name).

### Rust

A closure called immediately.

```rust
let total = (|price: f64, qty: i32| -> f64 { price * qty as f64 })(price, qty);
```

Values of `Copy` types are passed by value.
Other values the selection only reads are passed as `&T`, and values it mutates as `&mut T`, so nothing moves and writes go straight to the caller's variable (the body dereferences them where needed).
A value the selection moves is passed by value only if the move is unconditional.
`self` becomes a parameter named `this`.
Writes to `Copy` locals may instead be returned in a tuple and assigned with destructuring assignment, `(a, b) = (...)(a, b);`.
Parameter types are written where nameable; closure, `impl Trait` and other unnameable types are left to inference, and the type check confirms the inferred types are the same.

### Go

A function literal called immediately.

```go
total := func(price float64, qty int) float64 { return price * float64(qty) }(price, qty)
```

Writes use multiple return values: `a, b = func(a, b int) (int, int) { a++; b = a * 2; return a, b }(a, b)`.
A method receiver is an ordinary variable and is passed like any other.

## Preconditions (refuse otherwise)

- The selection is a complete expression, or complete statements in a single block.
- No non-local control flow leaves the selection: `return`, `break`, `continue`, `goto`, `yield`, labels.
  (Future refactorings will help users eliminate these first.)
- No `await` in the selection.
- The type of the expression and every parameter can be written out explicitly (no anonymous types in C#, etc.).
- Wrapping does not change the type, conversion, or overload resolution of the expression (target typing: `null`, `default`, lambdas, `new()`, collection expressions, implicit conversions).
- Evaluation order and evaluation count of every subexpression are unchanged.
- A closure created in the selection (lambda, local function, anonymous class, nested `def`) that uses an outer variable would capture the new parameter, a copy, instead of the variable.
  Refuse unless that variable is not written, inside or outside the selection, while the closure can still run.
- C#: no `ref` locals, `ref struct` values, `stackalloc`, pointers or `fixed` across the boundary; no `base.` access; definite-assignment state is preserved.
- C++: object lifetimes, temporary lifetime extension, copy elision, destructor timing, value category and `decltype` of the expression are unchanged.
- Python: no zero-argument `super()`, `__class__`, `locals()`, `globals()` mutation, `nonlocal`/`global` targets, or walrus targets crossing the boundary.
- Java: no `super.` access, `yield` (in a switch expression), or qualified `Outer.this`; no local class declared in the selection and used after it; definite-assignment state is preserved; a variable declared in the selection and used after it is declared before the call and assigned from the result.
- TypeScript: no `arguments`, `new.target`, `super`, `var` or function declarations (hoisted to the enclosing function) in the selection; no `let`, `const` or `class` declared in the selection and used after it (the temporal dead zone and `const` could not be kept); every expression in the selection keeps its type (contextual typing of lambdas and object literals, literal types of `const`); no `const enum` or decorator metadata whose emitted code would change.
- Rust: no `?`, `.await`, labeled `break` or `return` out of the selection; not in a `const` context (closures cannot be called there); no temporary with an observable `Drop` (a `MutexGuard`, a type with a user `Drop`) whose lifetime would end at the closure's return instead of the enclosing statement; no conditional move of a value with drop glue; the borrow checker accepts the result.
- Go: no `defer` (it would run when the function literal returns), `recover()`, `goto`, labels, `fallthrough`, `break`, `continue` or `return` out of the selection; not in a constant expression; an untyped constant expression keeps its type (write the return type it had in context, and refuse non-constant shifts whose type came from context); every type is nameable from the current package.
- Where a language cannot express the form in a given context, the tool refuses: Java where no statement can hold a needed local declaration, Rust `const` contexts, Go constant expressions.

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
