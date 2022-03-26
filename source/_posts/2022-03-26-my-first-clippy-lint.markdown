---
layout: post
title: "My First Clippy Lint"
titleen: "My First Clippy Lint"
titlesp: "My First Clippy Lint"
date: 2022-03-26 15:13:35 +0100
comments: false
meta: true
categories: [Rust]
---

Recently I wrote my first [Clippy](https://github.com/rust-lang/rust-clippy) lint. It was much easier to
implement and test than I had expected. In this post I'll review the
process of creating or contributing to a Clippy lint, the implementation itself and
how this reflects Rust's values of empowerment.


<!--more-->

Whilst writing some quick prototyping code back in October, I came
across an issue accidentally triggering a recursive definition of
the `Display` trait, with code similar to [the following](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=690cb006ae868d481ed830540bf1b52c):

```rust
struct TestType;

impl std::ops::Deref for TestType {
    type Target = str;

    fn deref(&self) -> &Self::Target {
        "test"
    }
}

impl std::fmt::Display for TestType {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", &*self)
    }
}
```

Note the subtle issue of the single deref in `fmt()`, the code works
correctly if `&*self` is replaced with `&**self` since `self` is already
a reference inside the method. This mistake leads to infinite recursion since it tries to format
`self` with `Display` whilst inside the definition for `Display`.

However, the real issue was that neither rustc nor Clippy gave any
warning about the infinite recursion. rustc does have
its own lint for [unconditional recursion](https://doc.rust-lang.org/rustc/lints/listing/warn-by-default.html#unconditional-recursion),
 however in this case it is not triggered due to it not being a direct
 call of the same function (i.e. there some layers of function calls due
 to the formatting machinery).

Being so used to rustc and Clippy detecting all sorts of issues, this
was disappointing. So I created [an issue for Clippy](https://github.com/rust-lang/rust-clippy/issues/7830)
and was referred to a [much earlier issue](https://github.com/rust-lang/rust-clippy/issues/2691) (from April 2018!)
for the underlying problem (the recursive Display implementation).

## Implementing the lint

The rough procedure for implementing a new lint is as follows:
- Decide on the lint type (early pass or late pass)
- Write test cases (including positive and negative cases, and likely
  false positives and negatives) - in this case they are UI tests since
  we want to check that the user-facing output of the lint is correct.
- Implementing the lint (checking similar lints for help and possible complications)
- Checking test output is correct (i.e. only triggers where wanted)
- Updating the expected test output and lint register

There are two lint pass types - early and late. The early lint pass is
faster but has no access to type lookups (only the Abstract Syntax
Tree (AST)). In this case we need the type information, so we use the late
pass.

When we implement the lint pass, we write code that will be executed for
each matching node in the AST. Note that we can store state in the Lint
struct itself, which is useful for keeping track of where we are in the
code (i.e. to keep information from outer parts of the AST) - here we
use this to only check expressions when inside the Display or Debug impl
block.

### Lint pass overview

To start with we need to be able to detect when we are inside a Display
or Debug impl block.

In this code set the `self.format_trait_impl` field
when we are inside the `Impl` of a format trait (Display or Debug)
and unset it once we leave that `Impl` block:

```rust
pub struct FormatImpl {
    // Whether we are inside a Display or Debug trait impl - None for neither
    format_trait_impl: Option<FormatTrait>,
}

impl<'tcx> LateLintPass<'tcx> for FormatImpl {
    fn check_impl_item(&mut self, cx: &LateContext<'_>, impl_item: &ImplItem<'_>) {
	// Check if Impl is for Display or Debug
        self.format_trait_impl = is_format_trait_impl(cx, impl_item);
    }

    fn check_impl_item_post(&mut self, cx: &LateContext<'_>, impl_item: &ImplItem<'_>) {
        // Assume no nested Impl of Debug and Display within eachother
        if is_format_trait_impl(cx, impl_item).is_some() {
            self.format_trait_impl = None;
        }
    }
    ...
```

`check_impl_item` is triggered at the start of an Impl block, and
`check_impl_item_post` when we leave it.

This use of state means that we can then do our Expression level checks
only when we're inside the relevant Impl blocks:

```rust
impl<'tcx> LateLintPass<'tcx> for FormatImpl {
    ...
    fn check_expr(&mut self, cx: &LateContext<'tcx>, expr: &'tcx Expr<'_>) {
        let Some(format_trait_impl) = self.format_trait_impl else { return };

        if format_trait_impl.name == sym::Display {
            check_to_string_in_display(cx, expr);
        }

        check_self_in_format_args(cx, expr, format_trait_impl);
        check_print_in_format_impl(cx, expr, format_trait_impl);
    }
}
```

The `to_string_in_display` check is only relevant for the Display trait,
so we check for it explicitly there.

Note the use of separate functions for different lint checks. This
allows us to combine checking related lints in the same lint pass whilst
keeping the code readable.

### to_string_in_display

The check for the use of `to_string()` on self whilst inside the Display
impl is quite straightforward:

```rust
fn check_to_string_in_display(cx: &LateContext<'_>, expr: &Expr<'_>) {
    if_chain! {
        // Get the hir_id of the object we are calling the method on
        if let ExprKind::MethodCall(path, [ref self_arg, ..], _) = expr.kind;
        // Is the method to_string() ?
        if path.ident.name == sym!(to_string);
        // Is the method a part of the ToString trait? (i.e. not to_string() implemented
        // separately)
        if let Some(expr_def_id) = cx.typeck_results().type_dependent_def_id(expr.hir_id);
        if is_diag_trait_item(cx, expr_def_id, sym::ToString);
        // Is the method is called on self?
        if let ExprKind::Path(QPath::Resolved(_, path)) = self_arg.kind;
        if let [segment] = path.segments;
        if segment.ident.name == kw::SelfLower;
        then {
            span_lint(
                cx,
                RECURSIVE_FORMAT_IMPL,
                expr.span,
                "using `self.to_string` in `fmt::Display` implementation will cause infinite recursion",
            );
        }
    }
}
```

Note the use of the [if_chain!](https://docs.rs/if_chain/latest/if_chain/) macro to 
chain if conditions without rightward drift (this is used throughout
Clippy).

The pattern `if let Some(XXX) = YYY;` is common when using if
chains, to allow us to refer to `XXX` later in the if chain (and stop
checking if it is None).

It is the `span_lint()` call that actually returns output for the
specific lint (i.e. `RECURSIVE_FORMAT_IMPL` in this case). This output
(to stderr) is what is compared in the UI test.

Also note the use of [diagnostic items](https://rustc-dev-guide.rust-lang.org/diagnostics/diagnostic-items.html) with the
`is_diag_trait_item()` check. This is [recommended over using hardcoded paths](https://github.com/rust-lang/rust-clippy/issues/5393).


#### UI Test example

One UI test (in `tests/ui/recursive_format_impl.rs`) for the above check is:

```rust
#![warn(clippy::recursive_format_impl)]
#![allow(
    clippy::inherent_to_string_shadow_display,
    clippy::to_string_in_format_args,
    clippy::deref_addrof
)]

use std::fmt;

struct A;
impl A {
    fn fmt(&self) {
        self.to_string();
    }
}

trait B {
    fn fmt(&self) {}
}

impl B for A {
    fn fmt(&self) {
        self.to_string();
    }
}

impl fmt::Display for A {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.to_string())
    }
}

fn fmt(a: A) {
    a.to_string();
}
```

With the expected output:

```
error: using `self.to_string` in `fmt::Display` implementation will cause infinite recursion
  --> $DIR/recursive_format_impl.rs:29:25
   |
LL |         write!(f, "{}", self.to_string())
   |                         ^^^^^^^^^^^^^^^^
   |
   = note: `-D clippy::recursive-format-impl` implied by `-D warnings`
```

i.e. it only triggers for the `fmt::Display` impl for A and not in the impl of
trait `B` for struct `A`.

### check_self_in_format_args

The original case to solve is slightly more complicated. First we need
to find the use of the format macro e.g. `write!(...)` inside the
Display/Debug impl, and then we need to check the arguments inside the macro call
(to check whether any of them resolve to `self`).

Here we use some of the shared Clippy utils like `root_macro_call_first_node()` which 
make it much easier to deal with macros. This makes the first part quite
straightforward:

```rust
fn check_self_in_format_args<'tcx>(cx: &LateContext<'tcx>, expr: &'tcx Expr<'_>, impl_trait: FormatTrait) {
    // First find the call of the format macro
    if_chain! {
        if let Some(outer_macro) = root_macro_call_first_node(cx, expr);
        if let macro_def_id = outer_macro.def_id;
        if let Some(format_args) = FormatArgsExpn::find_nested(cx, expr, outer_macro.expn);
        if is_format_macro(cx, macro_def_id);
        if let Some(args) = format_args.args();
        then {
            for arg in args {
		// It is okay to use a Display method in a Debug impl, and vice versa
                if arg.format_trait != impl_trait.name {
                    continue;
                }
		// Check if arg resolves to self
                check_format_arg_self(cx, expr, &arg, impl_trait);
            }
        }
    }
}
```

As noted in the original example, checking the arguments to the format
macro is slightly complicated by the fact we need to handle references
and de-references. Here this is done with the use of the
`peel_ref_operators()` utils function.

After that, it's just a case of checking whether the result is `self` or
not.

```rust
fn check_format_arg_self(cx: &LateContext<'_>, expr: &Expr<'_>, arg: &FormatArgsArg<'_>, impl_trait: FormatTrait) {
    // Check each arg in format calls - do we ever use Display on self (directly or via deref)?
    // Handle multiple dereferencing of references e.g. &&self
    // Handle dereference of &self -> self that is equivalent (i.e. via *self in fmt() impl)
    // Since the argument to fmt is itself a reference: &self
    let reference = peel_ref_operators(cx, arg.value);
    let map = cx.tcx.hir();
    // Is the reference self?
    if path_to_local(reference).map(|x| map.name(x)) == Some(kw::SelfLower) {
        let FormatTrait { name, .. } = impl_trait;
        span_lint(
            cx,
            RECURSIVE_FORMAT_IMPL,
            expr.span,
            &format!("using `self` as `{name}` in `impl {name}` will cause infinite recursion"),
        );
    }
}
```

### Things to consider

Note that we never execute the code itself, Clippy is entirely static
analysis. However, the fact that we have access to the type information
means we can still handle things like checking what the final type of an
expression will be after applying dereferences, etc. as done here.

However, one must take care not to create false positives by
accidentally over-simplifying the checks. For example, my original code
just compared the final type of the expression of the argument in the
format macro with the `Self` type in the impl block, to avoid dealing with references and de-references directly.

```rust
  let arg_ty = cx.typeck_results().expr_ty_adjusted(arg.value);
  let self_ty = cx.typeck_results().node_type(self_hir_id);
  if self_ty == arg_ty;
```

But this approach doesn't work with enums (thanks to [@mikerite spotting
this so quickly!](https://github.com/rust-lang/rust-clippy/pull/8188#issuecomment-1002426663))
such as in the following example (now a test case):

```rust
enum Tree {
    Leaf,
    Node(Vec<Tree>),
}

impl fmt::Display for Tree {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Tree::Leaf => write!(f, "*"),
            Tree::Node(children) => {
                write!(f, "(")?;
                for child in children.iter() {
                    write!(f, "{},", child)?;
                }
                write!(f, ")")
            }
        }
    }
}
```
This code is fine and won't trigger infinite recursion since Nodes will
always end in leaves, or have no children. But this would trigger a false positive
if we only compare the types, since both enum variants have the same type when checked
(since enum variants are not their own types yet - see the [Types for enum variants RFC](https://github.com/rust-lang/rfcs/pull/1450) and 
[Enum variant types RFC](https://github.com/rust-lang/rfcs/pull/2593) for details on possible future changes to that).

The corrected lint check works since `self` never refers to the same
entity as `child`.

Bear in mind that false negatives are greatly preferable to false
positives, so always try to think of possible edge cases for testing,
and lean towards more conservative solutions.

Note it is still possible to write code that will trigger infinite
recursion by similar means, but not be detected here. Such as calling
`to_string()` on a type inside our Display impl, whose Display impl in turn calls `to_string()`
on this type, etc. The aim is only to cover cases which are likely to
come up when writing code in practice (after all we cannot solve the [Halting problem](https://en.wikipedia.org/wiki/Halting_problem)!).

## Summary

If you run Clippy on [the original example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=690cb006ae868d481ed830540bf1b52c) you
will now see the following error:

```
error: using `self` as `Display` in `impl Display` will cause infinite recursion
  --> src/main.rs:15:9
   |
15 |         write!(f, "{}", &*self)
   |         ^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[deny(clippy::recursive_format_impl)]` on by default
   = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#recursive_format_impl
   = note: this error originates in the macro `write` (in Nightly builds, run with -Z macro-backtrace for more info)
```

It's a great feeling to go from first hitting this issue, all the way to
seeing the check running directly in the Rust playground.

At the time of writing, the main code for the lint is available at [clippy_lints/src/format_impl.rs](https://github.com/rust-lang/rust-clippy/blob/master/clippy_lints/src/format_impl.rs)
and the UI tests are available at [tests/ui/recursive_format_impl.rs](https://github.com/rust-lang/rust-clippy/blob/master/tests/ui/recursive_format_impl.rs) and [tests/ui/recursive_format_impl.stderr](https://github.com/rust-lang/rust-clippy/blob/master/tests/ui/recursive_format_impl.stderr).
My original Pull Request (with corrections) is [#8188](https://github.com/rust-lang/rust-clippy/pull/8188).


### Conclusion

Overall, I was very impressed by how easy it was to write the lint. The
Clippy team has done great work on providing examples, quick code
reviews and a lot of easy-to-use shared utils code.

If you ever find a similar nagging issue, no matter how small, I highly
encourage you to at least create an issue on Github so you can open
discussion around possible solutions and similar issues. In my case, it
took me almost two months to go from first seeing the issue to 
realising I could actually add the lint myself.

If you do wish to contribute a lint to Clippy, I recommend
reading the [Contribution guidelines](https://github.com/rust-lang/rust-clippy/blob/master/CONTRIBUTING.md),
the [basics for hacking on Clippy](https://github.com/rust-lang/rust-clippy/blob/master/doc/basics.md)
and the [documentation on adding lints](https://github.com/rust-lang/rust-clippy/blob/master/doc/adding_lints.md), 
and posting on the [clippy Zulip stream](https://rust-lang.zulipchat.com/#narrow/stream/257328-clippy) for additional help/discussion.
The [rustc dev guide](https://rustc-dev-guide.rust-lang.org/hir.html)
can be useful for additional documentation on the High-level
Intermediate Representation (HIR), Diagnostic Items, and other concepts.
Finally, the [Common tools for writing lints](https://github.com/rust-lang/rust-clippy/blob/master/doc/common_tools_writing_lints.md) 
is also useful for recognising common operations (as well as reading
through the existing lints and the `clippy_utils` crate).

In my opinion this whole experience perfectly demonstrates Rust's value
of empowerment for users and developers. I couldn't imagine making 
a similar contribution to the tooling of any other language, and it was greatly
facilitated by the efforts of reviewers and prior contributors.

I hope writing up this example as a first-time contributor will help
others to also start contributing. It is through thousands of
contributions like this (how ever small or large) that Rust has become
the least frustrating and most empowering programming language.
