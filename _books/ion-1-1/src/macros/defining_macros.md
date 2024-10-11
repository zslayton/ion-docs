## Defining macros

A macro is defined using a `macro` clause within a [module](../modules.md)'s [`macro_table` clause](../modules.md#macro_table).

### Syntax
```ion
(macro name signature template)
```

| Argument                                        | Description                                                                                               |
|-------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| [`name`](#macro-names)                          | A unique name assigned to the macro or--to construct an anonymous macro--`null`.                          |
| [`signature`](#macro-signatures)                | An s-expression enumerating the parameters this macro accepts.                                            |
| [`template`](#template-definition-language-tdl) | A template definition language (TDL) expression that can be evaluated to produce zero or more Ion values. |

#### Example macro clause
```ion
//      ┌─── name
//      │     ┌─── signature
//     ┌┴┐ ┌──┴──┐
(macro foo (x y z)
  {           // ─┐
    x: (%x),  //  │
    y: (%y),  //  ├─ template
    z: (%z),  //  │
  }           // ─┘
)
```

### Macro names

Syntactically, macro names are [identifiers](../modules.md#identifiers). Each macro name in a macro table must be unique.

In some circumstances, it may not make sense to name a macro. (For example, when the macro is generated automatically.) In such cases, authors may set the macro name to `null` or `null.symbol` to indicate that the macro does not have a name. Anonymous macros can only be referenced by their address in the macro table.

### Macro Parameters

A _parameter_ is a named stream of Ion values. The stream's contents are determined by the macro's invocation.
A macro's parameters are declared in the [macro signature](#macro-signatures).

Each parameter declaration is comprised of three elements:
1. A [name](#parameter-names)
2. An optional [encoding](#parameter-encodings)
3. An optional [cardinality](#parameter-cardinalities)

#### Example parameter declaration
```ion
//     ┌─── encoding
//     │      ┌─── name
//     │      │┌─── cardinality
// ┌───┴───┐  ││
   flex_uint::x*
```

#### Parameter names

A parameter's name is an [identifier](../modules.md#identifiers). The name is required; any non-identifier (including `null`, quoted symbols, `$0`, or a non-symbol) found in parameter-name position will cause the reader to raise an error.

All of a macro's parameters must have unique names.

#### Parameter encodings

In binary Ion, the default encoding for all parameters is _tagged_. Each argument passed into the macro from the callsite is prefixed by an [opcode](../binary/opcodes.md) (or "tag") that indicates the argument's type and length.

Parameters may choose to specify an alternative encoding to make the corresponding arguments' binary representation more compact and/or fixed width. These "tagless" encodings do not begin with an opcode, an arrangement which saves space but also limits the domain of values they can each represent. Arguments passed to tagless parameters cannot be `null`, cannot be annotated, and may have additional range restrictions.

To specify an encoding, the [parameter name](#parameter-names) is annotated with one of the following tokens:

| Tagless encodings                    | Description                                                       |
|--------------------------------------|-------------------------------------------------------------------|
| `flex_int`                           | Variable-width, signed int                                        |
| `flex_uint`                          | Variable-width, unsigned int                                      |
| `int8`  `int16`   `int32`   `int64`  | Fixed-width, signed int                                           |
| `uint8` `uint16`  `uint32`  `uint64` | Fixed-width, unsigned int                                         |
| `float16` `float32` `float64`        | Fixed-width float                                                 |
| `flex_symbol`                        | [`FlexSym`](../binary/primitives/flex_sym.md)-encoded SID or text |

When writing text Ion, the declared encoding does not affect how values are serialized.
However, it does constrain the domain of values that that parameter will accept.
When transcribing from text to binary, it must be possible to serialize all values passed as an argument using the parameter's declared encoding.
This means that parameters with a primitive encoding cannot be annotated or a `null` of any type.
If an `int` or a `float` is being passed to a parameter with a fixed-width encoding,
that value must fit within the range of values that can be represented by that width.
For example, the value `256` cannot be passed to a parameter with an encoding of `uint8` because a `uint8` can only represent values in the range `[0, 255]`.

#### Parameter cardinalities

A parameter name may optionally be followed by a _cardinality modifier_. This is a sigil that indicates how many values the parameter expects the corresponding argument expression to produce when it is evaluated.

| Modifier |         Cardinality |
|:--------:|--------------------:|
|   `?`    |   zero-or-one value |
|   `*`    | zero-or-more values |
|   `!`    |   exactly-one value |
|   `+`    |  one-or-more values |

If no modifier is specified, the parameter's cardinality will default to exactly-one.
An `exactly-one` parameter will always expand to a stream containing a single value.

Parameters with a cardinality other than `exactly-one` are called _variadic parameters_.

If an argument expression expands to a number of values that the cardinality forbids, the reader must raise an error.

##### Optional parameters

Parameters with a cardinality that can accept an empty expression group as an argument (`?` and `*`) are called
_optional parameters_. In text Ion, their corresponding arguments can be elided from e-expressions and TDL macro
invocations when they appear in tail position. When an argument is elided, it is treated as though an explicit
empty group `(::)` had been passed in its place.

In contrast, parameters with a cardinality that cannot accept an empty group (`!` and `+`) are called _required
parameters_. Required parameters can never be elided.

```ion
(:set_macros
    (foo (x y? z*) // `x` is required, `y` and `z` are optional
        [x, y, z]
    )
)

// `z` is a populated expression group
(:foo 1 2 (:: 3 4 5)) => [1, 2, 3, 4, 5]

// `z` is an empty expression group
(:foo 1 2 (::))       => [1, 2]

// `z` has been elided
(:foo 1 2)            => [1, 2]

// `y` and `z` have been elided
(:foo 1)              => [1]

// `x` cannot be elided
(:foo)                => ERROR: missing required argument `x`
```

Optional parameters that are _not_ in tail position cannot be elided, as this would
cause them to appear in a position corresponding to a different argument.

```ion
(:set_macros
    (foo (x? y) // `x` is optional, `y` is required
        [x, y]
    )
)

(:foo (::) 1) => [(::), 1] => [1]
(:foo 1)                   => ERROR: missing required argument `y`
```

### Macro signatures

A macro's _signature_ is the ordered sequence of parameters which an invocation of that macro must define.
Syntactically, the signature is an s-expression of [parameter declarations](#macro-parameters).

#### Example macro signature
```ion
(w flex_uint::x* float16::y? z+)
```

| Name |  Encoding   |  Cardinality   |
|:----:|:-----------:|:--------------:|
| `w`  |  `tagged`   | `exactly-one`  |
| `x`  | `flex_uint` | `zero-or-more` |
| `y`  |  `float16`  | `zero-or-one`  |
| `z`  |  `tagged`   | `one-or-more`  |

<!-- TODO; grammar and examples -->

### Template definition language (TDL)

<!-- TODO -->