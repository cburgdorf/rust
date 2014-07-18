% The Rust Pointer Guide

Rust's pointers are one of its more unique and compelling features. Pointers
are also one of the more confusing topics for newcomers to Rust. They can also
be confusing for people coming from other languages that support pointers, such
as C++. This guide will help you understand this important topic.

# An introduction

If you aren't familiar with the concept of pointers, here's a quick refresher.
Pointers are a very fundamental concept in systems programming languages, so it's
important to understand them.

## Pointer Basics

When you create a new variable binding, you're giving a name to a particular
place in memory. Like this:

```{rust}
let x = 5i;
let y = 8i;
```
```{notrust,ignore}
location  value
--------  -----
0xd3e010  5
0xd3e030  8
```

We're making up memory locations here, they're just sample values. Anyway, the
point is that `x`, the name we're using for our variable, corresponds to the
memory location `0xd3e010`, and the value stored in that place in memory is
`5`. When we refer to `x`, we get the value at its location. Hence, `x` is `5`.

Let's introduce a pointer. In some languages, there is just one type of
'pointer,' but in Rust, we have many types. In this case, we'll use a Rust
**reference**, which is the simplest kind of pointer.

```{rust}
let x = 5i;
let y = 8i;
let z = &y;
```
```{notrust,ignore}
location  value
--------  -----
0xd3e010  5
0xd3e030  8
0xd3e050  0xd3e030
```

See the difference? Rather than contain a value, the value of a pointer is a
location in memory. In this case, the location of `y`. `x` and `y` have the
type `int`, but `z` has the type `&int`. We can print this location using the
`{:p}` format string:

```{rust}
let x = 5i;
let y = 8i;
let z = &y;

println!("{:p}", z);
```

This would print `0xd3e030`, with our fictional memory addresses.

Because `int` and `&int` are different types, we can't, for example, add them
together:

```{rust,ignore}
let x = 5i;
let y = 8i;
let z = &y;

println!("{}", x + z);
```

This gives us an error:

```{notrust,ignore}
hello.rs:6:24: 6:25 error: mismatched types: expected `int` but found `&int` (expected int but found &-ptr)
hello.rs:6     println!("{}", x + z);
                                  ^
```

We can **dereference** the pointer by using the `*` operator. This will work:

```{rust}
let x = 5i;
let y = 8i;
let z = &y;

println!("{}", x + *z);
```

It prints `13`.

That's it! That's all pointers are: they point to some memory location. Not
much else to them. Now that we've discussed the 'what' of pointers, let's
talk about the 'why.'

## Pointer uses

Rust's pointers are quite useful, but in different ways than in other systems
languages. We'll talk about best practices for Rust pointers later in
the guide, but here are some ways that pointers are useful in other languages:

In C, strings are a pointer to a list of `char`acters, ending with a null byte.
The only way to use strings is to get quite familiar with pointers.

Pointers are useful to point to memory locations that are not on the stack. For
example, our example used two stack variables, so we were able to give them
names. But if we allocated some heap memory, we wouldn't have that name
available.  In C, `malloc` is used to allocate heap memory, and it returns a
pointer.

As a more general variant of the previous two points, any time you have a
structure that can change in size, you need a pointer. You can't tell at
compile time how much memory to allocate, so you've gotta use a pointer to
point at the memory where it will be allocated, and deal with it at run time.

Pointers are useful in languages that are pass-by-value, rather than
pass-by-reference. Basically, languages can make two choices (this is made
up syntax, it's not Rust):

```{notrust,ignore}
fn foo(x) {
    x = 5
}

fn main() {
    i = 1
    foo(i)
    // what is the value of i here?
}
```

In languages that are pass-by-value, `foo` will get a copy of `i`, and so
the original version of `i` is not modified. At the comment, `i` will still be
`1`. In a language that is pass-by-reference, `foo` will get a reference to `i`,
and therefore, can change its value. At the comment, `i` will be `5`.

So what do pointers have to do with this? Well, since pointers point to a
location in memory...

```{notrust,ignore}
fn foo(&int x) {
    *x = 5
}

fn main() {
    i = 1
    foo(&i)
    // what is the value of i here?
}
```

Even in a language which is pass by value, `i` will be `5` at the comment. You
see, because the argument `x` is a pointer, we do send a copy over to `foo`,
but because it points at a memory location, which we then assign to, the
original value is still changed. This pattern is called
'pass-reference-by-value.' Tricky!

## Common pointer problems

We've talked about pointers, and we've sung their praises. So what's the
downside? Well, Rust attempts to mitigate each of these kinds of problems,
but here are problems with pointers in other languages:

Uninitialized pointers can cause a problem. For example, what does this program
do?

```{notrust,ignore}
&int x;
*x = 5; // whoops!
```

Who knows? We just declare a pointer, but don't point it at anything, and then
set the memory location that it points at to be `5`. But which location? Nobody
knows. This might be harmless, and it might be catastrophic.

When you combine pointers and functions, it's easy to accidentally invalidate
the memory the pointer is pointing to. For example:

```{notrust,ignore}
fn make_pointer(): &int {
    x = 5;

    return &x;
}

fn main() {
    &int i = make_pointer();
    *i = 5; // uh oh!
}
```

`x` is local to the `make_pointer` function, and therefore, is invalid as soon
as `make_pointer` returns. But we return a pointer to its memory location, and
so back in `main`, we try to use that pointer, and it's a very similar
situation to our first one. Setting invalid memory locations is bad.

As one last example of a big problem with pointers, **aliasing** can be an
issue. Two pointers are said to alias when they point at the same location
in memory. Like this:

```{notrust,ignore}
fn mutate(&int i, int j) {
    *i = j;
}

fn main() {
  x = 5;
  y = &x;
  z = &x; //y and z are aliased


  run_in_new_thread(mutate, y, 1);
  run_in_new_thread(mutate, z, 100);

  // what is the value of x here?
}
```

In this made-up example, `run_in_new_thread` spins up a new thread, and calls
the given function name with its arguments. Since we have two threads, and
they're both operating on aliases to `x`, we can't tell which one finishes
first, and therefore, the value of `x` is actually non-deterministic. Worse,
what if one of them had invalidated the memory location they pointed to? We'd
have the same problem as before, where we'd be setting an invalid location.

## Conclusion

That's a basic overview of pointers as a general concept. As we alluded to
before, Rust has different kinds of pointers, rather than just one, and
mitigates all of the problems that we talked about, too. This does mean that
Rust pointers are slightly more complicated than in other languages, but
it's worth it to not have the problems that simple pointers have.

# References

The most basic type of pointer that Rust has is called a 'reference.' Rust
references look like this:

```{rust}
let x = 5i;
let y = &x;

println!("{}", *y);
println!("{:p}", y);
println!("{}", y);
```

We'd say "`y` is a reference to `x`." The first `println!` prints out the
value of `y`'s referent by using the dereference operator, `*`. The second
one prints out the memory location that `y` points to, by using the pointer
format string. The third `println!` *also* prints out the value of `y`'s
referent, because `println!` will automatically dereference it for us.

Here's a function that takes a reference:

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

You can also use `&` as an operator to create a reference, so we can
call this function in two different ways:

```{rust}
fn succ(x: &int) -> int { *x + 1 }

fn main() {

    let x = 5i;
    let y = &x;

    println!("{}", succ(y));
    println!("{}", succ(&x));
}
```

Both of these `println!`s will print out `6`.

Of course, if this were real code, we wouldn't bother with the reference, and
just write:

```{rust}
fn succ(x: int) -> int { x + 1 }
```

References are immutable by default:

```{rust,no_run}
let x = 5i;
let y = &x;

*y = 5; // error: cannot assign to immutable dereference of `&`-pointer `*y`
```

They can be made mutable with `mut`, but only if its referent is also mutable.
This works:

```{rust}
let mut x = 5i;
let y = &mut x;
```

This does not:

```{rust,no_run}
let x = 5i;
let y = &mut x; // error: cannot borrow immutable local variable `x` as mutable
```

Immutable pointers are allowed to alias:

```{rust}
let x = 5i;
let y = &x;
let z = &x;
```

Mutable ones, however, are not:

```{rust,no_run}
let x = 5i;
let y = &mut x;
let z = &mut x; // error: cannot borrow `x` as mutable more than once at a time
```

## Best practices

In general, prefer stack allocation over heap allocation. Using references to
stack allocated information is preferred whenever possible. Therefore,
references are the default pointer type you should use, unless you have
specific reason to use a different type. The other types of pointers cover when
they're appropriate to use in their own best practices sections.

Use references when you want to use a pointer, but do not want to take ownership.
References just borrow ownership, which is more polite if you don't need the
ownership. In other words, prefer:

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

to

```{rust}
fn succ(x: Box<int>) -> int { *x + 1 }
```

As a corollary to that rule, references allow you to accept a wide variety of
other pointers, and so are useful so that you don't have to write a number
of variants per pointer. In other words, prefer:

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

to

```{rust}
fn box_succ(x: Box<int>) -> int { *x + 1 }

fn gc_succ(x: std::gc::Gc<int>) -> int { *x + 1 }
```

Because it doesn't matter to your caller:

```{rust}
fn succ(x: &int) -> int { *x + 1 }

fn main() {

    let x = box 5i;
    let y = box(std::gc::GC) 5i;

    println!("{}", succ(&*x));
    println!("{}", succ(&*y));
}
```

The `&*` construction first dereferences the value inside the `Box<T>` or `Gc<T>`,
and then the `&` creates a reference.

# Boxes

`Box<T>` is Rust's 'boxed pointer' type. Boxes provide the simplest form of
heap allocation in Rust. Creating a box looks like this:

```{rust}
let x = box(std::boxed::HEAP) 5i;
```

`box` is a keyword that does 'placement new,' which we'll talk about in a bit.
`box` will be useful for creating a number of heap-allocated types, but is not
quite finished yet. In the meantime, `box`'s type defaults to `std::boxed::HEAP`,
and so you can leave it off:

```{rust}
let x = box 5i;
```

# Best practices

# Shared Ownership Pointer types

## Best practices

# Returning Pointers

## Best practices

# Creating your own Pointers

This part is coming soon.

## Best practices

This part is coming soon.

# Related resources

* [API documentation for Box](std/boxed/index.html)
* [Lifetimes guide](guide-lifetimes.html)

