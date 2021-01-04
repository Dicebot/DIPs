# Static arrays with automatically computed length

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Lucien Perregaux                                                |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

Allow `$` in the brackets of array declarations in place of integer literals.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

It is sometimes necessary to modify the contents of a static array initializer, for example,

```d
int[2] array = [2, 4];
```

may at some point be changed to

```d
int[3] array = [2, 4, 6];
```

The programmer must remember to also update the integer literal in the static array declaration, else face a compiler error.

The goal of this proposal is to eliminate the need to change the length specified in a static array declaration, with minimal impact on compile time, when modifying
the length of the array's initializer.

Allowing `$` in array declaration brackets will save time and potential compiler errors,
as the programmer will no longer need to change the integer literal in the array declaration.
This is especially beneficial when using `betterC`, where dynamic arrays are not available.

Conceptually, this new operator, `[$]`, documents that the programmer does not care about the array's length,
but only that the array is a static array and not a slice.

No control-flow analysis is needed for this feature.

## Prior Work

### C/C++

In C/C++, we can declare a static array like this:
```c
int arr[] = {1, 2, 3};
```
In D, it would become:
```d
int[] arr = [1, 2, 3];
```
which is a slice, and not a static array.

See the [documentation](https://en.cppreference.com/w/c/language/array_initialization).

### D
- The `$` operator, which is well implemented.
- [PR 3615](https://github.com/dlang/dmd/pull/3615)
  - Reverted due to partial deduction [PR 4373](https://github.com/dlang/dmd/pull/4373)
- [PR 590](https://github.com/dlang/dlang.org/pull/590)
- [Issue 14070](https://issues.dlang.org/show_bug.cgi?id=14070)
- [Issue 8008](https://issues.dlang.org/show_bug.cgi?id=8008)

## Description

This feature is compatible with `betterC` and is only available for variable declarations with initializers.
The `$` in the array declaration's brackets will be replaced, at compile time, with the array's length.

Currently, a static array must be declared like this:
```d
uint[2] arr = [ 2, 4 ];
```
When doing this, we have a DRY violation because we are spelling out the array's length
even though the compiler knows the length of the array literal used to initialize it.

With the proposed feature, the above array could be declared like this:
```d
uint[$] arr = [ 2, 4 ];    // $ = 2
```
There is no need to change the length of the static array every time we modify the length of its initializer.
Eliminating that requirement is the only goal of this proposal.

More examples:
```d
char[$] a1 = "cerise";              // OK, `$` is replaced with `6` at compile time

const int[] d = [1, 2, 3];
int[$] a2 = d;                      // OK, `$` = 3

auto a3 = cast(int[$]) [1, 2, 3];   // OK, `a3` is `int[3]` (same behavior as `cast(int[3])`)

int[$][] a4 = [[1, 2], [3, 4]];     // OK, same as `int[2][]`
foreach (int[$] a; a4) {}           // OK, we can deduce `$` because the length of `a4[0]` is equal to 2.

void foo(int[$] a5 = [1,2]);        // OK, `a5` has a default value, so we can deduce it's length

int[5] bar();
int[$] a6 = bar();                  // OK, `bar` returns a static array.
```

Those examples won't work:
```d
int[$] b1;                          // Error: we don't know the length of `b1` at compile-time.

int[$][] = [[1, 2],[1, 2, 3]];      // Error: cannot implicitly convert expression `[[1, 2], [1, 2, 3]]` of
                                    // type `int[][]` to `int[2][]`

/*
 * `$` is equal to `3`, because it's the length of `b2[0]`.
 * As `b2[1]`'s length is 4, a RangeViolation error is thrown.
 */
int[][] b2 = [[1, 2, 3], [1, 2, 3]];
foreach (int[$] b; b2) {}           // Error: we don't know the length of `b` at compile-time.

void foo(int[$] b3);                // Error: we don't know the length of `b3` at compile-time.

int[$] bar(int[2] arr)              // Error: not allowed in functions declarations
{
    return arr ~ [3, 4];
}

int baz(int[$] b5)(string s);       // Error: not allowed in templates declarations

int[] d = [1, 2];
int[$] b6 = d;                      // Error, we don't know the length of `d` at compile-time.
```

### Grammar changes

[declaration spec](https://dlang.org/spec/declaration.html)
```diff
TypeSuffix:
        *
        [ ]
+       [ $ ]
        [ AssignExpression ]
        [ AssignExpression .. AssignExpression ]
        [ Type ]
        delegate Parameters MemberFunctionAttributes
        delegate Parameters
        function Parameters FunctionAttributes
        function Parameters


2. Arrays read right to left as well:
 int[3] x;     // x is an array of 3 ints
 int[3][5] x;  // x is an array of 5 arrays of 3 ints
 int[3]*[5] x; // x is an array of 5 pointers to arrays of 3 ints
+int[$] x = [ 0, 1 ]; // x is a static array of 2 ints
+int[$][] x = [[0, 2], [1, 3]]; // x is a dynamic array of static arrays with a length of 2
```

## Reference


## Copyright & License
Copyright (c) 2020 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.