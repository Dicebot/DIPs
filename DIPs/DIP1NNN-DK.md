# @system variables

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0                                                               |
| Author:         | Dennis Korpel dkorpel@gmail.com                                 |
| Implementation: |                                                                 |
| Status:         |                                                                 |

## Abstract
When the `@system` attribute is attached to variables or fields, it means they cannot be written to in `@safe` functions.
If the variable has an unsafe type (type that has at least one pointer or `@system` field), it also cannot be read in `@safe` code.
This allows more safe encapsulation of low-level unsafe code using `@trusted`, since currently `@trusted` code cannot assume anything about the integrity of data.
It also fixes some existing issues with `@safe` code.

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

A distinction can be made between syntactic types and semantic types.
Syntactic types are the things the compiler understands: you can't assign a string to an int, you can't pass an integer array to a function that takes a float parameter etc.
While this helps catching bugs that assembly programmers encounter, it has its limitations with regards to data abstraction and unsafe features.
Often it is useful to make types that have additional constraints on the primitive types they consist of, or types that from the outside are safe and type-sound, but internally use unsafe operations that circumvent the type system.
This can only work if the language in which such types are defined has the needed encapsulation capabilities.

For a detailed explanation about this concept, refer to the article "[What type soundndess theorem do you really want to prove?](https://blog.sigplan.org/2019/10/17/what-type-soundness-theorem-do-you-really-want-to-prove/)".

## Rationale

### Need for encapsulation of @trusted code

D supports writing memory safe code using the `@safe` attribute, which disallows any operations that could corrupt memory.
Among these restricted operations are void-initialization and overlap of pointers:

```D
void main() @safe {
    int* a = void; // error: void initializers for pointers not allowed in safe functions
    *a = 3; // who knows what happens?
    union U {
        uint asNum = 0x8035FDF0;
        uint* asPtr;
    }
    U u;
    *u.asPtr = 3; // error: field U.asPtr cannot access pointers in @safe code that overlap other fields
}
```

However, sometimes using these operations is necessary for performance.
Take for example a [tagged union](https://en.wikipedia.org/wiki/Tagged_union).
To save on memory, it overlaps multiple fields of possibly different types, and then uses an integer to keep track which type of the union is currently valid.
As discussed above however, this is not allowed in `@safe` code when pointers, arrays or classes are among the union members.
One idea is to encapsulate a tagged union using a templated type and use `@trusted` getter methods that ensure the pointer members are only accessed when the tag ensures the underlying data is a valid pointer.
The package [sumtype](http://code.dlang.org/packages/sumtype) does this.

There is a loophole however: By meddling with the `tag` field, memory can be corrupted in `@safe` code:

```D
/+ dub.sdl:
dependency "sumtype" version="~>0.8.11"
+/
import sumtype;
import std;

void main() @safe {
	immutable(int)* immutPtr = new int(10);

 	SumType!(immutable(int)*, int*) sum = immutPtr; // assign immutable pointer

	__traits(getMember, sum, "tag") = 1; // ! ruin integrity of tagged union

	sum.match!(
		(ref immutable(int)* ptr) {}, // this one should be called
		(ref int* ptr) {*ptr = 1000;}, // but we made it call this one
	);

	writeln(*immutPtr); // prints '1000'. the immutable pointer got mutated!
}
```

It actually isn't possible to fix this currently, because:
- the tag having `private` visibility only encapsulates on the module level, while `@trusted` is applied on the function level, so functions in the same module can still violate it
- even outside the module, `__traits(getMember, )` [bypasses `private`](https://github.com/dlang/dmd/pull/9585)

A way of restricting access to the `tag` field from `@safe` code is needed to allow a `@safe` semantic type like this.
The tagged union is just one example, there are many other situations where data should only be modified by code that "knows what it's doing".
This is especially necessary for the recent push of `@safe` non-garbage collected data structures / system programming as evidenced by:
- [Ownsership and borrowing in D](https://dlang.org/blog/2019/07/15/ownership-and-borrowing-in-d/)
- [DIP1021](https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1021.md)
- [my vision of D's future](https://dlang.org/blog/2019/10/15/my-vision-of-ds-future/)

Take reference counting for example: As long as the reference count can be meddled with from `@safe` code, freeing the memory can't be `@trusted` and D won't support `@safe` reference counting.

### Existing holes in `@safe`

While constructing arbitrary pointers inside `@safe` code is not allowed, there is currently little defence around pointers initialized to an unsafe value outside any function:
```D
@safe int* x = cast(int*) 0x7FFE_E800_0000; // compiles

void main() @safe {
    *x = 3; // memory corruption
}
```

This has partially been fixed in DMD PR [#10056](https://github.com/dlang/dmd/pull/10056)

```D
@safe:
int* x = cast(int*) 0x7FFE_E800_0000; // error: cast from long to int* not allowed in safe code
```

It does not work when annotating the declaration as `@safe`, only when the variables is under a `@safe:` section.
It is also easily circumvented:

```D
auto getPtr() @system {return cast(int*) 0x7FFE_E800_0000;}

@safe:
int* x = getPtr();

void main() @safe {
    int y = *x; // = 3; // memory corruption
}
```

Another issue is that, unlike pointers, the `bool` type is regarded `@safe` to overlap or void-initialize.
The problem here is that the optimizer may assume it is always `0` or `1`, but the bool type can be any `ubyte` value in practice.
By constructing a `bool` that is larger than 1 and indexing it in an array with compile-time length 2, memory can be corrupted since the range check is elided.
See issue 19968: [@safe code can create invalid bools resulting in memory corruption](https://issues.dlang.org/show_bug.cgi?id=19968)

The issue is marked fixed after DMD PR [#10055](https://github.com/dlang/dmd/pull/10055) was merged, which defensively inserts a bitwise 'and' operation (`b & 1`) when `bool` values are promoted to integers.
This is not a complete solution however, since there are also other ways in which invalid booleans give undefined behavior: [void initializated bool can be both true and false](https://issues.dlang.org/show_bug.cgi?id=20148).
This happens because negation of a `bool` is simply toggling the least significant bit with the xor operator (`b ^ 1`), which only works if all other bits are 0.
This could, again, be fixed by inserting more `& 1` instructions, but it starts to defeat the purpose of a specialized `bool` primitive type when it can't be guaranteed to be `0` or `1`: you might as well make `bool` a wrapper around `ubyte` then.

And even if you accept this solution for booleans, the solution of ensuring validity on usage can't work on pointers, since there is no simple operation that type checks a pointer at run-time.
A proper solution should make it impossible for invalid values of types to even reach `@safe` code.

## Prior work

The need for encapsulation for memory safety has been mentioned in several discussions:

[#8035: tupleof ignoring private shouldn't be accepted in `@safe` code](https://github.com/dlang/dmd/pull/8035) (March 15, 2018)

[Re: shared - i need it to be useful](https://forum.dlang.org/post/pqleml$2kpg$1@digitalmars.com) (October 22, 2018)

[Re: Manu's `shared` vs the @trusted promise](https://forum.dlang.org/post/pqn0dc$2cq7$1@digitalmars.com) (October 23, 2018)

[Re: Both safe and wrong?](https://forum.dlang.org/post/cxupcgybvqwvkuvcokoz@forum.dlang.org) (February 7, 2019)

[#9585: __traits(getMember) should bypass the protection](https://github.com/dlang/dmd/pull/9585) (April 12, 2019)

[Should modifying private members be @system?](https://forum.dlang.org/thread/lobjvmjxvvmamklzfzhp@forum.dlang.org) (October 4, 2019)

[Borrowing and Ownership](https://forum.dlang.org/post/qp565f$2q48$1@digitalmars.com) (October 27, 2019)

### Other languages
Many other languages either don't allow systems programming at all (e.g. Java, Python) or don't support language enforced memory safety (e.g. C/C++).

A notable exception is Rust, where the equivalent of this DIP has been proposed multiple times before:
[Unsafe fields #381](https://github.com/rust-lang/rfcs/issues/381)

Some excerpts from the discussion there are:

> OTOH, privacy is primarily intended for abstraction (preventing users from depending on incidental details), not for protection (ensuring that invariants always hold). The fact that it can be used for protection is basically an happy accident.
> To clarify the difference, C strings have no abstraction whatever - they are a raw pointer to memory. However, they do have an invariant - they must point to a valid NUL-terminated string. Every place that constructs such a string must ensure it is valid, and every place that consumes it can rely on it.
> OTOH, a safe, say, buffered reader needs abstraction but doesn't need protection - it does not hold any critical invariant, but may want to change its internal representation.

[source](https://github.com/rust-lang/rfcs/issues/381#issuecomment-174955431)

> This doesn't seem very useful to me. Within a module I would expect the authors to know what they're doing, and the unit-tests to save them when they do not.
> For other users, you could simply introduce getters and setters, and functions/methods can already be marked unsafe.

[source](https://github.com/rust-lang/rfcs/pull/80#issuecomment-43489000)

Ultimately the proposal has not been accepted yet.
The idea of using `private` instead of `@system` variables for D will be discussed in the [alternatives](#alternatives) section.
More information about Rust's stand on unsafe functions can be found here:

- [safe unsafe meaning](https://doc.rust-lang.org/nightly/nomicon/safe-unsafe-meaning.html)
- [The scope of unsafe](https://www.ralfj.de/blog/2016/01/09/the-scope-of-unsafe.html)

## Description

### Existing rules for `@system`

Before the proposed changes, we quickly go over the relevant existing rules of what declarations can have the `@system` attribute:
```D
@system int w = 2; // compiles, does nothing
@system enum int x = 3; // compiles, does nothing
enum E {
    @system x, // error: @system is not a valid attribute for enum members
    y,
}
@system alias x = E; // compiles, does nothing
@system template T() {} // compiles, does nothing

void func(@system int x) // error: @system attribute for function parameter is not supported
{
    @system int x; // compiles, does nothing
}
template Temp(@system int x) {} // error: basic type expected, not @
```
In short, anything that can be marked `private` can also be marked `@system`.
Additionally, local variables can be marked `@system` (while they can't be marked `private`).

Any function attribute can be attached to a variable declaration, but they cannot be retrieved:
```D
@system @nogc pure nothrow int x;
pragma(msg, __traits(getFunctionAttributes, x)); // Error: first argument is not a function
pragma(msg, __traits(getAttributes, x)); // tuple()
```

### Unsafe types
In the proposed changes I use the term "unsafe types".
An unsafe type is a type which has underlying bit-patterns that risk causing memory corruption in `@safe` code.
The prime example of an unsafe type is a pointer, but any type that contains a pointer is also unsafe:
- classes
- arrays
- associative arrays
- a struct or union containing a pointer or one of the above types

Note that unsafe types can be used just fine in `@safe` code for the most part, there are only some restrictions to ensure their integrity:
- they can't be void-initialized
- they can't be overlapped in a union
- a `T[]` cannot be cast to a `U[]` when U is an unsafe type.
- certain operators (pointer arithmetic, unsafe casts) are disallowed

In the proposed changes both the list of unsafe types and the the list of restrictions will be extended.

### Proposed changes

**(0) _Writing_ to variables or fields marked `@system` is not allowed in `@safe` code**

This does not apply to local variables however, where `@system` remains ignored.
A function already has full control over the scope of its local variables, unless a `@safe` way of inspecting closures is introduced.

```D
@system int x;

struct S {
    @system int y;
}

S s;

void main() @safe {
    x += 10; // error: can't modify @system variable 'x'
    s.y += 10; // error: can't modify @system field 'y'

    @system int z;
    z += 1; // still allowed
}

// inferred as a @system function
auto foo() {
    x = 0;
}
```

Further operations disallowed in `@safe` code on `@system` variables or fields are:
- taking its address using `&`
- passing it as an argument to a function parameter marked `ref`
- returning it by `ref`

A struct or union with `@system` fields can not be constructed using the default constructor in `@safe` code.
A `@trusted` constructor can be defined to allow construction in `@safe` code.
Note that the `.init` value of a type is always usable in `@safe` code.

When using an `alias` to a `@system` field, that alias has the same restrictions regarding `@system`.

**(1) An aggregate with at least one `@system` field is an unsafe type**

It gets the same restrictions as existing unsafe types:

```D
struct Handle {
    @system int handle;
}

void main() @safe {
    Handle h = void; // error
    union U {
        Handle h;
        int i;
    }
    U u;
    u.i = 3; // error

    ubyte[Handle.sizeof] storage;
    auto array = cast(Handle[]) storage[]; // error
}
```

Without this, implicit writes to `@system` variables are still possible.

**(2) _Reading_ from variables or fields marked `@system` is not allowed in `@safe` code if their type is unsafe**

While writing to a `@system` variable is always unsafe, reading from one is only dangerous when it could yield an unsafe value.

```D
struct Handle {
    @system int handle;
}

// struct with @system field is an unsafe type
@safe   Handle safeHandle = Handle(1);
@system Handle systemHandle = Handle(-1);

// pointers are an unsafe type
@safe   immutable int* safePtr   = null;
@system immutable int* systemPtr = cast(int*) 0x8035FDF0;

// integers are a safe type
@safe   int safeInt   = 20;
@system int systemInt = 20;

void main() @safe {
    Handle h0 = safeHandle;        // allowed, @safe variable
    Handle h1 = systemHandle;      // error, reading @system var of unsafe type
    immutable int* p0 = safePtr;   // allowed, @safe variable
    immutable int* p1 = systemPtr; // error, reading @system var of unsafe type
    int i0 = safeInt;              // allowed
    int i1 = systemInt;            // allowed, not an unsafe type
}
```

**(3) Variables and fields without annotation are `@safe` unless their initial value is not `@safe`**

The rules regarding variables and fields are as follows:
- An initialization expression `x` is `@system` when the function `(() => x)` is inferred as `@system`.
- When marked `@system`, the result is always `@system` regardless of the type.
- When marked `@trusted`, the initialization expression `x` is treated as `(() @trusted => x)`.
- When marked `@safe`, the initialization expression must be `@safe`.
- In the absence of an annotation, the result is `@system` only if the type is unsafe and the initialization expression is `@system`.

```D
int* getPtr() @system {return cast(int*) 0x8035FDF0;}
int  getVal() @system {return -1;}

extern int* x0;                   // @safe by default
int* x1 = x0;                     // @safe, (() => x0) is @safe
int* x2 = cast(int*) 0x8035FDF0;  // @system, (() => cast(int*) 0x8035FDF0) is @system
int* x3 = getPtr();               // @system, (() => getPtr()) is @system
int  x4 = getVal();               // @safe, int is not an unsafe type
@system int x5 = 1;               // @system as requested
@trusted int* x6 = getPtr();      // @safe, the getPtr call gets trusted
@safe int* x7 = getPtr();         // error: cannot initialize @safe variable with @system initializer

struct S {
    // same rules for fields:
    int* x9 = x3; // @system
    int  x8 = x5; // @safe
}
```

An exception to the last rules is made on unsafe types when the compiler knows the resulting value is safe.
```D
int* getNull() pure @system {return null;}
int* n = getNull(); // despite unsafe type with @system initialization expression, inferred as @safe
```

Annotations with a scope (`@system {}`) or colon (`@system:`) affect variables just like they do functions.
```D
@system {
    int y0; // @system
}

@system:
int y1; // @system
```

**(4) `__traits(getFunctionAttributes)` may be called on variables and fields**

Currently it is possible to give function attributes to declarations that aren't functions.
It is not possible however to inspect any of them.

```D
@system @nogc pure nothrow int x;
pragma(msg, __traits(getFunctionAttributes, x)); // error: first argument is not a function
pragma(msg, __traits(getAttributes, x)); // tuple()
```

Since memory safety-related attributes now have an effect on variables and fields, it becomes useful to inspect them.
Therefor the restriction on the getFunctionAttributes trait gets lifted.

The name "function attributes" is a bit unfortunate in this case, but this DIP does not aim to fix that.

**(5) `bool` becomes an unsafe type**

As explained in the [rationale](#rationale) section, `bool` types either cause memory corruption when assumed to be always 0 or 1, or might as well be `ubyte` types when almost every operation needs to truncate the value.
By making `bool` an unsafe type, it can be both fast and `@safe` since all tricks to make invalid booleans are now disabled in `@safe` code.
The cost is that existing code might break.
Note that with the proposed rules a `bool` variable or field is only `@system` when the compiler cannot possibly prove that the initial value is 0 or 1, which should be a rare occurrence.
If there is code breakage, it will likely come from overlap or void initialization of booleans.
```D
immutable ubyte ub = 3;
bool getValidBool()   pure @system {return true;}
bool getInvalidBool() pure @system {return *(cast(bool*) ub);}

bool b0 = getValidBool();   // despite unsafe type with @system initialization expression, inferred as @safe
bool b1 = getInvalidBool(); // inferred as system

void main() @safe {
    bool b = void; // no longer compiles
}
```

### Grammar changes

There are no proposed grammar changes, since placing `@system` annotations is already allowed on the places where it's needed for this DIP.

## Examples

This section contains five examples that demonstrate use cases for the proposed changes.
Each example *wrongly uses `@trusted`* under *current semantics* followed by a `main` function demonstrating why.
After this DIP is implemented, these will all become correct usages of `@trusted` and the `main` functions will fail to compile.

### A tagged union
This example shows how to safely overlap an unsafe type with another type.
```D
struct Var {
    private @system union {
        string asString = "var";
        double asDouble;
    }
    enum Type : byte {
        Str,
        Num,
    }
    private @system Type type;

    this(double x) @trusted {
        this.type = Type.Num;
        this.asDouble = x;
    }

    this(string s) @trusted {
        this.type = Type.Str;
        this.asString = s;
    }

    double getDouble() const @trusted {
        return type == Type.Str ? asDouble : double.nan;
    }

    string getString() const @trusted {
        return type == Type.Str ? asString : null;
    }
}

import std;

void main() @safe {
    Var v = 1.0;
    v.type = Var.Type.Str; // not allowed after this DIP
    writeln(v.getString()); // memory corruption: the union messed up the string
}
```

### Using uninitialized memory
This simple stack data structure shows how to safely use an array with partially uninitialized memory.
In this case the stack data structure is using uninitialized stack memory, but the same idea applies to using malloc'ed memory.
Since destructors are still run on the entire array (also the uninitialized part), the stack has a restriction that its element type must be plain old data.
```D
private struct Stack(T, int size = 32) if (__traits(isPOD, T)) {
    private @system T[size] stack = void;
    private @system int _top = 0;

    T pop() @trusted {
        if (empty) assert(0); 
        return stack[--_top];
    }
    void push(T x) {
        if (full) assert(0);
        // T might have a @system opAssign, so only make retrieval @trusted
        delegate ref T() @trusted {return stack[_top++];}() = x;
    }
    bool empty() const {return _top == 0;}
    bool full() const {return _top == size;}
}

auto getStack(T)() @trusted {
    Stack!T result = void;
    result._top = 0;
    return result;
}

import std;

void main() @safe {
    auto stack = getStack!string();
    stack._top += 1; // not allowed after this DIP
    writeln(stack.pop); // memory corruption: garbage string is printed
}
```

### A custom allocator
Here we make a simple 'bump the pointer' allocator.
```D
@system private ubyte[4096] heap;
@system private size_t heapIndex;

/// allocate an array of T
/// never frees the memory
T[] customAlloc(T)(int length) @trusted {
    // round up heapIndex to multiple of T.alignof
    heapIndex = (heapIndex + T.alignof-1) & ~(T.alignof-1);
    auto result = cast(T[]) heap[heapIndex..heapIndex + T.sizeof*length];
    heapIndex += heapIndex + T.sizeof*length;
    return result;
}

import std;

void main() @safe {
    string[] strArr = customAlloc!string(3);
    heapIndex = 0; // mess up allocator integrity! not allowed after this DIP.
    int[] intArr = customAlloc!int(3);
    intArr[] = -1; // overwrites string array
    writeln(strArr[0]); // memory corruption: constructed pointer
}
```

### A custom slice type
Many arrays are small in length and in certain situations you do not want 4 or 8 bytes to store the length when 2 suffices.
A safe small slice type is created below.
It has room for extra fields but still fits in two registers.
```D
struct SmallSlice(T) {
    @system private T* ptr = null;
    @system private ushort _length = 0;
    ubyte[size_t.sizeof-2] extraSpace; // 2 on 32-bit, 6 on 64-bit

    this(T[] slice) @trusted {
        ptr = &slice[0];
        assert(slice.length <= ushort.max);
        _length = cast(ushort) slice.length;
    }

    T opIndex(size_t i) @trusted {
        return ptr[0.._length][i];
    }
}

import std;

void main() @safe {
    int[4] arr = [10, 20, 30, 40];
    auto slice = SmallSlice!int(arr[]);
    __traits(getMember, slice, "_length") = 100; // change private member
    writeln(slice[9]); // out of bounds memory access
}
```

### A zero-terminated string
String literals in D are zero-terminated, but as soon as you assign it to a `const(char)*` that type information is lost and using it becomes unsafe.
With `@system` fields we can make a `@safe` zero terminated string type.
```D
struct Cstring {
    @system const(char)* ptr = null;

    size_t length() const @trusted {
        import core.stdc.string: strlen;
        return ptr ? strlen(ptr) : 0;
    }
}

Cstring cStringLiteral(string s)() @trusted {
    static immutable string tmp = s;
    return Cstring(tmp.ptr); // static strings are guaranteed zero-terminated
}

import std;

void main() @safe {
    Cstring c = cStringLiteral!"hello";
    c.ptr = new char('D'); // not allowed after this DIP
    writeln(c.length); // memory corruption: strlen likely goes past the single 'D' character
}
```

## Alternatives

### Using `private`

On March 2018 a restriction was added to `.tupleof` such that it could not bypass `private`.
The restriction is only in effect with the flag `-dip1000` to avoid code breakage.
Walter Bright has stated the reason for this in [a comment](https://github.com/dlang/dmd/pull/8035#issuecomment-373627265):

> @safe code should not be accessing private members in other modules, and using tupleof to "work around" that restriction is a giant hole in the @safe system.
> Such code should be marked @trusted or @System.

> A struct may present an @safe interface to users, but internally have private unsafe pointers.
> tupleof allows any code to have access to those private members, and can mess up the invariants relied on by the struct.
> Even int fields are not safe to access, as they may be used as an index to a pointer.

While the need for giving a way of ensuring `struct` invariants in `@safe` code is in line with this DIP, the idea to use `private` for it is argued against.

First of all, disallowing bypassing `private` in `@safe` code is not sufficient for ensuring struct invariants.
As mentioned in the quote, sometimes invariants need to hold on types that are not unsafe such as `int`.
When there are no pointer members, then the private fields can still be indirectly written to using overlap in a union, void-initialization or array casting.

Secondly, it goes against the established 'zen' of `@safe` being the largest subset of the D language that ensures no memory corruption.
The [specification says](https://dlang.org/spec/memory-safe-d.html#limitations):

> Memory safety does not imply that code is portable, uses only sound programming practices, is free of byte order dependencies, or other bugs. It is focussed only on eliminating memory corruption possibilities. 

There have been suggestions to disallow certain operations that are risky but not sufficient for memory corruption in `@safe` code, but it [has been stated](https://forum.dlang.org/post/qdl954$hbb$1@digitalmars.com) that `@safe` does not mean "no bugs":

> Uninitialized non-pointers are @safe, because @safe refers to memory safety, not "no bugs".

Accessing a `private` field is just as memory safe as accessing a `public` field, so adding this restriction, while it potentially enables more `@trusted` code, is ultimately arbitrary. 
Defining `@system` variables is consistent with existing practice of allowing functions to be marked `@system`.

It also goes against the established [definition of trusted functions](https://dlang.org/spec/function.html#trusted-functions):

> Trusted functions are guaranteed to not exhibit any undefined behavior if called by a safe function. 
> Furthermore, calls to trusted functions cannot lead to undefined behavior in @safe code that is executed afterwards.
> It is the responsibility of the programmer to ensure that these guarantees are upheld.

`private` only acts on the module level, so a `@trusted` member function cannot assume anything about the state of the `this` parameter.

Finally, it would mean that circumventing visibility constraints using `__traits(getMember, ...)` must become `@system` or deprecated entirely similar to `.tupleof`.
This would break all (`@safe`) code that uses this feature, and re-introduces the problems of [issue 15371](https://issues.dlang.org/show_bug.cgi?id=15371).
All things considered, making `private` work with `@trusted` appears to be a bigger hassle than introducing checks for `@system` variables and fields.

### Not making bool an unsafe type
It has been proposed that `void` initialization should be disallowed in `@safe` code entirely, which would partially solve the problem of invalid `bool` values in `@safe` code.

See the comment on [Issue 20148 - void initializated bool can be both true and false](https://issues.dlang.org/show_bug.cgi?id=20148):
> So instead of closing the obvious hole of @safe functions using void initialization we're just poking at symptoms here and there?

This would still leave each `bool` in a union or a `bool[]` that was typecast from a `ubyte[]` vulnerable.
Truncating booleans in those situations only is a possible compromise, though treating the `bool` type as unsafe is still seen as an improvement.
The proposal to disallow `void` initialization in `@safe` code still has its merits besides preventing invalid booleans, and treating booleans as unsafe types has its merits even without the ability to void-initialize them.

## Breaking Changes and Deprecations

It is already allowed to attach the `@system` attribute to variables, but this didn't add any compiler checks.
The additional checks for `@system` variables can cause existing `@safe` code to break (note that `@system` code is completely unaffected by everything in this DIP).
However, since `@system` does not do anything, it is suspected that users didn't add this attribute to any variables at all, let alone variables that are meant to be used in `@safe` code.
The biggest risk here is that variables accidentily fall inside a `@system {}` block or under a `@system:` section.

```D
@system:

int x; // suddenly not writable in @safe code anymore
void unsafeFuncA() {};
void unsafeFuncB() {};

void main() @safe {
    x++; // not allowed anymore
}
```

Misconstructed pointers and `bool` variables can also be inferred `@system` under the new rules.
```D
struct S {
    int* a = cast(int*) 0x8035FDF0;
}

void main() @safe {
    S s;
    *s.a = 0; // this gives an error now? I thought this was safe!
    int[1] intArr = [-1];
    auto boolArr = cast(bool[]) intArr; // not even this?
}
```

Whenever this happens, there is a risk of memory corruption, so a compile error would be in its place.
In any case, a two-year deprecation period is proposed where instead of raising an error, a deprecation message is given whenever the new memory safety rules are broken.
A preview flag `-preview=systemVariables` can also be added that immediately raises errors for violations while leaving other deprecation messages as warnings.

## Reference

- [What type soundndess theorem do you really want to prove?](https://blog.sigplan.org/2019/10/17/what-type-soundness-theorem-do-you-really-want-to-prove/)
- [The scope of unsafe](https://www.ralfj.de/blog/2016/01/09/the-scope-of-unsafe.html)
- [safe unsafe meaning](https://doc.rust-lang.org/nightly/nomicon/safe-unsafe-meaning.html)

## Copyright & License

Copyright (c) 2019 by the D Language Foundation

Licensed under Creative Commons Zero 1.0

## Reviews
