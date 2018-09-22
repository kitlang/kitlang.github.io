---
layout: default
---

**Kit** is a programming language designed for creating concise, high performance cross-platform applications. Kit compiles to C, so it's highly portable; it can be used in addition to or as an alternative to C, and was designed with game development in mind.


~~~kit
include "stdio.h";

function main() {
    var s: CString = "Hello from Kit!";
    printf("%s\n", s);
}
~~~


**[See more code examples here](examples.html)**

**At a glance:**

* Kit has a **strong, static** type system to catch errors at compile-time.
* Kit is a **procedural language**, not object-oriented or functional; however, [traits](examples.html#traits), [boxes](examples.html#boxes) and [abstracts](examples.html#abstracts) can simulate object-oriented interfaces and polymorphism.
* Kit **compiles to C**, which then compiles to native libraries or executables.
* Memory management in Kit is **manual** (no automatic garbage collection), with some convenience features to make this easier.

*Kit is pre-alpha and not all features are fully implemented; see the [roadmap on Trello](https://trello.com/b/Bn9H0fzk/kit).*

**Why you should use Kit in place of:**

| C/C++ | a higher level language |
| --- | --- |
| Modern language features: type inference, [algebraic data types](examples.html#enumsalgebraic-data-types), [pattern matching](examples.html#match), explicit function [inlining](examples.html#inline), automatic [pointer dereferencing](examples.html#pointers), [generics](examples.html#generics), [implicits](examples.html#implicits). | Low-level control to optimize performance: [pointers](examples.html#pointers), manual memory management, no GC (unless you introduce it yourself, which is easy!) |
| A more expressive type system, including [traits](examples.html#traits) for polymorphism, and [abstract types](examples.html#abstracts), which provide custom compile-time behavioral and type checking semantics to existing types with no runtime cost. | Metaprogramming via a typed [term rewriting system](examples.html#term-rewriting); use rules to transform arbitrary expressions at compile time based on their type information. Create your own interface or DSL. |
| A sane, easy to use build system. Kit features modules, imports, and standard package structure, plus a simple but powerful build tool: manage your project via a simple YAML configuration file and `kit build`, `kit test`, or `kit run`. (coming soon...) | Zero-overhead [C interoperability](examples.html#c-interoperability). Take advantage of existing C libraries without any wrappers; just include the header and directly use types/functions/variables. |

**[See more comparisons here](comparisons.html)**
