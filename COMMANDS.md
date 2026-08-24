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

### `duplicate`
A two-phase, cross-file, whole-corpus clone detector - answers "where is this function body copy-pasted elsewhere in the codebase," not "is this specific construct a code smell" (that's what `lint`'s duplication-pattern rules, e.g. `duplicate-branch-code`, are for). Supports three detection algorithms (`hash`/`winnow`/`vector`).

### `astdump`
Parses a single file and dumps its raw Gosu AST tree - a debugging aid for understanding how the parser represents a given construct before writing a rule against it. Takes the file via `--file` (not positional) and can filter to one function via `--function`; emits JSON via `--format json`.

### `emit`
Compiles Gosu source files and writes `.class` files to disk (including inner/block/lambda classes), reusing the same `--include`/`--init-src-root`/`--additional-jars-file` setup as `lint`/`quality` - a one-step way to get `.class` files without invoking `gosuc` yourself, as long as that setup is correct.

## Diagnostics & Setup

### `diagnose`
Verifies the Gosu runtime environment without running a full scan: reports which `--additional-jars` loaded successfully vs. are missing, probes specific type names for resolution (pass via repeated `--type`), and lists which lint rules improve in accuracy once those types resolve. Useful for debugging `ErrorType`/type-incomplete results before a real run.

### `init-config`
Dumps every rule's default configuration to a `.properties` template - a starting point for customizing thresholds/severities without hand-writing the file from scratch.

### `rules`
Lists all available lint rules and their tokens - the same data that generates [RULES.md](RULES.md), but queryable from the CLI.
