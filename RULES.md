# Rules Reference

> **Rule philosophy:** This project ships a curated set of language-level rules that work across any Gosu codebase. It serves as a foundational component for larger code quality efforts and will not reach feature parity with general-purpose linters. If you're building a Gosu framework or application platform, write custom rules targeting your domain APIs.

85+ rules across several categories, listed alphabetically. Each section shows the CLI rule token, what it detects (including any configurable options), and FAIL/PASS examples drawn from the test fixtures.

<!-- BEGIN:rules -->
### dataflow

| Token | Detects |
|-------|---------|
| `ctor-delegation-param` | Flags constructor parameters not forwarded in explicit 'this(...)' delegation calls. |
| `dead-assignment` | Flags variables assigned but never read before being overwritten or going out of scope. |
| `identity-return` | Flags functions that perform work but return their input parameter unchanged. |
| `null-return` | Flags functions that return 'null' directly; prefer 'Optional' or an empty collection. |
| `unused-ctor-params` | Flags constructor parameters that are never read. |

<details>
<summary>ctor-delegation-param examples</summary>

Detects constructors that delegate to another overload via `this(...)`
without forwarding all of their parameters.

**Purpose:** Catch copy-paste errors where a constructor accepts a parameter
but hardcodes a value in the delegation call instead of forwarding the parameter:

```gosu
// BUG: askForConfirmation param is ignored; 'false' is hardcoded
construct(askForConfirmation : boolean) {
  this(Location.DEFAULT, false)   // should be: this(Location.DEFAULT, askForConfirmation)
}

construct(target : Location, askForConfirmation : boolean) {
  // real implementation
}
```

**Scope:** This rule checks that parameters either appear in the delegation
arguments OR are referenced elsewhere in the constructor body (e.g., assigned to
fields, logged, transformed). A parameter used in the body is not a forgotten
forwarding - it is intentionally consumed by the delegating constructor.

Example that WILL be flagged (parameter not in delegation AND not used in body):
```gosu
construct(foo : Foo) {
  this(Widget.DEFAULT)  // foo received but never used - likely a copy-paste bug
}
```

Example that will NOT be flagged (parameter not in delegation, but IS used in body):
```gosu
construct(foo : Foo) {
  this()               // delegates to no-arg overload for base init
  _foo = foo           // parameter consumed here - intentional
}
```

**Note:** This rule only checks `this(...)` delegation calls,
not `super(...)`, since parent constructors may have different signatures.

**AST quirk:** `dfs.getInitializer()` returns non-null for every
constructor - including those with no explicit delegation - because Gosu injects a
synthetic implicit `super()` call. The check below uses
`IConstructorFunctionSymbol` + `ISymbol.THIS` to distinguish a real
`this(...)` call from the synthetic `super()`.

</details>
<details>
<summary>dead-assignment examples</summary>

Flags local variables that are assigned a non-trivial value which is then
overwritten before being read - the first assignment was wasted.

### Flagged - non-trivial value assigned and immediately overwritten
```gosu
function example() {
  var result = computeResult()    // VIOLATION: overwritten before use
  result = computeOtherResult()
  return result
}
```

### Not flagged - trivial initializer (null, 0, false, "")
```gosu
function example() {
  var result : String = null      // OK: trivial sentinel initializer
  result = computeResult()
  return result
}
```

### Not flagged - variable is read before reassignment
```gosu
function example() {
  var x = computeResult()
  print(x)                        // read here - clears pending
  x = computeOtherResult()        // no dead assignment
}
```

### Not flagged - compound assignment reads the variable
```gosu
function example() {
  var x = computeResult()
  x = x + "_suffix"               // x is read on the RHS - not dead
}
```

### Algorithm

Walks the top-level statements of each function body in document order,
maintaining a map of variable name → most recent unread write. When a
variable is written again before it is read, the earlier write is flagged.
Control-flow statements (if, for, while, try, switch) conservatively flush
any variable they touch from the pending map - assignments inside branches
are not flagged to avoid false positives.

</details>
<details>
<summary>identity-return examples</summary>

Detects functions that accept a parameter, perform other work, and then return
that same parameter unchanged. This is likely a logic error where the computed
result was forgotten, e.g.:

```gosu
// BUG: does work but returns original param
function processNames(names : List) : List {
  var result = names.map(\n -> n.toUpperCase())  // work done
  return names   // VIOLATION: should be 'return result'
}
```

Pure pass-through functions with no other work are not flagged (intentional).

</details>
<details>
<summary>null-return examples</summary>

Detects functions whose declared return type is a `List` (or subtype) or an array,
and that contain a `return null` anywhere in their body.

Returning `null` from a collection-typed function is a common source of
`NullPointerException` at call sites that iterate the result without a null check.
The preferred alternatives are returning an empty collection or an empty array.

### Flagged - return type is List/array and body contains `return null`
```gosu
function getItems() : List {
  if (someCondition) {
    return null           // VIOLATION
  }
  return new ArrayList()
}

function getValues() : String[] {
  if (someCondition) {
    return null           // VIOLATION
  }
  return new String[0]
}
```

### Not flagged - empty collection returned instead
```gosu
function getItems() : List {
  if (someCondition) {
    return Collections.emptyList()    // OK
  }
  return items
}
```

### Not flagged - return type is not a collection or array
```gosu
function findFirst() : String {
  return null    // not checked - return type is not a collection/array
}
```

</details>
<details>
<summary>unused-ctor-params examples</summary>

Detects constructor parameters that are never assigned or used within the
constructor body.

A parameter is flagged when it does not appear as an identifier anywhere
inside the constructor body - i.e. it is declared but silently ignored:

```gosu
construct(name : String, value : int) {
  this._name = name    // OK - name is used
                       // VIOLATION - value is never referenced
}
```

Any reference to the parameter counts as "used": assignment to a field,
passing to a method, reading in an expression, etc.

```gosu
construct(name : String, value : int) {
  this._name = name
  this._value = value  // OK - both parameters used
}
```

</details>

### declarations

| Token | Detects |
|-------|---------|
| `delegate-member-conflict` | Flags class methods that shadow delegate-synthesized forwarding methods. |
| `duplicate-delegate` | Flags multiple delegate fields that represent the same interface. |
| `forbidden-imports` | Flags uses statements that import types from forbidden packages (supports wildcards like 'sun.*'). Requires 'packages' config key. *(disabled by default - activate with `--rules forbidden-imports`)* |
| `logger-static` | Flags logger fields that are not static (should be shared across instances). |
| `prefer-stringbuilder` | Flags StringBuffer usage; prefer StringBuilder which avoids unnecessary synchronization in single-threaded code. |
| `public-field` | Flags public instance fields; prefer private fields with public getter/setter. |
| `raw-type` | Flags use of raw generic types without type parameters (e.g., 'List' instead of 'List<T>'). |
| `static-hashmap` | Flags static HashMap/LinkedHashMap fields that are not thread-safe; prefer ConcurrentHashMap. |
| `strbuf-char-ctor` | Flags 'StringBuffer'/'StringBuilder' initialized with a 'char' (uses ASCII value, not char). |
| `threadlocal-static` | Flags 'ThreadLocal' fields that are not static (should be shared, not per-instance). |
| `too-many-args` | Flags functions with more than the configured number of parameters (default: 5). |

<details>
<summary>delegate-member-conflict examples</summary>

Detects delegate fields whose constituent interface methods conflict with
explicitly declared methods in the same class. When a class both declares
a method and delegates an interface containing the same method signature,
the explicit method silently shadows the delegated one, which can mask bugs
if the delegation was intended to forward calls.

### Flagged
```gosu
class Wrapper implements IFoo {
  delegate _inner represents IFoo
  function doWork() : String { return "local" }   // conflicts with IFoo.doWork()
}
```

### Not flagged
```gosu
class Wrapper implements IFoo {
  delegate _inner represents IFoo
  function helper() : String { return "ok" }   // no conflict with IFoo
}
```

Token: `delegate-member-conflict`

</details>
<details>
<summary>duplicate-delegate examples</summary>

Detects two or more `delegate` fields that represent the same interface,
which causes the Gosu compiler to synthesize duplicate forwarding methods for
all interface members. The last delegate wins, making the earlier one(s) dead code
and a source of confusion.

### Flagged
```gosu
delegate _a represents IFoo
delegate _b represents IFoo   // duplicate-delegate
```

### Not flagged
```gosu
delegate _a represents IFoo
delegate _b represents IBar   // distinct interfaces - OK
```

Token: `duplicate-delegate`

</details>
<details>
<summary>prefer-stringbuilder examples</summary>

Flags use of `StringBuffer` and recommends `StringBuilder` instead.

`StringBuffer` synchronizes every method on the instance monitor, which adds
overhead that is wasted in the overwhelmingly common case where the buffer is not
shared across threads. `StringBuilder` has an identical API but no
synchronization, making it faster in single-threaded code.

Only flag `StringBuffer` - not `StringBuilder` - since StringBuilder
is already the preferred type.

### Detection strategy

  - **Class-level fields** (via `.processVariable`): any field declared
      with type `StringBuffer`.
  - **Local variables** (within function/constructor bodies): any `var`
      statement whose declared or inferred type is `StringBuffer`.
  - **Standalone construction**: any `new StringBuffer(...)` expression
      whose line is not already covered by a local variable declaration above
      - catches `return new StringBuffer()` and argument-position usages without
      double-reporting lines like `var buf : StringBuffer = new StringBuffer()`.

### Flagged
```gosu
private var _buf : StringBuffer             // class field
var buf : StringBuffer = new StringBuffer() // local variable declaration
return new StringBuffer()                   // return without assignment
method(new StringBuffer())                  // argument without assignment
```

### Not flagged
```gosu
new StringBuilder()
var sb = new StringBuilder()
```

### Historical note - Matcher.appendReplacement / appendTail

In Java 8, `java.util.regex.Matcher.appendReplacement(StringBuffer, String)` and
`appendTail(StringBuffer)` only accepted `StringBuffer`, making it the only
context where `StringBuffer` was effectively required. Java 9 added `StringBuilder`
overloads for both methods. Since this project targets Java 11, no such forced use case
exists and the rule can safely flag all `StringBuffer` usage.

</details>
<details>
<summary>public-field examples</summary>

Detects class-level variables that are declared `public` instead of being
exposed through a private backing field with a `public` property accessor
(the idiomatic Gosu pattern).

### Flagged - bare public field
```gosu
public class Greeter {
  public var FirstName : String   // Noncompliant
}
```

### Not flagged - private field exposed via property alias
```gosu
public class Greeter {
  private var _firstName : String as FirstName   // OK
}
```

A field is considered compliant when it is `private` (or internal/protected)
AND declares a property name via the `as ` clause
(`IVarStatement.hasProperty()` returns `true`).

A field that is neither public nor has an `as` property clause
(e.g. a purely private backing field with no accessor) is also considered
compliant - the rule only fires on fields that are explicitly `public`.

</details>
<details>
<summary>raw-type examples</summary>

Detects class-level fields whose declared type is a generic class or interface
but is written without type parameters - a raw type.

### Flagged - raw type (no type arguments)
```gosu
private var _items : List          // Noncompliant - raw List
private var _lookup : Map          // Noncompliant - raw Map
```

### Not flagged - properly parameterised or non-generic
```gosu
private var _items : List  // OK - fully parameterised
private var _name  : String        // OK - String is not a generic type
private var _count : int           // OK - primitive
```

### Detection strategy

Uses the parse-tree approach rather than the type-system API alone, because the Gosu
compiler auto-parameterizes raw types (e.g. raw `List` → `List`) via
`TypeLord.deriveParameterizedTypeFromContext` before the symbol table is finalized.
Relying solely on `isParameterizedType()` would therefore produce false negatives.

Instead:

  - Obtain the `ITypeLiteralExpression` from `IVarStatement.getTypeLiteral()`.
  - Search its parse tree for an `ITypeParameterListClause` child. This clause is
      created during parsing (before auto-parameterization) and is present only when the
      programmer actually wrote `` in source.
  - If no clause is found, retrieve the `IType` from the literal expression
      (`typeLit.getType().getType()`) - this is the type as the parser first resolved
      it, before auto-parameterization may have replaced it.
  - If that type `isGenericType()`, the field is a raw generic declaration.

</details>
<details>
<summary>strbuf-char-ctor examples</summary>

Detects `new StringBuffer(char)` and `new StringBuilder(char)` constructions.

The `StringBuffer(int)` and `StringBuilder(int)` constructors accept an
initial capacity. Since `char` widens silently to `int` on the JVM,
passing a `char` literal or a `char`-typed expression compiles without error
but produces an empty buffer whose initial capacity equals the character's Unicode
code-point - not a buffer pre-loaded with that character.

### Detection strategy

For every `INewExpression` whose target type is `StringBuffer` or
`StringBuilder`:

  - If the sole argument is an `ICharLiteralExpression` (e.g. `'A'`),
      flag it immediately - this is the most common form of the bug.
  - Otherwise inspect the unwrapped argument expression's type; if it is `char`
      or `java.lang.Character`, flag it - this catches `char`-typed
      variable references that widen to `int`.

### Flagged
```gosu
new StringBuffer('A')        // char literal → capacity 65, not content "A"
new StringBuilder('\n')      // char literal → capacity 10, not a newline buffer
var c : char = 'X'
new StringBuffer(c)          // char variable → widened to int capacity
```

### Not flagged
```gosu
new StringBuffer("A")        // String constructor - correct
new StringBuffer(128)        // explicit int capacity - intentional
new StringBuffer()           // no-arg constructor - fine
```

</details>
<details>
<summary>too-many-args examples</summary>

Detects functions that declare more parameters than the configured limit.

Functions with too many parameters are a code-smell: they typically indicate
that the method is doing too much, or that a parameter object should be introduced.

The default limit is {@value #DEFAULT_MAX_ARGS}. A custom limit can be supplied
via the two-argument constructor.

### Not flagged (at default limit of 5)
```gosu
function doSomething(param1 : int, param2 : int, param3 : int,
                     param4 : String, param5 : long) { ... }  // OK - exactly 5
```

### Flagged
```gosu
function doTooMuch(a : int, b : int, c : int,
                   d : String, e : long, f : boolean) { ... }  // VIOLATION - 6 args
```

### Not flagged - overriding functions

Functions that override a supertype method are exempt: the signature is dictated
by the supertype contract, so the implementor has no freedom to reduce the argument
count.

</details>

### duplication

| Token | Detects |
|-------|---------|
| `redundant-block-code` | Flags unnecessary code blocks that could be removed without changing behavior. |

### exceptions

| Token | Detects |
|-------|---------|
| `catch-null-reference` | Flags catch blocks that catch 'NullPointerException' or 'NullReferenceException'. |
| `debug-only-catch` | Flags catch blocks containing only debug/trace log calls, which are (usually) silent in production and equivalent to empty catches. |
| `empty-catch` | Flags catch blocks with no statements (swallowed exceptions). |
| `exception-swallowed-in-finally` | Flags return or throw inside finally blocks that silently discard propagating exceptions. |
| `rethrow-catch` | Flags redundant 'catch'/'rethrow' patterns that should use 'throws' instead. |
| `too-many-throws` | Flags functions with more than the configured number of declared exceptions (default: 3). |

<details>
<summary>catch-null-reference examples</summary>

Flags `catch` blocks that catch `NullPointerException` or
`NullReferenceException`.

Catching a null-reference exception almost always masks a programming
error (an unexpected null). The correct fix is to guard with an explicit
`!= null` check, use the null-safe operator (`?.`), or use the
elvis operator (`?:`) before the value is dereferenced.

Example violation:
```gosu
try {
  var name = getRecord().Name   // may be null
} catch (e : NullPointerException) {  // VIOLATION - hide the bug instead of fixing it
  // ...
}
```

Preferred alternatives:
```gosu
var record = getRecord()
if (record != null) { var name = record.Name }

var name = getRecord()?.Name        // null-safe call
var name = getRecord()?.Name ?: ""  // elvis fallback
```

</details>
<details>
<summary>debug-only-catch examples</summary>

Flags catch blocks that contain only `log.debug()` or `log.trace()` calls.

Debug and trace logging is typically disabled in production, so a catch block that
contains only these calls is functionally silent - identical to an empty catch block
from a diagnostic perspective. The `empty-catch` rule does not fire on these
because they contain a method call, but the exception is still effectively swallowed.

### Flagged
```gosu
} catch (e : Exception) {
  log.debug("parse failed: " + e.getMessage())  // VIOLATION: silent in production
}
```

### Not flagged
```gosu
} catch (e : Exception) {
  log.warn("parse failed", e)   // OK: warn/info/error visible in production
}
} catch (e : Exception) {
  log.debug("rethrowing", e)
  throw new RuntimeException(e) // OK: throw is meaningful regardless of log level
}
```

### Detection

A catch block is flagged when:

  - It is not structurally empty (that is `empty-catch`'s responsibility).
  - It contains at least one `log.debug()` or `log.trace()` call.
  - It contains no production-meaningful statement: no throw, return,
      assignment, or non-debug/trace method call. Variable declarations alone do not
      count as meaningful (consistent with `empty-catch` semantics).

Logger receivers are identified by the same two-pronged strategy used in
`UnguardedLogCallRule`: resolved type name contains "log" (e.g.
`org.slf4j.Logger` → simple name `Logger`), falling back to the
receiver expression's text when the type is unresolved.

</details>
<details>
<summary>exception-swallowed-in-finally examples</summary>

Flags `return` or `throw` statements inside `finally` blocks.

A `return` or `throw` in a `finally` block silently discards
any exception propagating from the `try` or `catch` block. The original
failure disappears with no trace, making bugs extremely hard to diagnose.

```gosu
// BAD - the IOException from the try block is silently discarded
function readFile(path : String) : String {
  try {
    return FileUtil.readFile(path)  // throws IOException
  } finally {
    return ""                        // swallows the exception
  }
}
```

Token: `exception-swallowed-in-finally`

</details>
<details>
<summary>rethrow-catch examples</summary>

Detects `catch` blocks that do nothing except rethrow the caught exception.

A catch block is flagged when its body consists solely of a single
`throw` statement that references the catch parameter - i.e. it adds no
logging, wrapping, or other meaningful action before propagating the exception:

```gosu
try {
  doSomething()
} catch (ex : Exception) {
  throw ex          // VIOLATION - pure rethrow, no value added
}
```

Catch blocks that wrap the exception before rethrowing are not flagged:

```gosu
} catch (ex : Exception) {
  throw new RuntimeException("context", ex)   // OK - adds wrapping
}
```

</details>

### expansion

| Token | Detects |
|-------|---------|
| `chained-expansion` | Flags member expansion ('*.') operators where the root is also a '*.' expression. |
| `expansion-result-unused` | Flags member expansion ('*.') results that are computed but not used. |
| `expansion-void-call` | Flags member expansion ('*.') calls on void methods (returns nothing to expand). |

<details>
<summary>chained-expansion examples</summary>

Flags chained member-expansion (`*.`) calls, e.g.:
```gosu
words*.toUpperCase()*.toLowerCase()   // Noncompliant - chained *.
people*.Manager*.Name                 // Noncompliant - chained *.
```

A single `*.` is idiomatic and fine:
```gosu
words*.toUpperCase()                  // OK
people*.Name                          // OK
```

Chaining produces implicit intermediate arrays and obscures data flow.
The fix is to extract the intermediate result to a named variable:
```gosu
var upper = words*.toUpperCase()
var lower = upper*.toLowerCase()
```

### Detection
Two AST nodes represent `*.`:

  - `IMemberExpansionExpression` - property expansion (`people*.Name`)
  - `BeanMethodCallExpression` with `isExpansion()==true` - method expansion
      (`words*.toUpperCase()`)

A violation fires when either node's `getRootExpression()` is itself an expansion node.

CLI token: `chained-expansion`

</details>
<details>
<summary>expansion-result-unused examples</summary>

Flags `*.method()` calls where the method returns a non-void value
that is immediately discarded (the call is used as a statement).

```gosu
items*.transform()        // Noncompliant - String[] result is thrown away
```

This usually means the developer forgot to assign the result, or intended
a side-effect loop and should use `for/in` instead:
```gosu
var results = items*.transform()   // OK - result is used
return items*.transform()          // OK - result is returned
```

Void expansion calls are out of scope here - they are handled by
`ExpansionVoidCallRule` (`expansion-void-call`).

Detection: `IBeanMethodCallStatement` whose inner expression is an
expansion call (`isExpansion()==true`) with a non-`void` return type.
ErrorType receivers are skipped (`getMethodDescriptor()==null`).

CLI token: `expansion-result-unused`

</details>
<details>
<summary>expansion-void-call examples</summary>

Flags `*.method()` calls where the method returns `void`.

```gosu
items*.process()          // Noncompliant - void *. call; use for/in
```

Void expansion creates a hidden implicit array that is immediately discarded,
and the side-effect intent is clearer as an explicit loop:
```gosu
for (item in items) { item.process() }   // OK
```

Detection: `IBeanMethodCallStatement` whose inner expression is an
expansion call (`isExpansion()==true`) with a `void` return type.
ErrorType receivers are skipped (`getMethodDescriptor()==null`).

CLI token: `expansion-void-call`

</details>

### flow

| Token | Detects |
|-------|---------|
| `blank-line-before-else` | Flags else/else-if clauses separated from the closing '}' by a blank line. |
| `complex-boolean` | Flags boolean expressions with too many operators (default threshold: 3). |
| `duplicate-condition` | Flags if/else-if chains where the same condition appears more than once. |
| `logical-and-style` | Flags inconsistent spacing or style around logical AND ('&&') operators. |
| `logical-or-style` | Flags inconsistent spacing or style around logical OR ('||') operators. |
| `missing-braces` | Flags 'if'/'for'/'while' bodies not wrapped in braces (can cause statement-merging bugs). |
| `prefer-safe-navigation` | Flags null-guard chains that can be simplified using Gosu's '?.' safe-navigation operator. |
| `single-case-switch` | Flags switch statements with only one case clause - replace with an if statement. |
| `switch-default` | Flags 'switch' statements missing a 'default' branch. |
| `too-many-returns` | Flags functions with more than the configured number of 'return' statements (default: 5). |
| `while-literal-condition` | Flags while(true) (infinite loop) and while(false) (dead code) loop conditions. |

<details>
<summary>blank-line-before-else examples</summary>

Flags an `else` or `else if` clause that is separated from the closing
{@code }} of its `if` body by one or more blank lines.

A blank line between {@code }} and `else` visually disconnects the two
halves of the conditional and suggests they are unrelated, making the code harder
to follow. The `else` should either share the same line as the closing brace
(K&amp;R style) or appear on the very next line (Allman style):

### Flagged
```gosu
if (cond) {
  doA()
}
                 // ← blank line: VIOLATION
else {
  doB()
}
```

### Not flagged
```gosu
if (cond) { doA() } else { doB() }   // K&R same-line - OK

if (cond) {
  doA()
}
else {                               // Allman next-line - OK
  doB()
}

if (cond) {
  doA()
} else if (other) {                  // K&R else-if - OK
  doC()
}
```

### Detection

Rather than relying on AST line numbers to find the closing {@code }} of the
then-branch (which can misfire on compound statements like `try`), the rule
scans the source backwards from the `else` keyword. It walks through
blank lines until it hits the first non-blank line. If that line starts with
{@code }}, the blank lines are between the closing brace and the `else` - a
violation. If the first non-blank line is anything other than {@code }} (code,
comments, etc.), the blank lines are inside the then-body, not between
the brace and the `else`, so no violation is reported.

This approach handles all brace styles (K&amp;R, Allman) and all then-body
shapes (single-statement, multi-statement, deeply nested compound statements)
without depending on `IStatementList.getLastLine()` accuracy.

</details>
<details>
<summary>complex-boolean examples</summary>

Flags boolean expressions with too many `and`/`or` operators.

Each `and` or `or` operator in a single condition is counted.
When the count exceeds {@value #DEFAULT_MAX_OPERATORS} the expression is
flagged. The threshold is configurable via `configure(Map)` with key
`"maxOperators"`, or via the CLI flag
`--rule-config complex-boolean:maxOperators=2`.

Checked statement types: `if` / `else if`, `while`, `do-while`,
`switch` (discriminant), `return`, local `var` initializers, and
assignment statements.

Example violation (6 operators):
```gosu
if ((x and y) or (a == b) and (c == d) or (e != f) and (g == (h or i))) { ... }
```

Recommended fix - extract named booleans:
```gosu
var isXY  = x and y
var isABC = a == b and c == d
var isDEF = e != f and g == (h or i)
if (isXY or isABC or isDEF) { ... }
```

Token: `complex-boolean`

</details>
<details>
<summary>duplicate-condition examples</summary>

Detects `if / else if` chains where a condition expression is repeated -
making the duplicate branch unreachable (or at best misleading).

### Flagged example
```gosu
if (param == 1)
  openWindow()
else if (param == 2)
  closeWindow()
else if (param == 1)   // Noncompliant - same condition as the first branch
  moveWindowToTheBackground()
```

### Not flagged
```gosu
if (param == 1)
  openWindow()
else if (param == 2)
  closeWindow()
else if (param == 3)
  moveWindow()
```

Condition equality is determined by comparing the string representation returned by
`Object.toString()` on the condition expression (after normalizing whitespace).  This is sufficient for
the common cases of literal comparisons, boolean variables, and simple method calls.

</details>
<details>
<summary>missing-braces examples</summary>

to hide from RulesDocGenerator):

Detection strategy:
The Gosu parser strips curly braces from the AST; both braced and brace-free
single-statement bodies produce the same node type. Detection uses source line
numbers from the AST combined with raw source lines:

  If body: check if any source line in the range [ifLine, thenLine] contains '{'.
    thenLine is included because when the if condition spans multiple lines, the
    opening '{' appears on the last condition line, which is what IStatementList.getLineNum()
    returns. As a fallback, also checks for Allman-style braces: a standalone '{'
    on the line immediately after the condition ends (using the condition expression's
    line number to handle multi-line conditions).

  Else body: check if any source line in the range [thenLine + 1, elseLine] contains '{'.
    elseLine is included because when the else body is a StatementList, getLineNum()
    returns the '} else {' line itself, not the first statement inside.

Known limitation: a '{' character inside a string literal or comment on an otherwise
brace-free line will cause a false negative. This is uncommon in practice.
/
/**
Flags `if` and `else` branches that omit curly braces.

Brace-free single-statement branches are a common source of bugs - the
classic example being Apple's "goto fail" SSL vulnerability, where an
unbraced `if` caused security logic to be silently skipped:

```gosu
if (condition)
    doA()
    doB()     // always runs, even when condition is false
```

Both `if` bodies and `else` bodies are checked. `else if`
chains are handled naturally - each `if` in the chain is checked
independently by the AST collector.

Token: `missing-braces`

</details>
<details>
<summary>prefer-safe-navigation examples</summary>

Detects null-guard chains in `&&` conditions where a variable is null-checked
and then used via member access in the same chain. Such patterns can be rewritten
using Gosu's `?.` safe-navigation operator.

### Flagged example
```gosu
if (product != null && product.getPrice() != null && product.getPrice().Amount > 0)
```

### Preferred rewrite
```gosu
if (product?.Price?.Amount > 0)
```

Token: `prefer-safe-navigation`

</details>
<details>
<summary>single-case-switch examples</summary>

Detects `switch` statements that contain only a single `case` clause.

A switch with one case (with or without a `default`) is equivalent to a simple
`if` / `if-else` statement and should be rewritten as such. The switch
construct adds syntactic overhead without the clarity benefit it provides when there are
multiple branches.

### Flagged
```gosu
switch (code) {         // VIOLATION: only one case
  case 1:
    doSomething()
}

switch (status) {       // VIOLATION: one case + default ≡ if/else
  case "ACTIVE":
    activate()
  default:
    deactivate()
}
```

### Not flagged
```gosu
switch (code) {         // OK: two or more cases
  case 1: doA()
  case 2: doB()
  default: doC()
}

switch (code) {         // OK: default-only switch (no case clauses at all)
  default: doSomething()
}
```

</details>
<details>
<summary>while-literal-condition examples</summary>

Detects `while(true)` and `while(false)` loop conditions.

### Why flag these?

  - **`while(true)`** - an infinite loop with a boolean literal condition is
      almost always a design smell. Use a named sentinel variable or a structured exit
      condition so the intent is explicit and the termination condition is visible.
  - **`while(false)`** - a loop that never executes is dead code. The body
      will never run; remove it or fix the condition.

### Flagged
```gosu
while (true) {          // VIOLATION: literal infinite loop
  doWork()
  if (done) break
}

while (false) {         // VIOLATION: dead code - body never runs
  neverExecuted()
}
```

### Not flagged
```gosu
while (running) {       // OK: named boolean variable
  doWork()
}

while (!queue.empty()) { // OK: computed condition
  process(queue.poll())
}
```

</details>

### misuse

| Token | Detects |
|-------|---------|
| `bigdecimal-constant` | Flags new BigDecimal(0/1/10) and suggests BigDecimal.ZERO/ONE/TEN constants instead. |
| `bitshift` | Flags bitshift operators ('<<', '>>', '>>>') that are usually mistakes in business logic. |
| `bitwise-operator` | Flags bitwise operators ('|', '&', '~') that may be confused with logical operators. |
| `concurrenthashmap-contains` | Flags unsafe use of '.contains()' instead of '.containsKey()' on ConcurrentHashMaps. |
| `consecutive-log-calls` | Flags runs of 2+ consecutive same-level log calls on the same logger that should be merged into one. |
| `date-time-format` | Flags incorrect date/time format patterns (e.g., 'mm' instead of 'MM' for months). |
| `duplicate-null-check` | Flags redundant null checks on the same variable in the same code path. |
| `explicit-gc` | Flags System.gc() and Runtime.getRuntime().gc() calls (the JVM should manage GC; explicit calls cause unpredictable pauses). |
| `foreach-modify` | Flags collection mutations on the same collection being iterated in a foreach loop. |
| `immutable-collection-null` | Flags null arguments passed to immutable collection factories ('List.of', 'Set.of', 'Map.of'). |
| `indexed-removal` | Flags 'list.remove(i)' by index during for-each iteration (causes element skipping). |
| `interval-boundary-confusion` | Flags inclusive (..) ranges with .size()/.length upper bound - likely needs exclusive (..|). |
| `malformed-string-interpolation` | Flags string interpolation with mismatched or incorrect syntax. |
| `map-returns-collection` | Flags 'map()' operations that return nested collections that could be flattened with 'flatMap()'. |
| `process-spawn` | Flags Runtime.exec() and new ProcessBuilder(...) calls (OS process spawning is fragile and a source of command-injection vulnerabilities). |
| `redundant-size-check` | Flags redundant size comparisons that are always true or always false. |
| `reflection-use` | Flags Java java.lang.reflect.* calls and Gosu TypeSystem.* calls that bypass compile-time type safety. |
| `regex-bad-pattern` | Flags regex patterns with ReDoS risk, invalid character ranges, or trivially broad wildcards. |
| `regex-compile-in-loop` | Flags Pattern.compile() and implicit regex compilation (matches/replaceAll/replaceFirst) inside loops. |
| `string-literal-arg` | Flags raw string literals passed as arguments to configured method names. Requires 'methods' config key. *(disabled by default - activate with `--rules string-literal-arg`)* |
| `synchronized-method` | Flags 'synchronized' method modifiers; prefer explicit lock objects. |
| `system-exit` | Flags 'System.exit()' calls (usually a mistake in library code). |
| `thread-primitives` | Flags direct use of Thread, Runnable, ExecutorService and related threading primitives. Disabled by default: legitimate in framework code; enable for application-layer modules only. *(disabled by default - activate with `--rules thread-primitives`)* |
| `thread-sleep` | Flags 'Thread.sleep()' calls (usually a mistake; use events or futures instead). |
| `tomap-no-merge` | Flags 'toMap()' without a merge function on non-unique keys (throws exception at runtime). |
| `tostring-misuse` | Flags incorrect use of '.toString()' on arrays, streams, and optional objects. |
| `unguarded-log-call` | Flags log.debug() and log.trace() calls not wrapped in isDebugEnabled()/isTraceEnabled() guards. |
| `unsafe-static-formatter` | Flags static SimpleDateFormat/DateFormat fields; these are thread-unsafe and should be instance-level or use java.time.format.DateTimeFormatter. *(disabled by default - activate with `--rules unsafe-static-formatter`)* |

<details>
<summary>bigdecimal-constant examples</summary>

Detects `new BigDecimal(0)`, `new BigDecimal(1)`, and `new BigDecimal(10)`
(and their string-literal equivalents) and suggests the named constants instead.

`java.math.BigDecimal` ships with three named constants that are more readable,
allocation-free (implementations may cache them), and unambiguous about intent:
`BigDecimal.ZERO`, `BigDecimal.ONE`, and `BigDecimal.TEN`.

### Flagged → suggested replacement
```gosu
new BigDecimal(0)    →  BigDecimal.ZERO
new BigDecimal("0")  →  BigDecimal.ZERO
new BigDecimal(1)    →  BigDecimal.ONE
new BigDecimal("1")  →  BigDecimal.ONE
new BigDecimal(10)   →  BigDecimal.TEN
new BigDecimal("10") →  BigDecimal.TEN
```

### Not flagged
```gosu
new BigDecimal(42)        // no named constant for 42
new BigDecimal("3.14")    // no named constant for 3.14
new BigDecimal(someVar)   // not a literal
BigDecimal.ZERO           // already using the constant
```

</details>
<details>
<summary>bitwise-operator examples</summary>

Detects bitwise operators (`&`, `|`, `^`).

Bitwise operators on non-flag integer values are rare in high-level
business logic and are a common source of bugs - most often the developer
intended the short-circuit boolean operators (`&&` / `and`,
`||` / `or`) instead.

Common mistakes caught by this rule:
```gosu
if (a & b) { ... }     // likely meant a && b
if (x | y) { ... }     // likely meant x || y
var mask = flags ^ 0xFF // legitimate but worth flagging for review
```

Configuration:

  - `allowHashCode` (boolean, default `false`) - when `true`, bitwise
      operators inside a `hashCode` function are not flagged. Bitwise ops are idiomatic
      in manual `hashCode` implementations (e.g. `result = 31 * result ^ field`).

Token: `bitwise-operator`

</details>
<details>
<summary>consecutive-log-calls examples</summary>

Detects consecutive log calls at the same level on the same logger that should be merged
into a single call.

Each log statement - even a simple string literal - independently triggers timestamp
capture, level-threshold evaluation, and log-record allocation. Two consecutive
`LOG.debug("Hello")` / `LOG.debug("World")` calls pay that overhead twice;
merging them into `LOG.debug("Hello World")` (or a multi-line format string) halves it.

All six standard log levels are covered: `trace`, `debug`, `info`,
`warn`, `error`, and `fatal`.

A "run" is broken by any of:

  - A different log level (e.g. `debug` followed by `info`)
  - A different logger variable (e.g. `LOG.debug` followed by `AUDIT_LOG.debug`)
  - Any non-log statement between the calls

### Flagged
```gosu
LOG.debug("Hello")   // VIOLATION: start of a 2-call run
LOG.debug("World")
```

### Not flagged
```gosu
LOG.debug("only one")          // OK: single call, no run
LOG.debug("step")
LOG.info("done")               // OK: level change breaks run
LOG1.debug("a")
LOG2.debug("b")                // OK: different logger breaks run
LOG.debug("before")
doSomething()                  // OK: non-log statement breaks run
LOG.debug("after")
```

</details>
<details>
<summary>date-time-format examples</summary>

Detects common Java date/time format pattern mistakes in SimpleDateFormat and DateTimeFormatter.

Flags patterns that contain case-sensitive pattern letters used incorrectly:

  - **mm vs MM**: `mm` is minutes, `MM` is months
  - **hh vs HH**: `hh` is 12-hour, `HH` is 24-hour format
  - **ss vs SS**: `ss` is seconds, `SS` is milliseconds
  - **YYYY vs yyyy**: `YYYY` is week-based year, `yyyy` is calendar year
  - **D vs dd**: `D` is day of year (1-366), `dd` is day of month (1-31)
  - **Single d**: `d` is unpadded (1-31), `dd` is zero-padded (01-31)

### Flagged
```gosu
new SimpleDateFormat("yyyy-mm-dd")        // mm is minutes, not months
new SimpleDateFormat("hh:mm:ss")          // hh is 12-hour, likely wanted HH
DateTimeFormatter.ofPattern("YYYY-MM-dd") // YYYY is week-based year
```

### Not flagged
```gosu
new SimpleDateFormat("yyyy-MM-dd")        // correct month pattern
new SimpleDateFormat("HH:mm:ss")          // correct 24-hour format
new SimpleDateFormat("hh:mm:ss a")        // correct 12-hour with AM/PM
```

</details>
<details>
<summary>explicit-gc examples</summary>

Detects explicit garbage collection calls: `System.gc()` and
`Runtime.getRuntime().gc()`.

Explicit GC calls are at best a hint the JVM is free to ignore, and at worst trigger a
full stop-the-world GC pause in production at an uncontrolled moment, hurting latency and
throughput. Rely on the JVM's own GC heuristics instead.

### Flagged
```gosu
System.gc()                    // VIOLATION
Runtime.getRuntime().gc()      // VIOLATION
var rt = Runtime.getRuntime()
rt.gc()                        // VIOLATION
```

### Not flagged
```gosu
myObject.gc()   // custom method named gc() on a non-Runtime type
```

</details>
<details>
<summary>foreach-modify examples</summary>

Detects when a collection is mutated while being iterated in a for-each loop.

Calls to add, addAll, remove, removeAll, clear, set, retainAll, addFirst, addLast,
removeFirst, removeLast on the same collection variable that is the iterable in the
enclosing for-each loop cause ConcurrentModificationException.

### Flagged - modifying the iterable collection
```gosu
function example() {
  var list = new ArrayList(){"a", "b", "c"}
  for (item in list) {
    if (item == "b") {
      list.remove(item)  // VIOLATION: modifying 'list' during iteration
    }
  }
}
```

### Not flagged - modifying a different collection
```gosu
function example() {
  var source = new ArrayList(){"a", "b", "c"}
  var toRemove = new ArrayList()
  for (item in source) {
    if (item == "b") {
      toRemove.add(item)  // OK: modifying 'toRemove', not 'source'
    }
  }
}
```

### Scope limitation

Only detects when the iterable is a simple identifier (e.g. "list"). Field access
(e.g. "this._list" or "_items.subList()") are skipped as out of scope.

</details>
<details>
<summary>immutable-collection-null examples</summary>

Detects `null` arguments passed to immutable collection factory methods.

`List.of()`, `Set.of()`, and `Map.of()` (including `Map.ofEntries()`)
all throw `NullPointerException` if any argument is `null`. This is a deliberate
API contract - unlike their mutable counterparts (`ArrayList`, `HashMap`), the
immutable factories do not permit null elements, keys, or values.

### Flagged
```gosu
var list = List.of("A", null)          // NullPointerException at runtime
var set  = Set.of(null, "B")           // NullPointerException at runtime
var map  = Map.of("key", null)         // NullPointerException at runtime
```

### Not flagged
```gosu
var list = List.of("A", "B")           // no nulls - fine
var map  = new HashMap<>()             // mutable - nulls allowed
list.add(null)                         // mutable operation - not a factory call
```

Token: `immutable-collection-null`

</details>
<details>
<summary>indexed-removal examples</summary>

Detects when list.remove(index) is called where the index is the loop variable
of an enclosing for-each iteration.

Calling remove(index) during iteration by index causes index shifting: if you remove
the element at index 3, elements 4+ shift left, so the next iteration processes the
wrong element.

### Flagged - calling remove(i) where i is the for-each loop variable
```gosu
function example() {
  var list = new ArrayList(){"a", "b", "c"}
  for (i in 0..|list.Count) {
    if (someCondition) {
      list.remove(i)  // VIOLATION: index shifts after removal
    }
  }
}
```

### Not flagged - remove is outside the loop
```gosu
function example() {
  var list = new ArrayList(){"a", "b", "c"}
  var idx = -1
  for (i in 0..|list.Count) {
    if (someCondition) {
      idx = i
    }
  }
  if (idx >= 0) {
    list.remove(idx)  // OK: outside the loop
  }
}
```

### Not flagged - remove argument is a different variable
```gosu
function example() {
  var list = new ArrayList(){"a", "b", "c"}
  for (i in 0..|list.Count) {
    list.remove(0)  // OK: remove literal, not loop variable
  }
}
```

### Fix

Use backwards iteration for safe index-based removal:
```gosu
for (i in (list.Count - 1).downto(0)) {
  list.remove(i)  // Safe: iterating backwards prevents index shift
}
```

</details>
<details>
<summary>interval-boundary-confusion examples</summary>

Flags inclusive `..` range literals whose right-hand side is a
`.size()`, `.length()`, or `.Count` call.

Gosu provides `0..n` (inclusive upper bound) and `0..|n`
(exclusive upper bound). When the upper bound is a size or length, the intent
is almost always exclusive - iterating indices 0 through n-1. Using `..`
instead of `..|`  silently causes an off-by-one and typically an
`IndexOutOfBoundsException` at runtime.

```gosu
// BAD - iterates 0 through items.size() inclusive
for (i in 0..items.size()) { ... }

// GOOD - exclusive upper bound
for (i in 0..|items.size()) { ... }
```

Token: `interval-boundary-confusion`

</details>
<details>
<summary>map-returns-collection examples</summary>

Detects `map()` calls whose lambda returns a collection, producing a nested
collection that is never flattened.

When a `map()` lambda returns a `List`, `Set`, or other collection,
the result is a collection-of-collections (e.g. `List>`). If the caller
does not subsequently flatten the result, this is almost always unintentional and leads
to type errors or unexpected behaviour. The idiomatic fix is to replace `map + flatten`
with `flatMap`, or to explicitly call `.flatten()` on the result.

### Flagged
```gosu
var nested = widgets.map(\w -> w.getItems())  // returns List> - never flattened
```

### Not flagged
```gosu
var flat = widgets.map(\w -> w.getItems()).flatten()       // directly chained flatten
var xs   = widgets.map(\w -> w.getItems()); xs.flatten()  // flattened via local variable
var names = widgets.map(\w -> w.getName())                // lambda returns a scalar
```

Token: `map-returns-collection`

</details>
<details>
<summary>process-spawn examples</summary>

Detects code that spawns OS processes via `Runtime.exec()` or
`new ProcessBuilder(...)`.

Shelling out to OS processes is fragile (platform-specific paths, encoding
differences, resource leaks if the process is not properly waited on) and a
common source of command-injection vulnerabilities when command strings are
assembled from user input.

### Flagged
```gosu
Runtime.getRuntime().exec("ls -la")        // VIOLATION
var pb = new ProcessBuilder("cmd", "/c", "dir")  // VIOLATION
```

### Not flagged
```gosu
myService.exec(query)   // unrelated exec() call on a non-Runtime type
```

</details>
<details>
<summary>redundant-size-check examples</summary>

Detects redundant comparisons of collection.size() against non-negative bounds.

Since size() can never return a negative value, certain comparisons are always
true or always false:
- `size() >= 0` &rarr; always true (pointless guard)
- `size() > -1` &rarr; always true (same)
- `size() < 0` &rarr; always false (never happens)
- `size() <= -1` &rarr; always false (same)

Meaningful comparisons (`size() > 0`, `size() >= 1`, `size() == 0`) are not flagged.

### Flagged - always-true comparison
```gosu
function example() {
  var list = new ArrayList()
  if (list.size() >= 0) {  // VIOLATION: always true; size is never negative
    return true
  }
}
```

### Flagged - always-false comparison
```gosu
function example() {
  var list = new ArrayList()
  if (list.size() < 0) {  // VIOLATION: always false; size is never negative
    return true
  }
}
```

### Not flagged - meaningful comparison
```gosu
function example() {
  var list = new ArrayList()
  if (list.size() > 0) {  // OK: checks for non-empty; can be false
    return true
  }
}
```

</details>
<details>
<summary>reflection-use examples</summary>

Flags use of Java reflection (`java.lang.reflect.*`) and Gosu's
`TypeSystem` API. Both mechanisms bypass compile-time type safety:
they can't be verified at compile time, break silently when class/method
names change, and carry a measurable runtime overhead.

### What is flagged by default

  - **Java Class introspection:** `Class.forName(...)`,
      `.getMethod(...)`, `.getDeclaredField(...)`,
      `.getConstructor(...)`, etc. on `java.lang.Class`
  - **Java reflect invocation:** `method.invoke(...)`,
      `field.get(...)`, `field.set(...)`,
      `.setAccessible(true)`, `constructor.newInstance(...)`
  - **Gosu TypeSystem API:** `TypeSystem.getByFullName(...)`,
      `TypeSystem.getByFullNameIfValid(...)`, etc.

### Configuration
```gosu
--rule-config reflection-use:checkJavaReflect=true,checkGosuTypeSystem=false,allowedMethods=getClass
```

  - `checkJavaReflect` (boolean, default `true`) - flag Java
      `java.lang.reflect.*` calls
  - `checkGosuTypeSystem` (boolean, default `true`) - flag Gosu
      `TypeSystem.*` calls
  - `allowedMethods` (comma-separated, default `""`) - method
      names to skip regardless of declaring type, e.g. `"getClass,getMethod"`

Token: `reflection-use`

</details>
<details>
<summary>regex-compile-in-loop examples</summary>

Detects regex compilation that repeats unnecessarily on every loop iteration.

`Pattern.compile()` is relatively expensive - it parses and compiles
the regex into an internal automaton. Calling it inside a loop wastes CPU on
every iteration. The same applies to `String.matches()`,
`String.replaceAll()`, and `String.replaceFirst()`: each of those
methods silently calls `Pattern.compile()` internally every time they
are invoked.

The fix is to extract the compiled `Pattern` into a
`private static final` field so it is created once when the class loads
and reused on every call.

### Flagged - Pattern.compile() inside a for-each loop
```gosu
for (item in items) {
  var p = Pattern.compile("\\d+")   // VIOLATION: compiled every iteration
  if (p.matcher(item).matches()) { ... }
}
```

### Not flagged - pattern cached as a static field before the loop
```gosu
private static final DIGITS = Pattern.compile("\\d+")  // compiled once

function process(items : List) {
  for (item in items) {
    if (DIGITS.matcher(item).matches()) { ... }   // OK
  }
}
```

### Scope notes

  - Only flags when the regex argument is a string literal for
      `matches`/`replaceAll`/`replaceFirst`. When the regex
      comes from a variable it may already be a pre-compiled wrapper, so the
      call is skipped to avoid false positives.
  - `String.split()` is intentionally excluded - many split calls use
      trivial single-character delimiters where the overhead is negligible and
      flagging would produce too much noise.
  - Handles for-each, while, and do-while loops.

</details>
<details>
<summary>thread-primitives examples</summary>

Detects direct use of low-level threading primitives: `Thread`, `Runnable`,
`ExecutorService` and related types.

Raw threading primitives are error-prone and hard to test. Projects should prefer
higher-level concurrency abstractions (e.g. async frameworks, message queues, reactive
pipelines) and restrict direct thread management to a single dedicated layer.

This rule is disabled by default because threading primitives are legitimate
in infrastructure and framework code. Enable it selectively for application-layer modules
where direct thread management is an architectural violation.

### Flagged
```gosu
var _t : Thread = null                        // field of thread type
var t = new Thread(\-> { doWork() })           // Thread instantiation
var pool = Executors.newFixedThreadPool(4)    // Executors factory call
var ex : ExecutorService = getPool()          // local var of thread type
```

### Not flagged
```gosu
Thread.sleep(100)     // use thread-sleep rule for this
Thread.currentThread()
```

### Detection signals

  - Class-level field declarations whose explicit type is a thread primitive type
  - `new Thread()` / `new ThreadPoolExecutor()` instantiations
  - `Executors.newXxx()` factory calls
  - Local variable declarations with an explicit thread primitive type annotation

</details>
<details>
<summary>tomap-no-merge examples</summary>

Detects `toMap()` calls that omit a merge function, which throws
`IllegalStateException` at runtime if the key mapper produces
duplicate keys.

`Collectors.toMap(keyMapper, valueMapper)` (two-argument form) provides no
strategy for handling duplicate keys: the collector throws immediately. The three-argument
form `toMap(keyMapper, valueMapper, mergeFunction)` lets the caller decide how to
combine duplicate values, making it safe for data where key uniqueness is not guaranteed.

### Flagged
```gosu
items.stream().collect(Collectors.toMap(\i -> i.getId(), \i -> i))
// throws IllegalStateException if two items share an id
```

### Not flagged
```gosu
items.stream().collect(Collectors.toMap(\i -> i.getId(), \i -> i, \a, b -> b))
// merge function supplied - duplicate keys handled explicitly
```

Token: `tomap-no-merge`

</details>
<details>
<summary>unguarded-log-call examples</summary>

Detects `log.debug()` and `log.trace()` calls that are not wrapped in a
corresponding `isDebugEnabled()` / `isTraceEnabled()` guard.

Unguarded debug/trace calls cause argument expressions (string concatenation, method
invocations) to be evaluated even when the log level is disabled, wasting CPU in production.

Both the Java method-call form (`if (log.isDebugEnabled())`) and the Gosu property
form (`if (log.DebugEnabled)`) are recognized as valid guards.

### Flagged
```gosu
LOG.debug("user=" + user.getName())   // VIOLATION: no guard
LOG.trace("payload=" + serialize())   // VIOLATION: no guard
```

### Not flagged
```gosu
if (LOG.isDebugEnabled()) {
  LOG.debug("user=" + user.getName()) // OK: inside guard
}
if (LOG.DebugEnabled) {
  LOG.debug("user=" + user.getName()) // OK: Gosu property form
}
LOG.info("server started")            // OK: info/warn/error not guarded
```

### Configuration

  KeyTypeDefaultDescription
  
    `allowCheapArgs`
    boolean
    `false`
    When `true`, unguarded debug/trace calls whose arguments are all
        "cheap" (string/numeric/boolean literals, local variables of primitive or
        String type, concatenation of cheap values, or template strings with only
        cheap interpolations) are not flagged. Property accesses and method calls
        are always considered expensive.
  

Usage: `--rule-config unguarded-log-call:allowCheapArgs=true`

### Known limitation

Compound guard conditions such as `if (cond && log.isDebugEnabled())` are not
recognized - the call inside will be reported as unguarded (false positive).

</details>

### style

| Token | Detects |
|-------|---------|
| `ambiguous-call-args` | Flags call sites with 3+ boolean or null literal arguments; suggests named parameters. |
| `anonymous-class` | Flags 'new IFoo() { ... }' anonymous class instantiations of SAM interfaces.  |
| `block-parameter-ignored` | Flags lambda/block parameters that are declared but never referenced in the body. |
| `comment-density` | Flags source files whose comment-to-code ratio falls below the configured  *(disabled by default - activate with `--rules comment-density`)* |
| `constant-naming` | Flags constants not using UPPER_CASE naming convention. |
| `constructor-order` | Flags constructors that are not ordered from most-specific to least-specific parameters. |
| `enhancement-modifies-state` | Flags enhancement methods that mutate the enhanced type's fields via this.field = ... *(disabled by default - activate with `--rules enhancement-modifies-state`)* |
| `excessive-newlines` | Flags excessive consecutive blank lines in code (default threshold: 2). |
| `field-order` | Flags field declarations that appear after constructors or methods. |
| `file-header` | Checks that every source file has a comment header (// or /* */)  *(disabled by default - activate with `--rules file-header`)* |
| `foreach-index-math` | Flags manual counter variables incremented inside for-each loops - use the built-in 'index' clause instead. |
| `indent-alignment` | Flags statements whose indentation is inconsistent with the majority of their siblings in the same block. |
| `long-line` | Flags source lines exceeding the configured maximum length (default: 150 characters). |
| `max-file-size` | Flags source files exceeding the configured maximum size (default: 200 KB). *(disabled by default - activate with `--rules max-file-size`)* |
| `max-function-lines` | Flags functions or constructors whose source span exceeds the configured maximum (default: 75 lines). |
| `max-lines` | Flags source files exceeding the configured maximum number of lines (default: 25,000). *(disabled by default - activate with `--rules max-lines`)* |
| `max-ncss` | Flags source files whose non-commenting source statement count  *(disabled by default - activate with `--rules max-ncss`)* |
| `nested-ternary` | Flags ternary expressions whose then/else branch is itself a ternary; use if/else blocks instead. |
| `override-grouping` | Flags '@Override' methods that are not grouped together with other overrides. |
| `package-naming` | Flags package names containing uppercase characters. |
| `print-statement` | Flags 'print' and 'println' statements; use a logging framework instead. |
| `property-vs-method` | Flags trivial properties that should use Gosu's 'var X : T' shorthand. |
| `regex-simplifiable` | Flags str.matches() or str.replaceAll() where a built-in String method (equals, contains, startsWith, endsWith, replace) would be clearer and safer. |
| `string-plus-in-expansion` | Flags string concatenation (+) with a string literal inside ${} template expressions. |
| `this-consistency` | Flags inconsistent use of 'this' qualifier on member access within a class. |
| `undocumented-complex-function` | Flags functions whose cyclomatic complexity meets or exceeds minScore (default: 10)  *(disabled by default - activate with `--rules undocumented-complex-function`)* |

<details>
<summary>ambiguous-call-args examples</summary>

Flags call sites where 2 or more arguments are boolean or null literals,
making the call hard to read without knowing the function signature.

Example of the pattern this rule detects:
```gosu
someMethod(true, false, null, false, true)  // what do these mean?
```

The preferred Gosu style is named parameters at the call site:
```gosu
someMethod(:enableLogging = true, :debug = false,
           :contextObject = null, :trace = false,
           :summarizeTiming = true)
```

When the method descriptor resolves (Gosu-defined methods), the violation
message includes the ambiguous parameter names to make the fix obvious.
For Java library calls the message falls back to a generic suggestion.

The threshold is configurable via `--rule-config ambiguous-call-args:minAmbiguousArgs=N`
(default: 3). Setting it to 1 flags even single-boolean calls like `setEnabled(true)`.

CLI token: `ambiguous-call-args`

</details>
<details>
<summary>block-parameter-ignored examples</summary>

Flags lambda/block expressions with declared parameters that are never referenced
in the body.

A block parameter that goes unused is misleading - it implies the body depends
on each element when it does not. The cleaner form omits the parameter entirely
or uses a method reference. The smell also suggests a potential logic error where
the developer forgot to use the value.

```gosu
// BAD - `item` is declared but never used
var names = customers.map(\ item -> "Unknown")

// GOOD - parameter is actually used
var names = customers.map(\ item -> item.Name)
```

Configurable via `--rule-config block-parameter-ignored:allowTypes=Type1,Type2.method`:

  - `Type` - suppress for any method call on that receiver type
  - `Type.method` - suppress only for that specific method

Token: `block-parameter-ignored`

</details>
<details>
<summary>constant-naming examples</summary>

Detects static final variables (constants) that do not follow UPPER_SNAKE_CASE naming convention.

Constants should be named with all uppercase letters and underscores separating words.
This rule applies to class-level variables that are declared both `static` and `final`.

### Flagged - constant not in UPPER_SNAKE_CASE
```gosu
public static final int maxRetries = 3           // Noncompliant - should be MAX_RETRIES
private static final String appVersion = "1.0"   // Noncompliant - should be APP_VERSION
static final double piValue = 3.14               // Noncompliant - should be PI_VALUE
```

### Not flagged - correct UPPER_SNAKE_CASE
```gosu
public static final int MAX_RETRIES = 3
private static final String APP_VERSION = "1.0"
static final double PI_VALUE = 3.14
```

### Not flagged - not a constant (not static final)
```gosu
public final String title = "Title"              // OK - not static
public static String appName = "App"             // OK - not final
private var _value : int = 0                     // OK - neither static nor final
```

Usage:
```gosu
ConstantNamingRule checker = new ConstantNamingRule();
var result = checker.processFile(gsFile).getVariableResults();
result.getViolations().forEach(v -> System.out.println(v.getSummary()));
```

</details>
<details>
<summary>excessive-newlines examples</summary>

Flags runs of blank lines that exceed the configured limit when they appear
anywhere in the code: between statements, before closing braces, or at end of file.

A single blank line inside a function body is fine for grouping related statements,
but large gaps - whether trailing before a closing brace, or mixed in between code -
are almost always accidental editor artifacts that add noise without improving readability.

Three locations are checked:

  - **Between statements:** excessive blank lines in the middle of function logic.
  - **Before a closing brace:** blank lines immediately preceding a closing brace.
  - **End of file:** trailing blank lines after the last non-empty line.

The default limit is {@value #DEFAULT_MAX_BLANK_LINES} consecutive blank lines.
A custom limit can be supplied via the constructor or the `maxBlankLines` config key.

### Flagged
```gosu
function foo() {
  doSomething()

  doSomethingElse()  // ← four blank lines between statements
}

function bar() {
  doSomething()

}  // ← also flagged before closing brace
```

### Not flagged
```gosu
function foo() {
  doSomething()

  doSomethingElse()  // single blank line between statements - fine
}
```

Token: `excessive-newlines`

</details>
<details>
<summary>foreach-index-math examples</summary>

Flags manual counter variables declared before a for-each loop and incremented inside it.

Gosu's `for (item in collection index i)` syntax provides a built-in,
zero-based index variable. Maintaining a separate counter is redundant boilerplate
that can drift from the real index if `continue` or other control flow is used,
and it signals unfamiliarity with the language.

```gosu
// BAD - manually tracking an index that Gosu's `index` clause provides for free
var counter = 0
for (item in items) {
    print(counter + ": " + item)
    counter++
}

// GOOD - use the built-in index clause
for (item in items index i) {
    print(i + ": " + item)
}
```

The rule does NOT flag the counter if it is referenced outside the loop body,
since in that case it serves a broader purpose (e.g. accumulating a total) that
the built-in `index` variable cannot replace:
```gosu
// PASS - counter used after the loop; cannot be replaced by index clause
var counter = 0
for (item in items) {
    print(counter + ": " + item)
    counter++
}
print("Total: " + counter)
```

Token: `foreach-index-math`

</details>
<details>
<summary>indent-alignment examples</summary>

Flags indentation "hiccups" within function and constructor bodies.

A hiccup is a statement whose leading whitespace differs from the majority of
its siblings in the same block. The rule takes a majority vote across every statement
in a given `IStatementList`: the most common leading-whitespace prefix is the
"expected" indent, and any statement that deviates from it is flagged.

No particular indentation width is mandated - the rule only detects internal
inconsistency within a single block. Each nested block (inside an `if`,
`for`, `while`, etc.) is evaluated independently.

Statement positions are resolved via `ParseTree.getOffset()` rather than
`IStatement.getLineNum()`. Gosu's `getLineNum()` can misfire on compound
statements (for/if/while/try), returning a line inside the statement body instead of
the opening keyword line. Using the parse-tree offset avoids that quirk entirely.

### Flagged - one statement out of alignment
```gosu
function process() {
  var a = 1
  var b = 2
      var c = 3   // VIOLATION: 6 spaces when siblings use 2
  var d = 4
}
```

### Not flagged - consistent indentation
```gosu
function process() {
  var a = 1
  var b = 2
  var c = 3
}
```

Blocks with fewer than three non-blank statements are not checked - too few
peers to establish a reliable majority.

</details>
<details>
<summary>nested-ternary examples</summary>

Detects ternary expressions that have another ternary expression in their
`then` or `else` branch (nested ternaries).

Nested ternaries are harder to read than equivalent `if/else` blocks.
For example:
```gosu
// BAD - hard to follow at a glance
var name = (fs != null) ? fs.Name : (gs != null) ? gs.Name : "?"

// GOOD - the intent is immediately clear
var name : String
if (fs != null) {
  name = fs.Name
} else if (gs != null) {
  name = gs.Name
} else {
  name = "?"
}
```

Only the outer ternary in a chain is flagged; the rule does not double-flag
a single nesting level. Each additional nesting level produces one extra violation.

</details>
<details>
<summary>package-naming examples</summary>

Detects package declarations that contain uppercase characters.

Java and Gosu convention requires package names to be all lowercase
(e.g., `com.example.util`, not `com.Example.Util`).
Uppercase letters in package names can cause confusion and portability
issues across case-sensitive and case-insensitive file systems.

### Flagged - uppercase characters in package name
```gosu
package com.Example.Util    // Noncompliant - should be com.example.util
package myApp.core          // Noncompliant - should be myapp.core
```

### Not flagged - all lowercase
```gosu
package com.example.util    // Compliant
package myapp.core          // Compliant
```

</details>
<details>
<summary>property-vs-method examples</summary>

Detects trivial properties that simply wrap a backing field with no added logic.

A property whose getter returns a backing field and whose setter assigns to it
with no validation, transformation, or access-control logic adds no value beyond
the field itself. The backing field could be exposed as public, or Gosu's
`var X : T` shorthand could be used, which auto-generates equivalent getters
and setters. The boilerplate obscures intent and doubles the maintenance surface.

### How getter triviality is determined

The Gosu parser has two paths for `property get` declarations:

  - **New syntax** (`property get Name : Type`) - no parentheses after the name.
      The parser calls `parseNewPropertyDecl()`, which creates a
      `VarPropertyGetFunctionSymbol` backed by a `SyntheticFunctionStatement`.
      The body is opaque and must be inspected to determine triviality: with
      `--try-harder`, the body is re-parsed via `GetterBodyParser`; otherwise
      a source-text regex is used as a fast-path fallback.
  - **Old syntax** (`property get Name() : Type`) - parentheses present.
      The parser falls through to the regular `FunctionStatement` path with
      `eatStatementBlock()`, producing a real parsed body accessible via
      `getDeclFunctionStmt()`. Must be inspected for complexity.

### Flagged - Trivial property wrapper
```gosu
// BAD - auto-property getter, trivial setter
property get Name : String {
  return _name
}
property set Name(v : String) {
  _name = v
}

// GOOD - use Gosu shorthand instead
var _name : String as Name = "default"
```

### Not flagged - Property with logic
```gosu
property get DisplayName : String {
  return _name.toUpperCase()  // has transformation
}
property set Name(v : String) {
  _name = v.trim()            // has validation/transformation
}
```

</details>
<details>
<summary>string-plus-in-expansion examples</summary>

Flags string concatenation (`+`) inside a template interpolation expression
(`${`}) when one operand is a string literal.

Gosu string templates (`"Hello ${expr`"}) already handle interpolation.
Embedding a `+` concatenation inside `${`} where one side is a literal
is redundant - the literal can be moved outside the braces or inlined as a separate
interpolation segment, making the template cleaner and more readable.

```gosu
// BAD
var msg = "Result: ${"prefix_" + value}"

// Better
var msg = "Result: prefix_${value}"
```

Token: `string-plus-in-expansion`

</details>
<details>
<summary>this-consistency examples</summary>

Flags inconsistent use of the `this.` qualifier on member access within a class.

A class should use either `this.field` / `this.method()` throughout, or
bare `field` / `method()` throughout. Mixing the two styles within the same
class is a violation. The rule takes a majority vote across all instance methods and
constructors: accesses that use the minority style are flagged.

### Flagged - majority is bare, minority is this-qualified
```gosu
class Example {
  var _name : String
  var _value : int

  function setName(n : String) { _name = n }       // bare (majority)
  function setValue(v : int)   { this._value = v } // VIOLATION: this-qualified minority
}
```

### Not flagged - consistent bare style
```gosu
class Example {
  var _name : String
  function setName(n : String) { _name = n }
  function getName() : String  { return _name }
}
```

### Not flagged - consistent this-qualified style
```gosu
class Example {
  var _name : String
  function setName(n : String) { this._name = n }
  function getName() : String  { return this._name }
}
```

Static methods and constructors are excluded from analysis. Constructors frequently
need `this.` to disambiguate field names from parameter names, so including them
would skew the majority vote. A tie (equal number of this-qualified and bare accesses)
is not flagged.

</details>

### vendor

| Token | Detects |
|-------|---------|
| `cls01g-public-primitive-field` | Flags public primitive fields; prefer private fields with public getter/setter. |
| `cls01g-public-reference-field` | Flags public reference fields; prefer private fields with public getter/setter. |
| `cls01g-readonly-array` | Flags array properties without a copy-on-return to prevent external modification. |
| `cls01g-readonly-reference` | Flags reference properties without defensive copying to prevent external modification. |
| `err01g-throw-generic` | Flags throwing generic 'Exception' instead of a specific exception type. |
| `err01g-throw-string` | Flags throwing non-exception objects (strings, primitives) instead of exceptions. |
| `res00g-close-not-in-finally` | Flags resource close statements not in a 'finally' block or 'using' statement. |
| `res00g-not-in-using` | Flags closeable resources not used in a 'using' statement or try-with-resources. |
| `res00g-unsafe-close-in-finally` | Flags unsafe resource closing in finally blocks without null checks. |

<details>
<summary>cls01g-public-primitive-field examples</summary>

Detects public primitive fields that directly expose data without encapsulation.

CLS01-G guideline requires that primitive fields be private and exposed
only through property accessors, protecting against uncontrolled mutation.

### Flagged - public primitive field
```gosu
public var total : int      // Noncompliant - direct mutation possible
public var active : boolean // Noncompliant
```

### Not flagged - private field with property accessor
```gosu
private var _total : int as readonly Total   // OK
```

### Not flagged - reference type field (different rule concern)
```gosu
public var Name : String  // Separate rule; this rule only flags primitives
```

</details>
<details>
<summary>cls01g-public-reference-field examples</summary>

Detects public reference-type fields (mutable by any caller).

CLS01-G guideline requires that reference fields be private and exposed
only through carefully designed property accessors. Even a `final`
reference field allows callers to mutate the referenced object's state.

### Flagged - public reference field (non-final)
```gosu
public var owner : Object     // Noncompliant - both reference and object are mutable
```

### Flagged - public final reference field
```gosu
public final var config : Object  // Noncompliant - final prevents reassignment but not mutation
```

### Not flagged - private reference field with property accessor
```gosu
private var _owner : Object as Owner   // OK
```

### Not flagged - primitive field (different rule concern)
```gosu
public var count : int  // Separate rule; this rule only flags reference types
```

</details>
<details>
<summary>cls01g-readonly-array examples</summary>

Detects readonly array properties that expose always-mutable array types.

CLS01-G guideline requires that array properties return defensive copies
instead of direct array references. Arrays are always mutable: their elements
can be written to regardless of the `final` or `readonly` modifiers.

### Flagged - readonly array property
```gosu
private final var _rgb : String[] as readonly RGB    // Noncompliant
private var _flags : int[] as readonly Flags         // Noncompliant
```

### Not flagged - private array field without property (no external exposure)
```gosu
private final var _rgb : String[]  // OK - not exposed as property
```

### Not flagged - writable array property (caller can reassign, different concern)
```gosu
private var _tags : String[] as Tags  // Different issue; caller can reassign
```

### Not flagged - readonly reference property, non-array (covered by separate rule)
```gosu
private var _manager : Object as readonly Manager  // Separate rule
```

</details>
<details>
<summary>cls01g-readonly-reference examples</summary>

Detects readonly properties that expose mutable reference types.

CLS01-G guideline requires that readonly properties on mutable reference
types return defensive copies instead of direct references. The `readonly`
keyword only prevents the property reference from being reassigned through the
property; it does NOT prevent callers from mutating the referenced object itself.

### Flagged - readonly reference property (non-array, non-String)
```gosu
private final var _manager : Object as readonly Manager  // Noncompliant
private var _context : Config as readonly Context        // Noncompliant
```

### Not flagged - readonly String property (String is immutable)
```gosu
private var _label : String as readonly Label    // OK - String cannot be mutated
```

### Not flagged - readonly primitive property
```gosu
private var _count : int as readonly Count  // OK - primitive cannot be mutated externally
```

### Not flagged - writable property (different concern)
```gosu
private var _item : Object as Item  // Caller can reassign, intentional
```

### Not flagged - readonly array property (covered by separate rule)
```gosu
private var _tags : String[] as readonly Tags  // Separate rule
```

</details>
<details>
<summary>err01g-throw-generic examples</summary>

Detects methods throwing generic exception classes that obscure the actual error.

ERR01-G guideline requires throwing specific exceptions instead of
superclasses like `RuntimeException`, `Exception`, `Throwable`,
or `Error`. Generic exceptions prevent callers from catching and handling
specific error conditions, making error diagnosis and recovery difficult.

### Flagged - generic exception thrown
```gosu
if (s == null) throw new RuntimeException("Null String")  // Noncompliant
if (x < 0) throw new Exception("Invalid value")          // Noncompliant
throw new Throwable("Something failed")                   // Noncompliant
```

### Not flagged - specific exception thrown
```gosu
if (s == null) throw new NullPointerException()           // Compliant
if (x < 0) throw new IllegalArgumentException("Invalid") // Compliant
```

</details>
<details>
<summary>err01g-throw-string examples</summary>

Detects methods throwing non-exception objects (typically string literals).

ERR01-G guideline forbids throwing non-exception objects. When a string
is thrown in Gosu, it is implicitly coerced and wrapped in a `RuntimeException`
with the string as the message. This prevents specific exception catching
and violates the principle of throwing specific, typed exceptions.

### Flagged - non-exception thrown
```gosu
if (user.Age < 21) {
  throw "User is not allowed in the bar"    // Noncompliant - coerced to RuntimeException
}
```

### Not flagged - typed exception thrown
```gosu
if (user.Age < 21) {
  throw new IllegalArgumentException("User is not allowed in the bar")  // Compliant
}
```

</details>
<details>
<summary>res00g-close-not-in-finally examples</summary>

Detects `close()` calls inside a `try` body that has no `finally` block.

RES00-G guideline requires that resource cleanup be performed in a `finally`
block (or a Gosu `using` statement) so that `close()` is guaranteed to
execute even when an exception is thrown in the `try` body. A `close()`
placed directly in the `try` body is silently skipped if any preceding statement
throws, leaving the resource open.

### Flagged
```gosu
try {
  stream.write(data)
  stream.close()      // skipped if write() throws
} catch (Exception e) { ... }
```

### Not flagged - close() in finally
```gosu
try {
  stream.write(data)
} catch (Exception e) { ... }
finally {
  stream.close()      // always executes
}
```

### Not flagged - using statement
```gosu
using (var stream = new FileOutputStream(path)) {
  stream.write(data)  // closed automatically
}
```

Token: `res00g-close-not-in-finally`

</details>
<details>
<summary>res00g-not-in-using examples</summary>

Detects closeable resources that are declared as local variables but not managed
by a Gosu `using` statement.

RES00-G guideline requires that I/O streams, database connections, and other
`Closeable` resources be acquired inside a `using` block so that they
are guaranteed to be closed even if an exception is thrown. Unmanaged resources
lead to connection leaks, file handle exhaustion, and unpredictable behaviour under
load.

The following resource types are checked: `FileInputStream`,
`FileOutputStream`, `BufferedReader`, `BufferedWriter`,
`PrintWriter`, `PrintStream`, `InputStreamReader`,
`OutputStreamWriter`, `Connection`, `Statement`,
`PreparedStatement`, `CallableStatement`, and `ResultSet`.

### Flagged
```gosu
var reader = new BufferedReader(new FileReader(path))  // not in a using block
reader.readLine()
```

### Not flagged
```gosu
using (var reader = new BufferedReader(new FileReader(path))) {
  reader.readLine()
}
```

Token: `res00g-not-in-using`

</details>
<!-- END:rules -->
