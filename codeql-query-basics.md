# CodeQL query basics: a short orientation

Source of truth: the [CodeQL documentation](https://codeql.github.com/docs/), in particular [CodeQL queries](https://codeql.github.com/docs/writing-codeql-queries/codeql-queries/) and the [QL language reference](https://codeql.github.com/docs/ql-language-reference/). This page is a short orientation for someone writing their first queries. For using the command-line tool from a JetBrains IDE see [Writing CodeQL queries in a JetBrains IDE](codeql-in-jetbrains).

CodeQL treats code as data. You build a database from a codebase, then write queries in QL, a declarative, logic-based language, to ask questions about it.

## Anatomy of a query

A query file ends in `.ql`. It usually has a metadata comment, imports, and a `from … where … select` block:

```ql
/**
 * @name Empty if statement
 * @description Finds if statements with an empty body.
 * @kind problem
 * @problem.severity warning
 * @id example/empty-if
 */

import java

from IfStmt s
where s.getThen() instanceof EmptyStmt
select s, "This if statement has an empty body."
```

- `import java` brings in the Java library. Each supported language has its own library (`python`, `javascript`, `cpp`, `csharp`, `go`, `ruby` and others).
- `from` declares variables and their types, here `s` of type `IfStmt`.
- `where` is a logical condition over those variables.
- `select` says what to report. For `@kind problem` queries the result is an element followed by a message string.

## Common building blocks

| Piece | What it is | Example |
|---|---|---|
| Class | A set of values with member predicates | `IfStmt`, `MethodCall` |
| Predicate | A named relation, like a function that may return several results | `predicate isEmpty(Block b) { b.getNumStmt() = 0 }` |
| `exists` | "There is some value such that…" | `exists(Method m \| m.getName() = "run")` |
| `not`, `and`, `or` | Logical connectives | `not s.hasElse()` |
| `instanceof` | Type test | `e instanceof NullLiteral` |
| `forall` | "For all values…" | `forall(Parameter p \| p.isFinal())` |
| `result` | The return value of a predicate with a result type | `int getSize() { result = 3 }` |

Queries and `.qll` library files can define your own classes and predicates. A `.qll` file is a shared library; a `.ql` file is a query you run.

## Query kinds

The `@kind` tag in the metadata decides how results are shown:

- `problem`: an alert at one location with a message.
- `path-problem`: an alert with a data-flow path from a source to a sink (taint tracking).
- `metric` and `table`: values, not alerts.

## Packs

Queries live in a **pack**, described by `qlpack.yml`. The pack names its dependencies, such as the language library, and `codeql pack install` downloads them. Without a `qlpack.yml`, imports such as `import java` usually cannot be resolved.

## Running a query

With a database already built (`codeql database create`), run:

```sh
codeql query run --database=my-db path/to/query.ql
```

## Habits that help

- Start from a broad query and narrow the `where` clause step by step, running it each time.
- Use the language library's documentation (the CodeQL library reference for your language) to find the class you need before writing your own.
- Give every query an `@id` and a clear `@name`; tools rely on them.
- Check the licence terms before running the CodeQL CLI on a company codebase (see the [JetBrains page](codeql-in-jetbrains)).

---

Editing `.ql`, `.qll` and `.dbscheme` files in a JetBrains IDE? [Taintline](https://plugins.jetbrains.com/plugin/36120-taintline) adds highlighting, folding, a structure view and a run configuration, all offline.

Not affiliated with or endorsed by GitHub. CodeQL is a trademark of GitHub, Inc.
