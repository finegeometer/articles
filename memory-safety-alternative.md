Languages take different approaches to memory safety. Some languages don't bother, leaving it up to the programmer to ensure that they don't access invalid memory. Other languages use a garbage collector, automatically cleaning up memory once it is no longer used. Rust takes a third option: it uses its lifetime system to keep the programmer from destroying values that still have references to them. 

But surely these three solutions aren't the only ones. How *else* could memory safety be ensured?

In this article, I explore another possible design for a memory-safe language with references. <sup id="a1">[Note 1](#f1)</sup>
The design is fairly similar to Rust. But it's sufficiently different that before I explain how I handle memory safety, I need to explain a few other features of this hypothetical language.

# Existential types

First, my language has a feature commonly known as *existential types*. Existential types are kind of like the opposite of generics.

Consider something with a generic type, like `thing1: fn<T>(T) -> T`. (I would prefer to write this as `thing1: forall<T> fn(T) -> T`.) Then if `T` is substituted for *any* type, `thing1` will still have that type. For example, `thing1: fn(i32) -> i32`, or `thing1: fn(Vec<String>) -> Vec<String>`.

On the other hand, consider something with an existential type, like `thing2: exists<T> (T, fn(T) -> i32)`. This just says that there is *some* type `T` for which `thing2` has type `(T, fn(T) -> i32)`. Since `T` is unknown, the only way to use `thing2` is to plug the `T` into the `fn(T) -> i32`.

There is something similar to existentials in Rust: `dyn Trait`. If `thing: dyn Trait`, then `thing` has some unknown type that implements `Trait`. This sounds like an existential! In fact,  `dyn Trait` is equivalent to `exists<T: Trait> T`.
<sup id="a2">[Note 2](#f2)</sup>

But existentials have uses beyond replicating `dyn Trait`. For example, `Vec<T>` is conceptually equivalent to `exists<N> Box<[T; N]>`. It's a boxed array of some length, but the compiler doesn't know how long it is.
<sup id="a3">[Note 3](#f3)</sup>

# Move, Copy, Del

In Rust, only values of certain types can be copied. This is determined by the `Copy` trait. But Rust allows *any* value to freely moved or destroyed.

In my hypothetical language, as well as `Copy`, there are `Move` and `Del` traits. These control whether values can be moved and/or deleted.

There is one problem I haven't solved. I want functions to be able to change the type of objects, without moving them. But I don't know how to notate the type signatures of such functions.

I suppose in a less low-level language than Rust, one could simply box everything, and thus avoid the need for `Move`. But if that's not acceptable, I have no solution.

---

Now that that's out of the way, let's see how I handle memory safety!

# Memory Safety

The goal is to enforce memory safety without a garbage collector. In the absence of references, this is handled by Rust's ownership system. I like the ownership system, and I preserve it unchanged in my hypothetical language.

Things get more interesting when references are involved. The issue is that a memory-safe language must ensure that dangling pointers are never dereferenced.

Rust does this by preventing the programmer from *creating* dangling pointers. To do so, it uses its lifetime system.

My language instead prevents the programmer from *destroying* dangling pointers. So you can create a dangling pointer, but if you do, you can't finish the program!

Rust's lifetimes are added *on top* of the type system. My strategy, on the other hand, can be implemented *within* the type system. Here's the scheme:

    - There is a type `Borrowed<L, T>`, that represents a value of type `T` that has been (immutably) borrowed by something else. `Borrowed<L, T>` does not implement `Move`, `Copy`, or `Del`.

    - There is a type `Ref<L, T>`, that represents an (immutable) reference to a value of type `T`. `Ref<L, T>` implements `Move` and `Copy`, but not `Del`.

    - Borrowing something of type `T` returns `exists<L> (Borrowed<L, T>, Ref<L, T>)`, where the `Borrowed<L, T>` goes in the same location as the original `T`.

    - A `Borrowed<L, T>` can always be converted (in place) back into a `T`.

    - A reference `Ref<L, U>` can be returned to its owner `Borrowed<L, T>`, The `Borrowed<L, T>` continues to exist (in the same location); the `Ref<L, T>` does not.

<sup id="a4">[Note 4](#f4)</sup>

The `L` parameter prevents a reference from being returned to the wrong place. Suppose you try to do so:
- Create two strings, `a` and `b`.
- Borrow them as `ref_a` and `ref_b`.
- Destroy `a`.
- *`ref_a` is a dangling pointer here!*
- Return the references to `b`.
- Destroy `b`.

This won't compile. The compiler knows that `ref_a: Ref<L1, String>`, for some unknown type `L1`. And it knows that `b: Borrowed<L2, String>`, for some unknown type `L2`. But it has no reason to think `L1` and `L2` are the same! So returning `ref_a` to `b` will fail.

As a result, if a reference ever becomes dangling, *there is no way to destroy it*. Therefore, one cannot write a complete program in which a dangling pointer exists. And that ensures memory safety.

---

Well, almost. This reasoning fails if the program never terminates at all! So in my language, panicking, aborting, and infinite-looping have to be `unsafe`.

# Comparison to Rust

In my language, there is no (usable) equivalent to Rust's `fn longest(&'a str, &'a str) -> &'a str`.

On the other hand, self-referential structs are easy:
```
exists<L> {
    vec: Borrowed<L, Vec<T>>,
    slice: Ref<L, [T]>
}
```

It's unfortunate that I'm forced to make infinite loops unsafe. While code that hangs is annoying, I worry that forbidding it will have a side-effect of disallowing too many correct programs.

On the other hand, I like the absence of panics. I've seen several Rust APIs that are annoyingly complicated because they have to consider the possibility of a panic. Mutex poisoning is the first example that comes to mind, but I know I've seen others.

It's worth noting that getting rid of `panic!()` does *not* mean I have to get rid of `todo!()`. 
Instead, `todo!()` should be a signal to the compiler to only do typechecking, not code generation. Then `todo!()` *has* no runtime behavior, so it doesn't need to panic.

# Conclusion

This has been an exploration of one interesting way to ensure memory safety. Can you come up with any others?

---

<b id="f1">Note 1</b> I tried to implement this language, so a demonstration could accompany this article. But it turns out that making a language is hard! I got pretty far, but gave up on implementing the trait system. <sup id="a5">[Note 5](#f5)</sup> [↩](#a1)

<b id="f2">Note 2</b> For simplicity, I'm ignoring the issue of sized vs. unsized types. [↩](#a2)

<b id="f3">Note 3</b> I've sometimes wondered what it would take to be able to implement `Vec<T>` safely. I suspect existential types would be a requirement. [↩](#a3)

<b id="f4">Note 4</b> The error "cannot be moved out of borrowed content" becomes "`Borrowed<L, T>` does not implement `Move`"! [↩](#a4)

<b id="f5">Note 5</b> Thanks to [this page](https://github.com/seamusdemora/seamusdemora.github.io/blob/master/GFM_FootnotesWithReturnFeature.md) for showing me how to implement these footnotes! [↩](#a5)


