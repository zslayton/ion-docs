# Changes since 2023-08

This document summarizes the changes made to the specification in the past year.

## Macros

### Types

In earlier versions of the specification, macros were able to constrain the Ion types of their parameters and expansions.
The set of supported types was as follows:
```bnf
any-type       ::= tagged-type | tagless-type

tagged-type    ::= abstract-type | concrete-type

tagless-type   ::= primitive-type | macro-ref

concrete-type  ::= 'null' | bool   | timestamp | int  | decimal | float
                          | string | symbol    | blob | clob    | list
                          | sexp   | struct

abstract-type  ::= any | number | exact | text | lob | sequence

primitive-type ::= flex_symbol | flex_string | flex_int | flex_uint | uint8
                               | uint16      | uint32   | uint64    | int8
                               | int16       | int32    | int64     | float16
                               | float32     | float64
```

Specifying a tagged type other than `any` was optional as the encoding would be unchanged;
on the wire, a `string` would be encoded the same way a `number` would be: with a leading opcode.

This set of types has been pared back to only include those types that affect the wire encoding of parameters:
```bnf
any-type       ::= tagged-type | tagless-type

tagged-type    ::= any

tagless-type   ::= primitive-type | macro-ref

primitive-type ::= flex_symbol | flex_string | flex_int | flex_uint | uint8
                               | uint16      | uint32   | uint64    | int8
                               | int16       | int32    | int64     | float16
                               | float32     | float64
```
We now refer to these types as 'encodings' to highlight this distinction.

This set of types/encodings could be easily restored or expanded in a later version of Ion using existing syntax.

### Result specifications

Previously, macros could optionally provide a 'result specification' in their signature to enforce the type and cardinality of values in their expansion.

```bnf
result-spec ::= -> tagged-type tagged-cardinality
```

For example:
```ion
(macro concat [(seq sequence...)] -> sequence! /* body */)
//         result specification --^^^^^^^^^^^^
```

This feature became less powerful with the removal of types and so was removed for simplicity.
As this was already optional, restoring it in a later version of Ion would be straightforward.

### Arguments with multiple expressions

While tagged parameters can always accept an arbitrary number of expressions via the `(:values ...)` macro,
parameters using a tagless encoding do not have that option.
Without an opcode (a 'tag'), primitive encodings like `uint8` would not have a way to encode a macro invocation.

#### Grouped expressions

We previously addressed this issue by creating an 'expression group' syntax that could be used by both tagged and tagless types.
Macro parameters would opt into this via `[]` notation in their declaration.

For example:
```ion
(macro annotate (annotations [text]*) value) -> any
//        indicated a    ----^----^
//        grouped parameter
```
In this signature, the `annotations` parameter was a 'grouped' parameter.
In text, any number of expressions could be passed to `annotations`, but they had to be enclosed in a delimiting `[]` pair.

```ion
(:annotate ["a2"] a1::true) ⇒ a2::a1::true
//         ^----^--- This was an expression group, not a list
```

In binary, baking this into the macro definition meant that no additional bytes needed to be spent at the callsite to communicate
whether the argument was a single expression or a group of expressions--it was guaranteed to be a group.

However, it was not without drawbacks. In particular:
* In text, the usage of a delimiting `[]` pair that was not actually an Ion list was potentially confusing to human readers.
  It also precluded the possibility of passing a single expression because it would be ambiguous whether a `[]` was a list
  or a group.
* In binary, the group required one or more bytes to encode even if it was empty.
* It contributed to an overall sense that parameter declarations were too 'noisy'.

#### Expression Groups

Expression grouping has been removed from parameter declarations.
Passing an expression group as an argument is now opted into at the callsite.

In text, we have introduced a distinct syntax to indicate an argument expression group:
```ion
(:foo (:: 1 2 3))
//    ^^^------^--- Indicates an expression group
```

Expression groups are only legal in argument position, and are supported for any parameter with a cardinality other than `exactly-one`.

In binary, we refer to the argument encoding bitmap at the callsite to determine whether an argument has been encoding as a single expression,
an expression group, or no expression at all.

### Rest parameters

Rest parameter syntax no longer requires explicit opt-in in the macro signature, and the corresponding cardinality modifiers have been removed.

| Modifier | Cardinality                                  |
|:--------:|----------------------------------------------|
|   `!`    | exactly-one value                            |
|   `?`    | zero-or-one value                            |
|   `+`    | one-or-more values                           |
|   `*`    | zero-or-more values                          |
|  `...`   | ~~zero-or-more values, as "rest" arguments~~ |
|  `...+`  | ~~one-or-more values, as "rest" arguments~~  |

Instead, rest syntax is implicitly permitted for any `zero-or-more` or `one-or-more` parameter in tail position.
As before, rest parameter syntax only exists in text Ion.

### Simplified macro signatures

In their most explicit form, macro signatures could become visually dense. For example:

```ion
(macro foo
  [(bar [flex_uint+]), (baz sequence?), (quux any...)] -> sequence!
  [bar, baz, quux])
```

As described prior:
* [Types have been pared back to encodings](#types)
* [Result specifications have been removed](#result-specifications)
* [Grouping is longer part of a parameter's declaration](#arguments-with-multiple-expressions)
* [Rest syntax is now implicit](#rest-parameters)

Collectively, these changes greatly reduce the number of concepts to communicate in the macro signature.

To eliminate a layer of nesting in the signature, encodings are now written as an annotation on the corresponding parameter.
If no annotation is present, the encoding defaults to tagged.
To eliminate the need for delimiting `,`s between declaration elements, the parameter declaration sequence is now an s-expression rather than a list.

Subjectively, these changes make macro signatures simpler to parse for both humans and machines.
Using today's specification, the example macro definition shown above would instead be written as:
```ion
(macro foo
  (flex_uint::bar+ baz? quux*)
  [bar, baz, quux])
```

## Modules

A module is a `(symbol table, macro table)` pair.
The definition and handling of modules has been changed, as detailed in the sections below.

### Encoding directives

#### The encoding environment

Earlier versions of the specification included a construct called the _encoding environment_, which was comprised of:
* A map of names to "available" modules, described below.
* The current symbol table.
* The current macro table.

Each encoding directive was a set of one or more operations that would modify the encoding environment.
An encoding directive could:
* Define a new module, adding it to the available modules map.
* Import a shared module, adding it to the available modules.
* Build up a new symbol table by combining lists of text values with the symbol tables from available modules.
* Build up a new macro table by combining macro definitions with the macro tables from available modules.

Encoding directives were also responsible for enumerating which modules were still needed.
They did this by specifying a `retain` clause which listed the ones to keep in memory.
Modules that were not mentioned in the `retain` clause were removed from the available modules map.
Retaining all modules was done via `(retain *)`, but there was not a definedway to enumerate modules for eviction.

#### The encoding module

The 'encoding environment' has been supplanted by the concept of an 'encoding module'.
Like any other module, the encoding module is a `(symbol table, macro table)` pair.
The available modules map has been removed, and modules are now lexically scoped.
When an encoding directive defines a module or imports a shared one,
that module binding is available only within the body of that directive.

Rather than mutating the encoding environment, an encoding directive now defines an encoding module.
As before, it can build up symbol and macro tables from the modules that in scope.

This approach allows for improved code reuse because the encoding module is now "just another module".
It also eliminates the complexity of the `retain` system.
It does sacrifice the ability to define modules that persist over the lifetime of the stream,
treating the "available modules" map as a form of stream-local catalog construct.
This capability could be added in the future if a use case presents itself.

### Intra-module aliasing

### Tunneled modules removed

## Encoding

### Simplified arguments encoding bitmap

Previously, the [arguments encoding bitmap (AEB)](binary/e_expressions.md#argument-encoding-bitmap-aeb) assigned varying numbers of bits
to each argument according to both its cardinality and its grouping.

Now that [grouping has been removed](#arguments-with-multiple-expressions),
the AEB simply assigns two bits to any parameter with a cardinality other than `exactly-one`.

### Consolidated binary `struct` encodings

In Ion 1.0, struct field names are always encoded as a `VarUInt` symbol ID.
Ion 1.1 introduced the ability to encode one's struct field names using a [`FlexSym`](binary/primitives/flex_sym.md),
an encoding primitive capable of compactly representing either a symbol ID or inline text.

When representing a symbol ID, a `FlexSym` is marginally less compact than a `VarUInt`.
Experiments showed that naively transcoding struct-heavy Ion 1.0 data to Ion 1.1 with `FlexSym` field names would result in a data size increase of 5-7%.

In order to guarantee that users' data would be the same size after migrating to v1.1,
we originally preserved the Ion 1.0 struct encoding alongside the new `FlexSym` encoding.
Applications that wanted the flexibility to write field names as inline symbol text would opt into the somewhat less dense encoding.
However, this came at the cost of [opcode space](binary/opcodes.md).
Supporting both encodings meant that structs occupied 32 of the available 256 opcodes (16 per encoding).
Additionally, it was not easy for a writer to switch encodings in the middle of a struct,
so implementations were incentivized to always use the more flexible encoding anyway.

The specification now defines a single [unified struct encoding](binary/values/struct.md).
All structs begin assuming that they will encode field names as symbol IDs.
If in the course of encoding fields the writer determines that it needs to write a field name as inline text,
it can emit a sentinel symbol ID indicating that [all subsequent field names will be encoded as `FlexSym`s](binary/values/struct.md#optional-flexsym-field-name-encoding).
The sentinel field name is a single byte; if subsequent fields will include inline text, the cost of that byte becomes negligible.
The sentinel field name is symbol ID `$0`; if the writer wishes to encode the actual symbol ID `$0`, it can do so as a `FlexSym`.
The switch from symbol IDs to `FlexSym`s is one-way.

This arrangement preserves Ion 1.0 density where possible and gives writers the ability to opt into the flexible encoding in the middle of a struct.
It also freed up 16 opcodes that are now used to represent macro IDs.

### System address space

TDL introduced a number of keywords that are necessary to define an encoding context.
While they could be encoded as inline text, it would preferable to have an inexpensive way to encode them.
In an Ion 1.0-style encoding context, these keywords would be added to the system symbol table as a permanent prefix.
In Ion 1.1, however, [there are dozens of these keywords](modules/system_module.md#system-symbols)
and it is likely that many of them will not be referenced in any given stream.
This makes the cost of storing them in the active symbol table as a permanent prefix excessive;
user symbols are more likely to require two or more bytes to encode because the most valuable symbol IDs are taken.

Similarly, Ion 1.1 defines a number of system macros that we would like for users to be able to cheaply reference when boostrapping an encoding context,
but we do not want those macros to occupy valuable address space better spent on user data.

To satisfy the dual goals of A) making these constructs cheaply available when constructing an encoding context
and B) not occupying valuable address space with constructs the user doesn't need,
we have made changes to the way the encoding context works.

First: system macros and symbols have been given their own address space.

They are always available and can be referenced in any context.
In binary, the `0xEE` opcode indicates that a one-byte `FixedUInt` system symbol ID follows.
The `0xEF` opcode indicates that a one-byte `FixedUInt` system macro ID follows.
In text, system macros can be accessed by using the syntax `$ion::macro_name`.

Second: following an Ion version marker, the encoding context defaults to the system module.
This means that, like Ion 1.0, all of the constructs needed to bootstrap a custom encoding context are cheaply available at the outset of a stream.
Unlike Ion 1.0, the system constructs are not a permanent prefix.
The user is able to redefine the encoding context, up to and including removing the system macros/symbols.

```ion
$ion_1_1

// Following an IVM, the encoding module is the system module.
// This allows a user to, for example, invoke system macros to define macros
// of their own. System macro `set_macros` expands to an encoding directive
// that clears the macro table and adds the provided macro definitions.
(:set_macros
  (foo (a b c) /*...*/) // 0
  (bar (x)     /*...*/) // 1
  (baz ()      /*...*/) // 2
)
// Following this encoding directive, there are only 3 macro IDs in use: 0, 1, and 2.
// 62 more one-byte macro IDs are available.

// Calling `set_macros` also removed the `set_macros` from the macro table.
// However, we can still invoke it using system macro syntax:
(:$ion::set_macros
  (quux (d e) /*...*/)
  (quuz ()    /*...*/)
)
```

The symbol ID `$0` is always available in the active symbol table (as a permanent prefix, if that suits your mental model).