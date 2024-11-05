# Encoding modules

A stream's _encoding modules_ are an ordered sequence of modules whose symbols and macros are being used to read or write the current stream segment.

Available macros can be referenced by their qualified name (`module_name::foo`) or qualified address (`module_name::17`),
indicating in which of the encoding modules to begin resolution.

> [!TIP]
> Modules are often maintained and vended by different parties, making it likely that the same macro names will be used in multiple modules.
> Qualified name references allow modules to be developed in isolation and still coexist in the same stream without issue.

They can also be referenced via an unqualified address (`17`).
In this case, resolution is performed by viewing the encoding modules as forming a single concatenated macro table.
Note that the order of the modules is significant.

Consider the following encoding modules and their macro tables:
```ion
// == Module A ==
  foo // 0
  bar // 1
  baz // 2

// == Module B ==
  quux // 0
  quuz // 1

// == Module C ==
  bi   // 0
  bim  // 1
  bop  // 2
  foo  // 3
  quux // 4
```

Here are some examples of how unqualified addresses are resolved:

| Encoding modules | Unqualified address | Qualified name   |
|:----------------:|:-------------------:|------------------|
|     `A B C`      |         `1`         | `B::1`           |
|                  |         `3`         | `B::quux`        |
|                  |         `5`         | `C::bi`          |
|                  |         `9`         | `C::quux`        |
|     `C A B`      |         `1`         | `C::bim`         |
|                  |         `3`         | `C::foo`         |
|                  |         `5`         | `A::foo`         |
|                  |         `9`         | `B::quuz`        |
|      `C A`       |         `1`         | `C::bim`         |
|                  |         `3`         | `C::foo`         |
|                  |         `5`         | `A::foo`         |
|                  |         `9`         | _&lt;undefined>_ |

> [!NOTE]
> The 'concatenated' macro table view does not support resolving unqualified _names_,
> as it is possible for two modules to contain macros with the same name.
> To use unqualified names, see the _[default module](#default-module)_ section.

Symbols can only be referenced by their unqualified address,
which is resolved by viewing the encoding modules as forming a concatenated symbol table.

# Directives

_Directives_ define the encoding context that will be used to read and write the next segment of the stream.

Syntactically, a directive is a top-level s-expression annotated with `$ion`.
Its first child value is an operation name.
The operation determines what changes will be made to the encoding context and which clauses may legally follow.

```ion
$ion::
(operation_name
    (clause_a /*...*/)
    (clause_b /*...*/)
    (clause_c /*...*/))
```

In Ion v1.1, there are three supported directive operations:
1. [`module`](#module)
2. [`import`](#import)
3. [`encoding`](#encoding)

## `module`

```ion
$ion::
(module foo
    /*...submodules, if any...*/
    (macro_table /*...*/)
    (symbol_table /*...*/)
)
```

The `module` directive defines a new module with a binding whose scope is _the stream itself_.
This means that once created, module bindings at this level endure until the file ends or another Ion version marker is encountered.
While module bindings cannot be deleted, they can be redefined.

> [!IMPORTANT]
> **Users cannot create a binding whose name begins with `$`.**
> These are reserved for the system use.
> ```ion
> (import $ion /*...*/)      // ERROR
> (module $foo /*...*/)      // ERROR
> ```

## `import`

```ion
$ion::
(import
    bar               // Binding
    "com.example.bar" // Module name
    2)                // Module version
```

The _import_ directive looks up the module corresponding to the given `(name, value)` pair in the catalog.
Upon success, it creates a new binding to that module.
The binding endures until the file ends or another Ion version marker is encountered.

## `encoding`

The _encoding_ directive sets the [encoding modules](#encoding-modules) to the given sequence of module bindings.

```ion
$ion::
(encoding mod_a mod_b mod_c)
```

The new encoding module sequence goes into effect immediately after the directive
and remains the same until the next `encoding` directive or Ion version marker.

# Default module

While the [encoding module sequence](#encoding-modules) allows a writer to encode the stream using a variety of modules,
many streams will only use a few locally defined macros.
In such streams, naming a new module and adding that binding to the encoding module sequence is burdensome.
It also forces all named invocations of that module's macros to be qualified even though the writer knows there are no name collisions.

To simplify this use case, an empty stream-level module named `_` is available at the beginning of every stream.
Macros and symbols can be added to it by redefining `_`.
Like all modules, `_` can be redefined in terms of itself, making appends and prepends straightforward.

```ion
$ion_1_1

// `_` exists, but is empty

$ion::
(module _
    (macro_table
        (macro foo () /*...*/)))

// `_` now contains macro `foo`

$ion::
(module _
    (macro_table
        _ // Add all macros in `_` to its redefinition
        (macro bar () /*...*/)))

// `_` now contains macros `foo` and `bar`
```

In e-expressions, unqualified macro name references are always resolved in the default module.

```ion
// This...
(:foo 1 2 3)

//...is the same as this:
(_::foo 1 2 3)
```

> ZS: There's room to design this bit below further.

System macros like `add_symbols` and `add_macros` apply their changes to `_`:
```ion
(:add_macros
    (macro foo () /*...*/)
    (macro bar () /*...*/))
```

We will also offer versions that take a module name:
```ion
(:add_macros_to module_x
    (macro foo () /*...*/)
    (macro bar () /*...*/))
```

# Default encoding modules

At the beginning of every stream, the encoding module sequence contains two modules:
1. The default module, `_`, which is empty.
2. The system module, `$ion`, which is defined by the specification.

Notice that `_` is _before_ `$ion`.
This means that initially, the macro at unqualified address `0` is the first macro in the system macro table.
Adding a macro called `foo` to `_` would cause `0` to point to `foo`; the first macro in the system macro table is now `1`.

Both `_` and `$ion` can be removed from the encoding modules sequence.

# Changing encoding modules in place

When a module in the encoding module sequence is modified or redefined,
this change takes effect immediately after the directive that performed the modification.

```ion
$ion_1_1

// This modifies `_`, which is already in the encoding module sequence
(:append_macros
    (macro foo () /*...*/)
    (macro bar () /*...*/))

// We don't have to use an `encoding` directive to apply the change,
// it's already taken effect.

(:foo)
(:bar)
```
