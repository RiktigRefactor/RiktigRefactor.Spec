# Make method static

Turn an instance method into a static method on the same type.
The receiver (`this`, `self`) becomes an explicit parameter, or, for members that cannot change, the members the method uses become parameters.
Every call site is updated, across the whole workspace.

Follow it with [Move static member](move-static-member.md) to move the method to another type.
The inverse is [Move static method to instance method](move-to-instance-method.md).

## Input

- A cursor on the declaration of the method, or on any reference to it.
- `pass` (optional): `"receiver"` (default) passes the receiver as the first parameter; `"members"` passes each instance member the method uses as its own parameter.
- `receiverName` (optional): the name of the new receiver parameter.
  The default is the type's name in the language's local-variable style (`order` for `Order`, `line_item` for Python `LineItem`), except `self` in C++, `this` in Rust (where `self` is reserved), and the existing receiver name in Go.

## Generated form

### C#

```csharp
// Before
public decimal Total(int qty) => price * qty;
var t = order.Total(3);

// After
public static decimal Total(Order order, int qty) => order.price * qty;
var t = Order.Total(order, 3);
```

A struct receiver becomes `ref Order order` (call sites `Order.Total(ref o, 3)`), or `in Order order` when the method is `readonly` or the struct is a `readonly struct`.

### Java

```java
public static BigDecimal total(Order order, int qty) { return order.price.multiply(BigDecimal.valueOf(qty)); }
BigDecimal t = Order.total(order, 3);
```

### C++

The method stays a member, so it keeps access to private members.

```cpp
static double total(const Order& self, int qty) { return self.price * qty; }
double t = Order::total(order, 3);   // was order.total(3)
double u = Order::total(*p, 3);      // was p->total(3)
```

A `const` member function takes `const Order&`; a non-`const` one takes `Order&`.

### Python

```python
@staticmethod
def total(order, qty):
    return order.price * qty

t = Order.total(order, 3)
```

### TypeScript

```typescript
static total(order: Order, qty: number): number { return order.price * qty; }
const t = Order.total(order, 3);
```

### Rust

An associated function with no `self`; `Self` and call sites use the path form.

```rust
fn total(this: &Order, qty: u32) -> f64 { this.price * qty as f64 }
let t = Order::total(&order, 3);
```

`self`, `&self`, `&mut self`, `self: Box<Self>` and similar become `this: Order`, `this: &Order`, `this: &mut Order`, `this: Box<Order>`.
Call sites write the receiver adjustments that method-call syntax made implicitly (auto-reference and auto-dereference) explicitly, e.g. `Order::total(&*boxed, 3)`.

### Go

The method becomes a package-level function; the receiver becomes the first parameter.

```go
func Total(o *Order, qty int) float64 { return o.price * float64(qty) }
t := Total(order, 3)
```

A method expression (`(*Order).Total`) already has this function's type, so it is rewritten to `Total`.

### Passing members

With `pass: "members"`, each instance member the body uses becomes a parameter, and call sites read the members:

```csharp
public static decimal Total(decimal price, int qty) => price * qty;
var t = Order.Total(order.price, 3);
```

Call sites now read the members before evaluating the other arguments, and the body no longer sees later changes to them.
So this mode is allowed only when every member the body uses cannot change during the call: C# `readonly` fields, Java `final` fields, C++ `const` data members, TypeScript `readonly` properties assigned only in the constructor, Rust fields of a `&self` receiver without interior mutability.
Python and Go have no such members, so they support only `pass: "receiver"`.
The body must use the receiver for nothing else (no calls to other instance methods, no `this` as a value).

## Algorithm (normative outline)

1. Resolve the method.
   Refuse if it is not an instance method declared in the workspace, or fails a precondition below.
1. Record the binding of every identifier that could be affected: every reference to the method (calls, method groups, method references, delegate creation, `super`/`base` calls), every name in its body, and every reference to any member with the method's name on the type, its base types, and its subtypes.
1. Rewrite the declaration: add the `static` modifier (or move a Go method to a package-level function, or drop Rust's `self`), insert the new parameter(s), and rewrite every use of the receiver in the body, explicit (`this.x`, `self.x`, `this`) and implicit (`x` meaning `this.x`, a call `f()` meaning `this.f()`), including uses inside nested lambdas and local functions.
1. Rewrite every call site: `recv.M(args)` becomes `Type.M(recv, args)`, `M(args)` inside the type becomes `Type.M(this, args)`.
1. **Re-binding check**: every recorded reference binds to the same symbol as before, with references to the method now binding to the static method, and no other call changes overload.
   Refuse if any binding changed.
1. Check that the receiver and arguments are evaluated in the same order as before, and that a missing receiver fails at the same point (below).

## Evaluation order and the null receiver

In every language here, a call `recv.M(a, b)` evaluates `recv` before `a` and `b`, so `Type.M(recv, a, b)` keeps the order, except as noted:

- C++: the postfix expression `recv.M` is sequenced before the arguments, but in `Type::M(recv, a, b)` the arguments are indeterminately sequenced.
  Refuse unless `recv` is [independent](../principles.md#evaluation-order) of every argument.

A null receiver fails at a different point:

- C# and Java check for `null` after evaluating the arguments, at the call, so `recv.M(f())` runs `f()` and then throws before the body.
  `Type.M(recv, f())` runs `f()`, then the body, which may do something observable before it first dereferences the receiver, or never dereference it; and Java's helpful `NullPointerException` message would name a different expression.
- Python and TypeScript fail when they look up `M` on `None`/`undefined`, before evaluating any argument.
- C++: calling a member function through a null pointer is undefined behavior both before and after; `p->M()` becomes `Type::M(*p)`.
- Go: a pointer receiver may be `nil`; the call does not fail before the body in either form.
  A value-receiver method called through a pointer dereferences it at the call; refuse such call sites unless the pointer is provably non-nil.
- Rust references are never null.

So, in C#, Java, Python and TypeScript, every call site's receiver must be **provably non-null**: `this`/`self`, an object creation, a literal, a value of a non-nullable value type (struct), or a variable that flow analysis proves non-null at the call.
Nullable-reference annotations (C#), `@NonNull` (Java), type hints (Python) and `strictNullChecks` (TypeScript) are not proof, because they are not enforced at runtime.

## Preconditions (refuse otherwise)

- The method does not take part in dynamic dispatch: it is not `virtual`, `abstract`, `override`, an interface implementation, or a Rust trait method; in Java, TypeScript and Python (where methods are virtual by default) no type in the workspace overrides it and it overrides nothing.
  Go: no interface type in the program (workspace and dependencies, including the standard library and anonymous interfaces in type assertions) has a method with the same name, since removing the method changes the type's method set, which runtime type assertions observe.
- No reference that keeps the receiver bound: C# method groups and delegate creation (`Func<int> f = order.Total;`), Java bound method references (`order::total`), Python bound methods used as values (`f = order.total`), TypeScript method references (`const f = order.total`), Go method values (`order.Total`).
  Converting them to lambdas would move when the receiver is evaluated and when a null receiver fails.
  Java unbound references (`Order::total`) and Go method expressions are rewritten, since they already take the receiver first.
- Every call site's receiver is provably non-null where the language requires it (above).
- The body does not use `base`/`super` (a static method cannot), Python zero-argument `super()` or `__class__`, or TypeScript `this` as a type (`this` return types).
- Java: the body does not create inner-class instances that need the enclosing instance (`new Inner()`), and does not use `Outer.this` except for the receiver itself.
- C++: no pointer to the member function (`&Order::total`), whose type would change; no `volatile` or ref-qualified (`&&`) member function; no call site whose receiver is a prvalue when the method is non-`const`.
- C#: a struct receiver passed as `ref` must be a mutable variable at every call site (not a `readonly` field, an `in` parameter, or a temporary), because the original call operated on a copy in those cases and `ref` would not.
- Python: the method has no decorators; every reference to the attribute name in the workspace is resolved as for [Rename](rename.md) (no access through receivers of unknown type, `getattr`, or string literals).
- TypeScript: no access to the method through `any`, bracket access with a computed key, `Order.prototype`, or an interface or object type the class is assigned to (removing the instance member must not change what type-checks).
- Go: no promoted call through an embedding struct is ambiguous to rewrite (`outer.Total()` becomes `Total(&outer.Order)` only through a unique path).
- No new name collides: the static method's new signature does not clash with another overload, and Go's new package-level function name is not already declared in the package.

## Edge cases that must be handled (tests required)

- Implicit receiver uses: unqualified fields, properties and calls to other instance methods; `this` passed as an argument; uses inside lambdas, local functions, and anonymous classes.
- A null receiver with an argument that has a side effect (C#, Java, Python, TypeScript); tests record the trace.
- C++ receiver and argument that both have side effects.
- Struct receivers: a mutating method on a local, on a `readonly` field, on a temporary (C#); value and pointer receivers (Go); `self` by value, `&mut self`, and `Box<Self>` (Rust).
- Overloads: a new static overload that a call to a different overload would now select.
- Calls from subclasses, through `base.M()`, and through interfaces or supertypes (refused when dispatch is involved).
- Method groups, method references, bound methods and method values; Java `Order::total` and Go `(*Order).Total`.
- Generic methods and methods of generic types (type arguments now inferred from the receiver argument).
- `pass: "members"` with a mutable member (refused), and with a member assigned in the constructor before a call from the constructor.
- Go: a type that stops satisfying an interface used in a type switch; promoted methods through embedding.
- Python and TypeScript: a subclass defining the same name; access through `getattr` or a computed key.

## Refusal reasons

Each language defines its own refusal codes (see [protocol.md](../protocol.md)); this spec lists none, except `internal-verification-failed`.
