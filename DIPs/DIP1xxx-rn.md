# The Copy Constructor

| Field           | Value                                                                           |
|-----------------|---------------------------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                                          |
| Review Count:   | 0 (edited by DIP Manager)                                                       |
| Authors:        | Razvan Nitu - razvan.nitu1305@gmail.com, Andrei Alexandrescu - andrei@erdani.com|
| Implementation: | (links to implementation PR if any)                                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")                  |

## Abstract

This document proposes copy constructor semantics as an alternative
to the design flaws and inherent limitations of the postblit.

### Reference

* [1] https://github.com/dlang/dmd/pull/8032

* [2] https://forum.dlang.org/post/p9p64v$ue6$1@digitalmars.com

* [3] http://en.cppreference.com/w/cpp/language/copy_constructor

* [4] https://dlang.org/spec/struct.html#struct-postblit

* [5] https://news.ycombinator.com/item?id=15008636

* [6] https://dlang.org/spec/struct.html#struct-constructor

* [7] https://dlang.org/spec/struct.html#field-init

## Contents
* [Rationale and Motivation](#rationale-and-motivation)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale and Motivation

This section highlights the existing problems with the postblit and demonstrates
why the implementation of a copy constructor is more desirable than an attempt
to resolve all the postblit issues.

### Overview of this(this)

The postblit function is a non-overloadable, non-qualifiable
function. However, the compiler does not reject the
following code:

```D
struct A { this(this) const {} }
struct B { this(this) immutable {} }
struct C { this(this) shared {} }
```

Since the semantics of the postblit in the presence of qualifiers is
not defined and most likely not intended, a series of problems has arisen:

* `const` postblits are not able to modify any fields in the destination
* `immutable` postblits never get called (resulting in compilation errors)
* `shared` postblits cannot guarantee atomicity while blitting the fields

#### `const`/`immutable` postblits

The solution for `const` and `immutable` postblits is to type check them as normal
constructors, where the first assignment of a member is treated as an initialization
and subsequent assignments are treated as modifications. This is problematic because
after the blitting phase, the destination object is no longer in its initial state
and subsequent assignments to its fields will be regarded as modifications, making
it impossible to construct `immutable`/`const` objects in the postblit. In addition,
it is possible for multiple postblits to modify the same field. Consider:

```D
struct A
{
    immutable int a;
    this(this)
    {
        this.a += 2;  // modifying immutable or ok?
    }
}

struct B
{
    A a;
    this(this)
    {
        this.a.a += 2;  // modifying immutable error or ok?
    }
}

void main()
{
    B b = B(A(7));
    B c = b;
}
```

When `B c = b;` is encountered, the following actions are taken:

1. `b`'s fields are blitted to `c`
2. A's postblit is called
3. B's postblit is called

After `step 1`, the object `c` has the exact contents as `b`, but it is not
initialized (the postblits still need to fix it) nor uninitialized (the field
`B.a` does not have its initial value). From a type checking perspective
this is a problem, because the assignment inside A's postblit is breaking immutability.
This makes it impossible to postblit objects that have `immutable`/`const` fields.
To alleviate this problem we can consider that after the blitting phase the object is
in a raw state, therefore uninitialized; this way, the first assignment of `B.a.a`
is considered an initialization. However, after this step the field
`B.a.a` is considered initialized, therefore how is the assignment inside B's postblit
supposed to be type checked ? Is it breaking immutability or should
it be legal? Indeed, it is breaking immutability because it is changing an immutable
value. However as this is part of initialization (remember that `c` is initialized only
after all the postblits are ran) it should be legal, thus weakening the immutability
concept and creating a different strategy from that implemented by normal constructors.

#### Shared postblits

Shared postblits cannot guarantee atomicity while blitting the fields because
that part is done automatically and it does not involve any
synchronization techniques. The following example demonstrates the problem:

```D
shared struct A
{
    long a, b, c;
    this(this) { ... }
}

A a = A(1, 2, 3);

void fun()
{
    A b = a;
    /* do other work */
}
```

Let's consider the above code is run in a multithreaded environment
When `A b = a;` is encountered, the following actions are taken:

* `a`'s fields are copied to `b`
* the user code defined in `this(this)` is called

In the blitting phase, no synchronization mechanism is employed, which means
that while the copying is in progress, another thread may modify `a`'s data, resulting
in the corruption of `b`. There four possible approaches to fixing this issue:

1. Make shared objects larger than two words uncopyable. This solution cannot be
taken into account because it imposes a major arbitrary limitation: almost all
`struct`s will become uncopyable.

2. Allow incorrect copying and expect that the user will
do the necessary synchronization. Example:

```D
shared struct A
{
    Mutex m;
    long a, b, c;
    this(this) { ... }
}

A a = A(1, 2, 3);

void fun()
{
    A b;
    a.m.acquire();
    b = a;
    a.m.release();
    /* do other work */
}
```

Although this solution solves our synchronization problem, it does so in a manner
that requires unencapsulated attention at each copy site. Another problem is
represented by the fact that the mutex release occurs after the postblit is
run, which imposes some overhead. The release may occur immediately following
the blitting phase (the first line of the postblit) because the copy is
thread-local, but this results in non-scoped locking: the mutex is released
in a different scope than that in which it was acquired. Also, the mutex is
automatically (wrongfully) copied.

3. Introduce a preblit function that will be called before blitting the fields.
The purpose of the preblit is to offer the possibility of preparing data
before the blitting phase; acquiring the mutex on the `struct` that will be
copied is one of the operations that the preblit will be responsible for. Later
on, the mutex will be released in the postblit. This approach has the benefit of
minimizing the mutex-protected area in a manner that offers encapsulation, but
suffers the disadvantage of adding even more complexity by introducing a new
concept which requires type checking disparate sections of code (need to type check
across preblit, `mempcy` and postblit).

4. Use an implicit global mutex to synchronize the blitting of fields. This approach
has the advantage that the compiler will do all the work and synchronize all blitting
phases (even if the threads don't actually touch each other's data) at a cost to
performance. Python implements a global interpreter lock which was proven to cause
unscalable high contention; there are ongoing discussions of removing it from the Python
implementation [5].

### Introducing the copy constructor

As stated above, the fundamental problem with the postblit is the automatic blitting
of fields, which makes type checking impossible and cannot be synchronized without
additional costs.

As an alternative, this DIP proposes the implementation of a copy constructor. The benefits
of this approach are the following:

* it is a known concept. C++ has it [3];
* it can be type checked as a normal constructor (since no blitting is done, the data are
initialized as if they were in a normal constructor); this means that `const`/`immutable`/`shared`
copy constructors will be type checked exactly as their analogous constructors
* offers encapsulation

The downside of this solution is that the user must copy all fields by hand, and every
time a field is added to a `struct`, the copy constructor must be modified. However, this
can be easily bypassed by D's introspection mechanisms. For example, this simple code
may be used as a language idiom or library function:

```D
foreach (i, ref field; src.tupleof)
    this.tupleof[i] = field;
```

As shown above, the single benefit of the postblit can be easily substituted with a
few lines of code. On the other hand, to fix the limitations of the postblit it is
required that more complexity be added for little to no benefit. In these circumstances,
for the sake of uniformity and consistency, replacing the postblit constructor with a
copy constructor is a reasonable approach.

## Description

This section discusses all the technical aspects regarding the semantics
of the copy constructor.

### Syntax

A declaration is a copy constructor declaration if it is a constructor declaration that takes only one
parameter by reference that is of the same type as `typeof(this)`. Declaring a copy constructor in this
manner has the advantage that no parser modifications are required, thus leaving the language grammar
unchanged.

```d
import std.stdio;

struct A
{
    this(ref A rhs) { writeln("x"); }     // copy constructor
}

void main()
{
    A a;
    A b = a;        // calls copy constructor implicitly
    A c = A(b);     // calls constructor explicitly
}
```

The copy constructor may also be called explicitly since it is also a constructor.

The argument to the copy constructor is passed by reference in order to avoid infinite recursion (passing by
value would require a copy of the `struct`, which would be made by calling the copy constructor, thus leading
to an infinite chain of calls).

Type qualifiers may be applied to the parameter of the copy constructor, but also to the function itself,
in order to facilitate the ability to describe copies between objects of different mutability levels. The type
qualifiers are optional.

### Semantics

This section discusses all the aspects of the semantic analysis and interaction between the copy
constructor and other components.

#### Copy constructor and postblit cohabitation

In order to assure a smooth transition from postblit to copy constructor, the following
strategy is employed: if a `struct` defines a postblit (user-defined or generated) all
copy constructor definitions will be ignored for that particular `struct` and the postblit
will be preferred. Existing code bases that do not use the postblit may start using the
copy constructor without any problems, while codebases that rely on the postblit may
start writing new code using the copy constructor and remove the deprecated postblit
from their code.

The transition from postblit to copy constructor may be simply achieved by replacing the postblit
declaration `this(this)` with `this(ref inout(typeof(this)) rhs) inout`.

#### Copy constructor usage

The copy constructor is implicitly used by the compiler whenever a `struct` variable is initialized
as a copy of another variable of the same unqualified type:

1. When a variable is explicitly initialized:

```d
struct A
{
    this(ref A rhs) {}
}

void main()
{
    A a;
    A b = a; // copy constructor gets called
    b = a;   // assignment, not initialization
}
```

2. When a parameter is passed by value to a function:

```d
struct A
{
    this(ref A another) {}
}

void fun(A a) {}

void main()
{
    A a;
    fun(a);    // copy constructor gets called
}
```

3. When a parameter is returned by value from a function and NRVO cannot be performed:

```d
struct A
{
    this(ref A another)
    {
        writeln("cpCtor");
    }
}

A fun()
{
    A a;
    return a;
}

A a;
A gun()
{
    return a;
}

void main()
{
    A a;
    a = fun();      // NRVO - no copy constructor call
    A b;
    b = gun();      // NRVO cannot be performed, copy constructor is called
}
```

The parameter of the copy constructor is passed by reference, so initializations will be
to be lowered to copy constructor calls only if the source is an lvalue. Although this can be
worked around by declaring temporary lvalues which can be forwarded to the copy constructor,
binding rvalues to lvalues is beyond the scope of this DIP.

Note that when a function returns a `struct` instace that defines a copy constructor and NRVO cannot be
performed, the copy constructor is called at the callee site before returning. If NRVO
may be performed, then the copy is elided:

```d
struct A
{
    this(ref A rhs) {}
}

A a;
A fun()
{
    return a;      // lowered to return tmp.copyCtor(a)
    // return A();  // rvalue, no copyCtor called
}

void main()
{
    A b = fun();    // the return value of fun() is moved to the location of b
}
```

#### Type checking

The copy constructor type-check is identical to that of the constructor [[6](https://dlang.org/spec/struct.html#struct-constructor)]
[[7](https://dlang.org/spec/struct.html#field-init)].

Copy constructor overloads can be explicitly disabled:

```d
struct A
{
    @disable this(ref A rhs);
    this(ref immutable A rhs) {}
}

void main()
{
    A b;
    A a = b;     // error: disabled copy construction

    immutable A ia;
    A c = ia;  // ok

}
```

In order to disable copy construction, all copy constructor overloads must be disabled.
In the above example, only copies from mutable to mutable are disabled; the overload for
immutable to mutable copies is still callable.

#### Overloading

The copy constructor can be overloaded with different qualifiers applied to the parameter
(copying from a qualified source) or to the copy constructor itself (copying to a qualified
destination):

```d
struct A
{
    this(ref A another) {}                        // 1 - mutable source, mutable destination
    this(ref immutable A another) {}              // 2 - immutable source, mutable destination
    this(ref A another) immutable {}              // 3 - mutable source, immutable destination
    this(ref immutable A another) immutable {}    // 4 - immutable source, immutable destination
}

void main()
{
    A a;
    immutable A ia;

    A a2 = a;      // calls 1
    A a3 = ia;     // calls 2
    immutable A a4 = a;     // calls 3
    immutable A a5 = ia;    // calls 4
}
```
The proposed model enables the user to define the copy from an object of any qualified type
to an object of any qualified type: any combination of two among mutable, `const`, `immutable`, `shared`,
`const shared`.

The `inout` qualifier may be applied to the copy constructor parameter in order to specify that mutable, `const`, or `immutable` types are
treated the same:

```d
struct A
{
    this(ref inout A rhs) immutable {}
}

void main()
{
    A r1;
    const(A) r2;
    immutable(A) r3;

    // All call the same copy constructor because `inout` acts like a wildcard
    immutable(A) a = r1;
    immutable(A) b = r2;
    immutable(A) c = r3;
}
```

In the case of partial matching, existing overloading and implicit conversions
apply to the argument.

#### Copy constructor call vs. blitting

When a copy constructor is not defined for a `struct`, initializations are handled
by copying the contents from the memory location of the right-hand side expression
to the memory location of the left-hand side expression (i.e. "blitting"):

```d
struct A
{
    int[] a;
}

void main()
{
    A a = A([7]);
    A b = a;                 // mempcy(&b, &a)
    immutable A c = A([12]);
    immutable A d = c;       // memcpy(&d, &c)
}
```

When a copy constructor is defined for a `struct`, all implicit blitting is disabled for
that `struct`:

```d
struct A
{
    int[] a;
    this(ref A rhs) {}
}

void fun(immutable A) {}

void main()
{
    immutable A a;
    fun(a);          // error: copy constructor cannot be called with types (immutable) immutable
}
```

#### Interaction with `alias this`

A `struct` may define both an `alias this` and a copy constructor for which assignments to variables
of the `struct` type may lead to ambiguities:

```d
struct A
{
    int a;
    immutable(A) fun()
    {
        return immutable A(7);
    }

    alias fun this;

    this(ref A another) immutable {}
}

struct B
{
    int a;
    B fun()
    {
        return B(7);
    }

    alias fun this;

    this(ref B another) immutable {}
}

void main()
{
    immutable A ia;
    A a = ia;            // 1 - calls copy constructor

    B bc;
    B b = bc;            // 2 - calls B.fun
}
```

When both the copy constructor and `alias this` are suitable to resolve the assignment (1),
the copy constructor takes precedence over `alias this` as it is considered more
specialized (the sole purpose of the copy constructor is to create copies). If no
copy constructor in the overload set matches the exact qualified types of the source and the
destination, the `alias this` is preferred (2).

#### Generating copy constructors

A copy constructor is generated for a `struct S`  if any member of S defines
a copy constructor that S does not define.

A field of a `struct` may define multiple copy constructors. In this situation,
a copy constructor is generated for each overload.

```d
struct M
{
    this(ref immutable M) {}
    this(ref immutable M) shared {}
}

struct S
{
    M m;
    this(ref shared S) {}
    /* The copy constructors with the following signatures are generated:
        this(ref immutable S)
        this(ref immutable S) shared
     */
}
```

The body of all the generated copy constructors performs memberwise initialization for each field:

```d
{
    this.field1 = s.field1;
    this.field2 = s.field2;
    ...;
}
```

Inside the body of the generated copy constructors, the assignment will be rewritten as a call
to the copy constructor of each field that defines one; blitting is employed where possible for
each field which does not define a copy constructor. If any field is uncopyable (e.g. the
copy constructor is disabled), the generated copy constructor will be annotated with `@disable`.

If the copy constructors of any fields, a single copy constructor is generated:

```d
struct A
{
    int a;
    this(ref A rhs)
    {
        this.a = rhs.a;
    }
}

struct B
{
    int[] b;
    this(ref B rhs)
    {
        this.b = rhs.b.dup;
    }
}

struct C
{
    A a;
    B b;
    /* the following copy constructor is generated for C:

    this(ref C rhs)
    {
        this.a = rhs.a;       // which, in turn, is lowered to a.__copyCtor(rhs.a);
        this.b = rhs.b;       // which, in turn, is lowered to b.__copyCtor(rhs.b);
    }
    */
}
```

#### Generating `opAssign` from a copy constructor

The copy constructor is used to initialize an object from another object, whereas `opAssign` is
used to copy an object to another object that has already been initialized:

```d
struct A
{
    int a;
    immutable int id;
    this(int a, int b)
    {
        this.a = a;
        this.b = b;
    }
    this(ref A rhs)
    {
        this.a = rhs.a;
        this.b = rhs.b;
    }
    void opAssign(A rhs)
    {
        this.a = rhs.a;
    }
}

void main()
{
    A a = A(2);
    A b = a;      // calls copy constructor;
    b = a;        // calls opAssign;
}
```

Both the copy constructor and the `opAssign` method are needed because they are type checked
differently: `opAssign` is type checked as a normal function, whereas the copy constructor is type
checked as a constructor (where the initial assignment of non-mutable fields is allowed). In many
cases, the body of the copy constructor will be identical to that of `opAssign`:

```d
struct A
{
    int a;
    this(int a)
    {
        this.a = a;
    }
    this(ref A rhs)
    {
        this.a = rhs.a;
    }
    void opAssign(S rhs)
    {
        this.a = rhs.a;
    }
}

void main()
{
    A a = A(2);
    A b = a;      // calls copy constructor;
    b = a;        // calls opAssign;
}
```

In order to avoid such code duplication, the user could ideally define a single method that deals with
both copy construction and normal copying. This is possible if the following conditions are met:

1. If the `struct` contains any `const`/`immutable` fields, it is impossible to use the copy constructor
for `opAssign` as the copy constructor might initialize the fields. The compiler must analyze the function
body to very that the copy constructor does not modify `const`/`immutable` fields, which
is problematic in situations when the body is missing. In conclusion, `opAssign` can be generated from the
copy constructor when the `struct` contains only assignable (mutable) fields.

2. The copy constructor signature is: `this(ref $q1 S rhs) $q2`, where `q1` and `q2` represent the
qualifiers that can be applied to the function and its parameter (`const`, `immutable`, `shared`, etc.).
Depending on the values of `$q1` and `$q2`, what should the signature of `opAssign` be?
A solution might be to generate the counterpart `opAssign` for each copy constructor, e.g. `void opAssign(ref $q1 S rhs) $q2`.
However, when is a `const`/`immutable` `opAssign` needed? There might be obscure cases when it is useful, but those are
niche situations where the user must step in to clarify the desired outcome, and define its own `opAssign`. For
the sake of simplicity, `opAssign` will be generated solely for copy constructors that have a missing `$q2`.

Taking into account the above restrictions, we can generate a single `opAssign` function
and leverage the existing `opAssign` generation logic for the [[postblit](https://github.com/dlang/dmd/pull/8505)].
Because `opAssign` is generated only if there are copy constructors that create a mutable object, the copy
constructor will be called (if possible) when the source is passed as a parameter to the `opAssign`
function.

### POD (Plain Old Data)

A `struct` that defines a copy constructor is not POD.

## Breaking Changes and Deprecations

1. The parameter of the copy constructor is passed by a mutable reference to the
source object. This means that a call to the copy constructor may legally modify
the source object:

```d
struct A
{
    int[] a;
    this(ref A another)
    {
        another.a[2] = 3;
    }
}

void main()
{
    A a, b;
    a = b;    // b.a[2] is modified
}
```

A solution to this might be to make the reference `const`, but that would make code like
`this.a = another.a` inside the copy constructor illegal. This can be solved with a cast, e.g.
`this.a = cast(int[])another.a`.

2. This DIP changes the semantic of constructors which receive a parameter by reference of type
`typeof(this)`. The consequence is that existing constructors might be called implicitly in some
situations:

```d
struct C
{
    this(ref C another)    // normal constructor before DIP, copy constructor after
    {
        import std.stdio : writeln;
        writeln("Yo");
    }
}

void fun(C c) {}

void main()
{
    C c;
    fun(c);
}
```

This will print "Yo" subsequent to the implementation of this DIP, where before it printed nothing. A case can be
made that a constructor with the above definition could not be correctly used as anything else than a copy
constructor, in which case this DIP  actually fixes the code.

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
