# Give unit type semantics to void

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0                                                               |
| Author:         | Dennis Korpel dkorpel@gmail.com                                 |
| Implementation: |                                                                 |
| Status:         | Draft                                                           |

## Abstract
Declaring a variable of type `void` and assigning an expression of type `void` to it should be allowed.
Currently doing this results in a compile error, but this limitation came from D's C legacy.
Lifting this restriction makes `void` a useful degenerate type in meta-programming.

## Contents
* [Background](#background)
* [Rationale](#rationale)
* [Prior work](#prior-work)
* [Description](#description)
* [Alternatives](#alternatives)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Background
While some early languages made an explicit distinction between functions that do return a value and functions that do not, C unified the two. [[1](#reference)]

> C functions are like the subroutines and functions of Fortran or the procedures and functions of Pascal

In early C, returning no value would be done by specifying the return type as `int` and simply omitting a return statement.
Later, a special keyword was made for it: `void`.
While this `void` type is specified as simply a way to tell the compiler the function doesn't return a value, mathematically it bears resemblance to a [unit type](https://en.wikipedia.org/wiki/Unit_type).

A unit type is a type with only one value, therefore carrying no information and requiring 0 bits of storage.
Any expression of type `void` with no side-effects can freely discard its inputs, since the result is always the same.
In C, which is sometimes considered a portable assembler, using `void` as a type for variables makes little sense.
After all, `void` variables and assignments produce no assembly code, so why would anyone want to write that?

The D language inherits most semantics of its `void` type from C/C++, since it originated as a successor to them.

`void` is also used as a top-type pointer, since every pointer implicitly converts to `void*`.
This does not follow from a unit type, instead it can be seen as stepping outside and back inside the type-system, where everything inbetween is the programmer's responsibility.
Unlike C, D requires to explicitly cast from void* to any other pointer.

While a unit type requires no storage, `void.sizeof == 1` in D.
Consequently, `void[]` actually acts like a `ubyte[]`.
Similar to `void*`, any `T[]` does implicitly convert to a `void[]`, and a `void[]` is a chunk of data of which the type is a responsibility of the programmer, not the type system.

Finally, D has the notion of void initialization (`int a = void`), which has nothing to do with the unit type `void`.
The syntax was likely chosen because of the implementation detail that a return type of `void` in code generation usually results in leaving garbage in a return register.
Similarly, void initialization results in the initial value of a variable being garbage stack memory.

## Rationale
While disallowing variables of type `void` might make sense in C, in D it causes significant friction when doing meta-programming.
In generic functions, producing no code when a template argument involves the `void` type is usually desired.
Because of this restriction, using `void` is not a useful degenerate case but an annoying edge case that needs to be dealt with explicitly.

One situations where this pops up is in [std.functional : memoize](https://github.com/dlang/phobos/blob/f105cc49da008298f2fdad1305c158f53e724eb9/std/functional.d#L1123), simplified code:
```D
import std.traits: ReturnType, Parameters;
import std.typecons: Tuple;

ReturnType!f memoize(alias fun)(Parameters!fun args) {
    alias Args = Parameters!fun;
    static ReturnType!fun[Tuple!Args] memo;
    auto t = Tuple!Args(args);
    if (auto p = t in memo)
        return *p;
    return memo[t] = fun(args);
}
```
Because you can't make an associative array with value type `void`, this won't work on functions returning `void`.
This does work on functions that have no parameters, because an empty `Tuple` is a valid unit type that acts as a degenerate case.

Another example that works on basically everything but `void` is this transformation:
```D
void f(alias g)() {
    return g(); // possible
}

void f(alias g)() {
    auto result = g(); // breaks when g returns void
    return result;
}
```

An example "in the wild" is apache thrift where in module `codegen.client` [line 199](https://github.com/apache/thrift/blob/77d96c18c3729bf3faeadff67e57e7e429f1d3cd/lib/d/src/thrift/codegen/client.d#L199) and [line 421](https://github.com/apache/thrift/blob/77d96c18c3729bf3faeadff67e57e7e429f1d3cd/lib/d/src/thrift/codegen/client.d#L421) it can be seen how two `static if` statements are needed to _not_ include a `void` field and to _not_ assign it a `void` return value from a generic function.
Other examples are [stdx.allocator](https://github.com/dlang-community/stdx-allocator/blob/4903a249e83b9797cc1d5167a13701a6dfd111a7/source/stdx/allocator/common.d#L23) and [arsd.web](https://github.com/adamdruppe/arsd/blob/472236b9fa9159a36f3eacafcc50ad4759339db1/web.d#L2725).

## Prior work

I am not aware of any existing plans to change `void` in D, though the discussion of `void` being a weird type came up [during the review of DIP 1017](https://github.com/dlang/DIPs/pull/117#issuecomment-392331199), which tried to add a bottom type to D.
This DIP goes hand in hand with the [reboot](https://github.com/dkorpel/DIPs/blob/bottom-type/DIPs/DIP1NNN-DK.md) of that DIP, which is another effort to make D types more principled.

There are other languages where not returning anything is formalized in a unit type, such as Haskell, Scala, Rust and Zig.
The latter two, also being systems programming languages, will be discussed.

### Rust
In Rust both the default unit type and unit-value are called `()`.
It works similar to `void` in D after this DIP is implemented:
It is implicitly returned and can be the type of fields or variables.
Its size is 0 though, unlike `void` (which is stuck with being of size 1 in D due to legacy).
A pointer to `()` can be created, though it isn't used for opaque pointers like `void*`.

```rust
use std::mem;

struct Pair {
    x: i32,
    unit: (),
}

fn foo() -> () {
    (); // last expression is returned, return keyword is optional
}

fn main() { // -> () ommitted
    let p = Pair {x: 1, unit: foo()};
    let q: () = p.unit;
    assert_eq!(0, mem::size_of::<()>()); // assert(().sizeof == 0);
    assert_eq!(4, mem::size_of::<Pair>()); // assert(Pair.sizeof == 4);
    //return (); omitted
}
```

### Zig
[Zig's `void`](https://ziglang.org/documentation/master/#void) type works just like Rust's `()`: it has size 0 and can be used for fields, variables and expressions.
For interfacing with C's void type, a separate type exists:
```Zig
pub const c_void = @OpaqueType();
```
(Note that Zig has first-class types, in D this would be an `alias`)

### C++
Apart from the restrictive `void` type, there is a standard unit type `std::monostate` that can be used in templates, most notably here `std::variant`. A nullable `double` can be declared as `std::variant<std::monostate, int, double>` for example (though for one non-unit type you would probably use `std::optional` instead).

In D, `std.variant` allows directly assigning a value of any type "with very few restrictions (such as shared types and noncopyable types)".
Another restriction is, of course, a `void` type since the compiler complains "expression ... is void and has no value".

## Description
Four changes to `void` are proposed.
N.B. the untyped [SIMD types](https://dlang.org/spec/simd.html#x86_64_vec) `void16` and `void32` are unchanged.

**(0) Declaring variables of type void and assigning to them is allowed**

As discussed in the [rationale](#rationale), this removes special casing in generic functions.

Example of usage:
```D
void foo();

struct S {
    int x;
    void y;
}

void main() {
    void x = void; // note: the right void is void initialization, not the void type!
    S s;
    s.y = foo();
    return;
}
```

**(1) void.init becomes a valid expression**

Currently using `void.init` causes an error:
```D
void foo() {
    return void.init; // Error: void does not have a default initializer
}
```
This error shall be gone, and `void.init` becomes the standard way to access the unit value.

**(2) void.sizeof shall remain 1**

As discussed before, a unit type requires 0 bits storage.
Currently it holds that `void.sizeof == 1` however.
Note that this is not a contradiction: a type does not require to use only the minimum amount of bits to represent all possible values.
A `bool` has two values requiring 1 bit, but uses 8 bits so pointers can be made to it.
Only the values 0 and 1 are valid, all other 254 bit-patterns can exist but are invalid for a `bool`;
Similarly `void` can be seen as a unit type with 256 bit-patterns representing the same value.
It can also be seen as a type where everything except 0 is invalid, but that conflicts with the existing usage of `void[]`.

When adding a member of type `void` to a struct, it needlessly wastes 1 byte or more (depending on alignment constraints).
However, changing the size to 0 either breaks all uses of `void[]`, or it introduces new inconsistent behavior.
```D
void[] data = import("data.txt");
void[200] buffer = data[0..200];
string[] dataString = cast(string) buffer[];
```

And making a special case for `void[]` introduces new inconsistencies.
The following rule will not longer hold for example:
```D
T[n].sizeof == T.sizeof * n
```

When declaring a local variable with type `void`, the optimizer can likely automatically discard the unused stack space.
To shave off the wasted bytes when adding a `void` field to a struct, the user can still make a special case for `void`.

**(3) Associative arrays can have key/value type void**

A map with value type `void` can be seen as a set.
Currently this is not allowed:
```D
void[string] x; // Error: cannot have associative array of void
```
Similary, an associative array with key type `void` decays into an optional value.
It is proposed that for generality, `void` is allowed as the key/value type of an associative array.
(Note that it is not proposed that `void[T]` and `T[void]` become the preferred set- and optional types.)

## Alternatives
Since `void` is currently unprincipled, a new type can be made that is a proper unit type.
Functions with no return will be inferred as `Unit` instead of `void`, and `void` will implicitly convert to `Unit`.
```D
struct Unit {} // implicitly imported in every module

auto foo() {
    // return type inferred as Unit
}
```
However, since existing codebases are full of `is(T == void)` checks and functions explicitly annotated with `void` return type, this will be a really large transition.
The improvement is not large enough to justify the cost of that transition.

Another alternative is to throw in the towel and just leave `void` as the unprincipled type we've come accustomed to.
While fixing void completely is neigh impossible given its history and existing usage, the DIP argues the proposed changes are a good compromise between a full fix and maintaining backwards compatibility, though any future efforts to improve `void` are not discouraged.

## Breaking Changes and Deprecations

The changes are only lifting restrictions, so this should not break existing code unless that code explicitly assumes `void` cannot be instantiated (e.g. `assert(!__traits(compiles, () {void x; return x;}))`).

## Reference

[1] Brian Kernighan and Dennis Ritchie, "The C Programming language"

## Copyright & License

Copyright (c) 2019 by the D Language Foundation

Licensed under Creative Commons Zero 1.0

## Reviews