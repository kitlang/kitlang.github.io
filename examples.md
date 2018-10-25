---
title: Examples
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

Unless you know what you're doing, don't create a root-level prelude.kit file! The root-level prelude is how Kit imports its own standard library. You can, but probably shouldn't, override it.

### The main module

A single module will serve as the "main" module, which is the entry point for compilation; all modules imported from this module, and all modules imported from those modules recursively, will be compiled. If compiling an executable (the default), the main module should have a function called `main`, which can be typed as `Int16` or `Void`.


Types
-----

Types in Kit can define static fields/methods as well as instance methods.

### Basic Types

#### Numeric types

Numeric types include:

- Signed integers: `Int8`, `Int16` (`Short`), `Int32`, and `Int64` (`Long`)
- Unsigned integers: `Uint8` (`Byte`), `Uint16`, `Uint32`, `Uint64`
- Floating point numbers: `Float32` (`Float`), `Float64` (`Double`)
- `Char`, corresponding to C's `char`
- `Int`, corresponding to C's `int` (which may vary by platform)
- `Size`, corresponding to C's `size_t`, representing a pointer-sized value

All numeric types implement the builtin [trait](#traits) `Numeric`. Integer types also implement `Integral`, while floating point types implement `NumericMixed`.

#### Boolean

~~~kit
var a = true;
var b = false;
~~~

#### Pointers

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

#### Strings

Kit supports native C null-terminated strings using the `CString` type.

~~~kit
var s: CString = "hello!";
~~~

TODO: Kit length-prefixed strings

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

### CArrays

C arrays can be either explicitly sized or unsized:

~~~kit
var x: CArray[Int, 3] = [1, 2, 3];
~~~

~~~kit
// the "unsized array struct hack"
struct IntValues {
    var length: Size;
    var values: CArray[Int];
}
~~~

The special value `empty` can be used to initialize an explicitly empty array.

### Compound types

Compound types - structs, unions, enums and abstracts - are built from existing types and share a common set of features.

#### Structs and unions

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

The special value `empty` can initialize an empty struct or union.

#### Enums/algebraic data types

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

An enum may have an *underlying type*; variants of the enum which don't have internal data will be values of the specified type:

~~~kit
enum Status: CString {
    Success = "success";
    Error = "error";
    InProgress = "in progress";
}
~~~

#### Abstracts

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

Abstracts unify with their underlying type in only one direction. Given `abstract Color: Uint32`:

- A `Color` can be used anywhere a `Uint32` is expected.
- However, a `Uint32` must be explicitly cast to be promoted to a Color:

~~~kit
var c: Color = 0xffffff_u32 as Color;
~~~

- Types which share the same underlying type can also be converted via explicit casting:

~~~kit
abstract RgbaColor: Uint32;

var c: Color = 0xffffff_u32 as Color;
var a: RgbaColor = c as RgbaColor;
~~~

The underlying type of an abstract can be another abstract, in which case the true runtime type is the parent's runtime type; otherwise, the same abstract conversion rules apply.

### Compound type features

#### Instance methods

Compound types can declare instance methods:

~~~kit
abstract MyType: Int {
    public function printMe() {
        printf("%i", this);
    }
}
~~~

An instance method is a function which takes an instance of the type as an implicit first argument. There are two ways to call instance methods, which are equivalent:

~~~kit
var a = 1 as MyType;

// call on the instance
a.printMe();

// pass the instance explicitly
MyType.printMe(a);
~~~

The implicit first argument is passed in as a pointer. Within the method, `this` and `&this` can be used to refer to the value and the pointer respectively.

#### Static variables and methods

Compound types can declare static variables and static methods to group associated functions and variables that don't take an initial instance value within the same namespace.

~~~kit
struct MyData {
    // this is a static variable; it exists in only one place
    public static var b: Int;

    // this static field can't be reassigned
    public static const MY_CONST: Int = 123;

    // this is a struct field; each MyData value will have one
    public var a: Int;

    public static function staticMethod() {
        // this function does *not* take an instance of MyData
    }
}
~~~

Static variables and methods can be accessed using the name of the type:

~~~kit
MyData.staticMethod();
~~~

### Typedefs

Typedefs create aliases for existing types:

~~~kit
typedef StringList = List[String];
~~~

Typedefs are not unique types like [abstracts](#abstracts); they're simply a reference to the existing type.

### Self

`Self` is a special reference type:

- In a compound type's fields or methods, it refers to the type itself.
- In a trait definition, it refers to the trait.
- In a trait implementation, it refers to the implementing type.

`Self` can generally be used in the same places as [`this`](#this) and will have the same type as the value of `this`; an exception is static methods, where `Self` is allowed and `this` is not. Outside of these contexts, `Self` may not be used.


Expressions
-----------

### Variables

~~~kit
var a: Int = 1;
// uninitialized; will rely on type inference to determine the type
var b;
// single-assignment; can't be reassigned or declared without initializing
const c: CString = "hello";
~~~

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
// char
var alphabetSize = c'z' - c'a' + 1;
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

There are two types of for loops in Kit: numeric for loops, and iteration over collections.

`n...m` is a special intrinsic which, when used in a for loop, will compile to a simple C for loop:

~~~kit
for i in 1 ... 5 {
    printf("%i\n", i);
}
~~~

`for` loops can also be used to iterate over values which implement the [`Iterable` trait](#kititerator)

It's also possible to use [rewrite rules](#term-rewriting) to optimize for loops over specific types into some other form.

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
    const b: Int = 2;
    var c: Int = 3;
}

var s: MyStruct = struct MyStruct {
    a: 1,
    // we can provide a different value for `const b` but we won't be able to
    // reassign it afterward
    b: 4,
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

Traits, similar to [typeclasses](https://en.wikipedia.org/wiki/Type_class), are interfaces which can be implemented for a type. Once a trait is implemented for a given type, values of that type can be converted to "boxed" pointers (using the [`Box[T]`](#boxes) type) which look the same regardless of the type's identity; this enables open polymorphism in Kit:

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

Unlike traditional object-oriented interfaces,

- Traits can be implemented for *any* type, even basic types like `Int` or `String`.
- Trait implementations are separate from the type's definition, so you can implement a trait for a type you didn't define. This enables more extensible and modular code.

Traits can be used for both compile-time and runtime polymorphism.

Static dispatch to trait implementation methods is possible using the name of the trait and method:

~~~kit
implement MyTrait for Int {
    public function exclaim() {
        printf("hello from %i\n", this);
    }
}

function main() {
    var i = 1_i;
    i.MyTrait.exclaim();
}
~~~

### Boxes

The `Box[T]` type is used to create boxed pointers, which can call any of a trait's methods on a value implementing the trait.

~~~kit
function greet(w: Box[Writer]) {
    w.write("hello!");
}

function main() {
    var f = File.write("/tmp/greeting");
    greet(f);
}
~~~

Since a box contains a pointer to the value used to create it, the box's useful lifetime is the same as the underlying pointer. Care should be taken when retaining boxes of stack allocated values, or values in heap memory which may have been freed.

### Trait constraints

Using traits as type annotations creates a trait constraint, meaning a value will be some specific type which implements the trait.

~~~kit
var a: Writer;
~~~

A [generic](#generics) can also be constrained to types implementing a trait:

~~~kit
function greet[W: Writer](w: W) {
    w.Writer.write("hello");
}
~~~

This incurs no runtime cost as there is no boxing involved; at compile time, we will know the exact value of the type being used.

There's an important distinction between trait constraints and boxes:

- `var a: Trait`: variable `a` is *one specific type* which implements the trait.
- `var a: Box[Trait]`: variable `a` is a boxed pointer; underneath that pointer could be *any type* implementing the trait.

This means that a `List[Trait]` will contain members that are all the *same* type, but a `List[Box[Trait]]` can contain pointers to members of different types that all implement the same trait.

#### Specialization

Sometimes a value is constrained to types implementing a certain trait, but the compiler doesn't have enough information to determine *which* specific type should be used; the choice may be somewhat arbitrary. This happens frequently with numeric types.

~~~kit
// if we don't have any more information, what type should `a` be?
var a = 1;
~~~

In these cases, traits can be "specialized" to provide a default implementation when none is specified. Above, `Numeric` has been specialized as `Int`, so `Int` will be used by default. If we had encountered additional information about the value's type, the specialization may have gone differently:

~~~kit
var a = 1;
// given the next line, we know `a` must hold mixed numbers, so it will
// specialize to Float instead of Int
a += 2.5;
~~~

`Map` is another example of trait specialization in action, allowing users to create a `Map` value with `Map.new()` and have its type filled in automatically based on the key:

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

### Associated types

Sometimes parameters of generic traits that vary by trait implementation can be determined by the trait implemention itself, and shouldn't need to be specified whenever the trait is used. In these cases the trait can declare "associated types" - each implementation can specify a specific type for each associated type parameter, and these parameters will be implicit in the trait.

An example is `Iterable`:

~~~kit
// note parens instead of brackets
trait Iterable(IteratorT) {
    public function iterator(): Box[Iterator[IteratorT]];
}
~~~

`Iterable` allows a type to be iterated over using a `for` loop, by specifying how to get a boxed `Iterator` value for values of the given type. Because `Iterable` has no generic type parameters, only associated type parameters, each type may only implement `Iterable` once. The resulting iterator generates values of some type, which is a function of the collection type:

~~~kit
implement Iterable(Int) for MyList[Int] {
    public function iterator(): Box[Iterator[Int]] {
        // ...
    }
}
~~~

`MyList[Int]` is iterable, and always generates an iterator over `Int` values.

Without associated types, we would have a generic trait `Iterable[T]`; in addition to the verbosity of the additional parameter, it would be possible to implement `Iterable` multiple times for the same type, resulting in ambiguity:

~~~kit
trait Iterable[T];

// which instance should we use in a for loop?
implement Iterable[Int] for MyList[Int];
implement Iterable[Float] for MyList[Int];
~~~

Traits can declare both parameters and associated types:

~~~kit
trait MyTrait[ExplicitT, ExplicitU](AssociatedT, AssociatedU);
~~~


Generics
--------

Generics are parameterized functions, types or traits. In other words, they're function, type or trait "templates", which won't be complete until they're filled in with specific parameter types.

Generic type parameters are always **invariant**, meaning that types that can be implicitly converted between each other in some contexts *cannot* be used interchangeably as type parameters. For example, this means even though `Int` and `Float` can sometimes be used interchangeably (implicitly converted from one to the other), a `List[Int]` and a `List[Float]` are *not* the same type and will not unify in any context.

### Generic functions

To make a function generic, declare it with one or more named type parameters in square braces:

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

To access a static field or method on a generic type, you can specify the type parameters, or leave them blank to allow type inference to try to fill them in.

~~~kit
var l: List[Int] = Empty;
printf("%zu", List[Int].length(l));
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

### Using C from Kit

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

### Using Kit from C

You may also want to compile Kit into a library and use it from C code. Two things to keep in mind:

- Use the `--lib` flag to compile a library instead of an executable.
- Functions in Kit are namespaced, which involves name mangling. To expose them to C, use the `#[extern]` metadata on your function to prevent name mangling:

~~~kit
#[extern]
function myExternFunction() {
    // you can call this from C as `myExternFunction`
    printf("hello from Kit!\n");
}
~~~

Keep in mind that `#[extern]` creates a global name; care should be taken so that global names don't clash with each other, or with existing names in your C code.


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
        var a = (1_i, 2_i) + (3_i, 4_i);
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

To reduce ambiguity, implicit values are **invariant**, i.e. their types must match exactly. This means that even though the following is valid in some contexts:

~~~kit
function f(value: Float) {
    // ...
}

var x: Int = 1;
// this is fine, since Int normally unifies with Float
f(x);
~~~

it won't work with implicits:

~~~kit
function f(value: Float) {
    // ...
}

var x: Int = 1;
using implicit x {
    // this won't work!
    // we have an implicit Int in scope, but not an implicit Float
    f();
}
using implicit x as Float {
    // this is fine
    f();
}
~~~


Standard library
----------------

Kit includes a compact standard library; some modules of the standard library are imported by default via the root-level [`prelude` module](#prelude-modules)

### kit.array

TODO

### kit.cmp

TODO

### kit.common

Various core types such as `CArray` are defined here.

### kit.either

The `Either[A, B]` enum represents values which can take one of two types. Among other use cases, it can be used to return error information from a function.

### kit.io

TODO

### kit.iterator

The two traits defined in `kit.iterator`, `Iterable` and `Iterator`, are used in [`for` loops](#for).

~~~kit
var list: List[Int];
for item in list {
    // ...
}
~~~

For loops are often used to iterate over arrays, lists, or other collection types; Kit makes this usage easy by adding syntactic sugar via the `for` loop.

#### Iterable

A value that can be iterated over must implement `Iterable`:

~~~kit
implement Iterable(Int) for MyIntList {
    // ...
}
~~~

This trait has one associated type, which is the type of values produced by the iterator. `Iterable` has one method, `iterator`, which when called produces a value that implements the `Iterator[T]` trait. This trait is for iterator values, i.e. values which track the state of a single iteration pass.

#### Iterator

`Iterator[T]` has a single method, `next`:

~~~kit
trait Iterator[T] {
    public function next(): (Box[Iterator[T]], Option[T]);
}
~~~

The `next` method returns a tuple containing a new iterator (or the same iterator for iterators containing mutable state) and an [`Option` value](#kitoption), which could be `Some(value)` or `None` if the iterator is exhausted.

Given an implementor of `Iterable`, Kit will call `iterator()` once, then call `next()` repeatedly in a loop until a `None` value is returned.

### kit.list

The `List[T]` type is an enum representing a singly linked list. Each non-empty node contains a pointer to another `List[T]` node.

### kit.macro

TODO

### kit.map

TODO

### kit.mem

Contains helpers for memory allocation. Types that require heap allocation often provide a constructor that accepts an implicit `Box[Allocator]` as the first argument. This will use `malloc` by default, but allows easily swapping in a custom allocator in a given scope.

Basic `LinearAllocator` and `StackAllocator` implementations are also contained in this module.

### kit.numeric

This module contains numeric typeclasses such as `Numeric`, `Integral` and `NumericMixed`, which are used to describe the various builtin numeric types.

### kit.option

A single enum, `Option[T]`, is used to express optional values. Variant `Some(value)` is used when a value is present, and `None` is used otherwise.

### kit.pointer

TODO

### kit.random

TODO

### kit.string

TODO

### kit.sys

#### kit.sys.dir

Used to read directory contents:

~~~kit
import kit.sys.dir;

var dir: DirectoryReader = readDir("/tmp");
~~~

#### kit.sys.file

The helper type `File` is used for file input/output.

~~~kit
if !File.exists("temp.txt") {
    // open a file for writing
    var f = File.write("temp.txt");
    f.writeBytes("hello", 5);
    f.close();

    // open for reading
    var f2 = File.read("temp.txt");
    var buf = malloc(5);
    f2.readBytes(buf, 5);

    File.remove("temp.txt");
}
~~~

#### kit.sys.path

Abstraction over file paths. `Path` is an abstract over `CString`, but `CStrings` can be automatically promoted to `Path` without requiring a cast.

### kit.vector

TODO
