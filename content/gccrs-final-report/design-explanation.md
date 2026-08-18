---
title: "Design Note"
description: "The design and implementation of Rust Drop support in gccrs."
weight: 10
---

[&larr; Back to Final Report](/gccrs-final-report/)

**Jump to:**

- [Part 1: Manual Drop Emission](#part-1-manual-drop-emission)
  - [1.1 Tracking Drop candidates and LIFO order](#11-tracking-drop-candidates-and-lifo-order)
  - [1.2 Normal function exits](#12-normal-function-exits)
  - [1.3 Explicit returns](#13-explicit-returns)
  - [Part 1 pull requests and references](#part-1-pull-requests-and-references)
- [Part 2: Structured Drop Cleanup with `TRY_FINALLY_EXPR`](#part-2-structured-drop-cleanup-with-try_finally_expr)
  - [2.1 Moving cleanup into the scope](#21-moving-cleanup-into-the-scope)
  - [2.2 Using the GCC Go frontend as a reference](#22-using-the-gcc-go-frontend-as-a-reference)
  - [2.3 Handling the personality linker problem](#23-handling-the-personality-linker-problem)
  - [2.4 Applying structured cleanup to functions and returns](#24-applying-structured-cleanup-to-functions-and-returns)
  - [2.5 Extending cleanup to unlabeled `break` and `continue`](#25-extending-cleanup-to-unlabeled-break-and-continue)
  - [Part 2 pull requests and references](#part-2-pull-requests-and-references)
- [Part 3: BIR and CFG-Based Drop Analysis](#part-3-bir-and-cfg-based-drop-analysis)
  - [3.1 Why structured cleanup is not enough](#31-why-structured-cleanup-is-not-enough)
  - [3.2 Straight-line BIR Drop analysis](#32-straight-line-bir-drop-analysis)
  - [3.3 Using the analysis in backend cleanup](#33-using-the-analysis-in-backend-cleanup)
  - [3.4 Extending the analysis across a CFG — Merged](#34-extending-the-analysis-across-a-cfg--merged)
  - [3.5 Backend Drop flags for conditional moves — Merged](#35-backend-drop-flags-for-conditional-moves--merged)
  - [Part 3 pull requests and references](#part-3-pull-requests-and-references)

## Part 1: Manual Drop Emission

### 1.1 Tracking Drop candidates and LIFO order

When I started working on Drop support, the compiler first needed to know which variables should be dropped.

In the first version, gccrs only recorded an initialized local binding with a simple name, such as `let x = ...`. If its type implemented `Drop`, gccrs saved it as a Drop candidate in the current block scope. `ref` bindings and subpatterns were not supported yet.

**How the candidates were stored.** A [`DropCandidate`](https://github.com/Rust-GCC/gccrs/blob/ed80bf43fba235fdb53ea6522df261599c318b94/gcc/rust/backend/rust-compile-drop-candidate.h#L28-L35) stored the HIR ID of the binding and its source location:

```cpp
struct DropCandidate
{
  DropCandidate (HirId hirid, location_t locus)
    : hirid (hirid), locus (locus)
  {}

  HirId hirid;
  location_t locus;
};
```

[`Context::block_drop_candidates`](https://github.com/Rust-GCC/gccrs/blob/ed80bf43fba235fdb53ea6522df261599c318b94/gcc/rust/backend/rust-compile-context.h#L443-L447) kept one candidate list for each active block:

```cpp
std::vector<::std::vector<DropCandidate>> block_drop_candidates;
```

[`DropBuilder::note_simple_drop_candidate`](https://github.com/Rust-GCC/gccrs/blob/ed80bf43fba235fdb53ea6522df261599c318b94/gcc/rust/backend/rust-compile-drop-builder.cc#L27-L32) added a candidate to the current block:

```cpp
void
DropBuilder::note_simple_drop_candidate (HirId hirid, location_t locus)
{
  rust_assert (!ctx.block_drop_candidates.empty ());
  ctx.block_drop_candidates.back ().emplace_back (hirid, locus);
}
```

The outer vector acts as a stack of block scopes, and `back()` selects the current block. The HIR ID identifies the binding later. [`Context::push_block` and `Context::pop_block`](https://github.com/Rust-GCC/gccrs/blob/ed80bf43fba235fdb53ea6522df261599c318b94/gcc/rust/backend/rust-compile-context.h#L103-L123) create and remove the per-block lists.

---

At the end of the scope, the compiler added Drop calls in reverse declaration order. For example:

```rust
fn f() {
    let a = Droppable("a");
    let b = Droppable("b");
}
```

The compiler needed to run:

```text
drop(b)
drop(a)
```

The core loop in the gccrs source is small:

```cpp
for (auto it = drop_candidates.rbegin ();
     it != drop_candidates.rend (); ++it)
```

`rbegin()` visits the newest binding first, so gccrs emits `drop(b)` before `drop(a)`. [View the full implementation](https://github.com/Rust-GCC/gccrs/blob/ed80bf43fba235fdb53ea6522df261599c318b94/gcc/rust/backend/rust-compile-drop.cc#L94-L113).

The first block-scope implementation was added in [PR #4564](https://github.com/Rust-GCC/gccrs/pull/4564), with more tests for LIFO order and nested scopes in [PR #4632](https://github.com/Rust-GCC/gccrs/pull/4632).

[PR #4586](https://github.com/Rust-GCC/gccrs/pull/4586) moved candidate tracking behind `DropBuilder`. This refactoring did not change Drop behavior, but it gave later work a clearer interface.

I then used this manual approach to add Drop support for normal function exits and explicit returns.

### 1.2 Normal function exits

Normal function exit means that control reaches the end of the function body without an explicit `return`. This work covered three cases.

**1. Unit tail expressions**

```rust
fn unit_tail_call() {
    let _x = Droppable;
    foo()
}

fn unit_tail_literal() {
    let _x = Droppable;
    ()
}
```

```text
unit_tail_call():
    run foo()
    Drop _x
    function ends

unit_tail_literal():
    reach the () tail expression
    Drop _x
    function ends
```

[View `drop-function-scope-unit-tail.rs`](https://github.com/Rust-GCC/gccrs/blob/76397e896150e3bf04567fc8df45ef780df4e501/gcc/testsuite/rust/execute/drop-function-scope-unit-tail.rs), added in [PR #4591](https://github.com/Rust-GCC/gccrs/pull/4591).

**2. Function parameters**

Function parameters must also be dropped when the function ends. The body locals are dropped before the parameters:

```rust
fn named_param(_p: ParamDroppable) {
    let _l = LocalDroppable;
}
```

```text
Drop _l
Drop _p
function ends
```

[View `drop-function-params.rs`](https://github.com/Rust-GCC/gccrs/blob/76397e896150e3bf04567fc8df45ef780df4e501/gcc/testsuite/rust/execute/drop-function-params.rs), added in [PR #4591](https://github.com/Rust-GCC/gccrs/pull/4591). The testcase covers both named and wildcard parameters.

**3. Non-unit tail expression**

```rust
fn f() -> i32 {
    let _x = Droppable;
    foo()
}
```

```text
run foo()
save the result
Drop _x
return the saved result
```

[View `drop-function-scope-non-unit-tail.rs`](https://github.com/Rust-GCC/gccrs/blob/99edb212c204385d994c72b85e8098be6f464226/gcc/testsuite/rust/execute/drop-function-scope-non-unit-tail.rs), added in [PR #4602](https://github.com/Rust-GCC/gccrs/pull/4602).

### 1.3 Explicit returns

An explicit return uses the `return` keyword and leaves the function immediately. The testcase in [PR #4621](https://github.com/Rust-GCC/gccrs/pull/4621) covers four cases.

**1. Unit return**

```rust
fn unit_return() {
    let _x = UnitDroppable;
    return;
}
```

```text
Drop _x
return
```

**2. Unit return expression**

```rust
fn unit_return_expr() {
    let _x = UnitExprDroppable;
    return make_unit();
}
```

```text
run make_unit()
Drop _x
return ()
```

**3. Non-unit return expression**

```rust
fn non_unit_return() -> i32 {
    let _x = NonUnitDroppable;
    return make_value();
}
```

```text
run make_value()
save the result
Drop _x
return the saved result
```

**4. Return from a nested block**

```rust
fn nested_return() {
    let _outer = OuterDroppable;

    {
        let _inner = InnerDroppable;
        return;
    }
}
```

```text
Drop _inner
Drop _outer
return
```

[View `drop-explicit-return.rs`](https://github.com/Rust-GCC/gccrs/blob/adde96a15ced7011e432dadee73c19fc59eb9178/gcc/testsuite/rust/execute/drop-explicit-return.rs).

---

Philip Herron suggested using `TRY_FINALLY_EXPR` so that gccrs did not need to insert Drop calls directly at each `return`. Part 2 explains that design.

### Part 1 pull requests and references

- [PR #4559: Register the Drop lang item](https://github.com/Rust-GCC/gccrs/pull/4559)
- [PR #4564: Add Drop support for block-local variables](https://github.com/Rust-GCC/gccrs/pull/4564)
- [PR #4586: Refactor CompileDrop and add DropBuilder](https://github.com/Rust-GCC/gccrs/pull/4586)
- [PR #4591: Support function-scope Drops on normal function exit](https://github.com/Rust-GCC/gccrs/pull/4591)
- [PR #4602: Evaluate non-unit tail expressions before Drops](https://github.com/Rust-GCC/gccrs/pull/4602)
- [PR #4621: Handle explicit-return Drops](https://github.com/Rust-GCC/gccrs/pull/4621)
- [PR #4632: Add LIFO and nested-scope tests](https://github.com/Rust-GCC/gccrs/pull/4632)

## Part 2: Structured Drop Cleanup with `TRY_FINALLY_EXPR`

### 2.1 Moving cleanup into the scope

Philip's suggestion was to create a `TRY_FINALLY_EXPR` when gccrs lowered a `HIR::BlockExpr`.

The block body would be placed in the main region. The Drop calls for the block would be placed in the cleanup region:

```text
TRY_FINALLY_EXPR {
    block body
} cleanup {
    Drop the local values in this block
}
```

At the GCC tree level, the generic gccrs backend helper builds this node with `TRY_FINALLY_EXPR`. [The source takes the compiled body and cleanup as its two operands](https://github.com/Rust-GCC/gccrs/blob/2a1f86dfe935a7309198cc9b66f45d62b9b37cf9/gcc/rust/rust-gcc.cc#L1722-L1729).

gccrs now attaches the Drop calls to each scope once. Nested scopes create nested cleanup regions, so a return does not need to rebuild their Drop calls.

### 2.2 Using the GCC Go frontend as a reference

I then studied how the GCC Go frontend implements `defer`. Go's `defer` and Rust's Drop are different features. A deferred Go call runs when its function exits, not when any block ends. However, the Go frontend still gave me a useful example of how GCC represents code that must run when a function exits.

In the GCC Go backend, [a final cleanup is lowered with `TRY_FINALLY_EXPR`](https://github.com/gcc-mirror/gcc/blob/9e54d865af9361413b9c84e11dddfe652cb4ca81/gcc/go/go-gcc.cc#L2392-L2423).

Conceptually, nested Drop scopes can be represented like this:

```text
try {
    block body

    try {
        inner block body
    } finally {
        drop inner values
    }
} finally {
    drop outer values
}
```

This is a compiler IR structure. It does not add `try` or `finally` syntax to Rust. The inner cleanup runs first, which keeps the correct LIFO Drop order.

### 2.3 Handling the personality linker problem

When I tested the first `TRY_FINALLY_EXPR` version, the Drop order was correct, but `drop-nested-block-scope.rs` failed to link:

```text
undefined reference to `__gccrs_personality_v0`
```

Pierre-Emmanuel Patry explained that a [personality routine](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html#base-personality) is used during stack unwinding. gccrs does not support stack unwinding yet, so it does not provide this symbol.

Compiling the test with `-fno-exceptions` removed the reference, but this was only an experiment. It was not a suitable compiler-wide solution.

Pierre-Emmanuel then pointed me to the [GCC Internals documentation](https://gcc.gnu.org/onlinedocs/gccint/Cleanups.html) and the historical [rustc_codegen_gcc source](https://github.com/rust-lang/gcc/blob/6351b299c2f57e6ff60c41f161325bdd2987bd1b/gcc/jit/jit-playback.cc#L2385-L2404). They showed that `EH_ELSE_EXPR` can split normal cleanup from exceptional cleanup.

The key part I added to `Context::pop_block_impl` is:

```cpp
tree exceptional_cleanup = build_empty_stmt (cleanup_locus);
tree cleanup_selector
  = build2_loc (cleanup_locus, EH_ELSE_EXPR, void_type_node, cleanup,
                exceptional_cleanup);

tree try_finally
  = Backend::exception_handler_statement (body, NULL_TREE,
                                          cleanup_selector,
                                          cleanup_locus);
```

`cleanup` is the normal arm, and `exceptional_cleanup` is the empty exceptional arm. With this structure, the testcase linked without `__gccrs_personality_v0` and still printed the correct Drop order. [View the full gccrs source](https://github.com/Rust-GCC/gccrs/blob/2a1f86dfe935a7309198cc9b66f45d62b9b37cf9/gcc/rust/backend/rust-compile-context.h#L434-L452).

### 2.4 Applying structured cleanup to functions and returns

Once the block-scope cleanup was ready, I applied the same design to function scopes in [PR #4711](https://github.com/Rust-GCC/gccrs/pull/4711).

The final version of [PR #4621](https://github.com/Rust-GCC/gccrs/pull/4621) then removed the manual explicit-return Drop code. Normal function exits and explicit returns could now use the same cleanup structure.

The argument scope and function-body scope were later separated in [PR #4718](https://github.com/Rust-GCC/gccrs/pull/4718), which made the Drop scopes more closely match rustc.

{{< rustc-reference >}}
**rustc 1.49 reference.** In the [PR #4591 discussion](https://github.com/Rust-GCC/gccrs/pull/4591#discussion_r3499748547), I compared the gccrs scope structure with rustc 1.49. rustc creates an outer [`Arguments` scope](https://github.com/rust-lang/rust/blob/e1884a8e3c3e813aada8254edfa120e85bf5ffca/compiler/rustc_mir_build/src/build/mod.rs#L608-L628) and [schedules parameter Drops in that scope](https://github.com/rust-lang/rust/blob/e1884a8e3c3e813aada8254edfa120e85bf5ffca/compiler/rustc_mir_build/src/build/mod.rs#L882-L888). This makes body locals drop before function parameters.
{{< /rustc-reference >}}

### 2.5 Extending cleanup to unlabeled `break` and `continue`

`break` and `continue` can leave a block before its end, so the local values in that block must be dropped first.

[PR #4710](https://github.com/Rust-GCC/gccrs/pull/4710) includes this `break` testcase:

```rust
fn test_break() {
    loop {
        let _outer = BreakOuter;

        {
            let _inner = BreakInner;
            break;
        }
    }
}
```

```text
Drop _inner
Drop _outer
leave the loop
```

The same testcase also checks `continue`:

```rust
fn test_continue() {
    let mut done = false;

    loop {
        if done {
            break;
        }

        {
            let _outer = ContinueOuter;

            {
                let _inner = ContinueInner;
                done = true;
                continue;
            }
        }
    }
}
```

```text
Drop _inner
Drop _outer
start the next iteration
```

The lowered `break` or `continue` stays inside the `TRY_FINALLY_EXPR` body, so leaving that body runs the existing cleanup. The full testcase also covers tail-position forms, `while` loops, and `break` with a value. [View `drop-unlabeled-break-continue.rs`](https://github.com/Rust-GCC/gccrs/blob/c1150be2ca0ec323dc8ff7bad682551caf57a35f/gcc/testsuite/rust/execute/drop-unlabeled-break-continue.rs).

Labeled jumps and a `return` used as the final expression of a block were outside this PR.

This work also exposed [a separate double-Drop case](https://github.com/Rust-GCC/gccrs/issues/4717). One loop path could reach cleanup before a local value was initialized. This showed the next problem: structured cleanup knew where to run Drop, but it still could not decide whether a value should be dropped. Part 3 describes the analysis for that decision.

### Part 2 pull requests and references

- [PR #4685: Emit block-scope Drops through TRY_FINALLY_EXPR](https://github.com/Rust-GCC/gccrs/pull/4685)
- [PR #4711: Apply try/finally cleanup to function-scope Drops](https://github.com/Rust-GCC/gccrs/pull/4711)
- [PR #4621: Handle explicit-return Drops with try/finally cleanup](https://github.com/Rust-GCC/gccrs/pull/4621)
- [PR #4718: Separate argument and function-body Drop scopes](https://github.com/Rust-GCC/gccrs/pull/4718)
- [PR #4710: Run block cleanup for unlabeled break and continue](https://github.com/Rust-GCC/gccrs/pull/4710)
- [Itanium C++ ABI: Exception handling and personality routines](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html#base-personality)
- [GCC Internals documentation](https://gcc.gnu.org/onlinedocs/gcc-16.1.0/gccint.pdf)
- [`TRY_FINALLY_EXPR` and `EH_ELSE_EXPR` in the GCC fork used by rustc_codegen_gcc](https://github.com/rust-lang/gcc/blob/6351b299c2f57e6ff60c41f161325bdd2987bd1b/gcc/jit/jit-playback.cc#L2385-L2404)

## Part 3: BIR and CFG-Based Drop Analysis

### 3.1 Why structured cleanup is not enough

Structured cleanup tells the compiler where a Drop call should run. However, it does not tell the compiler whether a value still needs to be dropped.

For example:

```rust
fn f() {
    let x = Droppable("x");
    let y = x;
}
```

The value is moved from `x` to `y`. At the end of the function, the compiler should drop `y`, but it must not drop `x` again.

To handle this, gccrs needs an analysis that tracks whether a local variable still holds a value. I implemented this analysis in BIR, before the backend creates the final Drop cleanup. The current version handles direct moves of the whole variable. It does not handle moving only one field yet.

### 3.2 Straight-line BIR Drop analysis

The first version handles straight-line control flow. This means that the function has no branches or loops.

The analysis adds BIR Drop statements at scope exits. It then follows assignments and moves through the function. When a whole local variable is assigned a value, it becomes initialized. If BIR marks the assignment as a move from another whole local, the source becomes uninitialized.

The merged source updates the state in this way:

```cpp
PlaceId lhs = place;
AbstractExpr &expr = statement.get_expr ();

if (expr.get_kind () == ExprKind::ASSIGNMENT)
  {
    PlaceId rhs = static_cast<Assignment &> (expr).get_rhs ();
    const Place &rhs_place = function.place_db[rhs];

    if (rhs_place.kind == Place::VARIABLE
        && rhs_place.should_be_moved ())
      initialized[rhs.value] = false;
  }

initialized[lhs.value] = true;
```

[View the merged BIR analysis source](https://github.com/Rust-GCC/gccrs/blob/7a3c1de74ae6e3cf18b3ccf81f1a9f356f3e5c74/gcc/rust/checks/errors/borrowck/rust-bir-drop-analysis.cc#L85-L100).

Each Drop statement is then given a Drop style:

```text
Static: the value is initialized and should be dropped
Dead:   the value is uninitialized here and should not be dropped
```

For the earlier move example, the result is:

```text
Drop(y): Static
Drop(x): Dead
```

Function arguments start in the initialized state.

This straight-line analysis was added in [PR #4730](https://github.com/Rust-GCC/gccrs/pull/4730).

### 3.3 Using the analysis in backend cleanup

[PR #4748](https://github.com/Rust-GCC/gccrs/pull/4748) connected the analysis to backend cleanup. The backend emits `Static` Drops and skips `Dead` Drops. This support is now merged in the BIR/borrow-check path.

### 3.4 Extending the analysis across a CFG — Merged

**Status: Merged.** The BIR analysis in this section was merged in [PR #4777](https://github.com/Rust-GCC/gccrs/pull/4777).

Conditional control flow is more difficult because a value may be moved on one path but not another:

```rust
fn f(condition: bool) {
    let x = Droppable("x");

    if condition {
        let y = x;
    }
}
```

The control-flow graph can be viewed as:

```text
                 move x
                /      \
initialize x --          -- join -- Drop(x)
                \      /
                 keep x
```

At the start of every basic block, the analysis records whether each tracked local may be initialized and whether it may be uninitialized.

At a CFG join, `into` holds the state already stored for the block. `from` is the state from a new incoming path. The actual `merge_state` function is:

```cpp
static bool
merge_state (BlockInitializationState &into,
             const BlockInitializationState &from)
{
  bool changed = false;

  if (!into.reachable)
    {
      into = from;
      return !changed;
    }

  for (size_t i = 0; i < into.maybe_initialized.size (); i++)
    {
      bool maybe_initialized
        = into.maybe_initialized[i] || from.maybe_initialized[i];

      bool maybe_uninitialized
        = into.maybe_uninitialized[i] || from.maybe_uninitialized[i];

      changed |= maybe_initialized != into.maybe_initialized[i];
      changed |= maybe_uninitialized != into.maybe_uninitialized[i];

      into.maybe_initialized[i] = maybe_initialized;
      into.maybe_uninitialized[i] = maybe_uninitialized;
    }

  return changed;
}
```

The first incoming path copies the full state. Later paths use `||` to keep every possible state. If the result changes, the worklist processes the block again.

[View the merged implementation](https://github.com/Rust-GCC/gccrs/blob/7e91f41cc4c481b104d7fdc4dce795c82a5372c7/gcc/rust/checks/errors/borrowck/rust-bir-drop-analysis.cc#L101-L129).

The worklist sends each result to the block's successors:

```cpp
while (!worklist.empty ())
  {
    BasicBlockId block_id = worklist.back ();
    worklist.pop_back ();
    queued[block_id.value] = false;

    BlockInitializationState state = entry_states[block_id.value];
    BasicBlock &block = function.basic_blocks[block_id];

    for (Statement &statement : block.statements)
      update_state_for_statement (function, statement, state);

    for (BasicBlockId successor : block.successors)
      {
        bool state_changed =
          merge_state (entry_states[successor.value], state);

        if (state_changed && !queued[successor.value])
          {
            worklist.push_back (successor);
            queued[successor.value] = true;
          }
      }
  }
```

The worklist processes a successor again only when its entry state changes. `queued` prevents duplicate entries. An empty worklist means that the block-entry states are stable.

[View the worklist implementation](https://github.com/Rust-GCC/gccrs/blob/7e91f41cc4c481b104d7fdc4dce795c82a5372c7/gcc/rust/checks/errors/borrowck/rust-bir-drop-analysis.cc#L222-L245).

After the block-entry states stop changing, the analysis walks through each reachable block again. It chooses the Drop style from the state just before each Drop runs:

```text
initialized only                 -> Static
uninitialized only               -> Dead
initialized and uninitialized    -> Conditional
```

The [`classify_drop` implementation](https://github.com/Rust-GCC/gccrs/blob/7e91f41cc4c481b104d7fdc4dce795c82a5372c7/gcc/rust/checks/errors/borrowck/rust-bir-drop-analysis.cc#L176-L189) directly maps those two state bits to the three styles.

### 3.5 Backend Drop flags for conditional moves — Merged

**Status: Merged.** The backend implementation in this section was merged in [PR #4798](https://github.com/Rust-GCC/gccrs/pull/4798).

A `Conditional` Drop cannot always run and cannot always be skipped. The backend needs a runtime flag that records whether the value is still initialized on the path that was taken.

The backend then creates a Drop flag for `x`. Conceptually, the generated backend code for the example above behaves like this:

```text
try {
    bool x_drop_flag = false;
    let x = Droppable("x");
    x_drop_flag = true;

    if (condition) {
        let y = x;
        x_drop_flag = false;
    }
} finally {
    if (x_drop_flag) {
        x_drop_flag = false;
        Drop(x);
    }
}
```

The merged backend code follows the same lifecycle in four places:

1. [Create the flag before compiling the initializer](https://github.com/Rust-GCC/gccrs/blob/0fe4006f053bf14a90db1fb5c7524c001ce76714/gcc/rust/backend/rust-compile-stmt.cc#L73-L82).
2. [Set it after the initializer finishes](https://github.com/Rust-GCC/gccrs/blob/0fe4006f053bf14a90db1fb5c7524c001ce76714/gcc/rust/backend/rust-compile-pattern.cc#L1356-L1361).
3. [Clear the source flag when compiling a move](https://github.com/Rust-GCC/gccrs/blob/0fe4006f053bf14a90db1fb5c7524c001ce76714/gcc/rust/backend/rust-compile-stmt.cc#L99-L109).
4. [Guard the Drop call and clear the flag before dropping](https://github.com/Rust-GCC/gccrs/blob/0fe4006f053bf14a90db1fb5c7524c001ce76714/gcc/rust/backend/rust-compile-drop.cc#L120-L130).

In `let y = x`, the move expression and the source local have different HIR IDs. `move_sources` maps the expression ID to the local ID, so the backend can clear the correct Drop flag. [The PR discussion gives a concrete example](https://github.com/Rust-GCC/gccrs/pull/4798#discussion_r3802207514).

### Part 3 pull requests and references

- [PR #4730: Add straight-line BIR Drop-style analysis](https://github.com/Rust-GCC/gccrs/pull/4730) — merged
- [PR #4748: Use BIR Drop analysis in backend cleanup](https://github.com/Rust-GCC/gccrs/pull/4748) — merged
- [PR #4777: Handle conditional moves in BIR Drop analysis](https://github.com/Rust-GCC/gccrs/pull/4777) — merged
- [PR #4798: Generate backend Drop flags for conditional moves](https://github.com/Rust-GCC/gccrs/pull/4798) — merged
