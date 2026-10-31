# Other Commands

`gslint` ships as a single JAR with several sub-commands, dispatched by `Main`. `lint` (see [README.md](README.md)) is the default and primary command. This page briefly covers the others.

Each command has its own set of arguments, distinct from `lint`'s. Run `java -jar gslint.jar <command> --help` to see the full flag list for any command.

All commands accept `--include`/`--init-src-root` the same way `lint` does, unless noted otherwise. Within each section, commands are ordered from most to least commonly reached for.

## Analysis & Reporting

### `quality`
Produces a per-file risk score (0-N) by combining complexity, coupling (fan-in/fan-out), file size, doc coverage, and test coverage into one number. Where `lint` tells you what's specifically wrong on a line, `quality` tells you which files are the most expensive to maintain and change - every file gets a score even with zero lint violations. Supports `--auto-risk` (percentile-based weight calibration), `--rule-risk` (fold lint violations into the score), and rich filtering. Like `coverage`, its test-coverage-related factors rely on the same test-file naming convention (or `--test-class-pattern`) to identify test files - a mismatch here skews the risk score, not just a coverage number.

### `coverage`
Produces a per-file testability/coverage report from static analysis - a narrower, coverage-focused sibling of `quality`. By default it identifies test files by naming convention (simple name ending in `Test`/`Tests`); use `--test-class-pattern` if your codebase names tests differently.

### `complexity`
Reports cyclomatic and cognitive complexity per function. See [COMPLEXITY.md](COMPLEXITY.md) for the full metric reference.

### `constants`
Reports static-final constant usages with flow-control classification (e.g., is a constant used in a branch condition, a return value, a log message).

## Structure & Tooling Integration

### `outline`
Exports a structural JSON skeleton of each file - classes, members, line ranges, hashes, complexity - without full source. Always emits JSON (no `--format` flag).

```
java -jar gslint.jar outline --include src/ --init-src-root src/ --output outline.json
```

Output is one `outlines` entry per file (trimmed here; real entries also carry hashes, complexity and call indexes). Each member has a stable `id` you can use to track it across runs:

```json
{
  "metadata": { "tool": "gslint", "command": "outline", "filesInspected": 1 },
  "outlines": [{
    "file":  { "path": "Pricing.gs", "lineCount": 20, "language": "gosu" },
    "class": {
      "fqn": "Pricing", "kind": "class", "startLine": 1, "endLine": 20,
      "functions": [{
        "id": "Pricing#total(java.util.List<java.lang.Integer>)",
        "signature": "function total(items : java.util.List<java.lang.Integer>) : int",
        "returnType": "int", "startLine": 2
      }]
    }
  }]
}
```

### `duplicate`
A two-phase, cross-file, whole-corpus clone detector - answers "where is this function body copy-pasted elsewhere in the codebase," not "is this specific construct a code smell" (that's what `lint`'s duplication-pattern rules, e.g. `duplicate-branch-code`, are for). Supports three detection algorithms (`hash`/`winnow`/`vector`).

```
java -jar gslint.jar duplicate --include src/ --init-src-root src/ [--algorithm winnow]
```

The default `hash` algorithm reports exact structural matches (variable names are ignored). Here `totalCopy` is a copy-paste of `total`:

```
  Duplicate Group 1  [2 functions, exact match]
  ----------------------------------------------------------------
  src/Pricing.gs          tokens   17 | line    2
    total(List<Integer>)

  src/Pricing.gs          tokens   17 | line   11
    totalCopy(List<Integer>)

  Lines of code scanned    :     28
  Total functions analyzed :      3
  Duplicate groups found   :      1
  Redundant lines (approx) :      8
```

Use `--algorithm winnow` to also catch near-duplicates (copy-paste with edits).

### `astdump`
Parses a single file and dumps its raw Gosu AST tree - a debugging aid for understanding how the parser represents a given construct before writing a rule against it. Takes the file via `--file` (not positional) and can filter to one function via `--function`. `--format` is `text` (default), `json` or `dot` (Graphviz). Add `--resolve` to also see what the parser bound each call, `new` and field access to (the overload, its declaring owner, argument types), `--annotate-complexity cyclomatic|cognitive` to show complexity increments on each node, and `--max-depth` to cap the tree depth.

```
java -jar gslint.jar astdump --file Greeter.gs --init-src-root . --function greet
```

Each source line is printed once, with the AST nodes that start on it indented underneath. Node labels are the parser's interface names (`IIfStatement`, `IReturnStatement`), so they are the names to use when writing a rule.

```
AST Dump: Greeter.gs
2 | function greet(name : String) : String {
  |  IFunctionStatement  function greet(java.lang.String) -> java.lang.String
  |  └─ ParameterListClause
  |     └─ ParameterDeclaration  name : java.lang.String
  |  └─ IStatementList
3 | if (name == null) {
  |     └─ IIfStatement
  |        └─ EqualityExpression  ==
  |           └─ IIdentifierExpression  name
  |           └─ NullExpression
4 | return "Hello, stranger"
  |        └─ IReturnStatement
  |           └─ StringLiteral  "Hello, stranger"
6 | return "Hello, " + name
  |     └─ IReturnStatement
  |        └─ AdditiveExpression  +
  |           └─ StringLiteral  "Hello, "
  |           └─ IIdentifierExpression  name
```

(Trimmed here: the real output also lists `ModifierListClause`, `NameInDeclaration` and `TypeLiteral` nodes.)

### `emit`
Compiles Gosu source files and writes `.class` files to disk (including inner/block/lambda classes), reusing the same `--include`/`--init-src-root`/`--additional-jars-file` setup as `lint`/`quality` - a one-step way to get `.class` files without invoking `gosuc` yourself, as long as that setup is correct.

## Diagnostics & Setup

### `diagnose`
Verifies the Gosu runtime environment without running a full scan: reports which `--additional-jars` loaded successfully vs. are missing, probes specific type names for resolution (pass via repeated `--type`), and lists which lint rules improve in accuracy once those types resolve. Useful for debugging `ErrorType`/type-incomplete results before a real run.

```
java -jar gslint.jar diagnose --init-src-root src/ --type java.util.List --type com.example.Missing
```

```
Gosu runtime: initialized

Type resolution probes (1/2 resolved):
  java.util.List       [OK]  (JRE)
  com.example.Missing  [!!]  unresolved

Rules that require resolved types (significant degradation without):
  annotated-base, anonymous-class, command-injection, raw-type, sql-injection, ...
Rules that improve with resolved types (fallback exists):
  bigdecimal-constant, catch-throwable, float-equality, ...
```

An `[!!]` probe means that type is an `ErrorType` in a real run - add the JAR that provides it with `--additional-jars-file`.

### `init-config`
Dumps every rule's default configuration to a `.properties` template - a starting point for customizing thresholds/severities without hand-writing the file from scratch.

### `rules`
Lists all available lint rules and their tokens - the same data that generates [RULES.md](RULES.md), but queryable from the CLI.
