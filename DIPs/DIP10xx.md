# Add Unary Operator `...`

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | 10xx                                                            |
| Review Count:   | 0                                                               |
| Author:         | Manu Evans turkeyman@gmail.com                                  |
| Co-Author:      | Stefan Koch uplink.coder@gmail.com                              |
| Implementation: | https://github.com/TurkeyMan/dmd/tree/dotdotdot                 |
| Status:         | Draft                                                           |

## Abstract
Add an expression, `...`, to perform explicit tuple unpacking.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Compilation Performance](#compilation-performance)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)

## Rationale
Static transformations like _map_ and _fold_ are common and useful, but the mechanisms to implement them in D are awkward and have a high compile-time cost. The main struggle in producing efficient implementations is the tuple unpacking semantics, which usually necessitate recursive template expansions, often reaching quadratic complexity for relatively simple operations.

It is proposed that the language implement an expression to perform explicit tuple expansions at the expression level, which can express static map/fold operations efficiently and concisely, eliminating the necessity of using template expansion tricks to implement these patterns and avoid the associated compile-time costs.

This DIP proposes a unary `...` syntax, which explores an expression for tuples and expands to a tuple of expressions in which each tuple is replaced with its respective elements.

For example, given a tuple `Tup` containing three elements:
```d
(Tup*10)...  -->  ( Tup[0]*10, Tup[1]*10, Tup[2]*10 )
```

## Prior Work
C++11 implemented template parameter pack expansion with similar semantics, and it has been a great success in the language. Coupled with D's superior metaprogramming feature set, D users can gain even greater value from this novel feature.

## Description
Add a post-fix unary operator `...` which takes the form `expr...`, where the result is a tuple of expressions.

The implementation will explore `expr` for any tuples present in the expression tree, duplicate `expr` for each element in the discovered tuple, replacing the tuples with their respective elements. The result is a new tuple of `expr` duplicates with this substitution applied. If no tuples are discovered in the expression tree, an error is issued.

```d
alias Tup = AliasSeq!(1, 2, 3);
int[] myArr;
assert([ myArr[Tup + 1]... ] == [ myArr[Tup[0] + 1], myArr[Tup[1] + 1], myArr[Tup[2] + 1] ]);
```

This is an efficient and terse implementation of a static map applying a sequence of values to a common expression.

If multiple tuples are discovered beneath `expr`, they are expanded in parallel. They must have equal length, or an error is issued.

```d
alias Values = AliasSeq!(1, 2, 3);
alias Types = AliasSeq!(int, short, float);
pragma(msg, cast(Types)Values...);

> cast(int)1, cast(short)2, cast(float)3

alias OnlyTwo = AliasSeq!(10, 20);
pragma(msg, (Values + OnlyTwo)...);

> error: tuples beneath `...` expression have mismatching length
```

Notably, existing code can take immediate advantage of the improvements in compile time by upgrading the implementation of `staticMap` and, consequently, many parts of Phobos:
```d
alias staticMap(alias F, T...) = F!T...;
```

A second form shall exist which may implement a static reduce operation with the syntax `expr [BinOp] ...`, which will expand `expr` as above, but joins the resulting terms by a chain of `BinOp` operators. The result of this expansion is a single expression rather than a tuple of expressions.

```d
(Tup == 10) || ...  -->  ( Tup[0] == 10 || Tup[1] == 10 || Tup[2] == 10 )
```

For example:
```d
bool anyOnes = (Values == 1) || ...;
bool allOnes = (Values == 1) && ...;
int sum = Values + ...;
assert(anyOnes == true && allOnes == false && sum == 6);
```

The effect on user code will be increased readibility, a reduction in program logic indirections via 'utility' template definitions, and an improvement in compile time for sensitive applications.

Sensitive applications tend to include programs that perform systematic reflection, compile-time parsing, implementation of call-shims (i.e., foreign language bindings), or any compile-time preprocessing that cannot be strictly performed with CTFE.

Through application of the tools described here, the authors predict that a significant amount of boilerplate/cruft and recursive template instantions, which can constitute the majority of compile time in many applications, will cease to exist.

### Grammar Changes
```diff
PostfixExpression:
    PrimaryExpression
    PostfixExpression . Identifier
    PostfixExpression . TemplateInstance
    PostfixExpression . NewExpression
    PostfixExpression ++
    PostfixExpression --
+   PostfixExpression ...
+   PostfixExpression BinOp ...
    PostfixExpression ( ArgumentListopt )
    TypeCtorsopt BasicType ( ArgumentListopt )
    IndexExpression
    SliceExpression
```

## Compilation Performance
D projects can experience slow compilation due to explosive template expansion. Leading causes of template explosion tend to involve recursive expansion, and this most often looks like some form of static map (including `staticMap`), or static fold.

Existing solutions where `...` may be applied are implemented using recursive template instantiation. Each template instantiation populates the symbol table, and it is common that the arguments to such templates generate very long and expensive mangled names. By contrast, operator `...` generates no junk symbols, so avoids the associated cost in compile time. Experimental implementation has shown compile time improvement by orders of magnitude in sensitive programs.

The compile-time cost of operator `...` is negligible and strictly linear with the length of source tuples, whereas recursive template instantion has quadratic cost in compile time due to a growing number of symbol name and increased mangling costs with each iteration. In practice, this quadratic cost applies effective upper-limits on the length of lists that can be handled at compile time. The use of operator `...` will lift these practical limits substantially.

The experimental implementation has demonstrated the performance improvements described above.

## Breaking Changes and Deprecations
None

## Reference
[C++11 parameter pack expansion reference](https://en.cppreference.com/w/cpp/language/parameter_pack)

## Copyright & License
Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)