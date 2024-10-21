# Reworked modules sketch
<!--
## Module structure
* Internal
    * Shared reference to the `Catalog` of external shared modules
    * Map of names to nested module definitions
* Exported
    * Symbol table
    * Macro table

## Module bindings

In every scope:
* `(import module-name catalog-name catalog-version?)` statements are resolved in the `Catalog`. Upon success, adds `module-name` to the local module bindings.
* `(module module-name)` defines a module and adds `module-name` to the scope's module bindings.

```ion
(module greetings
    (module english
        (macro_table
            (macro greet () "hello")))
    (module french
        (macro_table
            (macro greet () "bonjour")
    (import german "com.example.language.german" 1)
    (macro_table
        (export english::greet english) // `greetings::english`
        (export german::greet german)   // `greetings::german`
        (export french::greet french))) // `greetings::french`
```

Module bindings are lexically scoped.
References to a module will be searched for first in the enclosing scope, and then recursively upward in each parent scope.

```ion
(module outer
    (module nested_a
        (macro_table
            (macro foo () /*...*/)))
    // binding 'nested_a' now exists inside 'outer'
    (module nested_b
        (macro_table
            // binding 'nested_a' is not found in 'nested_b',
            // but it is found in the parent scope.
            (macro bar () (.nested_a::foo))))
```

Module bindings in a scope cannot be deleted, but they can be shadowed.

```ion
(module outer
    (module nested_a
        (macro_table
            (macro foo1 () /*...*/)))
    // binding 'nested_a' now exists inside 'outer'
    (module nested_a
        (macro_table
            (macro foo2 () /*...*/)))
    // a new binding 'nested_a' now shadows the old 'nested_a'
    (module nested_b
        (macro_table
            (export nested_a::foo1)   // ERROR: invalid reference
            (export nested_a::foo2))) // OK
```

There is no qualified syntax for un-shadowing a module reference.
-->
<!--
## Macro bindings



Inside a `(module ...)` definition clause, `(macro macro-name ...)` statements define a macro and add its name to the current scope's macro bindings.

```ion
(module foo
    (macro bar (x) /*...*/)
    // Binding 'bar' now exists.
    (macro_table
        (macro baz () (.bar 0))
        // The macro table has a single macro, `baz`, which calls `bar`
    )
)
```

As with module bindings, a scope's macro bindings can be shadowed.
```ion
(module foo
    (macro bar () (.none))
    // Binding 'bar' now exists.
    (macro bar (x) /*...*/)
    // A new binding 'bar' has been creating, shadowing the old binding.
    (macro_table
        // The macro table has a single macro, baz, which calls
        // the most recently defined macro `bar`.
        (macro baz () (.bar 0))
    )
)
```

A macro defined inside a nested module's symbol table can be referred to using qualified syntax.
```ion
(module foo
    (macro bar () (.none))
    // Binding 'bar' now exists.
    (module inner
        (macro_table
            (macro bar () /*...*/)))
    (macro_table
        // Macros can refer
        (macro baz () (.bar 0))
    )
)
```

Inside a `(macro_table ...)` clause, `(macro macro-name ...)` defines a macro and adds it to the symbol table being constructed.
It does not add a name to the local bindings.
All macros within the same `(macro_table)` clause must have unique names.

```ion
(module foo
    (macro_table
        (macro bar (x) /*...*/)
        (macro baz () /*...*/)
        (macro baz () /*...*/))) // ERROR: a macro with name 'baz'
                                 //        already exists in this table
```
-->
## The top level

At the top level of a stream, the scope is the stream itself.
This means that once created, module bindings at this level endure until the file ends or another Ion version marker is encountered.
Module bindings are not cleared by further directives, though a module name may be shadowed by one.

One module bindings is automatically available at the outset of every stream: `$ion`, the system module.

### Directive syntax

Previously, there was a single encoding directive which defined modules as well as the encoding for the upcoming segment.
```ion
$ion_encoding::(
  (import /*...*/)
  (module /*...*/)
  (macro_table /*...*/)
  (symbol_table /*...*/)
)
// ...new segment begins using the above encoding module...
```

This proposal suggests making a more general "directive" syntax in the form:

```ion
$ion::
(operation_name
    (clause_a /*...*/)
    (clause_b /*...*/)
    (clause_c /*...*/))
```

This allows us to have multiple directive types while preserving the reader's ability to distinguish
between application values and system data at the top level with a single branch,
namely: "is it a top-level sexp annotated with `$ion`?


### Importing shared modules

This directive resolves the specified catalog key in the catalog, finding the corresponding module.

```ion
$ion::
(import foo "com.example.project.foo" 1)
```
Upon success, it adds the binding `foo` to the stream's map of names to module definitions.

## Named module definition

The `module` directive adds a module binding to the stream.

```ion
$ion::
(module foo
    (import mod_a) // Import from local catalog
    (import mod_b  // Import from global catalog
            "catalog-key" 1)
    (module mod_c  // Modules can have nested modules
            ...)
    (macro_table
        (macro bar () ...)
        (macro baz () ...)
        mod_a
        mod_b::specific macro))
    (symbol_table
        mod_c
        ["dog", "cat", "mouse"]
        mod_a
        ["cheese"]
        mod_b)
)
```

## The encoding directive

```ion
$ion::
(encoding
    foo
    bar
    baz
)
```

This statement loads the symbol and macro tables from each of the modules `foo`, `bar`, and `baz` into the encoding context for the next segment.
`encoding` operates on module names and definitions.

> [!TIP]
> You can reorganize the symbol and macro appends by creating a module with the appropriate exports.

You can also provide module definitions as arguments, which both creates a binding of that name and adds its contents to the encoding context.

```ion
$ion::
(encoding
    foo
    (module bar
        (import ...)
        (module nested_mod ...)
        (macro_table
            (macro quux () ...)
            (macro quuz () ...)))
    baz
)
```

## e-expressions

E-expressions can be qualified to specify which module in the encoding context a name refers to.