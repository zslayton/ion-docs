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
# Remaining issues

The current module specification has two gaps that we would like to close before finalizing the specification:
1. [Top-level module reuse](#top-level-module-reuse)
2. [Better ways to avoid namespace collisions](#better-ways-to-avoid-namespace-collisions)

## Top-level module reuse
It is currently not possible to define or import a shared module and make use of it for the duration of the stream.
As specified, module definitions and imports always appear within an `$ion_encoding::(...)` directive and go out of scope when the directive ends.

It would be nice to be able to define one or more 'core' sets of macros for a long-lived stream and periodically reset to some set of them,
reclaiming address space without discarding valuable encoding constructs.

## Better ways to avoid namespace collisions

While TDL supports qualified macro references, e-expressions do not.
This is because the binary encoding relies on the macro table being a flat address space;
there is no qualified e-expression syntax in binary.

To ensure that all macro names are unambiguous, module authors are currently required to resolve any naming conflicts in the module's `macro_table`.
Because e-expressions always invoked a macro from the encoding module's symbol table,
it is therefore guaranteed that all unqualified macro names referenced in an e-expression are unique and unambiguous.

However, when the module depends on other modules--especially those maintained by someone else--the
macro table uniqueness constraint can become quite onerous.

```ion
$ion_encoding::(
  (module foo
    (import mod_a "com.example.a" 2)
    (import mod_b "com.example.b" 4)
    (macro_table
        mod_a
        mod_b // conflict?
        /*...*/)))
```
In the above example, the author cannot easily know whether `mod_a` and `mod_b` contain any macros whose names conflict.
If there _is_ a conflict, the only mechanism that exists to resolve it would be to use the `(export ...)` operation to rename one of the conflicting macros.
However, doing so means that all of the other macros in that module would have to be re-exported individually--no bulk rename/re-export facility exists.

# Proposed changes

* [Directive syntax](#directive-syntax)
* [Top-level module bindings](#top-level-module-bindings)
* [Unambiguous macro references](#unambiguous-macro-references)

## Directive syntax

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

To address the need for [top-level module bindings](#top-level-module-bindings), the specification will be updated to support multiple top-level directives.
To accommodate this, we will introduce a more general "directive" syntax in the form:

```ion
$ion::
(operation_name
    (clause_a /*...*/)
    (clause_b /*...*/)
    (clause_c /*...*/))
```

This allows us to have multiple directive types while preserving the reader's ability to distinguish
between application values and system data at the top level with a single branch,
namely: "is it a top-level sexp annotated with `$ion`"?

## Top level module bindings

This proposal replaces the original `$ion_encoding::(...)` form with three top-level operations:
1. `(import ...)`
2. `(module ...)`
3. `(encoding ...)`

In TDL, the `(module ...)` and `(import ...)` operations create lexically-scoped module bindings.
At the top level of a stream, the new binding's lexical scope is _the stream itself_.
This means that once created, module bindings at this level endure until the file ends or another Ion version marker is encountered.

As with any lexically scoped binding, module names from the top scope are accessible in any nested TDL scope as long as the name is not shadowed.
This arrangement makes it possible for a writer to reference a module many times over the course of a stream,
even after the encoding module has been redefined.

```ion
$ion::
(module log_levels
    (symbol_table ["TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"]))
// ...
(:append_symbols ["abc123" "def456" "ghi789"])
// ...
(:set_symbols log_levels)
```

Module bindings are not removed/cleared by further directives, though a module name may be shadowed by a new binding.

### `$ion::(import ...)`

Resolves a `(name, version)` pair in the catalog.

```ion
$ion::
(import foo "com.example.project.foo" 1)
```
Upon success, it adds the binding `foo` to the stream's map of names to module definitions.

### `$ion::(module ...)`

Defines a new module and binds a name to it.

```ion
$ion::
(module mod_a /*...*/)

$ion::
(module foo
    (import mod_b  // Import from global catalog
            "catalog-key" 1)
    (module mod_c  // Modules can have nested modules
            ...)
    (macro_table
        (macro bar () /*...*/)
        mod_a // Visible from parent scope
        (export mod_b::specific_macro)))
    (symbol_table
        mod_c
        ["dog", "cat", "mouse"]
        mod_a))
```

### Reserved module names

Two module bindings are automatically available at the outset of every stream:
1. `$ion`
2. `$encoding`

`$ion` is the system module. It contains macros and symbols defined in the Ion specification.

`$encoding` is the encoding module. Its symbol table and macro table are used to encode the stream.

> [!IMPORTANT]
> **Users cannot create a binding whose name begins with `$`.**
> These are reserved for the system use.
> ```ion
> (import $ion /*...*/)      // ERROR
> (module $foo /*...*/)      // ERROR
> (import $encoding /*...*/) // ERROR
> ```

This means that the `$ion` and `$encoding` modules cannot be shadowed.

### `$ion::(encoding ...)`

The module `$encoding` is special in that its definition determines the encoding of data stream that follows.
To emphasize this, it is defined with its own directive:
```ion
$ion::
(encoding
    (macro_table
        (macro greet () "hello!")))
```

The `encoding` keyword is equivalent to `(module $encoding)` with two important distinctions:
* It enables the shadowing of a module whose name begins with `$`.
* It is only valid at the top level. This means that `$encoding` can be shadowed at the top, but not at deeper levels of nesting.

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



## E-expression macro resolution

E-expressions can only invoke macros in two modules: `$ion` and `$encoding`.
All user macros live in `$encoding`, which means it will be the most heavily referenced.

To streamline the common case, macros in `$encoding` can be referenced in an e-expression without qualification.
For example, consider this stream:

```ion
$ion_1_1

$ion::
(encoding
    (module boop
        (macro_table
            (macro shi (x) /*...*/)
            (macro shoo (y) /*...*/)))
    (macro_table
        boop
        (macro foo () /*...*/))))
```

Macro resolution begins in the `$encoding` module; writers do not need to include an `$encoding::` qualification:
```ion
(:foo) // OK
```
Qualified macros may also omit the leading `$encoding::`:
```ion
(:boop::shi) // OK
```

When the first module name in a qualified macro name begins with `$`, resolution instead begins in that module.
This allows system macros to be unambiguously invoked:
```ion
// Resolution begins in `$ion`
(:$ion::make_string foo bar baz)
```
Writers may optionally include the `$encoding` qualification under the same rule:
```ion
// Resolution begins in `$encoding`
(:$encoding::shoo)
```
