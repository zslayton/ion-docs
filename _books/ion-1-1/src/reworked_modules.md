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
# The problem

The current module specification has two gaps that we would like to close before finalizing the specification:
1. [Stream-level modules](#stream-level-modules)
2. [Better ways to avoid namespace collisions](#better-ways-to-avoid-namespace-collisions)

### Stream-level modules
First, it is currently not possible to define or import a shared module and make use of it for the duration of the stream.
As specified, module definitions and imports always appear within an `$ion_encoding::(...)` directive and go out of scope when the directive ends.

It would be nice to be able to define one or more 'core' sets of macros for a long-lived stream and periodically reset to some set of them,
reclaiming address space without discarding valuable encoding constructs.

### Better ways to avoid namespace collisions

While TDL supports qualified macro references, e-expressions do not.
This is because the binary encoding relies on the macro table being a flat address space;
there is no qualified e-expression syntax in binary.

To ensure that all macro names are unambiguous, module authors are currently required to resolve any naming conflicts in the module's `macro_table`.
However, when the module depends on other modules--especially those maintained by someone else--this can become quite onerous.

```ion
$ion_encoding::(
  (module foo
    (import mod_a "com.example.a" 2)
    (import mod_b "com.example.b" 4)
    (macro_table
        mod_a
        mod_b) // conflict?
    /*...*/
    ))
```
In the above example, the author cannot easily know whether `mod_a` and `mod_b` contain any macros whose names conflict.
If there _is_ a conflict, the only mechanism that exists to resolve it would be to use the `(export ...)` operation to rename one of the conflicting macros.
However, doing so means that all of the other macros in that module would have to be re-exported individually--no bulk rename/re-export facility exists.

# Supporting stream-level bindings
## Directive syntax

To address the need for [stream-level module bindings](#stream-level-modules), the specification will support multiple top-level directives.

Previously, there was a single encoding directive which defined modules as well as the encoding for the upcoming segment:
```ion
// (re-)defines the `$ion_encoding` module
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

It is also legal to group multiple operations into a single directive.
If the first expression in the s-expression is a name, it's a single form.
If the first expression is an s-expression, it's a sequence of operations.

```ion
// a directive with multiple operations
$ion::
((operation_a /*...*/)
 (operation_b /*...*/)
 (operation_c /*...*/))
```

## Top level module bindings

In TDL, the `(module ...)` and `(import ...)` operations create lexically-scoped module bindings.
This proposal replaces the original `$ion_encoding::(...)` form with these two operations,
making it possible for them to appear at the top level.

**`$ion::(import ...)`** resolves a `(named, version)` pair in the catalog.

```ion
$ion::
(import foo "com.example.project.foo" 1)
```
Upon success, it adds the binding `foo` to the stream's map of names to module definitions.


**`$ion::(module ...)`** which defines a new module and binds a name to it.

```ion
$ion::
(module foo
    (import mod_a) // Import from local catalog
    (import mod_b  // Import from global catalog
            "catalog-key" 1)
    (module mod_c  // Modules can have nested modules
            ...)
    (macro_table
        (macro bar () /*...*/)
        mod_a
        mod_b::specific_macro))
    (symbol_table
        mod_c
        ["dog", "cat", "mouse"]
        mod_a))
```

At the top level of a stream, the new binding's lexical scope is _the stream itself_.
This means that once created, module bindings at this level endure until the file ends or another Ion version marker is encountered.

Module bindings are not removed/cleared by further directives, though a module name may be shadowed by a new binding.

Two module bindings are automatically available at the outset of every stream:
1. `$ion`, the system module, which contains macros and symbols defined in the Ion specification
2. `$ion_encoding`, the encoding module

> [!WARNING]
> **Users cannot create a new module named `$ion` or whose name begins with `$ion_`.**
> These are reserved for the Ion specification.

As with any lexically scoped binding, module names from the top scope are accessible in any nested TDL scope as long as the name is not shadowed.

To modify the encoding context, writers will shadow the `$ion_encoding` module:
```ion
$ion::
(module $ion_encoding
    (macro_table
        (macro greet () "hello!")))
```

> [!TIP]
> Because users cannot create their own module called `$ion`, the system module can never be shadowed.

This arrangement makes it possible for a writer to reference a module many times over the course of a stream,
even after the `$ion_encoding` module has been redefined.

```ion
$ion::
(module log_levels
    (symbol_table ["TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"]))
// ...
(:append_symbols ["abc123" "def456" "ghi789"])
// ...
(:set_symbols log_levels)
```

# Unambiguous macro references

This section lays out a method of giving all macros an unambiguous qualified name while also constructing a flat macro address space for use in binary.

## Qualified macro exports

As before, referencing a module name inside a `(macro_table ...)` causes that module's macro table to be copied wholesale into the new macro table.
However, going forward it does _not_ cause that module's macro _names_ to be added to the new table's names.

Instead, the referenced macro name becomes eligible for qualified reference outside the module.

```ion
(module bar
    (import bim "com.example.bim" 1)
    (import bop "com.example.bop" 5)
    (module boop
        (macro_table
            (macro shi (x) /*...*/)
            (macro shoo (y) /*...*/)))
    (macro_table
           boop
        // ^^^^ Exporting a module's macros assigns them contiguous addresses.
        // However, this does _not_ add the names `shi` and `shoo` to `bar` directly.
        // Instead, it makes it possible to refer to `bar::boop::macro_name`.
        // Here:
        //   `bar::0` -> `bar::boop::shi`
        //   `bar::1` -> `bar::boop::shoo`
        // Invoking the macros from `boop` by name *requires* qualifying the
        // reference with `boop`.

        (macro quux () ...)  // `bar::2`, `bar::quux`
        (macro quuz () ...)) // `bar::3`, `bar::quuz`
    (symbol_table
        bim bop ["a", "b", "c"]))
```

If and when the author is confident there is not a naming conflict,
they may do a 'flattening' re-export using `module_name::*` syntax:

```ion
(module bar
    (import bim "com.example.bim" 1)
    (import bop "com.example.bop" 5)
    (module boop
        (macro_table
            (macro shi (x) /*...*/)
            (macro shoo (y) /*...*/)))
    (macro_table
           boop::*
        // ^^^^^^^ A flattening export assigns contiguous addresses
        // to the macros in `boop`. It also adds their names to the
        // parent namespace (in this example: `bar`),
        // allowing them to be referenced with fewer qualifications:
        //   `bar::0` -> `bar::shi`
        //   `bar::1` -> `bar::shoo`
        (macro quux () ...)  // `bar::2`, `bar::quux`
        (macro quuz () ...)) // `bar::3`, `bar::quuz`
    (symbol_table
        bim bop ["a", "b", "c"]))
```
Flattening imports do _not_ cause the nested module to become externally visible/addressable.

```ion
// Module binding `bar` exists at the top level
(module waldo
    (macro_table
        (macro foo() (.bar::boop::shoo)))) // ERROR: `bar::boop` is not accessible
```

As before, individual names can be added directly to the module via `(export module_name::macro_name [alias?])`.

```ion
(module bar
    (module boop
        (macro_table
            (macro shi (x) /*...*/)
            (macro shoo (y) /*...*/)))
    (macro_table
           (export boop::shoo)  // `bar::0`, `bar::shoo`
        // ^^^^^^^^^^^^^^^^^^^ Adds one macro to the parent table
        (macro quux () ...)  // `bar::1`, `bar::quux`
        (macro quuz () ...)) // `bar::2`, `bar::quuz`
    (symbol_table
        bim bop ["a", "b", "c"]))
```



## E-expressions

At the beginning of the stream, the `$ion_encoding` module is identical to the system module.

```ion
$ion_1_1

// The following directive defines a new `$ion_encoding` module, shadowing the original.
$ion::
(module $ion_encoding
    (module boop
        (macro_table
            (macro shi (x) /*...*/)
            (macro shoo (y) /*...*/)))
    (macro_table
        boop
        (macro foo () /*...*/)))
)
// The prior `$ion_encoding` is now shadowed, inaccessible.
// From here on, the stream will be encoded using the new `$ion_encoding`.
```

Macros in the `$ion_encoding` module can be referenced in an e-expression without qualification:
```ion
(:foo) // OK
```
...but macros that were re-exported via `$ion_encoding`'s `(macro_table ...)` are not:
```ion
(:shi) // ERROR: no macro named 'shi' in `$ion_encoding`
```
Instead, these macros require a qualified syntax:
```ion
(:boop::shi) // OK
```

### Reserved names

> [!WARNING]
> **It is illegal for users to define a module name which begins with `$ion_` or is exactly `$ion`.**

E-expression macro references almost begin the resolution process in the `$ion_encoding` module;
however, there is one exception:
if the outermost module name in a qualified e-expression starts with `$ion`,
the reader will attempt to resolve the macro name starting in the specified module name.

This allows macros in the system module to be invoked anywhere in the stream:
```ion
(:$ion::make_string a b c)
```
It also allows users to specify the 'absolute' path to a macro:
```ion
(:$ion_encoding::foo::bar)
```
