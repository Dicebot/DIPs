# Function pointers and Delegate Parameters Inherit Attributes from Function

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@digitalmars.com                            |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

When parameters are declared as delegates or function pointers, the `@safe`, `pure`,
`nothrow` and `@nogc` attributes that attached to the function type are ignored.
This DIP proposes that they become the defaults for the delegate and function pointer
types declared in the parameter declarations.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Consider the code:

```
@safe pure nothrow @nogc
int foo(int delegate() dg)
{
    return dg(); // Error
}
```

This fails because the delegate `dg` has none of those attributes. The user
will need to write:


```
@safe pure nothrow @nogc
int foo(int delegate() @safe pure nothrow @nogc dg)
{
    return dg(); // No error
}
```

The following also compiles without error:
```
@safe pure nothrow @nogc
{
    alias dg_t = int delegate();

    int foo(dg_t dg)
    {
        return dg(); // No error
    }
}
```

demonstrating that the `dg_t` declaration is picking up attributes that
are in scope.

Having the user repeat these attributes in the parameter declaration is
both surprising and burdensome. It is even more inconvenient because most
of the time a delegate is passed to a function is so the function can call it,
and the function cannot call it if it does not at least have the set of
`@safe`, `pure`, `nothrow`, and `@nogc` attributes that the function has.

Passing delegates to functions is a key feature of D and underpins a lot of
coding patterns. Making them easier to use by eliminating the need for a clutter
of attribute annotations will be welcome, and will lower the barrier to adding
annotation to functions in the first place.

A further reason is that the `lazy` function parameter attribute is underspecified
and a constant complication in the language specification. With this DIP, `lazy`
can move towards being defined in terms of an equivalent delegate parameter,
thereby simplifying the language by improving consistency.

This DIP proposes that delegate and function pointer types default to having
the same of the four attributes that the function has.


## Prior Work

No known.


## Description

Function pointer and delegate types defined in the function parameter list default
to the attributes of the function type that defines that parameter list.
The following will compile without error:
```
@safe pure nothrow @nogc
int foo(int delegate() dg, int function() fp)
{
    return dg() + fp();
}
```


## Breaking Changes and Deprecations

It is possible that a parameter declaration may require that a delegate or function pointer
parameter have fewer attributes than the function itself. This would only be possible if the
delegate or function pointer was never called, but was ignored or simply stored elsewhere.

Such can be accomplished by declaring the parameter type elsewhere:

```
alias dg_t = void delegate() dg_t;
alias fp_t = void function() fp_t;

dg_t global_dg;
fp_t global_fp;

@safe pure nothrow @nogc
void foo(dg_t dg, fp_t fp)
{
    global_dg = dg;
    global_fp = fp;
}
```

Because of possible code breakage, this feature will be initially available only via
the `-preview=attrdelegate` compiler switch.

## Reference

## Copyright & License
Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
