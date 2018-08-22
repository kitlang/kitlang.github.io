---
layout: default
---

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
var a = 1_u64;
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


Types
-----

All types in Kit can define static fields/methods as well as instance methods.

### Structs and unions

~~~kit
struct MyStruct {
    public var myField: Int;
}

function main() {
    // pointer auto-dereferencing on field access
    var x: Ptr[MyStruct] = struct MyStruct {
        myField: 1
    };

    printf("%li", x.myField);
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

Traits enable both compile-time and runtime polymorphism; a generic function can be constrained to types implementing a trait:

~~~kit
function greet[W: Writer](w: W) {
    w.write("hello");
}
~~~

Values can also be implicitly or explicitly cast to traits they implement, creating a "fat pointer" capable of calling the type's trait methods:

~~~kit
function greet(w: Box[Writer]) {
    w.write("hello!");
}

function main() {
    var f = File.write("/tmp/greeting");
    greet(f);
}
~~~

Traits can be "specialized" to provide a default implementation when none is specified.

~~~kit
trait Map[K, V] {
    function get(key: K): V;
    function set(key: K, value: V): V;
    function exists(key: K): Bool;
}

specialize Map[Bool, V] as BoolMap[V];
specialize Map[Integral, V] as IntMap[V];
specialize Map[String, V] as StringMap[V];

function main() {
    var myMap: Map[Int, String] = new Map();
    myMap[5] = "this just works!";
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
