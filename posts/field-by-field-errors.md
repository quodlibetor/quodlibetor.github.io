<!--
.. title: Field-by-Field errors
.. slug: field-by-field-errors
.. date: 2017-01-15 23:57:30 UTC-05:00
.. tags:
.. category:
.. link:
.. description:
.. type: text
-->

I come from more dynamic languages (mostly Python, Java, Javascript), where
types live in the object namespace. In general I consider runtime fiddling --
either introspection or, god forbid, modification -- with the type space to be
a wild antipattern. In other words, I don't like magic in production code.

That clean philosophy goes out the window when I'm writing tests. For tests I
want the fewest number of lines of code to give me maximum debuggability, which
often means type-and-object-introspection and all kinds of modification
shenanigans.

I've been wanting to play with the new Macros 1.1 (custom derive) system for
awhile, so I've been looking for something that really calls out for magic,
something that can't be done reasonably without observing the syntactic
structure of the code I'm working with.

One feature that I miss when working with large structs in Rust
is
[AssertJ's field-by-field assertions](http://joel-costigliola.github.io/assertj/assertj-core-features-highlight.html#field-by-field).
It's some Java magic that very quickly lets you determine which parts of
objects are distinct. This gives me a pretty nice workflow where I can create
an "expected object" in a setup function and get error messages that show me
specifically what is unexpected, while giving full context for the objects that
are busted. In particular, I love it when it tells me that two UUIDs don't
match by one bit inside of classes that have 10, 20, or 30 fields. (Nobody
wants classes with 30 fields, I know.)

So [I've written](https://github.com/quodlibetor/field-by-field) something like
it for Rust.

Given a struct and a test function that look like:

```rust
  #[derive(FieldByField, Debug)]
  struct Something {
      hello: String,
      val: i8,
      val2: i8,
      val3: i8,
  }

  #[test]
  fn assert_catches_differences() {
      let one = Something { hello: "one".into(), val: 1, val2: 5, val3: 3 };
      let two = Something { hello: "two".into(), val: 1, val2: 2, val3: 3 };

      one.assert_equal_field_by_field(&two);
  }
```

I can get a pretty good error message, that shows exactly which fields
weren't equal:

```
thread 'assert_catches_differences' panicked at '
      Items are not equal:
          hello: "one" != "two"
          val2: 5 != 2
      actually: Something { hello: "one", val: 1, val2: 5, val3: 3 }
      expected: Something { hello: "two", val: 1, val2: 2, val3: 3 }
  ', derive-struct.rs:11
```

For small structs like this it isn't essential, but with objects that contain
many fields, or objects that contain a lot of fields that look mostly similar
I find it to be a huge time-saver.

This isn't the story of that (as-yet-officially-unnamed) crate. This also isn't
the story of /learning/ macros 1.1,
[cbreedan has already covered that well](https://cbreeden.github.io/Macros11/).

It's a partial story of me learning to debug my custom derive code, and some
tools provided by the Rust ecosystem to do so.
