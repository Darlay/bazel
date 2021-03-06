---
layout: documentation
title: Extensions - Overview
---
# Overview

## Loading an extension

Extensions are files with the `.bzl` extension. Use the `load` statement to
import a symbol from an extension.

```python
load("//build_tools/rules:maprule.bzl", "maprule")
```

This code will load the file `build_tools/rules/maprule.bzl` and add the
`maprule` symbol to the environment. This can be used to load new rules,
functions or constants (e.g. a string, a list, etc.). Multiple symbols can be
imported by using additional arguments to the call to `load`. Arguments must
be string literals (no variable) and `load` statements must appear at
top-level, i.e. they cannot be in a function body.

`load` also supports aliases, i.e. you can assign different names to the
imported symbols.

```python
load("//build_tools/rules:maprule.bzl", maprule_alias = "maprule")
```

You can define multiple aliases within one `load` statement. Moreover, the
argument list can contain both aliases and regular symbol names. The following
example is perfectly legal (please note when to use quotation marks).

```python
load("/path/to:my_rules.bzl", "some_rule", nice_alias = "some_other_rule")
```

Symbols starting with `_` are private and cannot be loaded from other files.
Visibility doesn't affect loading: you don't need to use `exports_files` to make
a `.bzl` file visible.

## Macros and rules

A [macro](macros.md) is a function that instantiates rules. The
function is evaluated as soon as the BUILD file is read. Bazel has little
information about macros: if your macro generates a `genrule`, Bazel will behave
as if you wrote the `genrule`. As a result, `bazel query` will only list the
generated `genrule`.

A [rule](rules.md) is more powerful than a macro, as it can access
Bazel internals and have full control over what is going on. It may for example
pass information to other rules.

If a macro becomes complex, it is often a good idea to make it a rule.

## Evaluation model

A build consists of three phases.

* **Loading phase**. First, we load and evaluate all extensions and all BUILD
  files that are needed for the build. The execution of the BUILD files simply
  instantiates rules. This is where macros are evaluated.

* **Analysis phase**. The code of the rules is executed (their `implementation`
  function), and actions are instantiated. An action describes how to generate
  a set of outputs from a set of inputs, e.g. "run gcc on hello.c and get
  hello.o". It is important to note that we have to list explicitly which
  files will be generated before executing the actual commands.

* **Execution phase**. Actions are executed, when at least one of their outputs is
  required. If a file is missing or if a command fails to generate one output,
  the build fails. Tests are run during this phase, as they are actions.

Bazel uses parallelism to read, parse and evaluate the `.bzl` files and `BUILD`
files. A file is read at most once per build and the result of the evaluation is
cached and reused. A file is evaluated only once all its dependencies (`load()`
statements) have been resolved. By design, loading a `.bzl` file has no visible
side-effect, it only defines values and functions.

## Syntax

The extension language (Skylark) is a superset of the
[Core Build Language](/docs/build-ref.html#core_build_language)
and its syntax is a subset of Python.
It is designed to be simple, thread-safe and integrated with the
BUILD language. It is not a general-purpose language and most Python
features are not included.

The following constructs have been added to the Core Build Language: `if`
statements, `for` loops, and function definitions. They behave like in Python.
Here is an example to show the syntax:

```python
def fizz_buzz(n):
  """Print Fizz Buzz numbers from 1 to n."""
  for i in range(1, n + 1):
    s = ""
    if i % 3 == 0:
      s += "Fizz"
    if i % 5 == 0:
      s += "Buzz"
    print(s if s else i)

fizz_buzz(20)
```

## Mutability

Because evaluation of BUILD and .bzl files is performed in parallel, there are
some restrictions in order to guarantee thread-safety and determinism. Two
mutable data structures are available: [lists](lib/list.html) and
[dicts](lib/dict.html). Unlike in Python, [sets](lib/set.html) are not mutable.

In a build, there are many "evaluation contexts": each `.bzl` file and each
`BUILD` file is loaded in a different context. Each rule is also analyzed in a
separate context. We allow side-effects (e.g. appending a value to a list or
deleting an entry in a dictionary) only on objects created during the current
evaluation context.

For example, here is the content of the file `foo.bzl`:

```python
var = []

def fct():
  var.append(5)

fct()
```

The variable `var` is created when `foo.bzl` is loaded. `fct()` is called during
the same context, so it is safe. At the end of the evaluation, the definition
`var = [5]` is exported. Any other file can load it, and it is possible that
multiple files will load it at the same time. For this reason, the following
code is not legal:

```python
load(":foo.bzl", "var", "fct")

var.append(6)  # not allowed

fct()  # not allowed
```

Since the call to `fct()` attempts to mutate the shared variable `var`, it will
fail. `fct()` can only be called during the evaluation of `foo.bzl`. It cannot
be called from another file. It is also forbidden to call it during the analysis
phase (i.e. when a custom rule is analyzed).

## Differences with Python

In addition to the mutability restrictions, there are also differences with
Python:

* All global variables cannot be reassigned

* Sets and dictionaries have a deterministic order of iteration (see
  [documentation](lib/globals.html#set) for sets).

* Recursion is not allowed.

* Sets have reference equality semantics and can be stored in other sets.

* Modifying a collection during iteration is an error. You can avoid the error
  by iterating over a copy of the collection, e.g.
  `for x in list(my_list): ...`. You can still modify its deep contents
  regardless.

The following items are upcoming changes.

* Comprehensions currently "leak" the values of their loop variables into the
  surrounding scope (Python 2 semantics). This will be changed so that
  comprehension variables are local (Python 3 semantics).

* `load()` statements can currently appear anywhere at the top-level of a file
  so long as it is not in an indented block of code. In BUILD files, they may
  overwrite an existing variable with the loaded symbol. In the future, `load()`
  statements will be required to appear at the beginning of the file and will
  not be able to overwrite any names (use load aliases to avoid name clashes).

* Previously dictionaries were guaranteed to use sorted order for their keys.
  Going forward, there is no guarantee on order besides that it is
  deterministic. As an implementation matter, some kinds of dictionaries may
  continue to use sorted order while others may use insertion order.

* Some collections currently use reference equality (sometimes inconsistently).
  This will change to structural equality for all collections except sets.

* The `+=` operator and similar operators are currently syntactic sugar;
  `x += y` is the same as `x = x + y`. This will change to follow Python
  semantics, so that for mutable collection datatypes, `x += y` will be a
  mutation to the value of `x` rather than a rebinding of the variable `x`
  itself to a new value. E.g. for lists, `x += y` will be the same as
  `x.extend(y)`.

* The `+` operator is defined for dictionaries, returning an immutable
  concatenated dictionary created from the entries of the original
  dictionaries. This will be going away. The same result can be achieved using
  `dict(a.items() + b.items())`. Likewise, there is a `+` operator for sets that
  will be going away; users should use `|` instead.

The following Python features are not supported:

* `class` (see [`struct`](lib/globals.html#struct) function)
* `import` (see [`load`](#loading-a-skylark-extension) statement)
* `while`, `yield`
* `lambda` and nested functions
* `is` (use `==` instead)
* `try`, `raise`, `except`, `finally` (see [`fail`](lib/globals.html#fail)
  for fatal errors).
* most builtin functions, most methods


