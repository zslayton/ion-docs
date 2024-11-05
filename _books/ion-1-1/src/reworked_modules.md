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

## Module internals

Conceptually, each module has:
* a name
* an internal-only map of submodules
  * The only way to create new module bindings within a module is to define a submodule.
* a map of module exports
  * A module can re-export any module binding that's in scope, including its own submodules.
* an exported macro table, which is an insertion-order index map of names/addresses to their corresponding macro definitions
* a list of the macro, module, and flattened-module entries that appeared in the module definition's `(macro_table ...)` clause

These internals are illustrative, not prescriptive.
Implementations may structure their data as desired so long as they produce the required behavior.

**Example**
```ion
$ion_1_1
// `$ion` and `$encoding` are implicitly available
$ion::
(module a
    (module b
        (module c
            (macro_table
                (macro bi () /*...*/)))
        (macro_table
            (macro foo () /*...*/)
            (export c::bi)
            (macro bim () /*...*/)
            (macro bop () /*...*/)))
    (module d
        (macro_table
            (macro quux () /*...*/))
            (macro quuz () /*...*/)))
    (macro_table
        d::* // Flattening re-export of a sibling module's macros
        (macro bar () /*...*/)
        b // Re-exporting a sibling module whose binding is in scope
        (macro baz () /*...*/))
```

| Module |  Submodules  | Exported<br/>modules | Macro table                                               |
|:------:|:------------:|:--------------------:|-----------------------------------------------------------|
|  `a`   |   `b`, `d`   |         `b`          | `0 => quux`<br/>`1 => quuz`<br/>`2 => bar`<br/>`3 => baz` |
|  `b`   |     `c`      |     _&lt;empty>_     | `0 => foo`<br/>`1 => bi`<br/>`2 => bim`<br/>`3 => bop`    |
|  `c`   | _&lt;empty>_ |     _&lt;empty>_     | `0 => bi`                                                 |
|  `d`   | _&lt;empty>_ |     _&lt;empty>_     | `0 => quux`<br/>`1 => quuz`                               |
|  `e`   | _&lt;empty>_ |     _&lt;empty>_     | `0 => quuz`<br/>`1 => bar`<br/>`2 => baz`                 |

Things to notice:
* `a`'s `macro_table` clause...
    * does a flattening re-export of `d`. `d`'s macros appear in `a`'s macro table, but `d` does _not_ appear in `a`'s exported modules.
    * re-exports `b`. `a`'s exported macro table does _not_ include any of the macros defined in `b`, but `a`'s exported modules map _does_ include `b`.
* Each module tracks its submodules and exported modules, but can also refer to other modules that are in scope (e.g. previously defined sibling or top-level modules).
  This is because module bindings are resolved by recursively consulting the parent scope.

### Notes on competing models

#### Model 1: All modules are "statically linked"

In this model, every module copies its dependencies into its macro table, guaranteeing that it is self-contained.
This gives each module in the tree a flat macro address space, including the root of the tree: `$encoding`.

In this model, the encoding module (`$encoding`) is used as the encoding context.
All unqualified macro addresses and macro names in e-expressions are resolved in `$encoding`'s macro table.
Qualified names/addresses are resolved by consulting the appropriate module.

* Every module's macro table is self-contained. In the `(macro_table)` clause:
    * Module references (`foo`) re-export all of that module's macros. Referring to those macros requires qualification (`foo::bar`, `foo::7`).
    * Flattening references (`foo::*`) re-export all of that module's macros _and_ merge their names into the current module, raising an error on name conflicts.
      Re-exports that have been flattened cannot use qualified references to the original module (`foo::bar` is an error, `foo` is ok).
* Every module behaves the same, including the encoding module.
* It is not possible to know what module addresses macros will be assigned by looking at the `macro_table` clause alone. For example:
  ```ion
  (module mod_a
      (macro_table             // Local macro address
          (macro foo () ...)   // mod_a::0
          (macro bar () ...)   // mod_a::1
           mod_b               // addresses 2 through ??
          (macro baz () ...))) // ??
  ```
* If two modules re-export the same third module, the third module's macros will occupy extra address space.
    ```ion
  (module mod_a
      (macro_table             // Local macro address
          (macro foo () ...)   // mod_a::0
          (macro bar () ...)   // mod_a::1
          (macro baz () ...))) // ??
  (module mob_b (macro_table mod_a))
  (module mob_c (macro_table mod_a))
  $ion::
  (encoding (macro_table mod_b mod_c))

  // Macro table
  // 0: mod_b::mod_a::foo (aka mod_a::foo)
  // 1: mod_b::mod_a::bar (aka mod_a::bar)
  // 2: mod_b::mod_a::baz (aka mod_a::baz)
  // 3: mod_c::mod_a::foo (aka mod_a::foo)
  // 4: mod_c::mod_a::bar (aka mod_a::bar)
  // 5: mod_c::mod_a::baz (aka mod_a::baz)
  ```

#### Model 2: Modules are "dynamically linked"

> A note on terminology: typically we say "macro table" to refer to the joint data structure that allows for lookups both
> by address and by name. This section uses the term 'macro array' to refer to the macro-by-address half of a macro table.

In this model, `$ion::(encoding ...)` performs two functions:
1. As before, it defines the `$encoding` module.
2. It constructs a consolidated 'global' macro array.

In an e-expression:
* Qualified names and addresses are resolved using the appropriate module.
* Unqualified names are resolved using `$encoding`.
* The global macro array is used to resolve unqualified addresses.

When the reader compiles a module, it takes note of any non-flattening module references that appear in the module's `(macro_table ...)` clause.
Specifically, it records the number of contiguous macros that appear before it. For example:

```ion
(module mod_a
    (macro_table
        (macro foo ...)
        (macro bar ...)
        (macro baz ...)))
// ^^^ contains no module references

(module mod_b
    (macro_table
        (macro quux ...)
        (macro quuz ...)))
// ^^^ contains no module references

(module_c
    (macro_table
        (macro bi ...)
        mod_b::* // Flattening reference, counts as 2 macros
        (macro bim ...)
        mod_a    // Reference preceded by 4 contiguous macros
))
```

As the reader traverses the body of the `$ion::encoding()` directive,
it maintains a record of which module instances have already been copied into the global macro array.

The reader begins by processing the `$ion::(encoding ...)` directive's `macro_table` clause.
For each item encountered:
* **If that item is a macro definition or `(export)`,** the macro reference is added to the end of the encoding context's macro table.
* **If that item is a flattening re-export,** each macro in the referenced module is added to the end of the encoding context's macro table.


When constructing the encoding context, the reader reads the `$encoding` directive's macro_table,
walking the module reference tree in a depth-first search.
For each item encountered:
* **If that item is a macro definition or `(export)`,** the macro reference is added to the end of the encoding context's macro table.
* **If that item is a flattening re-export,** each macro in the referenced module is added to the end of the encoding context's macro table.



The global macro array holds exactly one copy of the macro array from each module transitively exported `$encoding`.





As a module constructs its macro table:
* Macro definitions and `(export)` statements append a macro to the end of the table.
* Flattening re-exports copy the referenced modules' macros to the end of the table.
* Non-flattening macro references do not modify the table; instead, the position of the reference within the `(macro_table ...)` is noted.

When constructing the encoding context, the reader reads the `$encoding` directive's macro_table,
walking the module reference tree in a depth-first search.
For each item encountered:
* **If that item is a macro definition or `(export)`,** the macro reference is added to the end of the encoding context's macro table.
* **If that item is a flattening re-export,** each macro in the referenced module is added to the end of the encoding context's macro table.

* A module's macro table contains only its own macro definitions.
  It can refer to other modules' macro tables, but does not copy their contents without explicit opt-in via a flattening re-export.
*

### Constructing the encoding context

The _encoding context_ is a `(symbol table, macro table)` pair used to encode the data stream.

The context's symbol table is the symbol table found in `$encoding`.

To keep the binary encoding compact, the encoding context uses a flat address space for its macros.
This allows any macro to be identified by a single integer.

Because symbol tables are already flat (they do )

When the `$encoding` module is constructed, it visits each expression in its `(macro_table)`


This pair is constructed by  walking the tree of modules formed by the `$encoding` module.

The symbol table is copied wholesale from the `$encoding` module.

The `$ion::(encoding ...)` directive defines the `$encoding` module, the root of the module tree that will be used to encode the data stream.

The symbol table is copied wholesale from the `$encoding` module.





It is constructed by walking the module body of an `$ion::(encoding ...)` directive and recursively flatten


### Address allocation

All modules have a macro table

Address allocation begins in the `(macro_table)` clause of the `encoding` directive.

For each item in the `macro_table`:
* **If it is a macro,** it receives the next available address.
* **If it is an `(export)`,** it receives the next available address.
* **If it is a module binding...**
  * **...and that module has not yet been added to the encoding context,** that module binding becomes publicly addressable.
    Its macros are assigned the next contiguous block of available addresses, and that address range is stored in the `Module`'s `addresses` field.
  * **...and that module has already been added to the encoding context,** then it already has addresses.
* **If it is a flattening re-export (`module_name::*`),** then all of the macros in `module_name` are added to the encoding context and
  assigned a contiguous block of fields.
## Examples

```ion
$ion::
(module a
    (module b
        (macro_table
            (macro foo () /*...*/)
            (macro bim () /*...*/)
            (macro bop () /*...*/)))
    (macro_table
        (macro bar () /*...*/)
        (macro baz () /*...*/))

$ion::
(encoding
    (macro_table
        (macro quux () /*...*/)
        a::b
        (macro quuz () /*...*/)))
```

Macro table
```ion
quux
a::b::bim
a::b::bop
```