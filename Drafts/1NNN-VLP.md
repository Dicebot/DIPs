# Struct literals with named arguments

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Victor Porton porton@narod.ru
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

Struct literals with named arguments would be useful both to shorten the literals and
to skip default initialized values.

Existing struct initializer syntax is sometimes not convenient, because it requires
a separate statement for each initializer. I want "inline" expressions instead.

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Struct literals with named arguments would be useful both to shorten the literals and
(more importantly) to skip default initialized values. This is especially useful for
structs with many fields to be initialized by default.

It is especially useful to allow to skip any default initialized values in struct
initializers.

## Description

I propose new syntax like:

```d
struct S {
    int x, y, z;
}

S s = S(x: 1, z: 3); // y is default initialized
```

By the way, the same can be done with function calls:

```d
void f(int x, int y, int z) { }

f(x: 1, z: 3);
```

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
