---
title: "Project Overview"
description: "A short summary of Rust Drop, the work completed in gccrs, and future work."
weight: 5
page_class: "project-overview-page"
---

[&larr; Back to Final Report](/gccrs-final-report/)

- **Project:** [Add infrastructure to handle the `Drop` trait](https://summerofcode.withgoogle.com/programs/2026/projects/rpCOU67B)
- **Organization:** [GNU Compiler Collection (GCC)](https://summerofcode.withgoogle.com/programs/2026/organizations/gnu-compiler-collection-gcc)
- **Mentors:** Arthur Cohen and Pierre-Emmanuel Patry

During Google Summer of Code 2026, I worked on adding the basic infrastructure for Rust's `Drop` trait to gccrs. The work involved C++, Rust, GCC tree expressions, BIR, control-flow graphs, and DejaGNU tests.

## 1. What is Drop?

Rust uses the `Drop` trait for value cleanup. The cleanup runs after a value is no longer needed.

```rust
fn example() {
    let a = Droppable("a");

    {
        let b = Droppable("b");
    }
}
```

In this example, the inner block ends first, so `b` is dropped first. Later, `a` is dropped at the end of the function. Values in the same scope are dropped in reverse declaration order. This is called LIFO order.

Early exits also need cleanup. Examples include `return`, `break`, and `continue`. A move transfers a value to a new location. The compiler must not drop the old location. The programmer writes the cleanup code in `Drop::drop`. The compiler inserts the call at the correct scope exit. It also skips the call for a moved value.

## 2. Implementation stages

Before this project, gccrs did not automatically insert `Drop` calls at scope exit.

The [original project plan](https://summerofcode.withgoogle.com/programs/2026/projects/rpCOU67B) started with a small and testable subset: simple, non-generic local variables. The project later added multiple locals, correct LIFO order, nested scopes, and whole-local moves in straight-line and conditional control flow. Temporaries, partial initialization, and generic cases were not completed and remain future work.

| Stage | Main result | Related pull requests |
| --- | --- | --- |
| **1. Initial manual Drop emission** | Added the first Drop calls for supported locals and parameters, including LIFO order, nested scopes, and normal function exits. The first explicit-return implementation also inserted Drop calls manually. | [#4559](https://github.com/Rust-GCC/gccrs/pull/4559), [#4564](https://github.com/Rust-GCC/gccrs/pull/4564), [#4591](https://github.com/Rust-GCC/gccrs/pull/4591), [#4602](https://github.com/Rust-GCC/gccrs/pull/4602), [#4621](https://github.com/Rust-GCC/gccrs/pull/4621), [#4632](https://github.com/Rust-GCC/gccrs/pull/4632) |
| **2. Structured cleanup with `TRY_FINALLY_EXPR`** | Replaced manual cleanup with one cleanup structure for block and function scopes, including explicit returns and supported unlabeled `break` and `continue` paths. | [#4621](https://github.com/Rust-GCC/gccrs/pull/4621), [#4685](https://github.com/Rust-GCC/gccrs/pull/4685), [#4710](https://github.com/Rust-GCC/gccrs/pull/4710), [#4711](https://github.com/Rust-GCC/gccrs/pull/4711), [#4718](https://github.com/Rust-GCC/gccrs/pull/4718) |
| **3. Move-aware cleanup with BIR and CFG** | Added Drop-state analysis for straight-line and conditional whole-local moves, then connected it to backend cleanup with Drop flags. | [#4730](https://github.com/Rust-GCC/gccrs/pull/4730), [#4748](https://github.com/Rust-GCC/gccrs/pull/4748), [#4777](https://github.com/Rust-GCC/gccrs/pull/4777), [#4798](https://github.com/Rust-GCC/gccrs/pull/4798) |

Supporting work included the `DropBuilder` refactoring in [PR #4586](https://github.com/Rust-GCC/gccrs/pull/4586) and additional execute-test coverage in [PR #4789](https://github.com/Rust-GCC/gccrs/pull/4789).

[PR #4621](https://github.com/Rust-GCC/gccrs/pull/4621) appears in both stages because it began with manual cleanup and its merged version uses structured cleanup.

This is a useful subset of Drop behavior, but it is not full Rust Drop support. The [Pull Request Notes](/gccrs-final-report/pull-request-notes/) show the exact implementation and tests behind each item.

## 3. Future work

### 3.1 Complete labeled `break` and `continue` cleanup

The current cleanup handles the tested unlabeled `break` and `continue` cases. One loop-body case can still run Drop twice when a `break` appears before a later local declaration. This is tracked in [issue #4717](https://github.com/Rust-GCC/gccrs/issues/4717).

Labeled `break` and `continue` are not supported yet. A labeled jump may leave several nested scopes. gccrs must clean every scope between the jump and its target.

### 3.2 Support partial moves

A move can take one field but leave the other field in place:

```rust
struct Pair {
    left: Droppable,
    right: Droppable,
}

let pair = Pair {
    left: Droppable("left"),
    right: Droppable("right"),
};

let a = pair.left;
```

`pair.left` was moved, but `pair.right` still needs cleanup. gccrs must track the fields separately. The same tracking is needed when only some fields have been initialized.

### 3.3 Support temporary values

A value can need cleanup even when it is not stored in a named local variable:

```rust
fn f() {
    make_droppable();
}
```

The result of `make_droppable()` is a temporary value. gccrs must Drop it at the end of its temporary scope.

### 3.4 Support generic Drop

The same generic function can receive types with different cleanup needs:

```rust
fn finish<T>(value: T) {
    // value is dropped when the function ends, if needed
}

fn main() {
    finish(42);               // i32 does not need Drop
    finish(Droppable("x"));   // Droppable must run Drop
}
```

gccrs must generate the correct cleanup for each concrete `T`. This still needs design and implementation work.

## 4. Reflection and Thanks

This project taught me about GCC tree expressions, structured cleanup, the boundary between the gccrs frontend and backend, BIR, CFG data-flow analysis, and compiler testing.

I wanted to work on generic Drop support. I did not reach this part during GSoC. BIR and CFG analysis took more time than I expected. I also spent time making the gccrs scope design closer to the rustc design. Both tasks were necessary for correct move handling. I chose to finish them first.

I would like to thank my mentors, Arthur Cohen and Pierre-Emmanuel Patry, for their reviews, explanations, and design feedback throughout the project.

[Read the Design Note &rarr;](/gccrs-final-report/design-explanation/) · [Browse the Pull Request Notes &rarr;](/gccrs-final-report/pull-request-notes/) · [View the Weekly Updates &rarr;](/gccrs-final-report/weekly-updates/)
