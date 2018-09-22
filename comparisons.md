---
title: Comparisons between Kit and other languages
layout: default
---

## C

Kit compiles to C, and layers many useful features on top. It's interoperable with C, so it's easy to replace all or part of a C codebase with a functionally equivalent but more concise version written in Kit.

Examples of features found in Kit but not C include:

* Tuples
* Traits
* Term rewriting
* "Method" function call sugar
* Namespacing
* Abstract types
* Generics
* much more

Kit aims to provide a superset of the capabilities of C. With that said, there are some things that aren't currently possible to do from Kit, including:

* Inline assembly.
* Specifying struct alignment (currently).
* Bitfields.

C's type system is weaker than Kit's; this means that some things that would be valid from C will throw compile-time errors in Kit. As an example, Kit features a true `Bool` type, and other types cannot be implicitly cast to bools, so `if 1 {}` will not compile in Kit. Kit's stance is that generally these usage patterns are more likely to be mistakes, so forbidding them reduces bugs.


## C++

C++ is an early example of a "better C" and is widely used in game development. Some comparisons:

* C++ features [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization); Kit does not.
* C++ provides object-oriented programming features, including classes; Kit provides traits and boxes (fat pointers.) A trait in Kit is similar to an interface, but can be implemented externally to type definitions, and can be implemented by even basic types (e.g. numeric types or tuples.) A box pointer must be explicitly created, so the overhead that boxes/objects add is entirely opt-in.
* C++ leverages much of the same build infrastructure as C. Kit provides its own compiler and build tool to make the build process simpler.


## Rust

[Rust](https://www.rust-lang.org/) is a safe systems programming language developed at Mozilla. Many features in Kit are inspired by Rust. In general, Rust is much stricter than Kit, which can result in a tradeoff of fewer bugs but longer iteration cycles.

* Rust, like C++, features [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization).
* One of Rust's key innovations is a borrow checker, which prevents certain kinds of simultaneous aliasing of values. This can greatly improve memory safety and prevent common bugs, but comes with a cost to ergonomics and makes some seemingly straightforward things (e.g. "doubly linked list") painful (but raw pointers can often be used as an escape hatch.) Kit doesn't provide this level of enforced safety.
* Rust disallows implementing external traits for external types, which closes off a big advantage of a trait system. For example, it's impossible to implement `Show` for `Path`, because the creators of `Path` decided they don't want you to treat it as a string. There is a valid argument against this, but in Kit the developer has the final say. As long as implementations are unambiguous, in Kit traits can be implemented for types arbitrarily.
* Rust uses a global allocator by default. Kit is designed to allow easy use of multiple custom allocators within the same codebase.


## Zig

If you want a safer, modern alternative to C, [Zig](https://ziglang.org/) is a great option. Like Kit, Zig features seamless C interop - just include a header and use it directly in your code. This means there's very little risk to trying Zig in place of C.

A couple key differences:

* *"Zig competes with C instead of depending on it."* Kit leverages C as a compile target and depends on parts of the C standard library; Kit intends to be C-compiler-independent, and offload portability concerns to the C compiler. Zig compiles to C ABI-compatible binaries, and can be built without any dependency on libc.

* *"[If Zig code doesn't look like it's jumping away to call a function, then it isn't.](https://github.com/ziglang/zig/wiki/Why-Zig-When-There-is-Already-CPP,-D,-and-Rust%3F#no-hidden-control-flow)"* Zig prioritizes readability (if we define readability here as "being able to read code and know what it will do at runtime.") Kit prefers to be concise and separate intent from mechanics; it provides metaprogramming that violate this definition of readability. Examples include [term rewriting](examples.html#term-rewriting) and [implicits](examples.html#implicits).

* Kit includes additional features inspired by functional langauges, including traits, implicits, algebraic data types and pattern matching.

* Kit uses [traits](examples.html#traits) as a mechanism for runtime polymorphism. Zig has no builtin mechanism for runtime polymorphism.

* Zig is mature; Kit is pre-alpha.


## Haxe

[Haxe](https://www.haxe.org) is a high-level language that compiles (or "transpiles") to 10+ other language targets, including C++, JavaScript and bytecode for various VMs. Haxe has been used for numerous successful games, and its use cases also include settop boxes, websites, and more.

* Haxe is a higher level language; it provides a runtime and is garbage collected. Kit has no garbage collector or runtime; your code is the only thing that runs.

* Haxe does not expose all of the functionality of the underlying platforms; for example, you can't manage memory manually in C++. Kit exposes C's functionality directly.

* Interop between Haxe and its targets requires bindings. Kit requires no bindings to leverage existing C libraries.

* Haxe is an object-oriented language. Kit's type system uses traits, and has no objects; [boxes](examples.html#boxes) must be created explicitly.


## Swift

TODO


## Nim

TODO


## Jai

As soon as it's publicly available, let's revisit.
