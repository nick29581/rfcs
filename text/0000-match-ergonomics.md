- Feature Name: default-pattern-binding-modes
- Start Date: 2016-08-12
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Better ergonomics for pattern-matching on references.

Currently:

```
match *x {
    Foo(ref x) => { ... }
    Bar(ref mut y, z) => { ... }
}
```

Proposed:

```
match x {
    Foo(x) => { ... }
    Bar(y, z) => { ... }
}
```

This is accomplished through automatic dereferencing and the introduction of
default binding modes.

# Motivation
[motivation]: #motivation

Rust is usually strict when distinguishing between value and reference types. In
particular, distinguishing borrowed and owned data. However, there is often a
trade-off between [explicit-ness and ergonomics](https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html),
and Rust errs on the side of ergonomics in some carefully selected places.
Notably when using the dot operator to call methods and access fields, and when
declaring closures.

The match expression is an extremely common expression and arguably, the most
important control flow mechanism in Rust. Borrowed data is probably the most
common form in the language. However, using match expressions and borrowed data
together can be frustrating: getting the correct combination of `*`, `&`, and
`ref` to satisfy the type and borrow checkers is a common problem, and one which
is often encountered early by Rust beginners. It is especially frustrating since
it seems that the compiler can guess what is needed but gives you error messages
instead of helping.

For example, consider the following program:

```
enum E { Foo(...), Bar }

fn f(e: &E) {
    match e { ... }
}

```

It is clear what we want to do here - we want to check which variant `e` is a
reference to. Annoyingly, we have two valid choices:

```
match e {
    &E::Foo(...) => { ... }
    &E::Bar => { ... }
}
```

and

```
match *e {
    E::Foo(...) => { ... }
    E::Bar => { ... }
}
```

The former is more obvious, but requires more noisey syntax (an `&` on every
arm). The latter can appear a bit magical to newcomers - the type checker treats
`*e` as a value, but the borrow checker treats the data as borrowed for the
duration of the match. It also does not work with nested types, `match (*e,)
...` for example is not allowed.

In either case if we further bind variables, we must ensure that we do not
attempt to move data, e.g.,

```
match *e {
    E::Foo(x) => { ... }
    E::Bar => { ... }
}
```

If the type of `x` does not have the `Copy` bound, then this will give a borrow
check error. We must use the `ref` keyword to take a reference: `E::Foo(ref x)`
(or `&E::Foo(ref x)` if we match `e` rather than `*e`).

The `ref` keyword is a pain for Rust beginners, and a bit of a wart for everyone
else. It violates the rule of patterns matching declarations, it is not found
anywhere outside of patterns, and it is often confused with `&`. (See for
example, https://github.com/rust-lang/rust-by-example/issues/390).

Match expressions are an area where programmers often end up playing 'type
Tetris': adding operators until the compiler stops complaining, without
understanding the underlying issues. This serves little benefit - we can make
match expressions much more ergonomic without sacrificing safety or readability.

Match ergonomics has been highlighted as an area for improvement in 2017:
[internals thread](https://internals.rust-lang.org/t/roadmap-2017-productivity-learning-curve-and-expressiveness/4097)
and [Rustconf keynote](https://www.youtube.com/watch?v=pTQxHIzGqFI&list=PLE7tQUdRKcybLShxegjn0xyTTDJeYwEkI&index=1).


# Detailed design
[design]: #detailed-design

This RFC is a refinement of
[the match ergonomics RFC](https://github.com/rust-lang/rfcs/pull/1944). Rather
than using auto-deref and auto-referencing, this RFC introduces _default binding
modes_ used when a reference value is matched by a non-reference pattern

In other words, we allow auto-dereferencing values during pattern-matching.
When an auto-dereference occurs, the compiler will automatically treat the inner
bindings as `ref` or `ref mut` bindings.

Example:

```rust
let x = Some(3);
let y: &Option<i32> = &x;
match y {
  Some(a) => {
    // `y` is dereferenced, and `a` is bound like `ref a`.
  }
  None => {}
}
```


## Definitions

A _non-reference pattern_ is any pattern that is not a binding, a wildcard (`_`),
a constant with reference type, or a reference pattern (a pattern beginning with
`&` or `&mut`).

 _Default binding mode_: either `move`, `ref`, or `ref mut`, is used
 to determine how to bind new pattern variables.
 When the compiler sees a variable binding not explicitly marked
 `ref`, `ref mut`, or `mut`, it uses the _default binding mode_
 to determine how the variable should be bound.
 Currently, the _default binding mode_ is always `move`.
 This RFC proposes to change that.

## Binding mode rules

The _default binding mode_ starts out as `move`. When matching a pattern, the
compiler starts from the outside of the pattern and works inwards.
Each time a reference is matched using a _non-reference pattern_,
it will automatically dereference the value and update the default binding mode:

1. If the reference encountered is `&Pat`, set the default binding mode to `ref`.
2. If the reference encountered is `&mut Pat`: if the current default
binding mode is `ref`, it should remain `ref`. Otherwise, set the current binding
mode to `ref mut T`.

```
                        Start                                
                          |                                  
                          v                                  
                +-----------------------+                     
                | Default Binding Mode: |                     
                |        move           |                     
                +-----------------------+                     
               /                        \                     
Encountered   /                          \  Encountered       
  &mut Pat   /                            \     &Pat
            v                              v                  
+-----------------------+        +-----------------------+    
| Default Binding Mode: |        | Default Binding Mode: |    
|        ref mut        |        |        ref            |    
+-----------------------+        +-----------------------+    
                          ----->                              
                        Encountered                           
                            &Pat                              
```

Note that there is no exit from the `ref` binding mode. This is because an
`&mut` inside of a `&` is still shared, and thus cannot be used to mutate the
underlying value.

Also note that no transitions are taken when using an explicit `ref` or
`ref mut` binding. The _default binding mode_ only changes when matching a
reference with a non-reference pattern.

The above rules and the examples that follow are drawn from @nikomatsakis's
[comment proposing this design](https://github.com/rust-lang/rfcs/pull/1944#issuecomment-296133645).

## Examples

No new behavior:
```rust
match &Some(3) {
    p => {
        // p is a variable binding. Hence, this is **not** a ref-defaulting match,
        // and `p` is move mode (and has type `&i32`).
    },
}
```

One match arm with new behavior:
```rust
match &Some(3) {
    Some(p) => {
        // This pattern is not a binding, `_`, or `&`-pattern, so this is a
        // "non-reference pattern." We dereference the `&` and shift the
        // default binding mode to `ref`. `p` is read as `ref p` and given
        // type `&i32`.
    },
    x => {
        // In this arm, we are still in move-mode by default, so `x` has type
        // `&Option<i32>`
    },
}

// Desugared:
match &Some(3) {
  &Some(ref P) => {
    ...
  },
  x => {
    ...
  },
}
```

`|`'d match:
```rust
let x = &Some((3, 3));
match x {
  // Here, each of the patterns are treated independently
  Some((x, 3)) | &Some((ref x, 5)) => { ... }
  _ => { ... }
}

// Desugared:
let x = &Some(3);
match x {
  &Some((ref x, 3)) | &Some((ref x, 5)) => { ... }
  None => { ... }
}
```

Multiple nested patterns with new and old behavior, respectively:
```rust
match (&Some(5), &Some(6)) {
    (Some(a), &Some(mut b)) => {
        // Here, the `a` will be `&i32`, because in the first half of the tuple
        // we hit a non-reference pattern and shifted into `ref` mode.
        //
        // In the second half of the tuple there's no non-reference pattern,
        // so `b` will be `i32` (bound with `move` mode). Moreover, `b` is
        // mutable.
    },
    _ => { ... }
}

// Desugared:
match (&Some(5), &Some(6)) {
  (&Some(ref a), &Some(mut b)) => {
    ...
  },
  _  => { ... },
}
```

Example with multiple dereferences:
```rust
let x = (1, &Some(5));
let y = &Some(x);
match y {
  Some((a, Some(b))) => { ... }
  _ => { ... }
}

// Desugared:
let x = (1, &Some(5));
let y = &Some(x);
match y {
  &Some((ref a, &Some(ref b))) => { ... }
  _ => { ... }
}
```

Example of new mutable reference behavior:
```rust
match &mut x {
    Some(y) => {
        // `y` is an `&mut` reference here, equivalent to `ref mut` before
    },
    None => { ... },
}

// Desugared:
match &mut x {
  &mut Some(ref mut y) => {
    ...
  },
  &mut None => { ... },
}
```

Example using `let`:
```rust
// Note that these rules apply to any pattern matching
// whether it be in a `match` or a `let`. So, for example,
// `x` here is a ref binding:
let Foo(x) = &Foo(3);

// Desugared:
let &Foo(ref x) = &Foo(3);
```


## Backwards compatibility

In order to guarantee backwards-compatibility, this proposal only modifies
pattern-matching when a reference is matched with a non-reference pattern,
which is an error today.

This reasoning requires that the compiler knows if the type being matched is a
reference, which isn't always true for inference variables.
If the type being matched may
or may not be a reference _and_ it is being matched by a _non-reference
pattern_, then the compiler will default to assuming that it is not a
reference, in which case the binding mode will default to `move` and it will
behave exactly as it does today.

Example:

```rust
let x = vec![];

match x[0] { // This will panic, but that doesn't matter for this example

    // When matching here, we don't know whether `x[0]` is `Option<_>` or
    // `&Option<_>`. `Some(y)` is a non-reference pattern, so we assume that
    // `x[0]` is not a reference
    Some(y) => {

        // Since we know `Vec::contains` takes `&T`, `x` must be of type
        // `Vec<Option<usize>>`. However, we couldn't have known that before
        // analyzing the match body.
        if x.contains(&Some(5)) {
            ...
        }
    }
    None => {}
}
```

# Drawbacks
[drawbacks]: #drawbacks

The major downside of this proposal is that it complicates the pattern-matching
logic. However, doing so allows common cases to "just work", making the beginner
experience more straightforward and requiring fewer manual reference gymnastics.

# Alternatives
[alternatives]: #alternatives

- We could support auto-ref / deref as suggested in
[the original match ergonomics RFC.](https://github.com/rust-lang/rfcs/pull/1944)
- We could allow writing `move` in patterns.
Without this, `move`, unlike `ref` and `ref mut`, would always be implicit,
leaving no way override a default binding mode of `ref` or `ref mut` and move
the value out from behind a reference.
However, moving a value out from behind a shared or mutable
reference is only possible for `Copy` types, so this would not be particularly
useful in practice, and would add unnecessary complexity to the language.

Both of these approaches have troublesome interaction with
backwards-compatibility, and it becomes more difficult for the user to reason
about whether they've borrowed or moved a value.
