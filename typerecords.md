# Type Format

An exposed type format is available to programmers. This can also be used internally in engines.

### Deterministic format

TODO: Identical types encode to the same record. (Define this algorithm). Basically expanding all the references to types should create identical records between the same types independent of their order in unions and intersections. Sorting should be somewhat sufficient?

### Union

```js
#{
  union: #[]
}
```

#### Literals

Any literals, including Symbols, can be used in a type.

```js
const T = type 'a' | 'b' | 'c';

#{
  union: #[
    'a'
    'b',
    'c'
  ]
}
```

```js
const T = type 0 | 1 | 1.5;

#{
  union: #[
    0,
    1,
    1.5
  ]
}
```

Numerical literals have no inherent type, so an intersection can be used to constrain them:

```js
const T = type float32 & (0 | 1 | 1.5);

#{
  intersection: #[
    float32,
    #{
      union: #[
        0,
        1,
        1.5
      ]
    }
  ]
}
```

This also handles tagged unions:

```js
const T = type
  | { kind: 'success', data: string }
  | { kind: 'error', message: string };

#{
  union: #[
    #{
      properties: #[
        #{ name: 'kind', type: 'success' },
        #{ name: 'data', type: string }
      ]
    },
    #{
      properties: #[
        #{ name: 'kind', type: 'error' },
        #{ name: 'message', type: string }
      ]
    }
  ]
}
```

### Intersection 
```js
#{
  intersection: #[]
]
```

### function type

Functions have a signature defined by a parameters record and a return type.

```js
const T = type function(x: number): string { return x.toString(); }

#{
  parameters: #{
    x: number
  },
  return: string
}
```

This use of a record means that these two functions have the same signature and the second would produce a TypeError:

```js
function f(x: number, y: string): void {}
function f(y: string, x: number): void {}
```

When using named parameters ```f(x: 0, y: 'abc')``` such calls would have been ambiguous also.

#### Optional parameter

```js
function f(x?: boolean): void {}
//function f(x: boolean = true): void {} // Identical signature

const T = type f;

#{
  parameters: #{
    x: #{
      type: boolean,
      optional: true
    }
  },
  return: void
}
```

(Note: ```optional``` is used because expanding these to unique signatures would mean a function with 8 optional parameters would have 256 signatures).

#### Overloaded functions

Overloaded functions are interesting because their type record can be quite massive, especially generic functions and decorators.

```js
function f(x: string): number {}
function f(x: number): string {}
function f(x: string, y: boolean): number {}

const T = type f;

#{
  union: #[
    #{
      parameters: #{
        x: string,
        y: #{
          type: boolean,
          optional: true
        }
      },
      return: number
    },
    #{
      parameters: #{
        x: number
      },
      return: string
    }
  ]
}
```

Note, I don't like this setup using a tuple. I would much rather use a set if they were added as the order of signatures doesn't matter.

#### Generic function

Tentatively all generic parameters are included in the parameters.

```js
function complex<T extends number, U extends Array<T>, V>(
  x: U,
  y: (t: T) => V,
  z: Map<V, T>
): U { ... }

#{
  parameters: #{
    T: #{
      type: type,
      constraint: number
    },
    U: #{
      type: type,
      constraint: #{
        type: Array,
        parameters: #[
          #{ parameter: 'T' }
        ]
      }
    },
    V: #{
      type: type
    },
    x: #{
      type: #{ parameter: 'U' }
    },
    y: #{
      type: #{
        parameters: #[
          #{
            name: 't',
            type: #{ parameter: 'T' }
          }
        ],
        return: #{ parameter: 'V' }
      }
    },
    z: #{
      type: #{
        type: Map,
        parameters: #{
          K: #{ parameter: 'V' },
          V: #{ parameter: 'T' }
        }
      }
    }
  ],
  return: #{ parameter: 'U' }
}
```

function f(x: number, y: string) {}
function f(y: string, x: number) {}

Is there any edge case where a parameter needs to be marked explicit/implicit?

### class type

A class 

```js
const T = type interface {
  x: number;
  f: (value: number, ...foo: [].<number>) => boolean;
  g: Generator<...>;
};

#{
  properties: #[
    #{
      name: 'x',
      type: number,
      public: true,
      private: false,
      static: false
    },
    #{
      name: 'f',
      parameters: #{
        value: number,
        foo: #{
          type: [].<number>,
          rest: true
        }
      ],
      public: true,
      private: false,
      static: false
    },
    #{
      name: 'g',
      type: Generator<...>,
      public: true,
      private: false,
      static: false
    }
  ]
}
```

As mentioned in the spec, async types are just a Promise<T, E>.
TODO: Include example

#### optional properties

```js
#{
  properties: #[
    #{
      name: 'x',
      type: #{
        union: #[
          number,
          undefined
        ]
      }
    }
  ]
}
```

#### Generic class

```js
class Pair<T, U> {
  first: T;
  second: U;
  swap(): Pair<U, T> {
    return new Pair(this.second, this.first);
  }
}

const T = type Pair;

#{
  parameters: #{
    T: type,
    U: type
  ],
  properties: #[
    #{
      name: 'first',
      type: #{ parameter: 'T' }
    },
    #{
      name: 'second',
      type: #{ parameter: 'U' }
    },
    #{
      name: 'swap',
      type: #{
        parameters: #{},
        return: #{
          type: Pair,
          parameters: #{
            T: #{ parameter: 'U' }
            U: #{ parameter: 'T' }
          }
        }
      }
    }
  ]
}
```

### enum type

```js
enum Count { Zero, One, Two }

#{
  values: #{
    Zero: 0,
    One: 1,
    Two: 2
  }
}
```

```js
enum Count: int32 { Zero, One, Two }

#{
  intersection: #[
    int32,
    #{
      values: #{
        Zero: 0,
        One: 1,
        Two: 2
      }
    }
  ]
}
```

```js
enum Count: float32 { Zero = 0, One = 100, Two = 200 }

#{
  intersection: #[
    float32,
    #{
      values: #{
        Zero: 0,
        One: 100,
        Two: 200
      }
    }
  ]
}
```

```js
enum Count: string { Zero = 'Zero', One = 'One', Two = 'Two', Three = 'Three' }

#{
  intersection: #[
    string
    #{
      values: #{
        Zero: 'Zero',
        One: 'One',
        Two: 'Two',
        Three: 'Three'
      }
    }
  ]
}
```

```js
enum Flags: uint32 { None = 0, Flag1 = 1, Flag2 = 2, Flag3 = 4 }

#{
  intersection: #[
    uint32,
    #{
      values: #{
        None: 0,
        Flag1: 1,
        Flag2: 2,
        Flag3: 4
      }
    }
  ]
}
```

### Recursive types


### Record operators

#### keyof

returns a type with all the property keys

```js
keyof T;

#{
  union: #[
    'a',
    'b',
    'f'
  ]
}
```

#### Get property type by property name

```js
TClass[propertyName]
```

#### Get parameter type by parameter name

```js
TMethod[parameterName]
```

Note: This works for generic parameters also

### Notes

I don't have a 'kind' applied to records. Should functions, classes, etc have a kind? Often their properties infer their kind. Is this sufficient?

Operators to check extends true or false between two type records?

If you add a new overload to a function dynamically, then previous type records would no longer match the new one. In practice what problems would this cause?

## Other Notes

### Getting parameter order or other metadata about functions?

I'm thinking that there would be a ```type.info``` operator that returns a more verbose reflection of the current type definition. This would include all the overloads including their default values or references to their initializers. This would not be a record. It could also include serialization information.
