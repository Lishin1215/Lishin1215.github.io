---
title: "Weekly Updates"
description: "A week-by-week record of my GSoC progress on Drop support in gccrs."
weight: 20
---

[&larr; Back to Final Report](/gccrs-final-report/)

This is the rolling agenda for my GSoC sync-ups on Drop support in gccrs.

[View the GSoC project page](https://summerofcode.withgoogle.com/programs/2026/projects/rpCOU67B)

## 19 August 2026

### Since the last update

- Pushed two patches:
  - [Add execute tests for supported Drop behavior](https://github.com/Rust-GCC/gccrs/pull/4789)
  - [Apply conditional BIR Drop-style analysis to backend cleanup](https://github.com/Rust-GCC/gccrs/pull/4798)

### This week

- Address review comments and focus on getting the existing PRs merged.

## 12 August 2026

### Since the last update

- Pushed a patch for [conditional BIR Drop-style analysis](https://github.com/Rust-GCC/gccrs/pull/4775).
- Temporarily closed the PR because I wanted to revise the design.

### This week

- Apply the conditional BIR Drop-style analysis to backend cleanup.
- Add more test cases for behavior that is already supported but not yet covered.
- Address review comments and focus on getting the existing PRs merged.

## 5 August 2026

### Last week

- Pushed two patches:
  - [Straight-line BIR drop-style analysis](https://github.com/Rust-GCC/gccrs/pull/4730)
  - [Apply straight-line BIR analysis to the backend](https://github.com/Rust-GCC/gccrs/pull/4748)

### This week

- Investigate conditional cases and implement conditional BIR drop-style analysis.
- Apply the analysis to the backend and add test cases. This will support conditional moves such as:

```rust
fn f(cond: bool) {
    let x = Droppable("x");
    if cond {
        let y = x;
    }
}
```

## 29 July 2026

### Last week

- Pushed a patch separating argument and function-body Drop scopes, making gccrs more closely aligned with rustc: [issue #4712](https://github.com/Rust-GCC/gccrs/issues/4712), [PR #4718](https://github.com/Rust-GCC/gccrs/pull/4718).
- Opened [issue #4717](https://github.com/Rust-GCC/gccrs/issues/4717) about a double Drop when implementing unlabeled `break` support.
- Investigated BIR in gccrs and how MIR implements control-flow graphs in rustc.

### This week

1. Implement straight-line BIR Drop-style analysis.
2. Use the BIR Drop analysis results to control backend Drop cleanup, supporting cases such as:

```rust
let x = Droppable { i: 1 };
let y = x;
```

## 22 July 2026

### Last week

- Took a few days off.
- Resolved conflicts and applied try/finally cleanup to function scope and explicit returns: [PR #4711](https://github.com/Rust-GCC/gccrs/pull/4711) (merged) and [PR #4621](https://github.com/Rust-GCC/gccrs/pull/4621) (under review).
- Added Drop support for unlabeled `break` and `continue`: [PR #4710](https://github.com/Rust-GCC/gccrs/pull/4710).

### This week

1. Open an issue and begin refactoring gccrs to align more closely with rustc, following [this review discussion](https://github.com/Rust-GCC/gccrs/pull/4591#discussion_r3499748547).
2. Decide whether to keep expanding Drop support for labeled `continue`, `break`, moves, and generics, or prioritize aligning the current gccrs Drop structure with rustc.

## 15 July 2026

### Last week

- Updated the try/finally PR to use `EH_ELSE_EXPR`. With this change, the tested program no longer referenced `__gccrs_personality_v0`: [PR #4685](https://github.com/Rust-GCC/gccrs/pull/4685).
- Applied try/finally cleanup to function scope and explicit returns locally, pending the merge of PR #4685.

### This week

1. Add support for unlabeled `break` and `continue`.
2. Open an issue and begin refactoring gccrs to align more closely with rustc, following [this review discussion](https://github.com/Rust-GCC/gccrs/pull/4591#discussion_r3499748547).
3. Address review feedback, resolve conflicts, and work towards merging the open PRs.

## 8 July 2026

### Last week

- Responded to feedback on the existing PRs:
  - [Non-unit tail function case](https://github.com/Rust-GCC/gccrs/pull/4602)
  - [Function-scope Drop support](https://github.com/Rust-GCC/gccrs/pull/4591)
- Studied how the Go frontend handles `defer` using try/finally-style cleanup and used it as a reference for Rust Drop cleanup.
- Implemented block-scope Drop cleanup using `TRY_FINALLY_EXPR` and opened [draft PR #4685](https://github.com/Rust-GCC/gccrs/pull/4685).

### This week

1. Discuss how to handle the `__gccrs_personality_v0` issue and follow up based on maintainer feedback.
2. Continue function-scope and explicit-return cleanup if the `TRY_FINALLY_EXPR` direction is confirmed.
3. Work on Drop support for unlabeled `break` and `continue`:

```rust
loop {
    let x = Droppable;

    if cond() {
        break; // drop x before leaving the loop
    }

    if other_cond() {
        continue; // drop x before the next iteration
    }

    work();
}
```

## 1 July 2026

### Last week

- Pushed PRs to:
  - [Support explicit returns](https://github.com/Rust-GCC/gccrs/pull/4621)
  - Add more test cases for the [non-unit tail function case](https://github.com/Rust-GCC/gccrs/pull/4602)
  - Test [LIFO Drop order and nested scopes](https://github.com/Rust-GCC/gccrs/pull/4632)

### This week

1. Address review comments, resolve conflicts, and work towards merging the open PRs.
2. Develop a try/finally-based design so Drop calls do not need to be emitted manually at every exit point. For example:

```rust
fn f() {
    let a = Droppable;
    {
        let x = Droppable;
    }
    work();
}
```

This could be lowered into a structure like:

```text
try {
  let a = Droppable;

  try {
    let x = Droppable;
  } finally {
    drop(x);
  }

  work();
} finally {
  drop(a);
}
```

## 24 June 2026

### Last week

- Pushed [PR #4586](https://github.com/Rust-GCC/gccrs/pull/4586) to refactor the Drop infrastructure.
- Started work on function-scope cases, including unit tail expressions and function parameter drops: [PR #4591](https://github.com/Rust-GCC/gccrs/pull/4591).

### This week

1. Update the Drop infrastructure refactor and function-scope support PRs based on feedback.
2. Start implementing Drop support for explicit return expressions:

```rust
// 1. Unit explicit return
fn f() {
    let _x = Droppable;
    return;
}

// 2. Unit explicit return expression
fn f() {
    let _x = Droppable;
    return make_unit();
}

// 3. Non-unit explicit return expression
fn f() -> i32 {
    let _x = Droppable;
    return make_value();
}

// 4. Nested explicit returns
fn f() {
    let _outer = Droppable;
    {
        let _inner = Droppable;
        return;
    }
}
```

## 17 June 2026

### Last week

- Studied rustc 1.49 and made a table of Drop insertion points.
- Pushed two commits:
  1. Supported the basic function-call case: `fn main() -> i32 { let _x = Droppable; 0 }`.
  2. Addressed feedback by adding missing copyright headers, fixing an include guard name, and asserting that Drop lookup returns a single function candidate.

### This week

1. Refactor `CompileDrop` to store and reuse `Context` as a class member instead of passing `Context *ctx` to each method.
2. Explore new `DropBuilder` APIs such as `peek_block_drop_candidates` and `note_simple_drop_candidate`.
3. Complete function-scope Drop support, including unit tail expressions and, possibly the following week, function parameters:

```rust
// Unit tail expression
fn foo() {}

fn f() {
    let _x = Droppable;
    foo()
}

// Function parameter
fn foo(x: Droppable) {}
```

## 10 June 2026

### Action items

- Study the rustc 1.49 source to locate the Drop infrastructure, establish a reference point, and understand the expected behavior in more detail.
- Consider suitable APIs for the Drop implementation.

### Last week

- Completed all scheduled tasks.
- Moved the Drop helper into separate files: [commit cf048a3](https://github.com/Rust-GCC/gccrs/commit/cf048a32a0628c36841c1ccb0e122703b86a7314).

### This week

- Work on function-call cases.
- Investigate Drop insertion points in rustc 1.49.

## 3 June 2026

### Last week

- Pushed a draft PR for the simple block-exit Drop implementation and added comments to explain the code.

### This week

1. Add more test cases for the current scope.
2. Update the draft PR based on feedback.
3. Investigate the backend Drop insertion point. Block cases are supported, but function-call cases such as the following are not yet supported:

```rust
fn main() -> i32 {
    let _x = Droppable;
    0
}
```

### API discussion

Arthur suggested that future Drop APIs could look something like this:

```cpp
auto dropper = DropBuilder();

/* ... */

dropper.note_simple_candidate(...);

/* ... */

dropper.peek_block_candidates();

/* ... */

for (auto stmt : dropper.compile_drop_calls())
    ctx->add_statement(stmt);
```

## 27 May 2026

- First introduction meeting. 🎉
- Opened [PR #4559](https://github.com/Rust-GCC/gccrs/pull/4559) to add Drop as a lang item; it was reviewed, approved, and merged.
- Planned a first WIP/draft PR implementing basic drops for the following week.
