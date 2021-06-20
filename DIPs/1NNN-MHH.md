# Introduce `__UDA_ATTACHED_TO_DECLS__`

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Max Haughton mh2410@[universityofbathdomain]                    |
| Implementation: | github.com/maxhaton/dmd  (too buggy for a PR as of writing)                          |
| Status:         | Draft  |

## Abstract
D's user-defined attributes are already great, this DIP simple change to make them better: Let the UDA see sideways, that is, let it see what it is attached to. Letting them do this lets them eliminate many tasks the preprocessor can do that D cannot. This is achieved by the introduction of a new  `SpecialKeyword` "\_\_UDA_ATTACHED_TO_DECLS\_\_" to the grammar. When used as a default initializer this shall be resolved to a `string[]` containing the fully qualified names of the declarations, if any, the expression's parent UDA declaration is attached to.

Note that the exact name (i.e. "UDA_ATTACHED_TO_DECLS") is easily changed.
## Contents
- [Let UDAs see what they are attached to](#)
  - [Abstract](#abstract)
  - [Contents](#contents)
  - [Rationale](#rationale)
  - [Prior Work](#prior-work)
  - [Description](#description)
  - [Reference](#reference)
  - [Copyright & License](#copyright--license)
  - [Reviews](#reviews)

## Rationale
For a UDA to do work in D, it currently must be accessed via the scope in which the declarations it is attached to are declared. For most uses this is ideal: for example when serializing a data structure using a UDA is an efficient and maintainable way of specifying the semantics of how the given data structure is to be serialized. 
### A simple example of a UDA - Serializing an exam result
```D
struct Serialize {
    string impl;
	this(string forWhom)
    {
    	impl = forWhom;
    }
}
struct ExamResult {
    //The student's name is needed in all reports
    @Serialize("ParentReport") @Serialize("InternalReport") 
    string studentName;
    //The examiner's name is only needed for internal moderation
    @Serialize("InternalReport") 
    string examinerName;
    //Same as the first
    @Serialize("ParentReport") @Serialize("InternalReport") 
    float overallScore;
    //Don't serialize this
    string dbEntry;
}
```

Clearly in this case there is a strictly top-down topology to the usage of these user-defined attributes, whenever this structure is serialized D's metaprogramming facilities makes it trivial to *iterate* (emphasis added for later relevance) over `ExamResult`'s members and look for any attributes relevant to our little library.
### A less simple example
There are, however, situations where a UDA is highly useful but there does not exist as clean of a hierarchy as in the previous example.

For example, an attribute will often be attached to a - in effect - free standing symbol like a `unittest`. 
```D
enum runMyTest;

@runMyTest unittest {

}
```
Since most of these declarations are at or near module scope, we can simply iterate over *all* the `unittest`s looking for one's matching our UDA. Although not technically required, it is still common to find these tests in a pre-compilation step - or by manually setting the test runner to work using a `mixin` statement.

This pattern is fine for `unittest`s: When used, there are usually many of them, and when there are not many the potential for a performance hit is mitigated by the compiler providing a list of `unittest`s within a given scope.

### The behaviour to be enabled by this DIP.
There are, however, scenarios where this pattern is less than ideal. Some UDAs maybe highly expressive and yet both sparse and not applied to `unittest`s, but the cost of using them may add up in a large D program.

If we setup a mechanism to get these UDA-ed symbols - assuming we know where they are i.e. a recursive search is currently not possible but scopes containing UDAs can be declared using a different UDA - we still have to iterate over *every* declaration in a given scope looking for those with our desired set of attributes. For a sparse declaration - that is, if we are looking for a few declarations out of a several thousand line file - this is not an efficient way of doing things.

For example, this DIP is motivated specifically by a desire to be able to declare benchmarks in the following manner (without needing the user to `mixin` anything):

```D
@Benchmark!(SomeInformationAboutTheBenchmark)
float dotp(float[3] vec1, float[3] vec2) pure
{
    return vec1[0] * vec2[0] + vec1[1] * vec2[1] + vec1[2] * vec2[2] 
}
//The library can then do what it wants with dotp, be that schedule it to run or add it to a table at compile time
```

This DIP proposes a simple solution to this problem: Let the UDA see sideways, that is, let it see what it is attached to. Rather than being some paradigm shift in how UDAs are used, this should simply be viewed as having the same effect of adding syntactic sugar for the following construct:
```D
    module home;
    enum MyUDA;
    template HandleTheUDA(alias handleThis) {/* impl */}
    @MyUDA void widget() {}

    mixin HandleTheUDA!handleThis;
    //^A mixin per symbol is ugly
    
    mixin HandleTheUDA!home;
    //^A searching mixin has the potential to be very slow; we don't want to search the entire module just for one UDA
```
i.e. This DIP enables the library writer to create a UDA that can manage the per-symbol `mixin` itself. In translating our example to this new idiom *MyUDA* would go from being an `enum` to a template as the code contained within *HandleTheUDA* is now done at the UDA instead. When the library is consumed, however, the API would be the same due to the use of default parameters (other than the lack of `mixin`s).

This does not add any new dragons or side effects beyond what the original construct could do.
## Prior Work
### A pattern that already does the job
The following pattern can be used within a UDA to find what said UDA is attached to.
```D
//Module is needed for obvious reasons, the line parameter makes it unique for most uses
template FinderUDA(string name, string m = __MODULE__, int l = __LINE__)
{
	import std.traits;
    enum FinderUDA;
    mixin("alias mod = " ~ m ~ ";");
    //Your implementation goes here
    pragma(msg, getSymbolsByUDA!(mod, FinderUDA));
}

@FinderUDA!"Hello" 
void wow() { }
```
In this case it prints "tuple(wow)".

This pattern has obvious flaws: Unless told where to look recursively, it can only work with symbols at module scope, the attribute declaration/s must be on separate lines to be uniquely identified, and more subtly in a big file we now have to search through every declaration that the UDA *could* be attached to. The compiler already has this information, let's use it.

### How the patterns enabled by this DIP are done in C++
Thanks to the simplicity of textual preprocessing, C++ can enable this pattern using a macro. However, this is extremely error-prone so a common way of achieving everything mentioned so far in this article is to have a separate compilation step to collect and use information from the source code.

For example, Epic Games' Unreal Engine uses the following syntax to interface between C++ and "Blueprints" 
```C++
class AGameActor : public AActor
{
  GENERATED_BODY()
  public:
  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Properties)
  FString Name;
  UFUNCTION(BlueprintCallable, Category = Properties)
  FString ToString();
};
```
The game then inserts hijacks the compile to read the (in our terms) user-defined attributes from the header file, however this not only hurts compilation times but is also fairly antithetical to the "D way" of doing things.
## Description
This DIP simply proposes adding to the language a new *SpecialKeyword* `__UDA_ATTACHED_TO_DECLS__` to the grammar. 
```diff
SpecialKeyword:
    __FILE__
    __FILE_FULL_PATH__
    __MODULE__
    __LINE__
    __FUNCTION__
    __PRETTY_FUNCTION__
+   __UDA_ATTACHED_TO_DECLS__
```
It will always resolve to a string array literal - A user-defined attribute may be attached to one *or more* declarations, necessitating the use of `string[]` rather than `string` (See examples below).
```D
void main(string[] args)
{
    const attr = __UDA_ATTACHED_TO_DECLS__;
    pragma(msg, typeof(attr), attr); //"const(string[]) and []"
}
```
This literal shall be empty unless `__UDA_ATTACHED_TO_DECLS__` is used as a default initializer, in which case it shall be resolved to either to be either empty or an array literal of the fully qualified names of all declarations a UDA is attached to, if and only if (subject to existing default initializer resolution behaviour) it is resolved to an expression within said *UserDefinedAttribute*

### Some specific examples:
A templated struct
```D
module testmodule;
struct Test(string name, string[] attr = __UDA_ATTACHED_TO_DECLS__)
{
    pragma(msg, name, " says: ", attr);
    this(int l) {/*Do work*/}  
}
@Test!"name"(1) 
int echo(int x)
{
    return x;
}
//"name says: ["testmodule.echo"]"
```
A simple function
```D
module testmodule;
auto just(string[] at = __UDA_ATTACHED_TO_DECLS__)
{
    return at;
}

@(just())
int x, y, z;

pragma(msg, pragma(msg, __traits(getAttributes, x)[0]));
//"["testmodule.x", "testmodule.y", "testmodule.z"]"
```
A class 
```D
class wow {
    string[] cont;
    this(string[] attr = __UDA_ATTACHED_TO_DECLS__)
    {
        cont = attr;
    }
}
@(new wow)
int cheese;
pragma(msg, __traits(getAttributes, cheese)[0]);
//"wow(["testmodule.cheese"])"
```
Finally, eliminating a `mixin`.
```D
template runThisFunction(string name, string[] attrs = __UDA_ATTACHED_TO_DECLS__)
{
    pragma(msg, "DRT", attrs);
    enum runThisFunction;

    shared static this()
    {
        import std.stdio;
        static foreach(at; attrs) {
            {
                mixin("alias theFunc = " ~ at ~ ";");
                writef!"%s is running %s"(name, at);
                theFunc();
            }
        }  
        
    }
}
@(runThisFunction!"Darth Vader")
void runMe()
{
    import std.stdio;
    writeln("dlang");
}
//Eliminates having to write @runThisFunction then using a mixin to actually do the work
```
## Reference
[The Unreal Engine property system](https://www.unrealengine.com/en-US/blog/unreal-property-system-reflection)


## Copyright & License
Copyright (c) 2020 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
