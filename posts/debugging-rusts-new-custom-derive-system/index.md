<!--
.. title: Debugging Rust's new Custom Derive system
.. slug: debugging-rusts-new-custom-derive-system
.. date: 2017-01-15 17:21:40 UTC-05:00
.. tags:
.. category:
.. link:
.. description:
.. type: text
-->

# Why a custom derive macro

I've been working
on [a couple of crates](https://github.com/quodlibetor/field-by-field) to copy
over
[the magic of AssertJ's field-by-field assertions](http://joel-costigliola.github.io/assertj/assertj-core-features-highlight.html#field-by-field) to
Rust. Partially as a way of learning Macros 1.1/custom derive, and partially
because it's one of my favorite testing tools and nobody should have to live
without it.

This post is about what I've been doing to *debug* my custom derive
implementation, which can require a couple of extra steps to get to the same
kind of good error messages that I'm used to from regular Rust code.

# Debugging

There are a couple
tips [in the `syn` crate's README](https://github.com/dtolnay/syn#testing),
which were the first thing that I read when I started working on
field-by-field. Most of this post is just me exploring the implications and
going into a bit/lots more detail.

## test files

I'm writing a testing tool, so of course the first thing that I want to do is
test that my code does what I want it to do. Plugins (custom-derive included)
cannot be tested using the standard in-module `#[test]` annotation that most
Rust code uses, but you can still use integration tests via the `tests/`
directory that Cargo knows about.

These modules are linked against the macro crate that you are writing, so
everything works basically the same except that you [end up using a
`#[macro_use] my_crate` to make it
work](https://github.com/quodlibetor/field-by-field/blob/28a5003fdcb58b350125197d0708844af8c7ec1b/field-by-field-macros/tests/derive-enum-struct.rs).

At that point the standard `#[test]` attribute does what you expect, and you
can use the common `cargo test` and `cargo test --test <integration-test-file>
<optional-test-filter` to do what you would expect:

```
$ cargo test
    Finished debug [unoptimized + debuginfo] target(s) in 0.0 secs
     Running .../derive_enum_mixed-d370953e520402cf
...
test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

     Running .../derive_enum_struct-8851e9b38fffd49d
...
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

     Running .../derive_enum_tuple-4b98f23c906ffa8d
...
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

     Running .../derive_enum_unit-f5f0a72af200d8c4
...
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

     Running .../derive_struct-116d0c78a0c6e62f
...
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured
```

Or to be more specific:

```
$ cargo test --test derive-struct assert_catches
    Finished debug [unoptimized + debuginfo] target(s) in 0.0 secs
     Running .../derive_struct-116d0c78a0c6e62f

running 1 test
test assert_catches_differences ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

Wheeee.

# Now we've got two kinds of problems

## Syntactically invalid code

There are two -- maybe two and half -- phases to compiling these integration
tests:

1. We have to compile the custom-derive library
2. We have to use the compiled lib to actually generate some code
   1. That compiled code might generate code that is syntactically unsound
   2. Or it could generate code that is syntactically fine, but violates some
      of rusts guarantees.

This means that there are three different kind of errors that we can get:
* Regular Rust errors
* Rust errors because some generated code is invalid
* Rust errors because the generated code doesn't type-check.

Syntactic errors end up causing *extremely* long error lines:

```rust
error: expected one of `=>`, `if`, or `|`, found `}`
 --> <proc-macro source code>:1:367
  |
1 | impl :: field_by_field :: EqualFieldByField for SingleUnitEnum { fn fields_not_equal ( & self , other : & Self ) -> Vec < :: field_by_field :: UnequalField > { # [ allow ( unused_mut ) ] let mut list : Vec < :: field_by_field :: UnequalField > = Vec :: new ( ) ; match ( self , other ) { log_syntax ! ( ( & SingleUnitEnum :: One , & SingleUnitEnum :: One ) => { } ) } list } fn assert_equal_field_by_field ( & self , other : & SingleUnitEnum ) { let errs = self . fields_not_equal ( other ) ; if errs . len ( ) > 0 { let mut errmsg = String :: from ( "\n    Items are not equal:\n" ) ; for field_err in errs { errmsg . push_str ( & format ! ( "        {}: {:?} != {:?}\n" , field_err . field_name , field_err . actually , field_err . expected ) ) ; } let ac_exp = format ! ( "    actually: {:?}\n    \
  |                                                                                                                                                                                                                                                                                                                                                                               ^
```

If you scroll all the way to the right you'll see where I omitted a match arm
entirely, causing Rust to just barf at me.

Perhaps the most interesting part of this is that these errors come from within
my procedural macro, but for some reason they result in error messages that
include the generated source, making them among the easiest proc-macro code to
debug.

# Syntactically valid errors

## An Error

Tests exist to make it possible to refactor and know that your code still
works, and if you do TDD to give you a nice incremental target to work towards.
They're great at helping write features, and the Rust compiler is great at
helping prevent stupid mistakes from creeping into prod.

But! When writing custom derive macros, these errors can not only prevent your
code from compiling, they can prevent your code from *even existing*. Let's say
I forget to unquote an ident somewhere. For
example,
[I've got a line](https://github.com/quodlibetor/field-by-field/blob/28a5003fdcb58b350125197d0708844af8c7ec1b/field-by-field-macros/src/lib.rs#L130) that
looks like:

```rust
    if is_multivariant {
        // long and simple code
    } else {
        quote! {
            ( &#name::#var_name, &#name::#var_name ) => {}
        }
    }
```

What happens if I forget to include one of the `#`s? What will that look like
to me as I'm writing it, and if I screw something up in a release, what will it
look like to my downstream users? What can we do to figure out the fat-fingered
mistake I've made?

Let's take a look:

The unexpanded error message (from above, where I forgot to unquote a token)
will be:

```console
$ cargo test --test derive-enum-unit
   Compiling field-by-field-macros v0.1.0 (file:///Users/bwm/projects/field-by-field/field-by-field-macros)
error: no associated item named `var_name` found for type `UnitEnum` in the current scope
 --> tests/derive-enum-unit.rs:9:10
  |
9 | #[derive(FieldByField, Debug)]
  |          ^^^^^^^^^^^^
```

Hm. Okay I know I should look for `var_name` in my code, somewhere. The rest of
this post is about getting better error messages.

## cargo expand

The [`cargo-expand`](https://crates.io/crates/cargo-expand) Cargo subcommand
(by the author of the `syn` and `quote` crates that are the bread and butter of
custom-derive implementations) is supremely helpful for figuring out what is
actually going on with your code.

Of particular note for the way that I've been hacking, `cargo-expand` works
with integration test files. I've found myself in an "edit - expand - compile -
test" loop to figure out what the heck I was trying to do. What that looks like
in more detail for me is something like:

```console
$ emacs src/lib.rs
$ # expanded code relies on some internal rust features:
$ echo '#![feature(box_syntax, test, fmt_internals)]' > tests/derive-enum-unit-expanded.rs
$ cargo expand --test derive-enum-struct > tests/derive-enum-struct-expanded.rs
$ cargo test --test derive-enum-struct-expanded
```

This results in wildly different error messages.

```console
$ cargo test --test derive-enum-unit-expanded
   Compiling field-by-field-macros v0.1.0 (file:///Users/bwm/projects/field-by-field/field-by-field-macros)
... 30 lines of errors about the format! macro
error: no associated item named `var_name` found for type `UnitEnum` in the current scope
  --> tests/derive-enum-unit-expanded.rs:41:15
   |
41 |             (&UnitEnum::var_name, &UnitEnum::One) => {}
   |               ^^^^^^^^^^^^^^^^^^

error: no associated item named `var_name` found for type `UnitEnum` in the current scope
  --> tests/derive-enum-unit-expanded.rs:72:15
   |
72 |             (&UnitEnum::var_name, &UnitEnum::Two) => {}
   |               ^^^^^^^^^^^^^^^^^^

... 30 lines of errors about the format! macro

error: aborting due to 6 previous errors

error: Could not compile `field-by-field-macros`.
```

In addition, this error will be in context, so it's more straightforward to see
exactly where in my generating code it is probably coming from. In particular,
for my code, this turns into:

```rust
impl ::field_by_field::EqualFieldByField for UnitEnum {
    fn fields_not_equal(&self, other: &Self) -> Vec<::field_by_field::UnequalField> {

        let mut list: Vec<::field_by_field::UnequalField> = Vec::new();
        match (self, other) {
            (&UnitEnum::var_name, &UnitEnum::One) => {}
```

Which makes it much more obvious to me where I should be looking.

It also means that I can edit the code in place and often get rid of that
specific error (although including `format!`/`println!`/etc anywhere in your
macro output seems to prevent it from ever succeeding building) which is also a
much faster edit loop, since emacs + flycheck will show me the error while I'm
editing, and will show me that I've fixed it when I have.

## println!

`cargo-expand` is the big guns, the first thing I reach for when I don't even
know what part of my code is causing the error. If I know approximately what
I'm doing, though, it's possible to get less verbose output by just using
`println!`.
[`Tokens` objects](https://docs.rs/quote/0.3.10/quote/struct.Tokens.html)
implement `Display` and `Debug`, so if you've combined several `quote!` macros,
or you aren't sure what the values of variables are, this is a pretty good
option. (Probably you could use a debugger here, but that idea hadn't even
occured to me until right now.)

Printed output just happens at compile time, so the same `cargo test --test
<file>` code works to narrow to exactly the use case you want to expand:

So if I've got the same error as above, I'll modify my code like:

```rust
let r = quote! {
    ( &#name::var_name, &#name::#var_name ) => {}
};
println!("in single variant: {}", r);
r
```

```console
$ cargo test --test derive-enum-unit
   Compiling field-by-field-macros v0.1.0 (file:///Users/bwm/projects/field-by-field/field-by-field-macros)
in single variant: ( & SingleUnitEnum :: var_name , & SingleUnitEnum :: One ) => { }
error: no associated item named `var_name` found for type `SingleUnitEnum` in the current scope
  --> tests/derive-enum-unit.rs:15:10
   |
15 | #[derive(FieldByField, Debug)]
   |          ^^^^^^^^^^^^

error: aborting due to previous error

error: Could not compile `field-by-field-macros`.
```

Which, when you look at what's actually there, after `in single variant`:

```rust
( & SingleUnitEnum :: var_name , & SingleUnitEnum :: One ) => { }
```

That looks just like a normal match arm, but that `var_name` is suspiciously
similar to the variable in my code, not in the code I'm decorating.

Aside: one of the things that caused me some headaches in my original
implementation of this was that I was using similar variable names in my tests
and in my code, and so these errors weren't occuring because I had the correct
literal text in my quasi quoted Rust. When I figured out what was going on it
was hilarious, believe me.

# Takeaways

This is actually pretty nice! Using `quote!` as a template language is pretty
convenient, even though the multiple phases of compilation mean that debugging
is a little rougher than writing "regular" code. I hope that we will eventually
get span information for procedurally-generated code, making the debug cycle
faster.

It would be nice if there was a way to pass rust the `--pretty expanded` option
but exclude some specific macros -- probably anything from inside std would be
the right thing, since those things seem to rely on dark magic that I can't get
to compile.

I would like to investigate actually using a debugger to hook into the
code-generation phase. I assume that's well-trod ground, and that I'm just not
familiar with it.

I'm very excited for the things that will be *<del>possible</del>* accessible now that
custom derive is stable, and if procedural macros end up as nice as custom
derive is then I have a combination of hope and trepidation surrounding the
kinds of magic that Rust will provide.
