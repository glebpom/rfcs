- Feature Name: `explicit_copy`
- Start Date: 2025-07-14
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

Introduce explicit `.copy` syntax to trigger copy semantics. All value transfers default to *move*, regardless of whether the type implements `Copy`. This removes implicit behavior differences tied to `Copy` and aligns with Rust's philosophy of semantic clarity and explicitness.

# Motivation

Rust currently has an implicit copying behavior for types that implement the `Copy` trait, which creates several problems for users:

## Eliminate Action-at-a-Distance

Adding or removing the `Copy` trait from a type currently changes all code using that type. This can silently break carefully designed move-based APIs:

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

This change is invisible and potentially harmful to API consumers.

## Make Language-Level Behavior Explicit

Rust already requires explicit syntax for sub-function-level operations that reflect fundamental language behavior — such as `.await` for suspension points, `?` for error propagation, and `unsafe` for bypassing safety guarantees. Implicit `Copy` behavior is similarly low-level: it controls ownership and memory transfer semantics. Making it explicit brings consistency with Rust's existing approach to exposing core language mechanics through syntax.

## Improve Code Review and Profiling

It's hard to see where expensive or important copies occur, especially in APIs using generic types. This makes performance analysis and code review more difficult.

## Simplify the Mental Model

Users must currently memorize which types are `Copy`. A single rule — all values move unless explicitly copied — is simpler and safer for developers to reason about.

## Improved Learning Clarity

By making copies explicit, developers — especially newcomers — will more easily understand ownership, `Copy` semantics, and the difference between `const`, literals, references, and owned values.

## Prevent Logical Errors from Accidental Reuse

When a value is moved, the compiler prevents further use, helping avoid bugs such as accidental reuse in a loop. If a type silently becomes `Copy`, such protections disappear — and implicit reuse may lead to subtle logic errors. Requiring `.copy` makes reuse visible and intentional.

# Guide-level explanation

With explicit copy syntax, all value transfers in Rust follow move semantics by default. To copy a value, you must explicitly use the `.copy` syntax:

```rust
let x = 42;
let y = x.copy; // explicit copy
let z = x;      // move (even though i32 is Copy)
```

## Basic Usage

The `.copy` syntax works anywhere you would normally use a value:

```rust
fn process(n: i32) {}

let x = 42;
process(x.copy);        // pass a copy
let result = x.copy + 5; // use in expressions
```

## Chaining

The `.copy` syntax chains naturally with method calls:

```rust
let point = Point { x: 1, y: 2 };
let result = point.copy.translate(3, 4);
```

## Error Handling

Using `.copy` on a non-`Copy` type results in a compile-time error:

```rust
let vec = vec![1, 2, 3];
let copy = vec.copy; // ERROR: Vec<i32> does not implement Copy
```

## Literals and Constants

Literals like `42`, `"hello"`, and `true` are embedded directly into expressions and don't require `.copy`:

```rust
let a = 42;        // ✅ OK: literal
let b = 1 + 2 + 3; // ✅ OK: arithmetic with literals
```

However, `const` values require `.copy` for consistency with normal bindings:

```rust
const MAX_SIZE: usize = 100;
let limit = MAX_SIZE.copy; // ✅ Explicit copy from const
```

## References

The `.copy` operator does not auto-dereference. It acts on the value as written:

```rust
let a = &42;
let x = a.copy;      // ✅ x: &i32 — copy the reference
let y = (*a).copy;   // ✅ y: i32  — copy the value behind the reference
```

This makes working with references clearer and reinforces Rust's principle that developers should write what they mean.

## Impact on Reading and Understanding Code

This change makes ownership transfers explicit and visible. When reading code, you can immediately see where values are being copied versus moved. This improves code comprehension and makes performance characteristics more apparent.

For existing Rust programmers, this represents a shift from implicit to explicit behavior. The mental model becomes simpler: assume everything moves unless you see `.copy`.

For new Rust programmers, this eliminates the need to memorize which types are `Copy` and provides a consistent rule for all value transfers.

## Migration

When migrating to a new edition that includes this feature, `cargo fix` will automatically insert `.copy` where needed to preserve existing behavior. The compiler will also provide helpful diagnostics to guide manual fixes where automatic migration isn't possible.

# Reference-level explanation

## Syntax and Semantics

The `.copy` syntax is a built-in operator, not a method call. It behaves similarly to `.await` or `?` as a syntactic construct that applies directly to a value.

### Contextual Keyword

The `.copy` syntax introduces `copy` as a **contextual keyword**, valid only in the context of `expr.copy`. It does not affect the ability to use `copy` as a variable, field, or function name outside of that context. This ensures backward compatibility and avoids namespace pollution.

### Grammar

```
copy_expr: expr '.' 'copy'
```

### Type Checking

Using `.copy` on an expression `e` of type `T` is valid if and only if `T: Copy`. The result type is `T`.

### Desugaring

The `.copy` operation desugars to a bitwise copy of the value, identical to the current implicit copying behavior for `Copy` types.

## Interaction with Other Features

### Closure Capture

Using `.copy` inside closures follows normal capture rules:

```rust
let a = MyCopyType(123);
let closure = || {
    let x = a.copy; // captured by reference
};

let closure = move || {
    let x = a.copy; // captured by move, then copied
};
```

### Pattern Matching

Pattern matching follows explicit copy semantics. To copy fields from a destructured `Copy` type, explicit `.copy` is required:

```rust
match point {
    Point(x, y) => {
        let x_copy = x.copy;
        let y_copy = y.copy;
        // use x_copy and y_copy
    }
}
```

### Generics

In generic contexts, `.copy` requires a `Copy` bound:

```rust
fn duplicate<T: Copy>(value: T) -> (T, T) {
    (value, value.copy)
}
```

## Const and Static Items

### Const Items

`const` items require `.copy` for value access:

```rust
const PI: f64 = 3.14159;
let circumference = 2.0 * PI.copy * radius;
```

### Static Items

`static` items are accessed as references. Copying follows normal reference rules:

```rust
static COUNTER: AtomicUsize = AtomicUsize::new(0);
let counter_ref = &COUNTER;     // reference to static
let ref_copy = counter_ref.copy; // copy the reference
```

## Error Messages

The compiler provides clear error messages for invalid `.copy` usage:

```rust
let vec = vec![1, 2, 3];
let copy = vec.copy;
// ERROR: the trait `Copy` is not implemented for `Vec<i32>`
// HELP: consider using `vec.clone()` to create a deep copy
```

## Corner Cases

### Borrowed Values

`.copy` works on borrowed values without auto-dereferencing:

```rust
let x = &42;
let y = x.copy; // copies the reference, not the value
```

### Temporary Values

`.copy` can be applied to temporary values:

```rust
let result = create_point().copy.translate(1, 2);
```

# Drawbacks

**Increased Verbosity**: Code that reuses `Copy` values must now do so explicitly with `.copy`. This adds syntactic overhead to common operations like arithmetic with variables.

**Learning Curve**: Existing Rust developers must adapt to explicit copy syntax, which changes established patterns.

**Compiler Implementation Complexity**: Primitive types are currently always treated as `Copy`, and the compiler may not track their move state. Supporting this change may require deeper changes to MIR and borrow checking to distinguish moves from copies even for scalars.

**Potential for Overuse**: Developers might reflexively add `.copy` without considering whether a move would be more appropriate, potentially leading to unnecessary copies.

# Rationale and alternatives

## Why This Design?

This design is the best because it:

- **Provides Consistency**: Aligns with Rust's existing philosophy of making semantically important operations explicit (like `.await`, `?`, `unsafe`)
- **Eliminates Surprises**: Removes the need to know which types are `Copy` to understand code behavior
- **Improves Readability**: Makes ownership transfers visible in code
- **Prevents Subtle Bugs**: Eliminates silent behavior changes when `Copy` is added or removed from types

## Alternative Designs Considered

### `copy x` Keyword Syntax

```rust
let y = copy x;
```

**Rejected because:**

- Requires a new keyword
- Doesn't chain well with method calls
- Less consistent with existing Rust syntax patterns

### Auto-dereferencing `.copy`

Making `.copy` auto-dereference like method calls.

**Rejected because:**

- Creates ambiguity about whether you're copying a reference or the referenced value
- Inconsistent with the goal of making ownership explicit
- Adds complexity to the mental model

### Status Quo (Keep Implicit Copy)

**Rejected because:**

- Maintains the action-at-a-distance problem
- Keeps the cognitive burden of remembering which types are `Copy`
- Doesn't align with Rust's general philosophy of explicit semantics

## Impact of Not Doing This

Without this change, Rust continues to have:

- Implicit behavior that can silently change when traits are added/removed
- Cognitive overhead in understanding which operations copy vs move
- Difficulty in code review and performance analysis
- Inconsistency with other explicit language features

## Library vs Language

This cannot be implemented as a library feature because it requires compiler support to:

- Override default move semantics
- Provide the `.copy` syntax
- Integrate with the type system and borrow checker

# Prior art

TODO

# Unresolved questions

**Pattern Matching Syntax**: Should future RFCs introduce special syntax for copying in patterns (e.g., `match x { MyStruct { copy a, ref b } => ... }`)?

**Diagnostic Quality**: What specific error messages and suggestions should the compiler provide for common mistakes?

**Generic Bounds**: How should the compiler handle complex generic scenarios where `Copy` bounds might be inferred?

**Performance Impact**: What is the compile-time cost of tracking move/copy semantics for all types, including primitives?

**IDE Integration**: How should language servers and IDEs highlight copy operations to improve developer experience?

# Future possibilities

## Pattern Matching Extensions

Future RFCs could introduce pattern-level copy syntax:

```rust
match point {
    Point { copy x, copy y } => {
        // x and y are copied automatically
    }
}
```

This would provide fine-grained control over ownership in pattern matching, similar to `ref` patterns.

## Lint Extensions

Additional lints could help identify:

- Unnecessary uses of `.copy` where moves would suffice
- Performance hotspots where many copies occur
- Opportunities to restructure code to avoid copies

## Generic Programming Enhancements

The explicit copy syntax could enable more sophisticated generic programming patterns where copy vs move behavior is controlled by type parameters or associated types.

## Tooling Integration

Build tools could provide reports on copy frequency and performance impact, helping developers optimize their code.

## Language Server Features

IDEs could provide visual indicators for copy operations, making them even more visible during development.

This change lays the foundation for a more explicit and predictable ownership model in Rust, while maintaining backward compatibility through edition-based migration.

