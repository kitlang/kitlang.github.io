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


Why you should use Kit in place of:

| C/C++ | a higher level language |
| --- | --- |
| *Modern language features*: type inference, algebraic data types, pattern matching, explicit function inlining, automatic pointer dereferencing, generics, implicits. | Full control over *performance*: pointers, manual memory management, no GC (unless you introduce it yourself, which is easy!) |
| A more expressive type system, including *traits* for polymorphism, and *abstract types*, which provide custom compile-time behavioral and type checking semantics to existing types with no runtime cost. | Metaprogramming via a typed *term rewriting system*; use rules to transform arbitrary expressions at compile time based on their type information. Create your own interface or DSL. |
| A sane, easy to use *build system*. Kit features modules, imports, and standard package structure, plus a simple but powerful build tool: manage your project via a simple YAML configuration file and `kit build`, `kit test`, or `kit run`. (coming soon...) | *Zero-overhead C interoperability*. Take advantage of existing C libraries without any wrappers; just include the header and directly use types/functions/variables. |

*Kit is pre-alpha and not all features are fully implemented; see the [roadmap on Trello](https://trello.com/b/Bn9H0fzk/kit).*
