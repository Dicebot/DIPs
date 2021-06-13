In-place struct initialization
===============================

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Cédric Picard (cpicard@openmailbox.org), Sebastian Wilzbach (seb[at]wilzba[dot]ch)  |
| Author:         | (your name and contact data)                                    |
| Implementation: | [#8460](https://github.com/dlang/dmd/pull/8460)                 |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

Abstract
--------

D supports static initialization for structs on declaration,
but not on assignment nor function calls. This DIP proposes to remove this
special case and support static struct initialization at any place where
calling a struct constructor would be possible.

### Reference

### Links

- [D grammar](https://dlang.org/spec/grammar.html)
- [Struct initialization language specification](https://dlang.org/spec/struct.html#static_struct_init)
- [Initial discussion on the NG](http://forum.dlang.org/post/firixmhzmnfoderhrbch@forum.dlang.org)
- [Issue 15692 - Allow struct member initializer everywhere](https://issues.dlang.org/show_bug.cgi?id=15692)

## Contents

- [Rationale](#rationale)
- [Description](#description)
- [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
- [Copyright & License](#copyright--license)
- [Reviews](#reviews)

Rationale
---------

Static struct initialization has great properties:

- It is explicit using named fields
- Order of declaration doesn't matter
- Not all attributes have to be specified

No current function call syntax provide those properties, and consequently
no constructor can benefit from it either. Allowing struct
initialization everywhere would enable current implementations to be simplified
and result in cleaner, more easily readable code.

The concept known as "orthogonality of language features" can be said to apply to this proposal.
Struct initialization already works for declarations in D, hence a user would expect them

An interesting application of in-place struct initialization is to use
structs to mimic keyword arguments for functions.
By encapsulating possible arguments in a struct it is possible to
use in-place initialization to provide a clean interface very similar to
keyword arguments such as seen in Python or Ruby.

At the moment complex argument sets can only accepted by a function via:

- generating lots of constructors for the different cases which is
messy
- setting a struct up before passing it to the function in a C-way
fashion

This change provides ways to design better high-level interfaces.

Description
-----------

Let S be a struct defined as:

```d
struct S {
    uint a;
    long b;
    int  c;
}
```

Currently static struct initialization is _only_ allowed in the line of the declaration
and for the example `S` it could look like this:

```d
S s = { a: 42, b: 10 };
```

The proposed change allows in-place struct initialization at any place where
calling a struct constructor would be possible as well.
In-place struct initialization behaves analogous to default constructor of structs
and thus is only allowed when there's no user-defined constructor.

Options
-------

However, there are multiple potential options for the new syntax:

<a name="option1">

### Option 1: curly braces

```diff
diff --git a/spec/expression.dd b/spec/expression.dd
@@ -950,6 +950,7 @@ $(GNAME PostfixExpression):
     $(I PostfixExpression) $(D .) $(GLINK NewExpression)
     $(I PostfixExpression) $(D ++)
     $(I PostfixExpression) $(D --)
+    $(I PostfixExpression) $(D $(LPAREN)) $(GLINK2 declaration, StructInitializer) $(D $(RPAREN))
     $(I PostfixExpression) $(D $(LPAREN)) $(GLINK ArgumentList)$(OPT) $(D $(RPAREN))
     $(GLINK2 declaration, TypeCtors)$(OPT) $(GLINK2 declaration, BasicType) $(D $(LPAREN)) $(GLINK ArgumentList)$(OPT) $(D $(RPAREN))
     $(GLINK IndexExpression)
```

Example:

    auto s = S({a:42, b:10});

This syntax matches the existing static struct initialization syntax the closest.
However, for people coming from other languages - especially where associative arrays
are defined with curly braces - this could look like passing a single argument
to the constructor of `S`. It is also the most verbose (in terms of required tokens)
of all three proposed options.

This syntax doesn't conflict with anonymous functions as every statement in a
function needs to end with a semi-colon:

    auto s = S({42}); // ERROR: found } when expecting ; following statement
    auto s = S({calc42()}); // ERROR: found } when expecting ; following statement

<a name="option2">

### Option 2: no braces

```diff
diff --git a/spec/expression.dd b/spec/expression.dd
@@ -950,6 +950,7 @@ $(GNAME PostfixExpression):
     $(I PostfixExpression) $(D .) $(GLINK NewExpression)
     $(I PostfixExpression) $(D ++)
     $(I PostfixExpression) $(D --)
+    $(I PostfixExpression) $(D $(LPAREN)) $(GLINK2 declaration, StructMemberInitializers) $(D $(RPAREN))
     $(I PostfixExpression) $(D $(LPAREN)) $(GLINK ArgumentList)$(OPT) $(D $(RPAREN))
     $(GLINK2 declaration, TypeCtors)$(OPT) $(GLINK2 declaration, BasicType) $(D $(LPAREN)) $(GLINK ArgumentList)$(OPT) $(D $(RPAREN))
     $(GLINK IndexExpression)

```

Example:

    s = S(c:10, b:20);

While this syntax looks the most aesthetically pleasing to the author, it
might be used in the future for named arguments.

<a name="option3">

### Option 3: curly braces without parentheses

```diff
diff --git a/spec/expression.dd b/spec/expression.dd
--- a/spec/expression.dd
+++ b/spec/expression.dd
@@ -950,6 +950,7 @@ $(GNAME PostfixExpression):
     $(I PostfixExpression) $(D .) $(GLINK NewExpression)
     $(I PostfixExpression) $(D ++)
     $(I PostfixExpression) $(D --)
+    $(I PostfixExpression) $(D {) $(GLINK2 declaration, StructMemberInitializers) $(D })
     $(I PostfixExpression) $(D $(LPAREN)) $(GLINK ArgumentList)$(OPT) $(D $(RPAREN))
     $(GLINK2 declaration, TypeCtors)$(OPT) $(GLINK2 declaration, BasicType) $(D $(LPAREN)) $(GLINK ArgumentList)$(OPT) $(D $(RPAREN))
     $(GLINK IndexExpression)
```

Example:

    s = S{c:10, b:20};


This syntax could be ambiguous with a token string and labels:

    s = q{c: 10};
    s = q{10};

A solution would be to disallow `NonVoidInitializers`:

```
$(GNAME StructMemberInitializersPostfix):
    $(GLINK StructMemberInitializePostfixr)
    $(GLINK StructMemberInitializerPostfix) $(D ,)
    $(GLINK StructMemberInitializerPostfix) $(D ,) $(I StructMemberInitializersPostfix)

$(GNAME StructMemberInitializerPostfix):
    $(I Identifier) $(D :) $(GLINK NonVoidInitializer)
```

[option1]: #option1
[option2]: #option2
[option3]: #option3

### Implementation

For example, `callMe(S({c: 10}))` could be implemented as a simple lowering:

    S __tmp1 = { c: 10 };
    callMe(__tmp1);

Examples
--------

These examples depend on a specific syntax option.
For simplicity, [option 2][option2] has been used for all examples.
First of all, a simple example is provided to make the reader familiar
with the proposed syntax and its advantages:

### Simple

```d
struct totalArgs {
    int tax;
    int discount;
}

int total(int subtotal, totalArgs args = totalArgs.init) {
    return subtotal + args.tax - args.discount;
}

unittest {
    assert(42.total == 42);
    assert(42.total(totalArgs(tax: 50)) == 92);
    assert(total(42, totalArgs(discount: 20, tax: 50)) == 72);

    int defaultTotal(int subtotal) {
        return total(subtotal, totalArgs(tax: 20));
    }
}
```

#### In-place struct initialization in Phobos

Many functions in Phobos provide multiple overloads to workaround the problem
of not having named arguments. For example, [`spawnProcess`](https://dlang.org/phobos/std_process.html#.spawnProcess)
offers these overloads to its user:

```
Pid spawnProcess(in char[][] args, File stdin = std.stdio.stdin, File stdout = std.stdio.stdout, File stderr = std.stdio.stderr, const string[string] env = null, Config config = Config.none, in char[] workDir = null);

Pid spawnProcess(in char[][] args, const string[string] env, Config config = Config.none, in char[] workDir = null);

Pid spawnProcess(in char[] program, File stdin = std.stdio.stdin, File stdout = std.stdio.stdout, File stderr = std.stdio.stderr, const string[string] env = null, Config config = Config.none, in char[] workDir = null);

Pid spawnProcess(in char[] program, const string[string] env, Config config = Config.none, in char[] workDir = null);
```

However, let's assume the user wants to change the working directory of the spawned process,
then he still needs to specify all other parameters are there's no relevant overload for this use case.

```d
import std.process;
auto pid = spawnProcess(["dmd", "myprog.d"],
                        std.stdio.stdin,
                        std.stdio.stdin,
                        std.stdio.stderr,
                        null,
                        Config.none,
                        "/my/custom/working/dir", // <- parameter that the user is interested in changing
);
```

With in-place struct initialization, it could look like this:

```d
auto pid = spawnProcess(["dmd", "myprog.d"], SpawnParams(workingdir: "/my/custom/workingdir"));
```

Also the overloads could be reduced to a single one:

```
struct SpawnParams
{
    File stdin = std.stdio.stdin;
    File stdout = std.stdio.stdout;
    File stderr = std.stdio.stderr;
    string[string] env;
    Config config = Config.none;
    char[] workDir;
}
Pid spawnProcess(in char[][] args, SpawnParams params);
```

#### In-place struct initialization in the wild

The following example of an interaction with Amazon Web Services
is an example for how in-place struct initialization would help
with the interaction with complex APIs:
At the moment an complex configuration objects can't be easily constructed in-place,
which gets pretty tedious for large objects:

```d
BucketOptions b1Options = {
    bucket: "MyBucket1",
    createBucketConfiguration: {
        locationConstraint: BucketLocationConstraint.EU_CENTRAL_1
    }
};
BucketOptions b1Options = {
    bucket: "MyBucket2",
    createBucketConfiguration: {
        locationConstraint: BucketLocationConstraint.EU_CENTRAL_1
    }
};
invoker.execute([
	new CreateBucketCommand(client, b1Options),
	new CreateBucketCommand(client, b2Options),
])
```

With in-place struct initialization the configuration object can be constructed
in-place and no intermediate states which are typically tedious to maintain are required:

```d
invoker.execute([
	new CreateBucketCommand(client, BucketOptions(
		bucket: "MyBucket1",
		createBucketConfiguration: {
			locationConstraint: BucketLocationConstraint.EU_CENTRAL_1
		}
	)),
	new CreateBucketCommand(client, BucketOptions(
		bucket: "MyBucket2",
		createBucketConfiguration: {
			locationConstraint: BucketLocationConstraint.EU_CENTRAL_1
		}
	))
]);
```

This gets clearer when we look at how typically API request are typically handled
in Vibe.d as in many cases a Json Object is created on the fly:

```d
class Example : ExampleAPI {
	Json get()
	{
		return serializeToJson(["id": 42, "message": "Hello D."]);
	}
}
```

With the new in-place struct initialization, returning a typed response will
be as convenient as returning a Json object:

```d
struct ExampleResponse
{
    int id;
    string message;
}

class Example : ExampleAPI {
    static int id;
	ExampleResponse get()
	{
		return ExampleReponse({"id": id++, "message": "Hello D."});
	}
}
```

Complex configuration objects is common day-to-day problem.
Here's another configuration example from [Vulkan](https://github.com/ColonelThirtyTwo/dvulkan):

```d
VkImageCreateInfo imgInfo = {
	imageType: VkImageType.VK_IMAGE_TYPE_2D,
	format: VkFormat.VK_FORMAT_R8G8B8A8_UNORM,
	extent: image.size,
	mipLevels: image.mipLevels,
	arrayLayers: image.layers,
	samples: VkSampleCountFlagBits.VK_SAMPLE_COUNT_1_BIT,
	tiling: VkImageTiling.VK_IMAGE_TILING_LINEAR,
	usage: VkImageUsageFlagBits.VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VkImageUsageFlagBits.VK_IMAGE_USAGE_SAMPLED_BIT,
	sharingMode: VkSharingMode.VK_SHARING_MODE_EXCLUSIVE,
	initialLayout: VkImageLayout.VK_IMAGE_LAYOUT_PREINITIALIZED,
};
enforceVK(vkCreateImage(device, &imgInfo, null, &image.img));
```

### User-defined attributes

Currently on UDAs it's not possible to specify only specific argument nor arbitrary order.
This DIP would allow this useful pattern:

```d
@S(c:42) void foo();
```

This is especially useful for serialization:

```d
struct CreateTableInput
{
	@FieldInfo({memberName: "TableName"})
	TableName tableName;

	@FieldInfo({memberName: "AttributeDefinitions", minLength: 1})
	AttributeDefinitions attributeDefinitions;
}
```

### Struct-initialization in other languages

#### Rust

One can (1) indicate another struct which should be copied as a base and
(2) if local variables have the same name as fields a simplified syntax can be used.

```rust
struct Point3d {
    x: i32,
    y: i32,
    z: i32,
}

fn main() {
    let mut point = Point3d { x: 0, y: 0, z: 0 };
    point = Point3d { y: 1, .. point };
    println!("({}, {}, {})", point.x, point.y, point.z);

    let x = 2, y = 2, y = 2;
    point = Point3d { x, y, z};
    println!("({}, {}, {})", point.x, point.y, point.z);
}
```

Thus, in Rust a typical pattern is as follows:

```rust
my_awesome_func(FooParameters {
    foo: 42,
    bar: "cake",
    .. FooParameters::default()
});
```

#### Go

Go allows named fields when initializing structs, omitted fields will be zero-valued.

```go
package main
import "fmt"
type person struct {
    name string
    age  int
}
func main() {
    fmt.Println(person{"Bob", 20})
    fmt.Println(person{name: "Alice", age: 30})
    fmt.Println(person{age: 30})
}
```

#### C

[C99 Compound Literals](http://en.cppreference.com/w/c/language/compound_literal)
allow assignment:

```c
#include <stdio.h>
struct point {
    int x, y, z;
};
int main() {
    struct point p;
    p = (struct point){ .z = 2 , .y = 1};
    printf("x: %d, y:%d, z:%d\n", p.x, p.y, p.z);
    return 0;
}
```

#### C++

C++11 supports aggregate initialization from [braced init lists](https://en.cppreference.com/w/cpp/language/aggregate_initialization):

```cpp
struct Foo
{
    int a;
    int b;
};

auto a = Foo{ .b = 4 };
```

#### Bonus 1: Function calls with the struct argument as sole parameter

Let's imagine we define a function `valid`:

```d
bool valid(S s)
{
    return s.c > 5;
}
```

With the current options (e.g. [Option 2][option2]) calling `valid` would be written like this:

```d
valid(S(c: 10));
```

Thus, a potential addition would be, if

- there's only one argument and
- it's a struct
- it doesn't have a user-defined constructor

yielding syntax like this:

```d
valid(c: 10);
```

This might complicate function overloading:

```d
struct Foo { int c; }
struct Bar { int c; }

bool valid(Foo s)
{
    return s.c > 5;
}

bool valid(Bar s)
{
    return s.c > 6;
}

valid(c: 10); // The Foo or Bar overload?
```

Though an easy way out would be to forbid the in-place syntax for such ambiguous cases.
However, as this syntax might be used for named arguments in the future
exploring this syntax goes over the scope of this DIP.
If this syntax change turns out to be beneficial it could always extended in a further DIP.

#### Bonus 2: Syntax sugar for in-place struct assignment

This DIP does _not_ try to introduce the following syntax as this DIP
proposes only the minimal version.
However, the in-place struct assignment could be allowed directly for struct assignments:

```diff
diff --git a/spec/expression.dd b/spec/expression.dd
@@ -55,6 +55,7 @@ $(GRAMMAR
 $(GNAME AssignExpression):
     $(GLINK ConditionalExpression)
     $(GLINK ConditionalExpression) $(D =) $(I AssignExpression)
+    $(GLINK ConditionalExpression) $(D =) $(D $({)) $(GLINK2 declaration, StructInitializer) $(D $(}))
     $(GLINK ConditionalExpression) $(D +=) $(I AssignExpression)
     $(GLINK ConditionalExpression) $(D -=) $(I AssignExpression)
     $(GLINK ConditionalExpression) $(D *=) $(I AssignExpression)
```

For example allowing this code:

```d
struct S
{
    int a, b, c;
}
S s;
s = {c: 10};
```

This might have similar overloading problems like the ones mentioned in Bonus 1
when `opAssign` is overload:

```d
struct A { int a; }
struct AB
{
    int a, b;
    auto opAssign(AB s)
    {
        this = s;
    }
    auto opAssign(A s)
    {
        a = s.a;
    }
}
AB s;
s = {a: 10}; // which opAssign should be called?
```

However, depending on the syntax option, the proposed syntax would at least allow the following:

    s = S(c: 10);

Justifying whether it's worth to go one step more, is a task for another DIP.

## Breaking Changes and Deprecations

No code breakage is expected as all changes are additive to the D grammar.

### Compatibility with older compilers

This change is completely retrocompatible in a nice way: the library
itself is just defining an argument struct and using it in its function
interface. Code using older compilers can setup the struct without in-place
initialization and modern compilers benefit from a cleaner interface.

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
