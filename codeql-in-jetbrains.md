# Writing CodeQL queries in a JetBrains IDE

Sources: the [CodeQL CLI manual](https://docs.github.com/en/code-security/codeql-cli/codeql-cli-manual) and the [CodeQL CLI binaries](https://github.com/github/codeql-cli-binaries) releases (the GitHub repository is `github/codeql-cli-binaries`). This page is a short, practical walk-through of using the CodeQL command-line tool alongside a JetBrains IDE. The official manual is the reference for every option.

CodeQL has three pieces you work with:

- **The CodeQL CLI** (`codeql`): the command-line tool that builds databases and runs queries. It runs on your machine.
- **A database**: a snapshot of a codebase, created once per project and language.
- **Queries and libraries**: `.ql` files (queries) and `.qll` files (shared libraries), usually organised in a *pack* described by `qlpack.yml`.

A JetBrains IDE gives you the editing side. [Taintline](https://plugins.jetbrains.com/plugin/36120-taintline) adds highlighting, folding, a structure view and a run configuration for `.ql`, `.qll` and `.dbscheme` files. It does not include the CodeQL CLI, so you install that yourself.

## 1. Install the CodeQL CLI

Download the zip for your platform from the CLI releases page (assets are named `codeql-osx64.zip`, `codeql-linux64.zip`, `codeql-win64.zip` and so on), unzip it somewhere permanent, and add the `codeql` folder to your `PATH`. Then check it works:

```sh
codeql --version
codeql resolve languages
```

**Check the licence terms first.** The CodeQL CLI is governed by the [GitHub CodeQL Terms and Conditions](https://github.com/github/codeql-cli-binaries/blob/main/LICENSE.md), which allow some uses (for example academic research, and analysing open-source codebases) and do not cover others, such as using the CLI on codebases that are not open source or generating databases in automated CI, unless you have a separate agreement with GitHub. Read them before using the tool on a company codebase.

If the IDE was started from a launcher that does not inherit your shell's `PATH`, it may not find `codeql`. In that case set the full path to the executable in the run configuration (below).

## 2. Create a database

From the root of the project you want to analyse:

```sh
codeql database create my-db --language=python --source-root=.
```

Replace `python` with the language identifier for your code (`codeql resolve languages` lists them). Compiled languages may also need `--command=<build command>`. Keep the resulting `my-db` folder; you reuse it for every query run.

## 3. Install the pack's dependencies

If your queries live in a pack with a `qlpack.yml` that lists dependencies, install them once:

```sh
codeql pack install
```

(run in the pack's folder, or pass the folder as an argument).

## 4. Run a query from the IDE

With Taintline installed:

1. Open a `.ql` file.
2. Right-click it and choose **Run**, or create a **QL Query** run configuration (Run | Edit Configurations).
3. Set the **database** folder (`my-db`). New configurations reuse the last one.
4. Optionally add extra arguments, and set the full path to `codeql` if it is not on the `PATH`.
5. Run. The plugin executes `codeql query run --database=<database> [extra arguments] <query.ql>` and shows the output in the Run console.

You can run the same thing in a terminal:

```sh
codeql query run --database=my-db path/to/query.ql
```

## Common problems

| Symptom | Likely cause and fix |
|---|---|
| "codeql: command not found" or the run fails to start | The IDE cannot see `codeql` on its `PATH`. Set the full path to the executable in the run configuration. |
| Imports such as `import python` cannot be resolved | The query is not inside a pack that depends on the language library. Create a `qlpack.yml` listing the dependency and run `codeql pack install`. |
| "Could not find database" | The database path is wrong or the database was not created. Point the run configuration at the folder created by `codeql database create`. |
| The query runs but returns nothing | The database may not contain what you expect. Check the language and, for compiled languages, that the build command ran during database creation. |
| Highlighting works but there are no completions or live errors | Taintline does not include a language server yet, so there is no completion or live error checking. Run the query to see the compiler's errors in the console. |

---

Taintline: free and offline, from the JetBrains Marketplace: [Taintline](https://plugins.jetbrains.com/plugin/36120-taintline).

Not affiliated with or endorsed by GitHub. CodeQL is a trademark of GitHub, Inc.
