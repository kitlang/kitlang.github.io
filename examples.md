---
layout: default
---


Modules
-------

Kit code is structured into ".kit" module files. Kit has a simple module system that mirrors the directory structure of your project.

### Source directories

By default, Kit will look for source files in the local "src" directory; you can customize this with one or more alternate source directories using the -s flag to kitc.

### Imports

Importing a module is necessary to reference declarations in that module. Modules are referred to by their path, which begins from the source directory. For example, the file src/pkg/a.kit would be imported from other modules as:

~~~kit
import pkg.a;
~~~

If you have multiple source directories, they'll be searched in priority order; the first "pkg/a.kit" found will be used.

Circular imports are allowed.

### Prelude modules

To reduce duplicate import statements across your project, you can create any number of `prelude.kit` files in the project's subdirectories. For every package (subdirectory) in your project, all files in that package or its children will automatically copy the contents of this package's `prelude` module if it exists. Importantly, they do not *import* the prelude file; they copy the contents. This means the prelude should not contain declarations; instead, `import`s and [`using` statements](#using) can be convenient in a prelude module when files in a package share common dependencies.

For example, a module "pkg1.pkg2.module" will look for preludes in the following locations:

- pkg1.pkg2.prelude
- pkg1.prelude
- prelude

Unless you know what you're doing, don't create a root-level prelude.kit file! The root-level prelude is how Kit imports its own standard library.

### The main module

A single module will serve as the "main" module, which is the entry point for compilation; all modules imported from this module, and all modules imported from those modules recursively, will be compiled. If compiling an executable (the default), the main module should have a function called `main`, which can be typed as `Int16` or `Void`.


Types
-----

Types in Kit can define static fields/methods as well as instance methods.

### Numeric types

Numeric types include:

- Signed integers: `Int8`, `Int16` (`Short`), `Int32`, and `Int64` (`Long`)
- `Char`, corresponding to C's `char`
- `Int`, corresponding to C's `int` (which may vary by platform)
- `Size`, corresponding to C's `size_t`, representing a pointer-sized value
- Unsigned integers: `Uint8` (`Byte`), `Uint16`, `Uint32`, `Uint64`
- Floating point numbers: `Float32` (`Float`), `Float64` (`Double`)

All numeric types implement the builtin [trait](#traits) `Numeric`. Integer types also implement `Integral`, while floating point types implement `NumericMixed`.

### Boolean

~~~kit
var a = true;
var b = false;
~~~

### Pointers

A value of type `Ptr[T]` is a pointer to a value of type T. They can be referenced and dereferenced using the `&` and `*` prefix operators.

~~~kit
var x: Int = 1;
var y: Ptr[Int] = &x;
var z: Int = *y;
~~~

You can usually let type inference handle pointer referencing/dereferencing for you automatically.

~~~kit
var x: Int = 1;
// these will work automatically
var y: Ptr[Int] = x;
var z: Int = y;
~~~

### Strings

Kit supports native C null-terminated strings using the `CString` type.

~~~kit
var s: CString = "hello!";
~~~

TODO: Kit length-prefixed strings

### Structs and unions

~~~kit
struct MyStruct {
    public var publicField: Int;
    var privateField: Int = 1;
}

function main() {
    // pointer auto-dereferencing on field access
    var x: Ptr[MyStruct] = struct MyStruct {
        publicField: 1,
    };

    printf("%i", x.publicField);
}
~~~

~~~kit
union MyUnion {
    var intField: Int;
    var floatField: Float;
    var stringField: CString;
}
~~~

### Enums/algebraic data types

Enums are "algebraic data types" and can optionally contain internal data.

Enums that don't have any values which contain internal data will be optimized to C enums.

Enums *cannot* contain themselves, since the size of the structure would be unknown; however, they can contain pointers to themselves.

~~~kit
/**
 * Represents a nullable value.
 */
enum Option[T] {
    Some(value: T);
    None;

    public function unwrap(): T {
        // enums can be destructured using match expressions
        match this {
            Some(value): value;
            default: throw "Attempt to unwrap an empty Option value";
        }
    }
}
~~~

An enum may have an *underlying type*; variants of the enum which don't have inernal data will be values of the specified type:

~~~kit
enum Status: CString {
    Success = "success";
    Error = "error";
    InProgress = "in progress";
}
~~~

### Tuples

Tuples are containers for heterogeneous types; they don't have a name and don't require a declaration.

~~~kit
var a: (Int, Float, CString) = (1, 2, "hello!");
// destructure tuples via assignment:
var b: Int;
(b, _, _) = a;
// ...or access fields directly using numeric constants:
printf("%.2f", a[1]);
~~~

### Abstracts

Abstract types wrap an existing type with additional compile-time semantics: they allow separating different contexts in which the same type is used, and can have instance methods. Abstract types are zero-cost abstractions; at runtime, they'll be indistinguishable from the underlying type.

~~~kit
/**
 * Provides convenience methods for working with colors. Variables can be
 * typed as Colors, but at runtime they'll be `uint_32t` with zero overhead.
 */
abstract Color: Uint32 {
    public function getRgb(): (Float, Float, Float) {
        return (this & 0xff0000 >> 16, this & 0xff00 >> 8, this & 0xff);
    }
}

function printRgb(c: Color) {
    print(c.getRgb());
}

function main() {
    printRgb(0xff8080);
}
~~~

Abstracts inherit some semantics from their underlying type by default, including trait implementations and methods. The abstract can declare its own methods or trait implementations which will take precedence over that of the underlying type.

### Typedefs

Typedefs are short names for existing types:

~~~kit
typedef StringList = List[String];
~~~

Typedefs are not unique types like [abstracts](#abstracts); they're simply a reference to the existing type.


Expressions
-----------

### Blocks

~~~kit
{
    printf("hello");
    printf(" world\n");
}
~~~

When used as an expression, the block's final expression is used as the block's value.

### Literals

~~~kit
// int
var a = 1;
// float
var b = 1.0;
// bool
var c = true;
// C (null-terminated) string
var d = "hello";
~~~

Generally type inference will type literals with the smallest type that would hold them and satisfy other program constraints. Literals can also be explicitly typed by using a type suffix:

~~~kit
// Uint64
var a = 1_u64;

// Char, Int, Size
var a = 1_c;
var b = 1_i;
var c = 1_s;
~~~

### this

`this` can be used in instance methods to reference the value on which the method was called.

~~~kit
struct MyStruct {
    var field1: Int;
    var field2: Float;

    public function myMethod() {
        printf("%i", this.field1);
    }
}
~~~

### for

TODO

### while

Kit supports both `while` and `do`-`while` loops.

~~~kit
var n = 1;
while true {
    if ++n > 10 {
        break;
    }
}

do {
    --n;
} while n > 0;
~~~

### if

`if` can be used as a statement, with optional `else`:

~~~kit
if false {
    printf("this can never happen!\n");
} else {
    printf("this will always happen!\n");
}
~~~

It can also be used as an expression; `if` expressions require an `else` clause, and the type of both must match:

~~~kit
var a = if true {
    "true";
} else {
    "false";
};
~~~

A shorter version of `if` expressions also exists when the branches don't need to be blocks; this requires the `then` keyword:

~~~kit
var a = if true then "true" else "false";
~~~

### inline

TODO

### Fields

Structs and unions have fields, which can be accessed by name:

~~~kit
var s = struct MyStruct {a: 1, b: 2};
var x = s.a;
~~~

Field access on pointers to structs will automatically dereference the pointer.

### Struct literals

~~~kit
struct MyStruct {
    var a: Int;
    var b: Int;
    var c: Int = 3;
}

var s: MyStruct = struct MyStruct {
    a: 1,
    b: 2,
    // since c has a default value, it is optional
};
~~~

### Enum literals

~~~kit
enum Value {
    BoolValue(b: Bool);
    IntValue(i: Int);
    FloatValue(f: Float);
    NullValue;
}

var a = BoolValue(true);
var b = NullValue;
~~~

### Tuple literals

~~~kit
var a: (Int, Float) = (1, 2);
// tuple members can be accessed by number, but the index must be a constant
var f: Float = a[1];
~~~

### match

`match` statements can be useful to avoid long if-else chains. They can include an optional `default` clause.

~~~kit
function f(i: Int) {
    match i {
        1 => printf("one\n");
        2 => printf("two\n");
        3 => printf("three\n");
        default => printf("???\n");
    }
}
~~~

It can also be used to access the inner values of enums depending on the variant:

~~~kit
match Value {
    BoolValue(true) => 1;
    FloatValue(f) => 2;
    IntValue(i) => i;
    default => 5;
}
~~~

`match` can be used as an expression; the types of all match bodies must match, and the match patterns must be exhaustive (or have a default clause.)

### defer

TODO

### box

TODO

### sizeof

`sizeof(Type)` will return the size in bytes of the given type:

~~~kit
var a = sizeof(Int);
var b = sizeof(MyStruct[Int, Int]);
~~~

### using

`using` can bring [rewrite rules](#term-rewriting) or [implicit values](#implicits) into scope:

~~~kit
using rules RuleSet1, implicit MyValue {
    // ...
}
~~~

`using` can be used at the module level, or to open an expression block:

~~~kit
using rules RuleSet1;

function main() {
    using implicit MyValue {
        // ...
    }
}
~~~


Traits
------

Traits (typeclasses) enable open polymorphism in Kit:

~~~kit
trait Writer {
    function write(s: String): Void;
}

implement Writer for File {
    function write(s) {
        fwrite(s, 1, s.length, this);
    }
}
~~~

Traits enable both compile-time and runtime polymorphism; a [generic](#generics) can be constrained to types implementing a trait:

~~~kit
function greet[W: Writer](w: W) {
    w.write("hello");
}
~~~

### Boxes

Values can be implicitly or explicitly cast to traits they implement, creating a "fat pointer" capable of calling the type's trait methods. This is done using the `Box[T]` type.

~~~kit
function greet(w: Box[Writer]) {
    w.write("hello!");
}

function main() {
    var f = File.write("/tmp/greeting");
    greet(f);
}
~~~

There's an important distinction between trait constraints and boxes:

- `var a: Trait`: variable `a` is *a specific type* which implements the trait
- `var a: Box[Trait]`: variable `a` could be *any type* implementing the trait

This means that a `List[Trait]` will contain members that are all the *same* type, but a `List[Box[Trait]]` can contain pointers to members of different types that all implement the same trait.

### Specialization

Sometimes a value is constrained to types implementing a certain trait, but the compiler doesn't have enough information to determine *which* specific type should be used; the choice may be somewhat arbitrary. This happens frequently with numeric types.

~~~kit
// if we don't have any more information, what type should `a` be?
var a = 1;
~~~

In these cases, traits can be "specialized" to provide a default implementation when none is specified.

~~~kit
trait Map[K, V] {
    function get(key: K): V;
    function set(key: K, value: V): V;
    function exists(key: K): Bool;
}

// when we know the key type, substitute a default implementation
specialize Map[Bool, V] as BoolMap[V];
specialize Map[Integral, V] as IntMap[V];
specialize Map[String, V] as StringMap[V];

function main() {
    var myMap: Map[Int, String] = Map.new();
    myMap[5] = "this just works!";

    var partialMap: Map[Int] = Map.new();
    partialMap[5] = "this works too thanks to parameter inference";
}
~~~


Generics
--------

Generics are parameterized functions, types or traits. In other words, they're functions, types or trait "templates", which won't be complete until they're filled in with specific parameter types.

### Generic functions

To make a function generic, declare it with one or more named type parameters:

~~~kit
function genericFunc[T](value: T) {
    // ...
}
~~~

The compiler will generate a new version of `genericFunc` every time it sees a new type used for T.

~~~kit
// these are calls to two different functions, since T is different
genericFunc(1);
genericFunc(true);
~~~

In this example, parameter `T` could be any type; a version will be generated with each specific type `T` observed during compilation. We know nothing about the capabilities of type `T`, since it could be anything; this means we can introduce compile-time errors by calling the function with an inappropriate type. For additional safety, type parameters can be constrained to specific trait members:

~~~kit
function add[T: Numeric](a: T, b: T): T {
    // this is safe; all T values will be Numeric, so they support addition
    return a + b;
}
~~~

### Generic types

`List` is a parameterized type which can hold values of any other type:

~~~kit
enum List[T] {
    Cons(head: T, tail: Ptr[List[T]]);
    Empty;
}
~~~

and `List[Int]` is a specific type of `List` containing `Int` values.

~~~kit
var x: List[Int] = Cons(1, Empty);
~~~

If you don't specify all of the type parameters, the compiler will try to infer the missing types:

~~~kit
// this works; it'll infer that this is a List[Int8]
var x: List = Cons(1, Empty);
~~~

### Generic traits

Generic traits are parameterized and can reference their type parameters in trait methods.

~~~kit
// allocates values of type T
trait Allocator[T] {
    // a unique version of alloc/free will be generated for every type that
    // matches constraint T
    function alloc[U: T](): Ptr[U];
    function free[U: T](ptr: Ptr[U]): Void;
}
~~~


C interoperability
------------------

Kit features seamless interoperability with existing C libraries. Kit compiles to C and its type system exposes C types directly. You can call C functions from Kit or Kit functions from C, by explicitly including header files, no additional bindings required.

~~~kit
include "stdio.h";

function main() {
    printf("%s\n", "Hello from Kit!");
}
~~~

Anything declared in the header will be type checked, and `#define` macros can also be used by providing type annotations at the usage site.

~~~kit
include "SDL2/SDL.h";

function helloSDL() {
    SDL_Init(${SDL_INIT_VIDEO: Uint});
    var window: Ptr[SDL_Window] = SDL_CreateWindow(
        "Hello from Kit!",
        ${SDL_WINDOWPOS_UNDEFINED: Uint},
        ${SDL_WINDOWPOS_UNDEFINED: Uint},
        640, 480,
        ${SDL_WINDOW_SHOWN: Uint}
    );
    SDL_Delay(750);
    SDL_DestroyWindow(window);
    SDL_Quit();
}
~~~


Term rewriting
--------------

Term rewriting rules allow compile-time syntax transformation, which lets you define optimizations:

~~~kit
import kit.math;

rules FastMath {
    (sin(pi/2)) => 1;
    (sin(pi)) => 0;
    (sin(1.5*pi)) => -1;
    (sin(2*pi)) => 0;

    (cos(pi/2)) => 0;
    (cos(pi)) => -1;
    (cos(1.5*pi)) => 0;
    (cos(2*pi)) => 1;
}

function main() {
    using rules FastMath {
        // this triggers a function call...
        var a = sin(pi * 0.75);
        // ...but this avoids one thanks to the rewrite rule
        var b = sin(pi / 2);
    }
}
~~~

as well as custom semantics:

~~~kit
rules TupleMath {
    // add tuples componentwise
    (${a: (Int, Int)} + ${n: (Int, Int)}) => ($a[0] + $n[0], $a[1] + $n[1]);
}

function main() {
    using rules TupleMath {
        var a = (1_i32, 2_i32) + (3_i32, 4_i32);
    }
}
~~~

Rules can match based on both the AST structure and value type information:

~~~kit
struct MyStruct {
    var myField: Int;
}

rules StructRules {
    (${s: MyStruct}.property) => $s.myField;
}

function main() {
    var s = struct MyStruct {myField: 5};
    using rules StructRules {
        printf("%li\n", s.property);
    }
}
~~~

Types can define their own rules, which automatically enter scope for expressions containing the type:

~~~kit
enum List[T] {
    Cons(head: T, tail: List[T]);
    Empty;

    rules {
        // Replace iteration over lists with an optimized version to avoid
        // creating an unnecessary ListIterator struct.
        (for $x in $this {$e}) => {
            var __rest = this;
            while !__rest.empty {
                var $x = __rest.head;
                $e;
                __rest = __rest.tail;
            }
        }

        (${head: T} :: ${tail: List[T]}) => Cons($head, $tail);

        // Define a custom `++` operator to combine lists
        ($this ++ ${other: Self}) => {
            var newList = Empty;
            for i in this {
                newList = i :: newList;
            }
            for i in other {
                newList = i :: newList;
            }
            return list.reverse();
        }
    }
}
~~~

Rules can also be brought into module scope:

~~~kit
using rules Reduce;

function main() {
    var a = 3;
    var b = 4;
    var c = pow(a + b, 2);
}
~~~

When relying on types to trigger rewrite rules, it's a best practice to include explicit type annotations. In some cases, such as when types are found via [trait specialization](#specialization), it's not guaranteed that any relevant rewrite rules will take effect.


Implicits
---------

When implicit values are in scope, they'll be used as arguments in functions automatically. A function will look for matching implicit values for each of its arguments from left to right; it will stop looking as soon as it fails to find an implicit for an argument, so implicit arguments must be contiguous and must be the first arguments of the function.

~~~kit
function getConfigSection(config: Config, sectionName: CString) {
    return config.get(sectionName);
}

function main() {
    var cfg = defaultConfig();

    using implicit cfg {
        var settings = getConfigSection("settings");
        var controls = getConfigSection("controls");
    }
}
~~~

Values can also be implicit at the module level; this can allow using global state as an overrideable default:

~~~kit
var x: Config = defaultConfig();
using implicit x;

function main() {
    var settings = getConfigSection("settings");
    // override the default temporarily
    var controls = using implicit otherConfig (getConfigSection("controls"));
}
~~~

~~~kit
using implicit malloc;

function main() {
    // allocate an object on the heap; MyObject's "constructor" takes an allocator as argument
    var myObject = MyObject.new();
}
~~~

You can also access the implicit in scope for a given type using an `implicit` expression:

~~~kit
var f: Float;
using implicit f {
    // since f is in scope as an implicit value, assigns g to f
    var g = implicit Float;
}
~~~
