# Extract object from arguments

(Also called *Introduce Parameter Object*.)

Group some of a method's parameters into a new immutable type, replace them with one parameter of that type, and update every call site to construct it from the same argument expressions, in the same order.

## Input

- A cursor on the declaration of the method, or on any reference to it.
- `parameters`: the names of the parameters to group, at least one.
- `typeName`: the name of the new type.
- `parameterName` (optional): the name of the new parameter; defaults to `typeName` in the language's local-variable style.

The new type is declared next to the method's containing type, in the same namespace, package, or module (Java: a nested `record` when the containing type is a class; Go and Rust: in the same package or module).
Its members keep the parameters' names, in the language's member style where it has one (C# `Price` for `price`; Go `Price`, exported, when a caller is in another package).
The new parameter takes the position of the first grouped parameter.

## Generated form

### C#

```csharp
// Before
decimal Quote(string sku, decimal price, int qty = 1) => price * qty;
var q = Quote(Sku(), Price(), Qty());

// After
public record QuoteLine(decimal Price, int Qty = 1);
decimal Quote(string sku, QuoteLine line) => line.Price * line.Qty;
var q = Quote(Sku(), new QuoteLine(Price(), Qty()));
```

Named arguments keep their written order: `Quote("a", qty: Q(), price: P())` becomes `Quote("a", new QuoteLine(Qty: Q(), Price: P()))`.

### Java

```java
record QuoteLine(BigDecimal price, int qty) {}
BigDecimal quote(String sku, QuoteLine line) { return line.price().multiply(BigDecimal.valueOf(line.qty())); }
var q = quote(sku(), new QuoteLine(price(), qty()));
```

### C++

```cpp
struct QuoteLine { double price; int qty = 1; };
double quote(const std::string& sku, QuoteLine line) { return line.price * line.qty; }
auto q = quote(sku(), QuoteLine{.price = price(), .qty = qty()});
```

### Python

```python
@dataclass(frozen=True)
class QuoteLine:
    price: Decimal
    qty: int = 1

def quote(sku, line):
    return line.price * line.qty

q = quote(sku(), QuoteLine(price(), qty()))
```

Keyword arguments stay keyword arguments: `quote("a", qty=q(), price=p())` becomes `quote("a", QuoteLine(qty=q(), price=p()))`.

### TypeScript

The new type is an interface (it has no runtime presence), and the parameter destructures it, so the body is unchanged.

```typescript
interface QuoteLine { readonly price: number; readonly qty?: number }
function quote(sku: string, { price, qty = 1 }: QuoteLine): number { return price * qty; }
const q = quote(sku(), { price: price(), qty: qty() });
```

A default applies when the property is `undefined`, exactly as a default parameter applies when the argument is `undefined`.

### Rust

A pattern parameter destructures the struct, so the body is unchanged.

```rust
struct QuoteLine { price: f64, qty: u32 }
fn quote(sku: &str, QuoteLine { price, qty }: QuoteLine) -> f64 { price * qty as f64 }
let q = quote(&sku(), QuoteLine { price: price(), qty: qty() });
```

`mut` parameters become `mut` bindings in the pattern.
Reference parameters give the struct lifetime parameters (`struct QuoteLine<'a> { sku: &'a str }`).

### Go

```go
type QuoteLine struct {
	Price float64
	Qty   int
}

func Quote(sku string, line QuoteLine) float64 {
	price, qty := line.Price, line.Qty
	return price * float64(qty)
}

q := Quote(sku(), QuoteLine{Price: price(), Qty: qty()})
```

Go copies the fields into locals with the parameters' names, so the body is unchanged.

### The body

Where the language has no destructuring parameter (C#, Java, C++, Python), references to a grouped parameter become member accesses on the new parameter (`price` becomes `line.Price`).
The object is created fresh at every call and is immutable, so a member access always gives the value the parameter had.
If the body assigns a grouped parameter, takes its address, or passes it by reference, that parameter is instead copied into a local with its name at the start of the body.

## Algorithm (normative outline)

1. Resolve the method, and every method it must change with: all overrides, the members it implements, and the interface or base declarations.
   Refuse if any of them is outside the workspace, or if the hierarchy cannot change together (an implementation may refuse any method in a hierarchy).
1. Check the preconditions below for each grouped parameter and each call site.
1. Record the binding of every identifier that could be affected: every reference to the method, every name in its body, every call to any overload of the method's name, and every reference to `typeName` and `parameterName` in the scopes where they are introduced.
1. Declare the new type.
   Rewrite the signature (and those of the hierarchy), and the body.
1. Rewrite every call site, putting the grouped arguments, in their written order, into one construction expression at the position of the first grouped argument.
   Omitted optional arguments are omitted from the construction too.
1. **Re-binding check**: every recorded reference binds to the same symbol as before, with calls to the method binding to the new signature; no call to another overload changes; the new type's name and the new parameter's name capture nothing.
   Refuse if any binding changed.
1. Check that every argument expression is evaluated the same number of times, in the same order, and converted to the same type as before.

## Evaluation order

Each call site's grouped arguments move into the construction expression, which sits where the first grouped argument was.

- If the grouped parameters are contiguous at the call site (in written order), nothing is reordered: object creation in C#, Java, Python, TypeScript and Rust evaluates its arguments or fields left to right.
- If they are not contiguous, the arguments written between them move after the construction.
  Refuse unless those arguments are [independent](../principles.md#evaluation-order) of the grouped arguments they now follow.
- C++: function arguments are indeterminately sequenced, while a braced initializer evaluates left to right.
  Refuse unless the grouped arguments are independent of each other and of the other arguments.
- Go: calls in a composite literal and in an argument list are both evaluated left to right, and other operand reads are unordered relative to them in both.
  The same independence rule as C++ applies to arguments that are not calls.

## Preconditions (refuse otherwise)

- Every grouped parameter is an ordinary value parameter: not `ref`/`out`/`in` (C#), not `params`/varargs/rest/`*args`/`**kwargs`, not a C# `ref struct` or `scoped` parameter, not a caller-information parameter (`[CallerMemberName]`, `[CallerArgumentExpression]` and similar).
- Every default value can move to the new type with the same meaning: C# and C++ constant defaults; Python defaults that are immutable literals (a default is evaluated once, at `def` time, and a mutable default object is shared); TypeScript defaults that do not refer to other parameters or have side effects.
  C++ defaults that are not constants are refused, since a default member initializer looks names up in a different scope.
- No call site passes grouped arguments through unpacking (`*args`, `**kwargs`, spread) or relies on `arguments` (TypeScript).
- No reference uses the method's shape: method groups and delegates, method references, function values, function pointers, and `Function.length` or `inspect.signature` are refused or listed in [limitations.md](../limitations.md).
- Each argument converts to the member's type exactly as it converted to the parameter's type.
  C#: target-typed arguments (`null`, `default`, `new()`, lambdas, collection expressions) are fine, since the constructor parameter has the same type, but grouped parameters whose types depend on a generic method's inferred type arguments are refused.
  C++: a narrowing conversion is ill-formed in aggregate initialization; the type check refuses it.
- C++: grouped parameters of types with non-trivial destructors are refused, since parameter and member destruction happen at different points.
- Rust: grouped parameters of types whose `Drop` has an observable effect are refused, since bindings from a pattern drop in a different order than parameters.
- No name collides: `typeName` is new in its namespace, the member names are distinct, and `parameterName` does not capture or shadow a name the body uses.
- The new signature does not collide with another overload.
- Every call site can name the new type without a new import that could run code (Python, TypeScript side-effect imports) or change other bindings; see [Move static member](move-static-member.md#imports).

## Edge cases that must be handled (tests required)

- Non-contiguous grouped parameters with side-effecting arguments in between; tests record the trace.
- C# named arguments and Python keyword arguments written out of declaration order.
- Optional parameters omitted at some call sites and given at others; TypeScript `undefined` passed explicitly.
- A body that assigns a grouped parameter; a lambda in the body that captures one.
- Overrides and interface implementations changed together.
- An existing overload that a rewritten call, or another call, would now select.
- Arguments that need conversions: numeric widening, `null`, lambdas, C++ braced lists, narrowing.
- Generic parameter types; Rust references needing lifetime parameters.
- Recursive calls inside the body.
- Python mutable default; Python caller in another module that would need a new import (refused, see [Move static member](move-static-member.md#imports)).
- Go caller in another package (exported member names).

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
