# RFC: Explicit Copy Syntax

* **Feature Name**: `explicit_copy`
* **Start Date**: 2025-07-14
* **RFC PR**: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
* **Rust Issue**: [rust-lang/rust#0000](https://github.com/rust-lang/rfcs/rust/issues/0000)

---

## Summary

Introduce explicit `.copy` syntax to trigger copy semantics. All value transfers default to *move*, regardless of whether the type implements `Copy`. This removes implicit behavior differences tied to `Copy` and aligns with Rust’s philosophy of semantic clarity and explicitness.

> This change does **not** alter the underlying behavior of the language or how copying works. Instead, it exposes what is currently an **implicit operation** as an **explicit one**, in line with Rust’s general design principle: operations with semantic cost or meaning should be visible in code.

---

## Motivation

### 1. Eliminate Action-at-a-Distance

Adding or removing the `Copy` trait from a type currently changes all code using that type. This can silently break carefully designed move-based APIs.

```rust
#[derive(Clone)]
struct Resource;

let x = Resource {};
let y = x; // move

// Later, Copy is added:
#[derive(Clone, Copy)]
struct Resource;

let x = Resource {};
let y = x; // now it's a copy
```

This change is invisible and potentially harmful.

### 2. Make Language-Level Behavior Explicit

Rust already requires explicit syntax for sub-function-level operations that reflect fundamental language behavior — such as `.await` for suspension points, `?` for error propagation, and `unsafe` for bypassing safety guarantees.

Implicit `Copy` behavior is similarly low-level: it controls ownership and memory transfer semantics. Making it explicit brings consistency with Rust’s existing approach to exposing core language mechanics through syntax.

### 3. Improve Code Review and Profiling

It's hard to see where expensive or important copies occur, especially in APIs using generic types.

### 4. Simplify the Mental Model

Users must currently memorize which types are `Copy`. A single rule — all values move unless explicitly copied — is simpler and safer.

### 5. Improved Learning Clarity

By making copies explicit, developers — especially newcomers — will more easily understand ownership, `Copy` semantics, and the difference between `const`, literals, references, and owned values.

### 6. Prevent Logical Errors from Accidental Reuse

When a value is moved, the compiler prevents further use, helping avoid bugs such as accidental reuse in a loop. If a type silently becomes `Copy`, such protections disappear — and implicit reuse may lead to subtle logic errors. Requiring `.copy` makes reuse visible and intentional.

---

## Design

### Syntax

```rust
let x = 42;
let y = x.copy; // explicit copy
let z = x;      // move (even though i32 is Copy)
```

### Rules

* **Move is always the default**, even for `Copy` types.
* **`.copy` is required** to trigger a copy.
* **Using `.copy` on a non-`Copy` type is a compile-time error**.
* Works in function arguments and call chains:

```rust
fn process(n: i32) {}
process(x.copy);

let result = point.copy.translate(3, 4);
```

---

## Consts and Literals

### Literals

Literals like `42`, `"hi"`, and `true` are embedded directly into expressions. They do **not** require `.copy`:

```rust
let a = 42;        // ✅ OK: literal
let b = 1 + 2 + 3; // ✅ OK
```

### Const Bindings

`const` values are always `Copy`. To maintain consistency with normal bindings, `.copy` is required when accessing them by value:

```rust
const A: i32 = 100;
let x = A.copy; // ✅ Explicit copy
```

This makes the distinction between values and bindings consistent, without changing `const` behavior.

### Static Bindings

Unlike `const`, `static` items are accessed as references to a fixed memory location. A `static` value is not itself `Copy`, but you can copy its reference, or dereference and copy the value if it supports `Copy`:

```rust
static B: i32 = 42;
let x = &B;        // ✅ reference
let y = x.copy;    // ✅ copy the reference
let z = (*x).copy; // ✅ copy the value behind the reference
```

No special rule applies to `static`; `.copy` works as usual based on what the user writes.

---

## Semantics of `.copy`

### `.copy` Is Not a Function

The `.copy` operator is not a function, and does not follow trait resolution. It behaves like `?` or `.await`: a syntactic construct that applies directly to a value without auto-dereferencing.

This is a deliberate design decision:

* It avoids subtle type-driven behavior.
* It makes ownership transitions predictable.
* It removes ambiguity between copying a reference and copying the underlying value.

#### Example: Reference Clarity

```rust
let a = &42;

let x = a.copy;      // ✅ x: &i32 — copy the reference
let y = (*a).copy;   // ✅ y: i32  — copy the value behind the reference
```

`.copy` does not auto-deref — it acts on the value as written. This makes working with references significantly clearer and reinforces Rust’s principle that developers should write what they mean.

---

## Closure Capture Semantics

Using `.copy` inside closures or `async` blocks does not change capture semantics. The behavior is consistent with normal read access:

* If `.copy` is called on a variable inside a closure, it is treated as a read.
* The closure will capture the variable **by reference** unless explicitly marked `move`.
* If the closure is marked `move`, the variable is moved into the closure, and `.copy` applies to the moved-in value.

Examples:

```rust
let a = MyCopyType(123);
let closure = || {
    let x = a.copy; // ✅ captured by reference
};

let closure = move || {
    let x = a.copy; // ✅ captured by move
};
```

This behavior is compatible with the no-auto-deref rule: `.copy` applies only to what is written, and dereferencing must be explicit.

---

## Pattern Matching Semantics

This RFC requires that all copies be explicit — including within pattern matching. As a result, destructuring a `Copy` type by value must use `.copy` if the user intends to copy fields. Normal move semantics remain valid:

```rust
match point {
    Point(x, y) => {
        let x = x.copy;
        let y = y.copy;
        // ...
    } // ✅ explicit copy after destructure
}
```

### Future Consideration: `copy` in Pattern Matching

This RFC does not introduce special syntax for `copy` in patterns, but it lays the foundation for such an extension.

A future RFC may explore pattern binding syntax like:

```rust
if let MyStruct { copy a, ref b } = value { ... }
```

This would allow fine-grained control over ownership in pattern matching — similar in spirit to `ref` and the previously proposed `box` pattern syntax.

While this pattern-level control may improve ergonomics, it is not required. Equivalent behavior can be achieved today through explicit destructuring and `.copy` calls.

---

## Migration

This change is edition-based (e.g. **Rust 2027**).

### Migration Strategy

* `cargo fix --edition 2027` will automatically insert `.copy` where needed to preserve behavior.
* If a move would be valid and preferable, the tool should favor move semantics and avoid adding `.copy`.
* In ambiguous or sensitive contexts — such as loop reuse, closure capture, or match destructuring — `cargo fix` may choose **not** to rewrite the code automatically, instead emitting a warning or suggestion.

  * **Examples:**

    * A loop where the variable is reused: `for x in items { ... let y = x; ... }`
    * Code with conditional reuse: `if flag { let _ = val; } let z = val;`
* Compiler lints will help detect and suggest fixes.
* Additional lints may help identify unnecessary uses of `.copy`.

---

## Drawbacks

* **Slightly Increased Verbosity**: Code that reuses `Copy` values now must do so explicitly with `.copy`, even in simple arithmetic. However, this only applies where a value is used more than once — single-use values follow normal move semantics.
* **Compiler Implementation Effort**: Primitive types are currently always treated as `Copy`, and the compiler may not track their move state at all. Supporting this change may require deeper changes to MIR and borrow checking to distinguish moves from copies even for scalars.

---

## Alternatives

### `copy x` Keyword

```rust
let y = copy x;
```

* Rejected due to awkward syntax, poor chaining, and need for a new keyword.

---

## Unresolved Questions

1. **Pattern Matching**: Should `.copy` be allowed or required in match arms or destructuring? (We recommend explicit `.copy`, and leave pattern-level syntax to a future RFC.)
2. **Diagnostics**: What compiler messages and suggestions should be shown?
3. **Generics**: The compiler must enforce `T: Copy` bounds in generic contexts if `.copy` is used.
