# Type Records

An exposed type record is available to programmers. This can also be used internally in engines.

### Deterministic format

TODO: Identical types should encode to the same record. (Define this algorithm). Basically expanding all the references to types should create identical records between the same types independent of their order in unions and intersections. Sorting should be somewhat sufficient?

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

Functions have a signature defined by a parameter tuple and a return type.

```js
const T = type function(x: number): string { return x.toString(); }

#{
  parameters: #[
    #{
      name: 'x',
      type: number
    }
  ],
  return: string
}
```

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
      parameters: #[#{ name: 'x', type: string }],
      return: number
    },
    #{
      parameters: #[#{ name: 'x', type: number }],
      return: string
    },
    #{
      parameters: #[
        #{ name: 'x', type: string },
        #{ name: 'y', type: boolean }
      ],
      return: number
    }
  ]
}
```

#### Generic function

Tentatively all generic parameters are included in the parameters.

```js
function complex<T extends number, U extends Array<T>, V>(
  x: U,
  y: (t: T) => V,
  z: Map<V, T>
): U { ... }

#{
  parameters: #[
    #{ name: 'T', type: type, constraint: number },
    #{ name: 'U', type: type, constraint: Array<T> },
    #{ name: 'V', type: type },
    #{ name: 'x', type: U },
    #{ name: 'y', type: type (T) => V },
    #{ name: 'z', type: type Map<V, T> }
  ],
  return: U
}

// Expanding the types further yields:

#{
  parameters: #[
    #{ name: 'T', type: type, constraint: number},
    #{ name: 'U', type: type, constraint: #{
      type: Array,
      parameters: #[T]
    } },
    #{ name: 'V', type: type},
    #{ name: 'x', type: U},
    #{ name: 'y', type: #{
      parameters: #[#{type: T}],
      return: V
    } },
    #{ name: 'z', type: #{
      type: Map,
      parameters: #[V, T]
    } }
  ],
  return: U
}
```

I'm unsure how the generic parameter syntax should look. You'll notice I'm using T and U as variables here. They could be ```#{ parameter: 'T' }```?

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
      parameters: #[
        #{
          name: 'value',
          type: number
        },
        #{
          name: 'value',
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
  parameters: #[
    #{name: 'T', type: type},
    #{name: 'U', type: type}
  ],
  properties: #[
    #{name: 'first', type: T},
    #{name: 'second', type: U},
    #{
      name: 'swap',
      type: #{
        parameters: #[],
        return: #{
          type: Pair,
          parameters: #[U, T]
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
