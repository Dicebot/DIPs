# Evaluate Pure Functions With CTFE

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Walter Bright walter@digitalmars.com                            |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

If the arguments to a call to a `pure` function are all literals, and the return
type is a literal, evaluate the call using CTFE.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Consider the [`std.conv.hexString`](https://dlang.org/phobos/std_conv.html#hexString) template.
It accepts a string literal, interprets the string literal as hex data, and returns a string
composed of the hex data:

```
auto string1 = hexString!"304A314B";
writeln(string1); // "0J1K"
```
The [implementation](https://github.com/dlang/phobos/blob/master/std/conv.d) looks like:
```
template hexString(string hexData)
if (hexData.isHexLiteral)
{
    enum hexString = mixin(hexToString(hexData));
}
```
This results in the instantiation of the following templates:

* `hexString`
* `isHexLiteral`
* `hexStringLiteral`
* `ElementEncodingType`

each with the string as an argument. Instantiation of the templates requires the encoding of
the string literal into the instantiation name. If the string literal is long, the template names
can get very long. This is slow, memory intensive, and can result in those names being written
to the generated object file.

While these four template instantiations can (and should) be reduced to one, the fundamental
problem remains.

This appears to be a common pattern in Phobos.

An alternative method is to use a function:
```
auto string1 = hexString("304A314B");
```
If it is a static variable being assigned to, `hexString()` is run at compile time,
and neither the function call nor the `"304A314B"` appear in the generated object
file, which is ideal. Unfortunately, if `string1` is a local variable, `hexString()`
gets run at runtime, and `"304A314B"` appears in the object file. One solution
is to use a two-step method:

```
enum tmp = hexString("304A314B");
auto string1 = tmp;
```
which forces the `hexString` call to be run at compile time and delivers the desired
result. This two-step approach can also be done using a template or a `mixin`.
But the two-step approach is awkward, extra work, and the cost of not using it will
be bloat and inefficiency that would likely go unnoticed by most programmers - one
cannot determine if it is even there unless one looks at the object file output.

This DIP proposes to expand the circumstances under which `pure` functions will be run
at compile time in order to eliminate many needs for two-step initializations and
unwieldly template instantiations along with their associated bloat.

It will further encourage the use of `pure` functions as many calls will be able to be
performed at compile time with no additional effort on the part of the programmer.


## Prior Work

Not known.


## Description

A `Literal` is defined as one of the following:

* `this`
* `super`
* `null`
* `true`
* `false`
* IntegerLiteral
* FloatLiteral
* CharacterLiteral
* StringLiterals
* ArrayLiteral, where each of its Arguments are also Literals
* AssocArrayLiteral, where each of its KeyValuePairs are also Literals
* StructLiteral, where each of its Arguments are also Literals

For a call to a `pure` function, if the function arguments are Literals or are Expressions
that are representable by Literals after the normal semantic processing of them, and the
return type of the function can be represented by a Literal, then the function is
evaluated at compile time using CTFE.

If the call cannot be evaluated using CTFE, such as if the `pure` function contains impure
`debug` statements in the path of CTFE execution, no error is generated, the function is simply
evaluated at run time instead.


No grammar changes are required.

## Alternatives

Don Clugston has proposed earlier (via email) an alternative in the form of using the existing `__ctfe`
keyword as a function attribute that will:

* force all uses of the function to be done at compile time
* prevent emission of the function to the object file

Disadvantages are:

* two versions of the function will need to exist if one also wants to use it at runtime
* `__ctfe` would add to the forest of existing function attributes
* some confusion over when a function should be `pure`, `__ctfe`, or `pure __ctfe`


## Breaking Changes and Deprecations

If a pure function generates a large literal using small inputs, this could cause
the object file and resulting executable file to grow accordingly. Runtime computation
can be restored by making at least one of the function's arguments be a variable,
even if that variable is replaced by the literal in a later compiler optimization.


## Reference

## Copyright & License
Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
