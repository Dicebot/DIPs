# Aliases of member functions in template arguments

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Victor Porton porton@narod.ru                                   |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

Member functions should be passeable as alias template arguments.

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

"Regular" aliases can be assigned to a member function like this:

```d
struct S {
  void f() { }
}
S s;
alias f = s.f;
```

But it does not work for template arguments:

```d
import std.stdio, std.meta;

struct S {
    void f() { }
}

void t(alias f)() { f(); writeln ("hello world"); }
void main()
{
    S s;
    t!(s.f); // does not compile
}
```

## Description

I propose to allow member functions as template alias arguments.

The benefits of this change are obvious.

## Copyright & License

Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
