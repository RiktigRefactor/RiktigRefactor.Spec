# Move static method to instance method

Turn a static method (or free function) into an instance method on the type of one of its parameters.
That parameter becomes the receiver (`this`, `self`), and every call site passes it as the receiver instead of as an argument.

This is the inverse of [Make method static](make-method-static.md).
To move a static method to another type first, use [Move static member](move-static-member.md).

## Input

- A cursor on the declaration of the method, or on any reference to it.
- `receiver`: the name of the parameter that becomes the receiver.

The parameter's type `T` must be a class, struct, record, or (Rust, Go) named type declared in the workspace.

## Generated form

### C#

```csharp
// Before
public static class Pricing { public static decimal Total(Order order, int qty) => order.Price * qty; }
var t = Pricing.Total(order, 3);

// After
public partial class Order { public decimal Total(int qty) => Price * qty; }
var t = order.Total(3);
```

The method goes into `T`'s declaration (in a C# `partial` part only if `T` already is partial).
For a struct `T`, the method is `readonly` (see the preconditions on aliasing).

### Java

```java
public BigDecimal total(int qty) { return price.multiply(BigDecimal.valueOf(qty)); }
BigDecimal t = order.total(3);
```

The new method is `final` when `T` has subclasses, so no subclass can override it later without a compile error.

### C++

```cpp
double total(int qty) const { return price * qty; }   // from: static double total(const Order& o, int qty)
double t = order.total(3);
```

A `const T&` parameter gives a `const` member function, `T&` a non-`const` one.

### Python

```python
def total(self, qty):
    return self.price * qty

t = order.total(3)
```

### TypeScript

```typescript
total(qty: number): number { return this.price * qty; }
const t = order.total(3);
```

### Rust

A new inherent `impl` block is placed next to the original function, in the same module, so the body's access to module-private items is unchanged.

```rust
impl Order {
    fn total(&self, qty: u32) -> f64 { self.price * qty as f64 }
}
let t = order.total(3);
```

`x: T`, `x: &T`, `x: &mut T`, `x: Box<T>`, `x: Rc<T>`, `x: Arc<T>` and `x: Pin<&mut T>` become `self`, `&self`, `&mut self`, `self: Box<Self>`, and so on.

### Go

The function must be in `T`'s package (use [Move static member](move-static-member.md) first).

```go
func (o *Order) Total(qty int) float64 { return o.price * float64(qty) }
t := order.Total(3)
```

The receiver keeps the parameter's name and type (`*Order` or `Order`).

## Algorithm (normative outline)

1. Resolve the method and the receiver parameter.
   Refuse if the method is not static, the parameter's type is not one listed above, or a precondition below fails.
1. Record the binding of every identifier that could be affected: every reference to the method, every name in its body, every reference to any member of `T`, its base types and its subtypes with the method's name, and every implicit binding that depends on `T`'s members (C# `foreach`, `await`, `using`, deconstruction and collection initializers; Python special methods; TypeScript and JavaScript protocol members; Go interface satisfaction).
1. Move the declaration into `T` as an instance method, without the receiver parameter.
   Rewrite uses of the parameter in the body to the receiver (`this.x` or implicit `x`, `self.x`), and qualify names that resolved through the old container's scope (`Pricing.Rate`).
1. Rewrite every call site: `S.M(a, recv, b)` becomes `recv.M(a, b)`.
   Function references are rewritten where the language has an unbound form with the same shape: Java `Pricing::total` becomes `Order::total`, Python `Pricing.total` becomes `Order.total`, Go `Total` becomes `(*Order).Total`; this needs the receiver to be the first parameter.
1. **Re-binding check**: every recorded reference binds to the same symbol as before, with references to the method now binding to the instance method; no other call, override, or implicit binding changed.
   Refuse if any binding changed.
1. Check evaluation order, the null receiver, and dispatch (below).

## Evaluation order

`S.M(a, recv, b)` evaluates `a` before `recv`; `recv.M(a, b)` evaluates `recv` first.
When the receiver is not the first argument, refuse unless every argument before it is [independent](../principles.md#evaluation-order) of the receiver expression.
The same applies to the order the arguments are written in, with C# named arguments and Python keyword arguments.

C++ evaluates arguments in an unspecified order but sequences `recv.M` before them, so refuse unless the receiver expression is independent of every other argument.

## The null receiver

`S.M(null, f())` runs `f()` and then the body; the body may never use the parameter.
After the move, the call fails before the body:

- C# and Java throw after evaluating the arguments, before the body.
- Python and TypeScript fail on looking up `M`, before evaluating the arguments.
- C++: a call through a null pointer is undefined behavior.

So every call site's receiver argument must be provably non-null, as defined in [Make method static](make-method-static.md#evaluation-order-and-the-null-receiver).
Go pointer receivers may be `nil` in both forms; Rust references are never null.

When a C# receiver may be null, a C# extension method keeps the static call's null behavior; the refusal suggests that form.

## Dispatch and new members

Adding a member to `T` can change code that never called the moved method:

- **Overriding.** In Java, TypeScript and Python, a subtype member with the same name and a compatible signature would override the new method; a supertype member with the same name would be overridden by it.
  In C#, the new member would hide a base member.
  Refuse if any type in `T`'s hierarchy declares a member with the method's name (Python: also any assignment of that attribute on any object, since an instance attribute hides a method).
- **Protocols.** Some names are looked up implicitly.
  Refuse names the runtime or the standard library looks up on objects: Python special (`__dunder__`) names; JavaScript `toString`, `valueOf`, `toJSON`, `then` (which makes an object awaitable), `constructor`, and symbol-keyed members; Java and C# members of `Object`/`object`.
  C# pattern-based lookups (`GetEnumerator`, `GetAwaiter`, `Dispose`, `Deconstruct`, `Add`) are compile-time bindings and are covered by the re-binding check.
- **Go method sets.** Adding a method can make `T` satisfy an interface it did not before, which runtime type assertions and type switches observe (a `String() string` method changes what `fmt` prints).
  Refuse if any interface type in the program (workspace and dependencies, including the standard library and anonymous interfaces) has a method with the same name, or if `T` has a field or promoted member with that name.
- **Rust method resolution.** An inherent method takes priority over a trait method of the same name at the same auto-reference step; the re-binding check catches calls that would change.

## Preconditions (refuse otherwise)

- `T` is declared in the workspace and is not an interface, a trait, a type parameter, a C# enum (which cannot declare methods), or a type alias.
  Rust: `T` is defined in the same crate.
  Go: the function is in `T`'s package, and `T` is not a pointer or interface type.
- The receiver parameter is not `ref`/`out`/`in` (C#), `params`/varargs/rest, optional or defaulted, keyword-only or `*args` (Python), or a pointer (C++, unless every call site's argument is provably non-null).
- No call site passes the receiver through argument unpacking (`*args`, spread).
- The body does not assign the receiver parameter (`order = other;`), take its address, or pass it by reference.
- Struct receivers (C# struct, C++ class passed by value, Go value receiver of a type whose methods mutate it): the original parameter was a copy, and `this` may alias the caller's variable.
  Refuse unless at every call site the argument is a temporary, or a local that nothing else can read or write during the call (not captured, not passed by reference, address not taken).
  C++: refuse a by-value parameter whose copy constructor or destructor is not trivial, since the copy disappears.
- The receiver argument's conversion to `T` is an identity or reference conversion (no user-defined conversion, boxing, or numeric conversion, which would now happen at a different time or not at all).
- The method's type parameters, if any, are used as `T`'s type arguments in order (`static <E> E first(Box<E> b)` becomes `E first()` on `Box<E>`); any other use of type parameters is refused.
- Every call site can access the method on `T` with its current accessibility, and the body does not use anything the new location cannot access.
- No reference the tool cannot see statically (reflection, `getattr`, computed keys; see [limitations.md](../limitations.md)).
- C++: a lambda in the body that captures the parameter by copy would copy the object; after the move it would capture `this`.
  Refuse.
- C#: method groups and delegates over the method (`Func<Order, int, decimal> f = Pricing.Total;`) are refused; there is no unbound instance method group.
- TypeScript: references to the function as a value are refused, since `Order.prototype.total` needs `this`.

## Edge cases that must be handled (tests required)

- The receiver is the second parameter, with a side-effecting first argument; with named or keyword arguments written out of order.
- A null receiver with a side-effecting argument, in each language that checks for null; a body that never uses the parameter.
- A subclass of `T` with a member of the same name; a base class of `T` with one; Python instance attribute with the same name.
- Go: `String`, `Error`, and a name matching an interface used only in a type switch.
- JavaScript `then`, `toJSON`; Python `__len__`, `__eq__`.
- C# struct receiver where the method body triggers a write to the caller's variable (through a closure or `ref`), with and without the struct being `readonly`.
- C++ by-value parameter with a copy constructor that logs; a lambda capturing the parameter.
- Rust: `&self`, `&mut self`, `self` by value, `Box<Self>`; a trait method of the same name in scope; module-private items used in the body.
- Generic method over a generic `T`.
- Function references in Java, Python, Go (rewritten) and C#, TypeScript (refused).
- Recursive calls in the body (`Total(order.Next, qty)` becomes `order.Next.Total(qty)`, under the same null rule).

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
Where another refactoring can remove the obstacle, the suggestion names it (for example [Move static member](move-static-member.md) to bring a Go function into `T`'s package).
