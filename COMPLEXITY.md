# Complexity Analysis

This document covers the complexity metrics available in gslint. These are measurements rather than rules-they report complexity scores rather than flagging violations. Complexity metrics are accessed via the separate `complexity` command, not the `lint` command.

## Cyclomatic Complexity

Counts the number of independent paths through a function. Each branch point (`if`, `else if`, `for`, `while`, `catch`, `case`, ternary `?:`) adds 1. The default threshold is **10**.

By default, lambda bodies are excluded from the count. Use `--metric-config cyclomatic:includeLambdas=true` to include lambda bodies-useful when you want to measure all code paths including nested lambdas.

```bash
# Default: excludes lambda bodies
java -jar gslint.jar complexity src --init-src-root src --metric cyclomatic

# Include lambda bodies in the count
java -jar gslint.jar complexity src --init-src-root src --metric cyclomatic --metric-config "cyclomatic:includeLambdas=true"
```

## Cognitive Complexity

Measures complexity differently than cyclomatic: nesting multiplies the increment. A deeply nested `if` costs more than a flat one. This better captures perceived complexity. The default threshold is **15**.

### cognitive

```bash
java -jar gslint.jar complexity src --init-src-root src --metric cognitive
```

## Ranking Functions by Complexity (`--limit N`)

When analyzing an entire project or folder, use `--limit N` to see the **top N most complex functions ranked across all scanned files**. This is useful for identifying hotspots and prioritizing refactoring efforts.

```bash
# Show the top 10 most complex functions (cyclomatic) across the entire src/ tree
java -jar gslint.jar complexity --pattern "src/**/*.gs" --init-src-root src --metric cyclomatic --limit 10

# Show the top 5 most complex functions (cognitive) in JSON format
java -jar gslint.jar complexity --pattern "src/**/*.gs" --init-src-root src --metric cognitive --limit 5 --format json

# Combine with --min-score to filter before ranking (excludes functions below the threshold)
java -jar gslint.jar complexity --pattern "src/**/*.gs" --init-src-root src --metric cyclomatic --limit 20 --min-score 8
```

**Output (text format):**
```
-- Top 10 Cyclomatic Complexity across 25 files --

  Rank  Score  File                       Function
  ──────────────────────────────────────────────────────────────
     1     24  OrderService.gs            processOrder(Order,User)@142
     2     19  PaymentProcessor.gs        executePayment(Payment)@87
     3     15  ReportGenerator.gs         buildReport(String)@33
     ...
```

**Notes:**
- `--limit` and `--explain` are mutually exclusive (pick one)
- `--min-score` filters functions before ranking (only counted functions >= score)
- `--limit 0` (default) disables ranking and shows per-file results

<!-- BEGIN:cyclomatic -->
<details>
<summary>cyclomatic complexity examples</summary>

Calculates cyclomatic complexity for every function in a Gosu source file.

Cyclomatic complexity starts at 1 per function and increments by 1 for each decision
point: `if`, `while`, `do-while`, `for`/`foreach`,
`case` clause, `catch` clause, ternary (`?:`), `&&`, and `||`.

Lambda/block expressions (`IBlockExpression`) are
handled according to the `includeLambdaComplexity` flag passed to the constructor:

  - `true` - decision points inside lambdas are counted and added to the
      enclosing function's total (used by the `complexity` CLI command).
  - `false` - lambda bodies are not descended into; only the lambda count is
      tracked (used by the `complexity-no-lambdas` variant).

Results are returned as a `ComplexityResults` containing one `ComplexityMetrics`
entry per function, keyed internally by a collision-safe string of the form
`functionName(paramTypes)@lineNumber`. Display output strips this to just
the function name via `displayName(key)`.
Inner classes are processed recursively; anonymous classes are skipped.

### Examples

```gosu
// FAIL: cyclomatic complexity = 6
function complexFunction(a : int, b : int) : int {
  var result = 0
  if (a > 0) {
    result += a
  } else if (b > 0) {
    result += b
  }
  for (i in 1..10) {
    if (i % 2 == 0) {
      result += i
    }
  }
  while (result < 100) {
    result += 10
  }
  return result
}

// PASS: cyclomatic complexity = 1
function simpleFunction(x : int) : int {
  return x > 0 ? x : -x
}
```

</details>
<!-- END:cyclomatic -->

<!-- BEGIN:cognitive -->
<details>
<summary>cognitive complexity examples</summary>

Calculates SonarSource-style Cognitive Complexity for each member function in a Gosu class.

Based on the
SonarSource Cognitive Complexity specification.

When `includeLambdaComplexity` is true, complexity inside lambda/block expressions
is included and lambdas act as nesting contributors (B2). When false, lambda subtrees are
skipped entirely.

### Examples

```gosu
// FAIL: nesting multiplies cost
function deeplyNested(x : int, y : int) : int {
  var total = 0
  if (x > 0) {                         // +1
    while (y > 0) {                    // +2 (nesting=1)
      if (y % 2 == 0) {               // +3 (nesting=2)
        total += 10
      }
      y--
    }
  }
  return total                         // cognitive score: 6
}

// PASS: flat conditions, low nesting cost
function flatConditions(a : int, b : int) : String {
  if (a == 1) { return "one" }         // +1
  if (b == 2) { return "two" }         // +1
  return "other"                        // score: 2
}
```

</details>
<!-- END:cognitive -->

---

## Drill-Down: Why Is This Function Complex?

Two commands let you inspect the exact lines that drive a function's score. Both rely on the same increment-tracking infrastructure built into the complexity calculators-every decision point records its source line, node type, amount, and reason alongside the total score.

---

### `complexity --explain` - annotated source

Prints the raw source lines of the function with each complexity-contributing line annotated on the right. A running total in `[N]` shows how the score accumulates as you read down.

```bash
# Annotate every function in a file
java -jar gslint.jar complexity src/CyclomaticComplexityExample.gs \
    --init-src-root src/ --metric cyclomatic --explain

# Narrow to one function (substring match)
java -jar gslint.jar complexity src/CyclomaticComplexityExample.gs \
    --init-src-root src/ --metric cyclomatic --explain --function complexFunction

# Cognitive complexity, with lambdas included
java -jar gslint.jar complexity src/CognitiveComplexityExample.gs \
    --init-src-root src/ --metric cognitive --explain \
    --metric-config "cognitive:includeLambdas=true"
```

**Output** - `complexFunction` from `CyclomaticComplexityExample.gs` (`--metric cyclomatic`):

```
CyclomaticComplexityExample.gs · complexFunction(int, int)@30 · cyclomatic = 6

 30 │ function complexFunction(a : int, b : int) : int {
 31 │   var result = 0
 32 │
 33 │   if (a > 0) {                                     // +1   if                             [2]
 34 │     result += a
 35 │   } else if (b > 0) {                              // +1   if                             [3]
 36 │     result += b
 37 │   }
 38 │
 39 │   for (i in 1..10) {                               // +1   foreach                        [4]
 40 │     if (i % 2 == 0) {                              // +1   if                             [5]
 41 │       result += i
 42 │     }
 43 │   }
 44 │
 45 │   while (result < 100) {                           // +1   while                          [6]
 46 │     result += 10
 47 │   }
 48 │
 49 │   return result
 50 │ }
```

**Output** - `example` from `CognitiveComplexityExample.gs` (`--metric cognitive`):

```
CognitiveComplexityExample.gs · example · cognitive = 13

 10 │ function example(x : int, y : int) : int {
 11 │   var total = 0
 12 │
 14 │   if (x > 0) {                                     // +1   if                             [2]
 16 │   } else if (x < 0) {                              // +1   else-if                        [3]
 19 │   } else {                                         // +1   else                           [4]
 25 │   while (y > 0) {                                  // +1   while                          [5]
 27 │     if (y % 2 == 0) {                              // +2   if (nesting=1)                 [7]
 34 │   total += (x == 1 ? 1 : 2)                        // +1   ternary                        [8]
 37 │   if ((x > 0 && y > 0 && total > 0) || (x == 0)) { // +3   if; &&; ||                    [11]
 43 │     if (x == 99) {                                 // +1   if                             [12]
 46 │   } catch (e : RuntimeException) {                 // +1   catch                          [13]
 58 │   return total
 59 │ }
```

Reading the `[N]` column: the score starts at 1 (the per-function base), increases at each annotated line, and equals the reported total at the last annotation. Lines with no annotation contribute nothing.

For cognitive complexity, multiple increments on the same line are merged into one annotation (line 37 above combines `if`, `&&`, and `||`).

---

### `astdump --annotate-complexity` - annotated AST tree

Overlays complexity annotations on the parse-tree structure. Useful for seeing exactly which AST node the linter is counting-particularly helpful when `&&`/`||` chains, ternary expressions, or `catch` clauses are easy to miss at the source level.

```bash
# Full class tree, cyclomatic annotations
java -jar gslint.jar astdump \
    --file src/CyclomaticComplexityExample.gs \
    --init-src-root src/ \
    --annotate-complexity cyclomatic

# Drill into one function (substring match against function label)
java -jar gslint.jar astdump \
    --file src/CyclomaticComplexityExample.gs \
    --init-src-root src/ \
    --function complexFunction \
    --annotate-complexity cyclomatic

# Cognitive, with lambdas, JSON output
java -jar gslint.jar astdump \
    --file src/CognitiveComplexityExample.gs \
    --init-src-root src/ \
    --function example \
    --annotate-complexity cognitive \
    --metric-config "cognitive:includeLambdas=true" \
    --format json
```

**Output** - `complexFunction` with `--annotate-complexity cyclomatic`:

Each `IFunctionStatement` node shows the function's total score. Every decision-point node shows `← +N  reason`.

```
AST Dump: CyclomaticComplexityExample.gs
IFunctionStatement  function complexFunction() -> int  [line 30]  ← cyclomatic=6
├─ IVarStatement  var result  [line 31]
├─ IIfStatement  [line 33]  ← +1  if
│  ├─ IRelationalExpression  [line 33]
│  ├─ IStatementList  [line 33]
│  │  └─ IAssignmentStatement  [line 34]
│  └─ IIfStatement  [line 35]  ← +1  if
│     ├─ IRelationalExpression  [line 35]
│     └─ IStatementList  [line 35]
│        └─ IAssignmentStatement  [line 36]
├─ IForEachStatement  [line 39]  ← +1  foreach
│  └─ IStatementList  [line 39]
│     └─ IIfStatement  [line 40]  ← +1  if
│        ├─ IRelationalExpression  [line 40]
│        └─ IStatementList  [line 40]
│           └─ IAssignmentStatement  [line 41]
├─ IWhileStatement  [line 45]  ← +1  while
│  └─ IStatementList  [line 45]
│     └─ IAssignmentStatement  [line 46]
└─ IReturnStatement  [line 49]
```

**Output** - `example` with `--annotate-complexity cognitive`:

The cognitive tree makes nesting costs visible: `IIfStatement [line 27]` is annotated `+2` because it sits one level inside the `while`. The `||` and `&&` chains on line 37 each appear as separate annotated nodes. `ICatchClause` appears as a sibling of the try body-`try` itself carries no annotation because it is not a nesting contributor.

```
AST Dump: CognitiveComplexityExample.gs
IFunctionStatement  function example() -> int  [line 10]  ← cognitive=13
├─ IVarStatement  var total  [line 11]
├─ IIfStatement  [line 14]  ← +1  if
│  ├─ IRelationalExpression  [line 14]
│  ├─ IStatementList  [line 14]
│  └─ IIfStatement  [line 16]  ← +1  else-if
│     ├─ IRelationalExpression  [line 16]
│     └─ IStatementList  [line 16]
├─ IWhileStatement  [line 25]  ← +1  while
│  └─ IStatementList  [line 25]
│     └─ IIfStatement  [line 27]  ← +2  if (nesting=1)
│        ├─ IRelationalExpression  [line 27]
│        └─ IStatementList  [line 27]
├─ IConditionalTernaryExpression  [line 34]  ← +1  ternary
├─ IIfStatement  [line 37]  ← +1  if
│  └─ IConditionalOrExpression  [line 37]  ← +1  ||
│     ├─ IConditionalAndExpression  [line 37]  ← +1  &&
│     └─ IRelationalExpression  [line 37]
├─ ITryCatchFinallyStatement  [line 42]
│  ├─ IStatementList  [line 42]
│  │  └─ IIfStatement  [line 43]  ← +1  if
│  └─ ICatchClause  catch (RuntimeException e)  [line 46]  ← +1  catch
└─ IReturnStatement  [line 58]
```

#### Reading the tree output

| Annotation | Meaning |
|---|---|
| `← cyclomatic=6` on `IFunctionStatement` | Total cyclomatic score for this function |
| `← cognitive=13` on `IFunctionStatement` | Total cognitive score for this function |
| `← +1  if` | This node contributes 1 to the score; reason: if-branch |
| `← +2  if (nesting=1)` | Nested inside one nesting contributor; costs 1 + 1 = 2 |
| `← +1  &&` | First `&&` in a new logical-operator sequence |
| `← +1  catch` | Catch clause, nesting=0; costs 1 |

Nodes with no annotation (`IRelationalExpression`, `IStatementList`, `IReturnStatement`, `IVarStatement`, etc.) are structural-they contribute nothing to the score.

#### `--function` filter

`--function` does a substring match against the function's label in the AST (e.g., `--function process` matches `function processOrder() -> void`). If no match is found, the command lists available function names and exits. Works identically for both `complexity --explain` and `astdump --annotate-complexity`.

#### JSON output

With `--format json` (astdump only), annotated nodes gain a `"complexity"` field:

```json
{
  "type": "IFunctionStatement",
  "label": "function complexFunction() -> int",
  "line": 30,
  "col": 2,
  "complexity": "cyclomatic=6",
  "children": [
    {
      "type": "IIfStatement",
      "line": 33,
      "col": 4,
      "complexity": "+1  if",
      "children": [ ... ]
    }
  ]
}
```

#### `--metric-config includeLambdas`

Both commands accept `--metric-config` using the same format as the regular `complexity` command:

```bash
--metric-config "cyclomatic:includeLambdas=true"
--metric-config "cognitive:includeLambdas=true"
```

When lambdas are included, `IBlockExpression` nodes become nesting contributors in cognitive mode, and decision points inside lambdas are counted and annotated in both modes.
