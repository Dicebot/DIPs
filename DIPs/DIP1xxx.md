# Expression and Block Statement Template Constraints

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0                                                               |
| Author:         | Nicholas Wilson                                                 |
| Implementation: | https://github.com/thewilsonator/dmd/tree/template-constraint-dip |
| Status:         | Draft                                                           |

## Abstract

Allow multiple `if` template constraints, for the expression form allow an optional message to be printed in the 
event that overload resolution fails (similar to `static assert`), as well as a block statement
form of template constraint that allows the use of `static foreach`.
That is to say, template constraint `if` becomes the static form of contract precondition `in`.

The template is considered a valid overload iff each of the constraints is satified.

Expression form:
```D
template all(alias pred)
{
    bool all(Range)(Range range)
    if (isInputRange!Range)
    if (is(typeof(unaryFun!pred(range.front))),
        "`" ~ pred.stringof[1..$-1] ~ "` isn't a unary predicate function for range.front")
    {
    }
}
```

Block statement form:
```D
ptrdiff_t countUntil(alias pred = "a == b", R, Rs...)(R haystack, Rs needles)
if 
{
    static assert(isForwardRange!R);
    static assert(Rs.length > 0, "need at least one needle as stop condition");
    static foreach (alias N; Rs) 
        static assert(isForwardRange!(N) == isInputRange!(N), "needles that are ranges must be forward ranges");
    static foreach (n; needles)
        static assert(is(typeof(startsWith!pred(haystack, n))), 
                      "predicate `" ~ pred.stringof "` must be valid for `startsWith!pred(haystack, "~n.stringof~"`));
}
```

Mixed:

```D
ptrdiff_t countUntil(alias pred = "a == b", R, Rs...)(R haystack, Rs needles)
if (isForwardRange!R)
if (Rs.length > 0, "need a needle to countUntil with")
if 
{
    static foreach (alias N; Rs) 
        static assert(isForwardRange!(N) == isInputRange!(N), "needles that are ranges must be forward ranges");
    static foreach (n; needles)
        static assert(is(typeof(startsWith!pred(haystack, n))), 
                "predicate `" ~ pred.stringof "` must be valid for `startsWith!pred(haystack, "~n.stringof~"`));
}
```

### Reference

https://github.com/dlang/phobos/pull/6607

https://issues.dlang.org/show_bug.cgi?id=13683

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

It is well known that compilation error messages due to template contraint overload resolution 
are particularly difficult to decipher. This is not helped by the number of overloads and very 
precice (and thus complex) constraints placed on the overloads. When overload resolution fails
the compiler will print out all the in scope overloads and their constraints, without indication
of which constraints have failed.

While it is not possible in the general case to provide useful information as to what constraints
have failed and why, because a constraint may have an arbitrary combination of logic, the vast 
majority of constraints are expressed in Conjunctive Normal Form (CNF). In this case it is definitely 
possible to provide better daignostics as to which clauses have failed. However the current grammar
provides no way to translate particularly verbose constraints to a user not intimately familiar with 
the constraint.

This DIP therefore proposes to formalise the use of CNF constraints by allowing multiple `if` constraints,
the expression form with an optional message (similar to what was done with contracts in DIP1009), as well as block statements that 
allows the use of `static foreach` and to declare `alias`es and `enum`s to eliminate the need for recursive templates in template constraints 
(similar to the `in` contract form prior to DIP1009).
This will put the compiler in a much better position to provide useful diagnostics, such as indicating which clauses are not satisfied 
and allowing the template author to provide messages in the case of non-intuitive formulations of constraints
e.g. `if (isForwardRange!(R) == isInputRange!(R), "needles that are ranges must be forward ranges")`.

Using the particularly egregious example of the first overload of `std.algorithm.searching.countUntil`,
its current signature of

```D
ptrdiff_t countUntil(alias pred = "a == b", R, Rs...)(R haystack, Rs needles)
if (isForwardRange!R
&& Rs.length > 0
&& isForwardRange!(Rs[0]) == isInputRange!(Rs[0])
&& is(typeof(startsWith!pred(haystack, needles[0])))
&& (Rs.length == 1
|| is(typeof(countUntil!pred(haystack, needles[1 .. $])))))
```

would be be written using the block statement form to eliminate the recursive constraint as  

```D
ptrdiff_t countUntil(alias pred = "a == b", R, Rs...)(R haystack, Rs needles)
if (isForwardRange!R)
if (Rs.length > 0, "need a needle to countUntil with") // example message, probably not needed for something this simple
if 
{
    static foreach (n; needles)
    {
        static assert(isForwardRange!(typeof(n)) == isInputRange!(typeof(n)), 
                        "`"~n.stringof ~"`: needles that are ranges must be forward ranges");
        static assert(is(typeof(startsWith!pred(haystack, n))), 
                        "predicate `" ~ pred.stringof "` must be valid for"~
                        "`startsWith!pred(haystack, "~n.stringof~"`));
    }
}
```
the first two constraints do not require the use of the block statement form to use a static foreach so 
they can be done in the expression style.

This could print on error using `countUntil("foo", inputRangeOfInt)` 
```
example.d(42): Error: template `std.algorithm.searching.countUntil` cannot deduce function from argument types !()(string,NotARange), candidates are: 
/path/to/std/algorithm/searching.d(747): std.algorithm.searching.countUntil(alias pred = "a == b", R, Rs...)(R haystack, Rs needles)
        not satisfied: `inputRangeOfInt`: needles that are ranges must be forward ranges
        not satisfied: predicate `a == b` must be valid for `startsWith!pred(haystack, inputRangeOfInt)`
/path/to/std/algorithm/searching.d(835): std.algorithm.searching.countUntil(alias pred = "a == b", R, N)(R haystack, N needle) if (isInputRange!R && is(typeof(binaryFun!pred(haystack.front, needle)) : bool))
/path/to/std/algorithm/searching.d(913): std.algorithm.searching.countUntil(alias pred, R)(R haystack) if (isInputRange!R && is(typeof(unaryFun!pred(haystack.front)) : bool))
```

## Description

Template constraints are changed to allow multiple `if` template constraints. 
All constraints must be satisfied for the template to be viable.
Examples given are for `template`s but also apply to constraints on template mixins, functions, methods, classes, 
interfaces, structs and unions.

### Expression Form

The expression form is:
```D
template foo(T) 
if (constraint1!T) 
if (constraint2!T)
if (constraint3!T) { ... }
```

An optional constraint message can be used to provide a more easily understood description of why a 
constraint has not been met.

```D
template foo(T) 
if (isForwardRange!T == isInputRange!T, T.stringof ~" must be a forward range if it is a range") 
```
### Block Statement Form

The block statement form is:
```D
template foo(T) 
if 
{
   ...constraints...
}
```
where `...constraints...` may contain only `static assert`, `static foreach` and `static if` statements and `enum` and `alias` declarations. 
The declarations are local to the scope of the constraint or `static foreach` or `static if` they are declared in.
Each `static assert` in the block statement in satisfied `static if` statements,
including those in unrolled `static foreach` statements, must pass for the constraint to be satisfied.

### Grammar changes

```diff
+Constraints:
+   Constraint
+   Constraint Constraints

Constraint:
-  if ( Expression )
+   if ( AssertArguments )
+   if BlockStatement

FuncDeclaratorSuffix:
    Parameters MemberFunctionAttributes[opt]
-   TemplateParameters Parameters MemberFunctionAttributes[opt] Constraint[opt]
+   TemplateParameters Parameters MemberFunctionAttributes[opt] Constraints[opt]

TemplateDeclaration:
-   template Identifier TemplateParameters Constraint[opt] { DeclDefs[opt] }
+   template Identifier TemplateParameters Constraints[opt] { DeclDefs[opt] }

ConstructorTemplate:
-   this TemplateParameters Parameters MemberFunctionAttributes[opt] Constraint[opt] :
-   this TemplateParameters Parameters MemberFunctionAttributes[opt] Constraint[opt] FunctionBody
+   this TemplateParameters Parameters MemberFunctionAttributes[opt] Constraints[opt] :
+   this TemplateParameters Parameters MemberFunctionAttributes[opt] Constraints[opt] FunctionBody

ClassTemplateDeclaration:
-   class Identifier TemplateParameters Constraint[opt] BaseClassList[opt] AggregateBody
-   class Identifier TemplateParameters BaseClassList[opt] Constraint[opt] AggregateBody
+   class Identifier TemplateParameters Constraints[opt] BaseClassList[opt] AggregateBody
+   class Identifier TemplateParameters BaseClassList[opt] Constraints[opt] AggregateBody

InterfaceTemplateDeclaration:
-   interface Identifier TemplateParameters Constraint[opt] BaseInterfaceList[opt] AggregateBody
-   interface Identifier TemplateParameters BaseInterfaceList Constraint AggregateBody
+   interface Identifier TemplateParameters Constraints[opt] BaseInterfaceList[opt] AggregateBody
+   interface Identifier TemplateParameters BaseInterfaceList Constraints AggregateBody

StructTemplateDeclaration:
-   struct Identifier TemplateParameters Constraint[opt] AggregateBody
+   struct Identifier TemplateParameters Constraints[opt] AggregateBody

UnionTemplateDeclaration:
-   union Identifier TemplateParameters Constraint[opt] AggregateBody
+   union Identifier TemplateParameters Constraints[opt] AggregateBody

TemplateMixinDeclaration:
-   mixin template Identifier TemplateParameters Constraint[opt] { DeclDefs[opt] }
+   mixin template Identifier TemplateParameters Constraints[opt] { DeclDefs[opt] }
```


## Breaking Changes and Deprecations

N/A. The current template constraint syntax becomes a single expression constraint.

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
