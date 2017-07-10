# extern(delegate)

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | 1xxx                                                            |
| RC#             | 0                                                               |
| Author:         | Jonathan Marler - johnnymarler@gmail.com                        |
| Implementation: | None                                                            |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

It is proposed that

* `extern(delegate)` be added as a linkage type in order to create functions that are ABI-compatible with delegates
* taking the address of a UFCS call to an `extern(delegate)` function creates a delegate
* member function be implicitly convertible to `extern(delegate)` functions

### Links

General forum discussion [Address of UFCS call implicity converts to Delegate](http://forum.dlang.org/post/qtaiotodqqxqoqmozgrq@forum.dlang.org)

## Description

It is proposed that `extern(delegate)` be added as a linkage type, i.e.
```D
extern(delegate) void bar(Foo foo);
```

This linkage type modifies the ABI of the function by causing the first parameter to be passed in the same way a context pointer would be passed to a delegate function.  If the first parameter of an `extern(delegate)` function is a class or a reference to a struct, then it will have the same ABI as a member function of that type.  Every function in the following example has the same ABI:
```D
class SomeClass
{
    void memberFunc(int x, float y)
    {
    }
}
extern(delegate) void notMemberFunc(SomeClass s, int x, float y)
{
}
struct SomeStruct
{
    void memberFunc(int x, float y)
    {
    }
}
extern(delegate) void notMemberFunc(ref SomeStruct s, int x, float y)
{
}
extern(delegate) void notMemberFunc(SomeStruct* s, int x, float y)
{
}
```

A delegate can be retreived to an `extern(delegate)` function using UFCS, i.e.
```D
extern(delegate) void bar(Foo foo)
{
    // ...
}

Foo foo;
void delegate() dg = &foo.bar;   // uses UFCS to get a "void delegate()" with the context pointer set to foo
```

By using UFCS to retrieve a delegate, the same syntax is used for `extern(delegate)` functions and member functions, namely, `&<object>.<function>`.  This allows templates and mixins to work with both kinds.  It also maintains type safety between the context pointer and the first parameter of the function by "piggy-backing" off the type checking done by the UFCS call.

For completeness, a member function should be implicitly convertible to an `extern(delegate)` function i.e.
```D
struct FooStruct
{
    void bar() // ...
}
class FooClass
{
    void bar() // ...
}

struct FooStruct fooStruct;
struct FooClass fooClass;

extern(delegate) function(ref FooStruct foo) fp1 = &fooStruct.bar;
extern(delegate) function(FooClass foo) fp2 = &fooClass.bar;
```

## Rationale

The first reason for this change is that it fills a hole in the semantics, namely, taking the addres of a UFCS call.  This proposal assigns a meaning to it that is consistent with existing semantics.

The most prominent use case for `extern(delegate)` is defining a delegate function (a "delegate function" being a function that can be called as a delegate) that uses an external type for the context pointer.

> Note: this use case is different from the std.functional.toDelegate method which can create a delegate from a function but does not allow the function to make use of the context pointer.

Say we want to create a delegate function that takes a `std.stdio.File` reference as the context pointer.  Let's call it `writelnWithTime` and have it write a line to the file prefixed with the current time. Normally new delegate functions are added for a type by adding member functions, but we can't do this with `std.stdio.File` because it is defined in another library.  One solution is to create a wrapper type and define our new delegate function inside the wrapper, i.e.

```D
struct FileWrapper
{
    File* file;
    void writelnWithTime(string msg)
    {
        file.writeln(Clock.currTime, " ", msg);
    }
}

File file;
auto wrapper = FileWrapper(&file);
auto ourDelegate = &wrapper.writelnWithTime;
```

This works but comes at a cost. It adds complexity in ownership semantics, potential for scoping errors, unnecessary runtime overhead, an extra level of indirection and requires extra boilerplate code at the call site to retreive the delegate.  The worst problem with this solution is it requires a proliferation of wrapper types.  If we define this wrapper in our own library that another application uses, and that application needs to add it's own delegate functions, then it must create a "wrapper-wrapper" type introducing another level of indirection.  All this adds no value to the program but does introduce overhead and complexity.

Another solution is to throw type safety out the window, i.e.

```D
struct DummyType
{
    void writelnWithTime(string msg)
    {
        File* file = cast(File*)&this;
        file.writeln(Clock.currTime, " ", msg);
    }
}

File file;
DummyType dummy;
auto ourDelegate = &dummy.writelnWithTime;
ourDelegate.ptr = cast(void*)&file;
```

This also has a cost and I've listed the problems at both the definition site and the call site.

#### At the function definition site:

* requires the function to be defined inside a dummy wrapper type
* requires a cast of the dummy type context pointer to the actual type you want, this throws away any type safety of the context pointer type and incurs some runtime overhead

#### At the function call site:

* requires an instance of the dummy type to be declared. note that this declaration has no purpose other than to get a delegate to the function
* once the delegate is retreived, the ptr is set as an independent statement which creates a new place for a runtime error
* to set the delegate context pointer, the type must be cast to a void*, like the definition site this throws away any type safety for the context pointer type and incurs some runtime overhead

With `extern(delegate)` the solution is simple, clear, and type-safe. No dummy wrapper type, no casting at the definition site or the call site, no runtime overhead, and delegate retrieval happens in one type-safe statement.

```D
extern(delegate) void writelnWithTime(ref File file, string msg)
{
    file.writeln(Clock.currTime, " ", msg);
}

File file;
auto ourDelegate = &file.writelnWithTime;
```

### Breaking changes / deprecation process

No breakage. It requires new syntax that if used previously would have caused a compilation error.

### Examples
Example to demonstrate the usage:
```D
struct Foo
{
    void bar(int x)
    {
    }
}
extern(delegate) void baz(ref Foo foo, int x)
{
}

// Note that even though `bar` is defined as a member function inside
// the struct Foo, the `baz` function has an identical ABI with `bar`.

void delegate(int) dg;
Foo foo;

dg = &foo.bar;  // a normal member function delegate
dg(42);         // calls foo.bar(42)

dg = &foo.baz;  // using UFCS to retreive a delegate to the extern(delegate) baz function
dg(42);         // calls baz(foo, 42);

dg = &baz;      // Error: cannot implicitly convert expression (baz) of type extern (delegate) void function(ref Foo, int) to void delegate(int)
```

A more realistic example:
```D
import std.stdio, std.datetime;

extern(delegate) void writelnWithTime(T)(ref File foo, T[] buffer)
{
    foo.writeln(Clock.currTime, " ", buffer);
}

void main()
{
    dumpInfo(&stdout.writeln!(const(char)[]));

    // Using UFCS to get a delegate to the extern(delegate) writelnWithTime function
    dumpInfo(&stdout.writelnWithTime!(const(char)[]))
}

void dumpInfo(void delegate(const(char)[]) writer)
{
    writer("Some Info:");
    writer("  2 = 2");
    writer("  3 != 4");
}
```

It may be worth noting that `extern(delegate)` functions can be imitated with existing semantics at the cost of type-safety and odd looking code with unclear intentions.  The previous example could be "emulated" today like this:
```D
import std.stdio, std.datetime;

// Since there's currently no way to define a function that is ABI-compatible
// with a delegate, we must define our function as a method on a "dummy" type.
// The odd part is that we will be casting the "this" argument to the real
// type we are going to operate on, which in this case is a File*.
struct DummyType
{
    void writelnWithTime(T)(T[] buffer)
    {
        File* file = cast(File*)&this;
        file.writeln(Clock.currTime, " ", buffer);
    }
}

void main()
{
    dumpInfo(&stdout.writeln!(const(char)[]));

    // The DummyType "dummy" variable has no fields and takes no memory.
    // It's only purpose is to create a delegate to the method we want to call.
    DummyType dummy;
    auto writer = &dummy.writelnWithTime!(const(char));

    // Here's where we override the dummy ptr with the real object
    // we want to call the delegate with.
    writer.ptr = &stdout;

    dumpInfo(writer);
}

void dumpInfo(void delegate(const(char)[]) writer)
{
    writer("Some Info:");
    writer("  2 = 2");
    writer("  3 != 4");
}
```

### Compile-time errors

Not all types can be passed in like a context pointer, i.e. they may be larger than a `void*`. If the first parameter of an `extern(delegate)` function is not compatible with a context pointer, a compile-time error should be asserted with a message like this:
```
function '<function>' cannot be declared extern(delegate) because the first parameter <reason>. [<suggested-fix>]
```
If the first parameter is a struct passed-by-value that is larger than `void*`, a good message would be:
```
function '<function>' cannot be declared extern(delegate) because the first parameter is too large. Consider adding the 'ref' attribute or making it a pointer.
```

Delegates only have one context pointer which means functions that already use the context pointer like member functions or nested functions cannot be declared `extern(delegate)`.  This violation should yield a compile-time error:
```D
struct Foo
{
    // Error: foo is a member function which cannot be declared extern(delegate) unless it is static
    extern(delegate) void foo(Bar bar)
    {
    }
    // OK because it is static
    extern(delegate) static void foo2(Bar bar)
    {
        // Error: foo3 is a nested function which cannot be declared extern(delegate) unless it is static
        extern(delegate) void foo3(Bar bar)
        {
        }
        // OK because it is static
        extern(delegate) static void foo4(Bar bar)
        {
        }
    }
}
```

Taking the address of an `extern(delegate)` function should return a function pointer just like a normal function.  The difference is that the function pointer will have the `extern(delegate)` linkage type meaning the type system will prevent it from being assigned and called like a regular function pointer.
```D
extern(delegate) void function(Foo foo) x;
void function(Foo foo) y = x; // Error: cannot implicitly convert expression (x) of type extern (delegate) void function(Foo) to void function(Foo)
```

This type of error already occurs with other linkage types so it should just work.

# Extras

It might be worth looking into the how this can be used to improve the delegate type.  It has two fields `ptr` and a `funcptr` where `funcptr` is a normal function pointer.  It might make more sense to have the function pointer use the `extern(delegate)` linkage type.  Should this deem useful, it's unlikely that changing `funcptr` would be worth breaking existing code, but a new property could be added that uses the `extern(delegate)` linkage type, i.e.
```D
struct __current_delegate_type__
{
    void* ptr;
    void function(<arg_types>) funcptr;
}
struct __new_delegate_type__
{
    void* ptr;
    union
    {
        void function(<arg_types>) funcptr;
        extern(delegate) void function(void*, <arg_types>) dgptr;
    }
}
```

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

Will contain comments / requests from language authors once review is complete,
filled out by the DIP manager - can be both inline and linking to external
document.
