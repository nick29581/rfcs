- Feature Name: default-pattern-binding-modes
- Start Date: 2016-08-12
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

I'm writing comments inline (not making changes), hopefully they'll stand out in the diff.

# Summary
[summary]: #summary

Better ergonomics for pattern-matching on references.

AIUI, this will only apply to match and similar expressions, not pattern matching in general?

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
than using auto-deref and autoreferencing, however, this proposal introduces
the idea of "reference inversion". Reference inversion would allow references
to structures to be matched as though they were structures of references.

I'm not sure if it is useful to cause this reference inversion - it seems a bit like jargon to me. "Default binding modes" (used in the intro) sounds better to me

Example:

```rust
match &Some(3) { // `&Option<i32>`
it might be clearer to match a variable rather than an expression
  Some(a) => {
    // `&Some(3)` is matched as though it were `Some(&3)`.
    I don't feel that the above comment is useful - it doesn't feel quite accurate
    // The value is dereferenced, and `a` is bound like `ref a`.
    You could then use the variable name instead of "the value" here.
  }
  None => {}
}
```

I would try and give an intuition here (before getting into formalities) about when this change applies and roughly what it does

## Definitions

A _non-reference pattern_ is a pattern that cannot currently match a reference.
A non-reference pattern is any pattern that is not a binding, a wildcard (`_`),
a value of a reference type, or a reference pattern (a pattern beginning with
what is "a value of a reference type" - I don't understand a value in a pattern which doesn't match the next clause
`&` or `&mut`).

 _Default binding mode_: this mode, either `move`, `ref`, or `ref mut`, is used
 to determine how to bind new pattern variables. When we see a variable binding
 not explicitly marked `ref`, `ref mut`, or `mut`, we use the _default binding
 mode_ to determine how it should be bound.

It might be worth noting that the default binding mode for current match arms is always move, and that unless there is a reference pattern, that will continue to be the case

## Binding mode rules

The _default binding mode_ starts out as `move.` When matching a reference with
a _non-reference pattern_, we will automatically dereference the value and
update the default binding mode:

1. If the reference encountered is `&T`, set the default binding mode to `ref`.
2. If the reference encountered is `&mut T`: if the current
binding mode is `&T`, it should remain `&T`. Otherwise, set the current binding
mode to `&mut T`.

I think the above could be clearer - not clear to me what "the reference" is or what the "current binding more" is. I think you need to specify what is being iterated over and how.

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
  &mut T     /                            \     &T            
            v                              v                  
+-----------------------+        +-----------------------+    
| Default Binding Mode: |        | Default Binding Mode: |    
|        ref mut        |        |        ref            |    
+-----------------------+        +-----------------------+    
                          ----->                              
                        Encountered                           
                            &T                                
```

Note that there is no exit from the `ref` binding mode. This is because an
`&mut` inside of a `&` is still shared, and thus cannot be used to mutate the
underlying value.

The above rules and the examples that follow are drawn from @nikomatsakis's
[comment proposing this design](https://github.com/rust-lang/rfcs/pull/1944#issuecomment-296133645).

Could you specifiy what happens if there is an explicit `ref`? Is it possible to opt out of this, e.g, by using `move` to cause a move binding?

## Examples

for these examples, I think it would be useful to give the 'desugared' version

```rust
match &3 {
    p => {
        // p is a variable binding. Hence, this is **not** a ref-defaulting match,
        // and `p` is move mode (and has type `&i32`).
    }
}
```

```rust
match &Some(3) {
    Some(p) => {
        // This pattern is not a binding, `_`, or `&`-pattern, so this is a
        // "non-reference pattern." We dereference the `&` and shift the
        // default binding mode to `ref`. `p` is read as `ref p` and given
        // type `&i32`.
    }
    e.g., &Some(ref p) 
    x => {
        // In this arm, we are still in move-mode by default, so `x` has type
        // `&Option<i32>`
    }
}
```

```rust
match (&Some(5), &Some(6)) {
    (Some(a), &Some(mut b)) => {
        // Here, the `a` will be `&i32`, because in the first half of the tuple
        // we hit a non-reference pattern and shifted into `ref` mode.
        //
        // In the second half of the tuple there's no non-reference pattern,
        // so `b` will be `i32` (bound with `move` mode). Moreover, `b` is
        // mutable.
    }
    (_, None) | (None, _) => ...,
    (None, None) => ...,
}
```

```rust
match &mut x {
    Some(y) => {
        // `y` is an `&mut` reference here, equivalent to `ref mut` before
    }
    None => { ... }
}
```

```rust
// Note that these rules apply to any match that occurs,
// whether it be in a `match` or a `let`. So, for example,
// `x` here is a ref binding:
if let Some(x) = &Some(3) { ... }
```


## Backwards compatibility

In order to guarantee backwards-compatibility, this proposal aims only modify
typo: only to modify, but I would be more bold and drop "aims"
pattern-matching a reference with a non-reference pattern, which is an error
today.

This reasoning requires that we know if the type being matched is a reference,
which isn't always true for inference variables. If the type being matched may
or may not be a reference, then we should default to assuming that it is not a
reference, in which case the binding mode will default to `move` and it will
behave exactly as it does today.

Could you give an example of when this would happen

# Drawbacks
[drawbacks]: #drawbacks

The major downside of this proposal is that it complicates the pattern-matching
logic. However, doing so allows common cases to "just work", making the beginner
experience more straightforward and requiring fewer manual reference-gymnastics.

# Alternatives
[alternatives]: #alternatives

* We could support auto-ref / deref as suggested in
[the original match ergonomics RFC.](https://github.com/rust-lang/rfcs/pull/1944)
- We could analyze the binding site and try to pick the best default based on
context.

Both of these approaches have troublesome interaction with
backwards-compatibility, and it becomes more difficult for the user to reason
about whether they've borrowed or moved a value.
