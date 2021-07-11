# Shortened Method Syntax

| Field           | Value                                                          |
|-----------------|----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                         |
| Review Count:   | 0 (edited by DIP Manager)                                      |
| Author:         | Max Haughton, Adam Ruppe                                       |
| Implementation: | https://github.com/dlang/dmd/pull/11833                        |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected") |

## Abstract

This DIP unifies the syntax of function literals and function definitions.
The following syntax is proposed: 
```D
int add(int x, int y) pure => x + y;
```

Thanks to a failure of process, this feature is already present inside the implementation of the 
D programming language. This DIP will disambiguate the status of this feature, i.e. after thorough 
consideration a decision can be made and acted upon rather than leaving features in limbo.

## Contents
* [Rationale](#rationale)
* [PriorWork](#prior-work)
* [Description](#description)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale
There already exists [syntactic sugar](https://en.wikipedia.org/wiki/Syntactic_sugar) for function literals in the D programming language, for example
```D
const succ = (int x) => x + 1;
```
is equivalent to
```D
const succ = function(int x) { return x + 1; };
```
No such syntactic sugar exists for functions and method definitions - therefore, a trivial change to the language unifies the syntax of 
those constructs and saves a little typing for the programmer.

## Prior Work
As mentioned previously, this DIP has the potential to result in a post-facto blessing of 
a feature already present (available through a preview flag) in the compiler, this implementation
was written by Adam Ruppe - in recognition of this he is listed as a coauthor.

This feature is present within the C# programming language, where instances of its use are referred to as 
[Expression-bodied members](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/expression-bodied-members).
This feature has been available for use since either C# version 6 or 7 depending on context - for reference the most recent stable release is version 9.0.

The C# documentation linked above gives an example of idiomatic use of this feature (reproduced below). In considering this piece of code, we 
should be careful to remember that C# (to some degree of approximation at least) encourages access to state through methods (Getters and Setters, properties, etc.),
so the use of this feature may be more profitable in C# than in D where the aforementioned idiom is not as popular - that is to say that in the authors experience 
D programs with many single line functions bodies are significantly less prevalent than in C#.
```c#
public class Location
{
   private string locationName;

   public Location(string name)
   {
      locationName = name;
   }

   public string Name => locationName;
}
```
## Description
### Syntax 
The grammar changes are as follows:
```diff
FunctionBody:
     SpecifiedFunctionBody
     MissingFunctionBody
+    ShortenedFunctionBody
...

+ShortenedFunctionBody:
+    => AssignExpression ;
```
### Semantics 
This proposed semantics are a simple example of what is often referred to as lowering:
```
FunctionDeclarator => AssignExpression;
```
shall be rewritten to
```
FunctionDeclarator
{
	return AssignExpression;
}
```
Although valid syntax, this construct is semantically nonsense in the case of a constructor or destructor (i.e. they cannot have a return value) and 
use in these contexts should be explicitly rejected early by the implementation to ensure a meaningful error message for the programmer. 

The crux of the [implementation](https://github.com/dlang/dmd/blob/master/src/dmd/parse.d#L5139) is listed with annotations below:
```D
const returnloc = token.loc;
nextToken();//Walk past the =>
// ↓ the where the lowered result goes     |  ↓ Ingest the expression following the =>
f.fbody = new AST.ReturnStatement(returnloc, parseExpression());
f.endloc = token.loc;
check(TOK.semicolon);
```
Although this listing is informative, the *location* of this implementation within the parser (within `parseContracts`) should not go unquestioned (elaborated in the following section). It should 
also be noted that the above code uses `parseExpression` rather than parsing an assign expression specifically - the author does not know if this matters in practice (or as an exercise).

## Reference
Recently, [it has been raised](https://github.com/dlang/dlang.org/pull/3059) that the implementation currently allows the use of function contracts with 
this syntax. This is not possible with function literals, so it has not been included in the body of this DIP however it is mentioned here both for future 
reference and in case, upon review, it is a desirable feature.

### Trivia 

* [The syntax is mentioned on bugzilla a decade ago](https://issues.dlang.org/show_bug.cgi?id=7176). The dialectics in this thread
  can be considered a pre-review.
* [Shortened Methods are documented in the specification](https://github.com/dlang/dlang.org/pull/2956)
* [A dmd changelog entry is added](https://github.com/dlang/dmd/pull/12241)
## Copyright & License
Copyright (c) 2020 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
