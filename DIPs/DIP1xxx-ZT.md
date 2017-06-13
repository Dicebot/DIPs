# Improve Contract Usability

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | 1xxx                                                            |
| RC#             | 0                                                               |
| Author:         | Zach Tollen - reachzach@gmail.com                        |
| Implementation: | None                                                            |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

This DIP proposes the addition of a new syntax for contracts in D, with the aim of making them easier to read and write.

### Links

[Related forum discussion](http://forum.dlang.org/post/cklhgfbnpajbeefmwjrf@forum.dlang.org)
[First announcement of this DIP](http://forum.dlang.org/post/tuzdqqpcoguatepgxupq@forum.dlang.org)

## Rationale

If a new syntax could be devised that removed the "extra plumbing" that `in` and `out` contracts currently require, their use would probably increase. Aptly characterizing the problem, [Jonathan M. Davis says,](http://forum.dlang.org/post/mailman.2288.1494811099.31550.digitalmars-d@puremagic.com) "The amount of extra code required around in/out contracts is part of why I almost never use them. In most cases, it just makes more sense to put the assertions in the function body and not have all of that extra plumbing right after the function signature."

This sentiment is probably very common. Even a code base that uses contracts, like SDC, must adhere to a strict formatting style to keep them manageable. [For example:](https://github.com/SDC-Developers/SDC/blob/master/src/d/parser/expression.d#L1059)
```d
ulong strToDecInt(string s) in {
    assert(s.length > 0, "s must not be empty");
} body {
    ulong ret = 0;
    ...
}  
```
The same contract in any other formatting style, including the standard style of dmd and phobos, would cause them to take up so much space that they would not be worthwhile for that reason alone:
```d
ulong strToDecInt(string s)
in
{
    assert(s.length > 0, "s must not be empty");
}
body
{
    ulong ret = 0;
}
```

Yet `in` and `out` contracts still have their uses. And history shows that a small change in syntax can dramatically affect programmers' willingness to use a given construct.

(Author's note: The recent acceptance of [DIP1003](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1003.md) has changed the `body` keyword to `do`, but for simplicity's sake, I'll keep using `body` in the examples.)

## Description
### Preliminary

The main proposal requires the introduction of the following three syntactic enhancements:

#### 1. allow `in` and `out` statements with semicolons

Expand the grammar of `in` and `out` statements to include regular statements in addition to bracketed block statements. Therefore this:
```d
in assert(x);
out(r) assert(r);
```
would be lowered to this:
```d
in { assert(x); }
out(r) { assert(r); }
```
This enhancement preserves the basic idea of contracts as statements. Added to D, it would arguably be an improvement by itself, as many programmers are deterred from writing one-line contracts simply because of the brackets and formatting they require.

#### 2. allow multiple `in` and `out` statements

The compiler groups all `in` statements, which must be consecutive, into a single contract, with multiple statements maintaining lexical order in the lowered code. It does the same thing with `out` statements, with one restriction: There may be only one `out` statement that defines a return identifier. That identifier may still be used in the other `out` statements, if desired.

By themselves, multiple such `in` and `out` statements seem unnecessary. But when unbracketed, readability can improve:
```d
in assert(x);
in assert(y);
out(r) assert(r);
out assert(z);

```
would create these contracts:
```d
in {
    assert(x);
    assert(y);
}
out(r) {
    assert(r);
    assert(z);
}
```
#### 3. allow contracts within the function body

They must occur at the beginning of the function, and act as syntax sugar for external contracts:
```d
int fun(int x)
{
    in { assert(x); }
    out(r) { assert(r); }
    ...
}
```
gets lowered to this:
```d
int fun(int x)
in { assert(x); }
out(r) { assert(r); }
{
    ...
}
```

Virtual interface functions cannot use this syntax, as the existence of contracts in a function body implies the existence of that body.  To generate documentation, rewrite the contracts using the old style, if necessary.

Note that it's possible to allow enhancements 1 and 2 only within function bodies.

### Proposal

Combining enhancements 1, 2, and 3 above, this example of current D:
```d
int noZeroAdd(int a, int b)
in {
    assert(a);
    assert(b);
}
out(r) {
    assert(r);
}
body {
    return a + b;
}
```
can now be written like this:
```d
int noZeroAdd(int a, int b) {
    in assert(a);
    in assert(b);
    out(r) assert(r);

    return a + b;
}
```
The latter is lowered to the former. The more familiar use with brackets is still allowed, of course, even in the body:
```d
int noZeroAdd(int a, int b) {
    in {
        assert(a);
        assert(b);
    }
    out(r) {
        assert(r);
    }

    return a + b;
}
```
## Alternatives

Any of the three enhancements above can be combined separately to produce alternatives. In particular, if enhancement 3 is deemed too extravagant, enhancement 1, or the combination of enhancements 1 and 2, may still be desirable.

Additionally, a variety of alternatives has been suggested in the forums. The details can be found in the following links:

- [allow contracts as naked expressions](http://forum.dlang.org/post/qxlihdtknfobguubugym@forum.dlang.org):
- [imply `assert` in contracts](http://forum.dlang.org/post/mailman.2337.1494961562.31550.digitalmars-d@puremagic.com):
- [@uda based contracts](http://forum.dlang.org/post/ckznaermqjadwamqaewd@forum.dlang.org)
- [inspiration from the Dafny programming language](http://forum.dlang.org/post/ckznaermqjadwamqaewd@forum.dlang.org)

## Analysis

In addition to the current need to use the keyword `body` (or `do`; see [DIP1003](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1003.md)) before the body , brackets are the main thing that make current D's contracts unwieldy to use in practice. Any variation of a new construction that allows contracts to be written without brackets will likely be an improvement. Enhancements 1, 2 have this advantage. Critiques of each enhancement follow.

#### Critique 1

Within a function body, semicolon statements are normal and expected. But in contracts outside of a function body they would be surprising. This is an argument in favor of allowing them only within function bodies. 

#### Critique 2

The pros and cons of allowing multiple `in` and `out` statements are stated in the original description. The reason for allowing only one `out` statement to declare a return identifier is to avoid any potential confusion during the lowering.

#### Critique 3

Allowing contracts inside function bodies is likely the most controversial of the listed proposals.

Downsides:
* One downside is that seeing them has produced a cognitive dissonance in at least one programmer, [who states his objection thusly:](http://forum.dlang.org/post/ofe315$11bc$1@digitalmars.com) "Contracts are not part of the function body, they are part of the function signature. That's the point."
* The second downside is that documentation generators which are accustomed to skipping through the body of a function while parsing will have to be modified to look past the opening brace for `in` and `out` contracts first. This is an implementation detail, but it might annoy some people who already experience the cognitive dissonance described above, if they had to implement it.

Benefits:
* By contrast, writing contracts at the top of the function body seems like a natural place to put them and, for many programmers, would be part of their natural workflow.
* Converting existing `assert`s to contracts and back will be easier if all they require is a prefix.
* Those programmers who dislike the requirement of having to write the word `body` (or `do`) for all functions with contracts will no longer have to do so.
* Finally, the awkwardness of semicolon statements appearing outside function bodies, as decribed in critique 1, would not be an issue if such statements were only allowed within function bodies.

#### Critique of Alternatives

When analyzing the alternatives, it will help to bear in mind that the main proposal has the following advantages which, for practical purposes, weigh heavily in its favor:

- contracts are simple to write, with minimal "extra plumbing", i.e. without brackets and parentheses
- they utilize existing contract infrastructure
- there is minimal divergence from D's existing syntax and semantics

Because of this, it is probably beyond the scope of this DIP to analyze the alternatives. This will change if compelling reasons to do so arise.

### Grammar

With semicolon statements allowed only inside the function body, the grammar would change as follows:

- the existing `InStatement` would be renamed to `InBlockStatement`
- existing `OutStatement` would be renamed to `OutBlockStatement`
- `InStatement` and `OutStatement` would be defined in Statement section of the grammar, under:
```
NonEmptyStatementNoCaseNoDefault:
    ...
    InStatement
    OutStatement

InStatement:
    in Statement
OutStatement:
    out Statement
    out ( Identifier ) Statement
```
### Code Breakage

No code is expected to break.

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

(Will contain comments / requests from language authors once review is complete,
filled out by the DIP manager - can be both inline and linking to external
document.)