# gslint - Gosu Linter

Source-level static analysis for the [Gosu](https://gosu-lang.github.io/) programming language. Parses `.gs`, `.gsx`, `.gst`, and `.gsp` source files using the Gosu compiler's own AST API (`gw.lang.parser.*`) and reports violations in Text, JSON, SARIF.

- Tested Gosu versions: 1.14.16, 1.14.29, 1.15.7, 1.17.13, **1.18.5**, 1.18.7
- Gosu has been a stable language for the past 10+ years so there should be flexibility with running a linter built for **1.18.5** against files designed for a previous version
- Java 11 is required to build (keeps parity with `gosu-core`); developed on IntelliJ 2024.1.5 to match IDE recommendation of popular Gosu application platforms
- Verified on Java 17 with Gosu 1.17.13, **1.18.5**, 1.18.7
- `gslint.jar` bundles `gosu-core`; also supports running without the bundled `gosu-core`

> **Rule philosophy:** This project ships a curated set of language-level rules. It serves as a foundational component for larger code quality efforts and will not reach feature parity with general-purpose linters. If you're building a Gosu framework or application platform, write custom rules targeting your domain APIs.

## How It Works

The linter depends directly on `gosu-core`. `GosuClassParser.parseFull()` produces a `IGosuClass` AST with fully resolved symbols; rules then walk that hierarchy.
Some complex rules use Complex rules that require type information (e.g., `raw-type`, `identity-return`) can safely navigate the AST and inspect types without worrying about missing information or API instability.
This project tries to use just the `gw.lang.*` interface hierarchy but occasional dips into `gw.internal.lang.*` which has been stable for many years.
If `gosu-core` ever bumps the minimal Java version, the `gslint` will update to match it.

## ErrorType and Type Resolution

When the Gosu compiler cannot resolve a referenced type because its JAR is not on the classpath, it substitutes `gw.lang.reflect.IErrorType`.
Rules that inspect types will see `ErrorType` instead of the real type, which means type-dependent rules (e.g., `raw-type`, `res00g-*`) may silently miss violations or produce lower-confidence results.

### Specifying source roots: `--init-src-root`

Specify the source root directory where Gosu files are located. This is required for the Gosu runtime to properly resolve types and package structures:

```bash
java -jar gslint.jar --init-src-root src
```

```bash
java -jar gslint.jar --pattern "src/**/*.gs" --init-src-root src
```

For multiple source roots (e.g., separate source and test directories), specify `--init-src-root` multiple times:

```bash
java -jar gslint.jar --patterns "src/**/*.gs","test/**/*.gs" --init-src-root src --init-src-root test --additional-jars-file dependencies.txt
```

### Providing dependency JARs: `--additional-jars-file` (strongly recommended)

Pass a plain-text file with one relative or absolute JAR path per line (or semicolons between paths on the same line):

```
lib/foo.jar
lib/bar.jar
```

> **Local Maven repositories:** It's not recommend to generate the file by including all JAR files from a local Maven repository. Local repositories may track multiple versions of the same JAR. You should determine the runtime dependencies for your project and only include those JARs.

The linter reports a resolution confidence summary at the end of each run:

```
-- Resolution Confidence --
Fully Resolved:   42 violations ( 87%)
Type-Incomplete:   6 violations ( 13%)
Tip: Fix imports & re-run for complete analysis
```

Individual violations are flagged with `[⚠ type-incomplete]` in text output and `"fullyResolved": false` in JSON.

**Recommendation:** Treat type-incomplete violations as lower-confidence findings.

## Rule Reference

See [RULES.md](RULES.md) for a comprehensive reference of all 70+ linting rules. For complexity metrics (cyclomatic and cognitive complexity), see [COMPLEXITY.md](COMPLEXITY.md).

## Command-Line Usage

```bash
# Common case: analyze a source directory recursively
java -jar gslint.jar --pattern src

# Common case: pass a glob pattern directly (quote to prevent shell expansion)
java -jar gslint.jar --pattern "src/**/*.gs"

# Single file
java -jar gslint.jar --pattern src/foo/Bar.gs

# Mixed: multiple patterns and a single file
java -jar gslint.jar --pattern "src/main/**/*.gs" --pattern "src/integration/**/*.gs" --pattern src/Util.gs

java -jar gslint.jar --pattern "/c/tmp/src/com.example/**/*.gs" --init-src-root /c/tmp/src
```

### Path separators in patterns (Windows)

Use **forward slashes** (`/`) in all patterns. They work identically on Windows, Linux, and macOS, making patterns portable across CI environments without modification.

```bash
# Recommended - portable everywhere
java -jar gslint.jar "src/**/*.gs" --init-src-root src

# Also accepted on Windows, but not portable to Unix CI and not religiously tested
java -jar gslint.jar "src\**\*.gs" --init-src-root src
```

### Running without the uber JAR

When you want to avoid the uber JAR - for example, when embedding the linter in a larger build that already has gosu-core on the classpath - use `-cp` to assemble the classpath yourself:

```bash
# With your own dependency JARs on the classpath for full type resolution
java -cp "gslint.jar:gosu-core-1.18.5.jar:gosu-core-api-1.18.5.jar:<other jars>:lib/myapp.jar" org.gslint.Mainn --pattern src --init-src-root src

# Windows (semicolon separators)
java -cp "gslint.jar;gosu-core-1.18.5.jar;gosu-core-api-1.18.5.jar;<other jars>;lib/myapp.jar" org.gslint.Mainn --pattern src --init-src-root src
```

### Rule selection

```bash
# Run only specific rules
java -jar gslint.jar --pattern src --rules empty-catch,dead-assignment

# Run all rules except the listed ones
java -jar gslint.jar --pattern src --skip-rules duplicate-code,complexity

# Mix built-in and custom rules
java -jar gslint.jar --pattern src --rules "empty-catch,com.example.MyRule,dead-assignment"
```

### Output formats

```bash
# Default text output
java -jar gslint.jar --pattern src

# JSON - suitable for CI result parsing
java -jar gslint.jar --pattern src --format json

# SARIF - for GitHub Code Scanning, VS Code, and other SARIF consumers
java -jar gslint.jar --pattern src --format sarif
```

All of the linter’s output go to `stdout`, so pipe it into a file (`... > report.txt`) or into another processor such as `jq` when you need structured filtering or CI artifact capture.

### Progress tracking

For long-running analyses, use `--show-progress` to track analysis progress in real time. Progress is printed to `stderr` every ~10% of files and a final summary showing total files analyzed and violations found.

```bash
# Enable progress output (useful for CI and interactive runs)
java -jar gslint.jar --pattern src --show-progress

# Combine with JSON output - progress goes to stderr, JSON to stdout
java -jar gslint.jar --pattern src --format json --show-progress > report.json
```

Progress output is sent to `stderr` so it doesn’t interfere with structured output (`--format json` or `--format sarif`) piped to `stdout`.

### Rule configuration via command line

```bash
# Single rule
java -jar gslint.jar --pattern src --rule-config complexity:maxComplexity=8

# Multiple rules: semicolons separate rules, commas separate key=value pairs
java -jar gslint.jar --pattern src \
  --rule-config "empty-catch:allowCommentedCatch=true;too-many-args:maxArgs=4;too-many-returns:maxReturns=4"

# Multiple --rule-config flags (equivalent to the semicolon form)
java -jar gslint.jar --pattern src \
  --rule-config complex-boolean:maxOperators=2 \
  --rule-config excessive-newlines:maxBlankLines=2 \
  --rule-config cognitive-complexity:maxComplexity=10
  
# Single rule with --skip-rules (other rules still run)
java -jar gslint.jar src --skip-rules complexity
```

### Rule configuration via properties file

```bash
cat <<'EOF' > /tmp/lint.properties
# this is a comment
rule.empty-catch.allowCommentedCatch=true
rule.too-many-args.maxArgs=4
rule.too-many-returns.maxReturns=4
rule.complex-boolean.maxOperators=2
rule.complexity.maxComplexity=8
rule.cognitive-complexity.maxComplexity=10
rule.excessive-newlines.maxBlankLines=2
rule.too-many-throws.maxThrows=2
EOF
java -jar gslint.jar Give a--init-src-root src --config lint.properties
```

### Custom severity

Each rule violation can carry a custom severity label. Severity values are free-form strings - define whatever scheme fits your project (e.g., `critical/high/low`, `blocking/warning/info`, `P0/P1/P2`).

**Priority:** CLI `--rule-config` overrides properties file; properties file overrides the `@RuleToken` annotation default on a custom rule.

**Via command line:**

```bash
# Mark a single rule's violations as "critical"
java -jar gslint.jar --init-src-root src --rule-config complexity:severity=critical

# Multiple rules with different severities (combine with other config keys freely)
java -jar gslint.jar --init-src-root src \
  --rule-config "empty-catch:severity=blocking;dead-assignment:severity=warning;print-statement:severity=info"

# Severity and a threshold in the same rule-config entry
java -jar gslint.jar --init-src-root src \
  --rule-config "complexity:maxComplexity=8,severity=critical"
```

**Via properties file:**

```properties
# lint.properties

# Per-rule severity overrides
rule.empty-catch.severity=blocking
rule.dead-assignment.severity=warning
rule.print-statement.severity=info
# severity can be combined with other configurable keys for the same rule
rule.complexity.maxComplexity=8
rule.complexity.severity=critical

# Skip rules - comma-separated tokens; glob patterns are supported
# Merged with --skip-rules before rule resolution.
# Ignored when --rules is active on the CLI (explicit include-list takes precedence).
skip-rules=duplicate-code,print-statement,manifold*
```

```bash
java -jar gslint.jar --config lint.properties
```

Severity is surfaced in all three output formats:

- **Text:** `[severity: critical]` appended to the violation line
- **JSON:** `"severity": "critical"` field on each violation object
- **SARIF:** the severity value is used as the SARIF `level` field

**Via `@RuleToken` (for custom rules):** Set a compile-time default severity in the annotation. It applies to every violation from that rule and can be overridden at runtime via `--rule-config` or a properties file.

```java
@RuleToken(value = "my-rule", displayName = "My Rule", severity = "high")
public class MyRule extends BaseRule<MyRule.Violation> { ... }
```

### Type resolution via classpath style file (`:` for Linux, `;` for Windows -or- newline separated)

```bash
cat <<'EOF' > /tmp/dependencies.txt
libs/jackson-databind-0.0.0.jar
libs/jackson-core-0.0.0.jar
libs/jackson-annotations-0.0.0.jar
EOF
java -jar gslint.jar --additional-jars-file /tmp/dependencies.txt
```

```bash
cat <<'EOF' > /tmp/dependencies.txt
# this is a comment
libs/jackson-databind-0.0.0.jar
libs/jackson-core-0.0.0.jar
libs/jackson-annotations-0.0.0.jar
// this is a comment too
EOF
java -jar gslint.jar --pattern src/ --additional-jars-file /tmp/dependencies.txt
```

### All flags

| Flag | Description | Default |
|------|-------------|---------|
| `--rules <list>` | Comma-separated rule tokens (or fully-qualified class names) to run. Supports glob patterns: `manifold*`, `*expansion*` | All rules |
| `--skip-rules <list>` | Comma-separated rule tokens to exclude. Supports glob patterns: `manifold*`, `*expansion*` | - |
| `--rules-only-custom` | Skip all built-in rules; use only with `--rules` to specify custom rules (incompatible with `--skip-rules`) | `false` |
| `--enable-experimental-rules` | Enable experimental Manifold-awareness rules | `false` |
| `--config <file>` | Properties file with rule configuration | - |
| `--rule-config <str>` | Inline rule configuration (`rule:key=value;rule2:key=value`) | - |
| `--format <fmt>` | Output format: `text`, `json`, `sarif` | `text` |
| `--init-src-root <path>` | Source root for Gosu runtime/type resolution. Required to properly resolve types and package structures. Can be specified multiple times for multiple roots. | - |
| `--additional-jars-file <file>` | JAR paths for type resolution (one per line or semicolon-separated) | - |
| `--include-generated` | Analyze `@Generated` classes (skipped by default) | `false` |
| `--show-progress` | Print progress every ~10% of files and a final summary | `false` |
| `--log-level <level>` | Logger level: `SEVERE`, `WARNING`, `INFO`, `FINE`, `FINER`, `FINEST` | `INFO` |
| `--help` | Print usage and exit | - |

## Extending with Custom Rules

This project ships a focused set of rules for open-source Gosu. If you work on a platform built on top of Gosu you will almost certainly have patterns worth linting that are invisible to general-purpose rules. Write those rules yourself and load them alongside the built-in ones.

Add `gslint.jar` to your compile classpath, extend `BaseRule<V extends Violation>`, annotate the class with `@RuleToken`, and package your rules into a separate JAR.

The runner distinguishes built-in tokens from custom class names by the presence of a `.` - anything containing a `.` is treated as a fully-qualified class name and loaded via reflection.

```bash
# Custom rule only (skip all built-ins explicitly)
java -jar gslint.jar --init-src-root src \
  --rules com.example.rules.MyRule \
  --rules-only-custom

# Mix built-in tokens and custom class names freely
java -jar gslint.jar --init-src-root src \
  --rules "empty-catch,com.example.rules.MyRule,dead-assignment"

# --skip-rules works the same way
java -jar gslint.jar --init-src-root src \
  --skip-rules com.example.rules.MyRule

# Custom rule with configuration
java -jar gslint.jar --init-src-root src \
  --rules com.example.rules.MyRule \
  --rules-only-custom \
  --rule-config "com.example.rules.MyRule:threshold=5"
```

Your JAR must be on the classpath at runtime. Use `-cp` or merge it with the uber JAR at build time:

```bash
# Unix: colon-separated classpath
java -cp "gslint.jar:my-rules.jar" org.gslint.Main \
  --init-src-root src --rules com.example.rules.MyRule

# Windows: semicolon-separated classpath
java -cp "gslint.jar;my-rules.jar" org.gslint.Main \
  --init-src-root src --rules com.example.rules.MyRule

# With explicit gosu JARs (no uber JAR)
java -cp "gslint.jar:/lib/gosu-core-1.18.5.jar:gosu-core-api-1.18.5.jar:<other jars>:my-rules.jar" \
  org.gslint.Main \
  --init-src-root src --rules com.example.rules.MyRule
```

## Notes

- Portions of this project were generated with the assistance of coding agents