# New proposed syntax for mixins

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Victor Porton <porton@narod.ru>                                              |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

The programmer sometimes needs to instantiate mixin templates with the same arguments repeatedly, like:

```
mixin template A(int x, float y) {
  // ...
}

mixin template B(int x, float y) {
  // ...
}

I propose to shorten the above code with the following syntax:

template composite(int x, float y) {
  mixin A {
    // ...
  }
  mixin B {
    // ...
  }
}

// ...

alias ourMixins = composite(1, 2.0)

struct Test {
  // put two mixins into Test
  ourMixins.A;
  ourMixins.B;
}
```

### Reference

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Don't repeat (as in the first example above) the template arguments multiple times.
It may shorten code, and more importante lessen the errors, as when arguments change,
we would need to change only in one place of the code, not say two times (for an
example above).

## Description

Allow to define "plain" (non-templated) mixins inside templates. (BTW, we can also allow to define mixins not in a template at all;
it could be useful, too.)

```
template composite(int x, float y) {
  mixin A {
    // ...
  }
  mixin B {
    // ...
  }
}
```

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
