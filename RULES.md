# Rules Reference

> **Rule philosophy:** This project ships a curated set of language-level rules that work across any Gosu codebase. It serves as a foundational component for larger code quality efforts and will not reach feature parity with general-purpose linters. If you're building a Gosu framework or application platform, write custom rules targeting your domain APIs.

70+ rules across several categories, listed alphabetically. Each section shows the CLI rule token, what it detects (including any configurable options), and FAIL/PASS examples drawn from the test fixtures.

### dataflow

| Token | Detects |
|-------|---------|
| `ctor-delegation-param` | A constructor that delegates to another constructor via `this(...)` but fails to forward one or more of its own parameters - the omitted parameter is silently lost. |
| `dead-assignment` | A local variable is assigned a non-trivial value (not `null`, `0`, `false`, or `""`) that is then overwritten before being read. The first assignment's result is discarded - it is either a wasted computation or a latent bug. Trivial sentinel values (`null`, `0`) are exempt. |
| `identity-return` | A function that accepts a single parameter and returns that exact parameter without transformation. This is almost always a sign that a local variable holding the transformed result is mistakenly replaced with the original parameter in the `return` statement. |
| `null-return` | A function whose return type is a collection (`List`, `Set`, `Map`) or an array returns `null`. Callers must then guard against `null`; returning an empty collection or array is almost always the correct alternative. |
| `unused-constructor-params` | A constructor parameter that is never assigned to a field or referenced anywhere in the constructor body. |

<details>
<summary>dead-assignment examples</summary>

```gosu
// FAIL: non-trivial initializer is never read before overwrite
function buildLabel() : String {
  var result = computeString()   // value computed, then discarded
  result = computeOther()        // overwrites without reading
  return result
}

// FAIL: assigned mid-function, overwritten without reading
function processValue(input : String) : String {
  var work = input.trim()        // non-trivial
  work = input.toUpperCase()     // dead assignment above
  return work
}

// PASS: variable is read before reassignment
function buildLabel() : String {
  var result = computeString()
  print(result)                  // read - clears dead-assignment tracking
  result = computeOther()
  return result
}

// PASS: trivial null sentinel is not flagged
function buildLabel() : String {
  var result : String = null     // sentinel - OK
  result = computeString()
  return result
}

// PASS: compound assignment reads the variable on the RHS
function append(base : String) : String {
  var s = computeString()
  s = s + "_suffix"              // reads s on the right
  return s
}
```

</details>

<details>
<summary>null-return examples</summary>

```gosu
// FAIL: List-returning function returns null
function getNames(flag : boolean) : List<String> {
  if (!flag) {
    return null
  }
  return new ArrayList<String>()
}

// FAIL: array-returning function returns null
function getValues(flag : boolean) : String[] {
  if (!flag) {
    return null
  }
  return new String[0]
}

// FAIL: unconditional null return
function alwaysNull() : List<String> {
  return null
}

// PASS: returns empty list instead of null
function getNames(flag : boolean) : List<String> {
  if (!flag) {
    return new ArrayList<String>()
  }
  return new ArrayList<String>()
}

// PASS: String return type - not a collection, not flagged
function findFirst() : String {
  return null
}
```

</details>

<details>
<summary>identity-return / unused-constructor-params / ctor-delegation-param examples</summary>

```gosu
// FAIL: identity-return - works done on `names` but original param is returned
function processNames(names : List<String>) : List<String> {
  var result = names.map(\n -> n.toUpperCase())
  return names   // bug: should return `result`
}

// PASS: returns transformed value, not the param
function processNames(names : List<String>) : List<String> {
  return names.map(\n -> n.toUpperCase())
}

// FAIL: unused-constructor-params - `value` is never referenced
construct(name : String, value : int) {
  _name = name
  // value is silently dropped
}

// PASS: all params assigned
construct(name : String, value : int) {
  _name = name
  _value = value
}

// FAIL: ctor-delegation-param - flag_me not forwarded
construct(flag_me : boolean) {
  this(Location.DEFAULT, false)   // bug: should pass flag_me
}

// PASS: param forwarded to delegate constructor
construct(flag_me : boolean) {
  this(Location.DEFAULT, flag_me)
}
```

</details>

### declarations

| Token | Detects |
|-------|---------|
| `logger-static` | Logger fields (`org.slf4j.Logger`, `java.util.logging.Logger`, and common variants) that are not declared `static`. Loggers are per-class, not per-instance; a non-static logger wastes memory and is created on every object construction. |
| `public-field` | Non-constant public instance fields. In Gosu, the idiomatic pattern is a private backing field exposed through a property accessor (`private var _x : T as X`). A raw `public var` bypasses encapsulation entirely. |
| `raw-type` | Field declarations using a generic type without its type parameters (e.g., `List` instead of `List<String>`). Raw types defeat type checking and are equivalent to `List<Object>` at the compiler level. |
| `stringbuffer-char-constructor` | `StringBuffer` or `StringBuilder` constructed with a single `char` argument. The `char` is silently coerced to `int` and interpreted as an initial *capacity*, not as the first character - a common source of subtle bugs. |
| `threadlocal-static` | `ThreadLocal` fields that are not `static`. A `ThreadLocal` is inherently a class-level construct; a non-static `ThreadLocal` creates a new instance per object and defeats its purpose. |
| `too-many-args` | Functions with more parameters than the configured threshold. Long parameter lists are hard to read and call correctly; consider grouping parameters into a parameter object. **Configurable:** `maxArgs` (default: **5**) |

<details>
<summary>declarations examples</summary>

```gosu
// FAIL: public-field
public var FirstName : String
public var Age : int

// PASS: private backing field with property accessor
private var _lastName : String as LastName
private var _email : String

// FAIL: logger-static - non-static logger
var _logger : org.slf4j.Logger = LoggerFactory.getLogger(MyClass.Type.Name)

// PASS: static logger
static var LOG : org.slf4j.Logger = LoggerFactory.getLogger(MyClass.Type.Name)

// FAIL: raw-type
private var _rawHandlers : Map
private var _rawQueue : List

// PASS: parameterized types
private var _handlers : Map<String, List<String>>
private var _queue : List<String>

// FAIL: threadlocal-static - not static
var myThreadLocal : ThreadLocal<String> = new ThreadLocal<String>()

// PASS: static ThreadLocal
static var THREAD_CTX : ThreadLocal<String> = new ThreadLocal<String>()

// FAIL: too-many-args (default threshold: 5)
function register(a : int, b : int, c : int, d : String, e : long, f : boolean) : void {}

// PASS: at threshold
function register(a : int, b : int, c : int, d : String, e : long) : void {}

// FAIL: stringbuffer-char-constructor - 'A' coerces to int 65, used as capacity
var sb = new StringBuilder('A')

// PASS: string argument
var sb = new StringBuilder("A")
```

```bash
java -jar gslint.jar src --rule-config too-many-args:maxArgs=4
```

</details>

---

### duplication

| Token | Detects |
|-------|---------|
| `delegate-member-conflict` | Delegate fields whose constituent interface methods conflict with explicitly declared methods in the same class. When a class both declares a method and delegates an interface containing the same method signature, the explicit method silently shadows the delegated one. |
| `duplicate-code` | Sequences of 3 or more identical (or structurally equivalent) statements that appear more than once within a function. Variable names are normalised during comparison, so `var host = "x"` and `var hostname = "x"` are treated as the same statement. Repeated code is a maintenance hazard and a sign that the logic should be extracted. |
| `duplicate-delegate` | Two or more `delegate` fields that represent the same interface, which causes duplicate forwarding methods to be synthesized for all interface members. The last delegate wins, making the earlier one(s) dead code. |
| `redundant-block-code` | Block-form lambdas (`\x -> { return expr }`) that contain only a single expression or return statement and could be simplified to expression-form (`\x -> expr`). Unnecessary block syntax adds noise without adding clarity. |

<details>
<summary>duplication examples</summary>

```gosu
// FAIL: duplicate-code - same 3-statement sequence appears twice
function processItem(item : Item) {
  item.validate()
  item.persist()
  item.notify()
  // ... other work ...
  item.validate()   // duplicate
  item.persist()
  item.notify()
}

// FAIL: duplicate-code - variable names differ but structure is identical
function setup() {
  var host = "localhost"
  var port = 5432
  var timeout = 30
  // ...
  var hostname = "localhost"  // same structure, normalised
  var portNum = 5432
  var timeoutSecs = 30
}

// PASS: sequences are not long enough (< 3 statements)
function short() {
  var a = 1
  var b = 1
}

// FAIL: redundant-block-code - single-return block
function redundant(items : List<Integer>) : List<Integer> {
  return items.map(\x -> { return x * 2 })
}

// PASS: expression lambda - no block needed
function clean(items : List<Integer>) : List<Integer> {
  return items.map(\x -> x * 2)
}

// PASS: multi-statement block - block is required
function multi(items : List<Integer>) : List<Integer> {
  return items.map(\x -> {
    var doubled = x * 2
    return doubled
  })
}

// FAIL: duplicate-delegate - two delegates for the same interface
class Wrapper implements IFoo {
  delegate _a represents IFoo
  delegate _b represents IFoo   // duplicate-delegate
}

// PASS: distinct interfaces
class Wrapper implements IFoo, IBar {
  delegate _a represents IFoo
  delegate _b represents IBar   // distinct interfaces, no conflict
}

// FAIL: delegate-member-conflict - explicit method shadows delegate method
class Wrapper implements IStringList {
  delegate _inner represents IStringList
  function charAt(index : int) : String {    // conflicts with IStringList.charAt()
    return "local"
  }
}

// PASS: no conflicting methods
class Wrapper implements IStringList {
  delegate _inner represents IStringList
  function helper() : String {    // no conflict with IStringList
    return "ok"
  }
}
```

</details>

---

### exceptions

| Token | Detects |
|-------|---------|
| `catch-null-reference` | Catch blocks that catch `NullPointerException` or `NullReferenceException` directly. Catching null-pointer exceptions masks the root cause and is almost always a sign that a null check or null-safe operator (`?.`) should be used instead. |
| `empty-catch` | Catch blocks whose body contains no statement with side effects (no method calls, throws, or assignments beyond local variables). Silently swallowing exceptions hides bugs. A catch block that contains only a comment can optionally be allowed. **Configurable:** `allowCommentedCatch` (default: **false**) |
| `exception-swallowed-in-finally` | `return` or `throw` statements inside `finally` blocks that silently discard any exception propagating from the `try` or `catch` block. The original failure disappears with no trace, making bugs extremely hard to diagnose. |
| `rethrow-catch` | Catch blocks whose sole statement is `throw ex` - the caught exception is rethrown unchanged with no wrapping, logging, or additional context. These blocks add indentation cost for zero value; they should either be removed or enriched. |
| `too-many-throws` | Functions that throw more distinct exception types than the configured threshold, making callers hard to write correctly. **Configurable:** `maxThrows` (default: **3**) |

<details>
<summary>exceptions examples</summary>

```gosu
// FAIL: empty-catch - exception is silently swallowed
function parseAge(s : String) {
  try {
    var age = Integer.parseInt(s)
  } catch (e : Exception) {
  }
}

// FAIL: empty-catch - body declares a variable but has no side effects
function varOnlyCatch() {
  try {
    var x = Integer.parseInt("oops")
  } catch (e : Exception) {
    var msg = "Caught: " + e.getMessage()   // no call, no throw
  }
}

// PASS: body throws - meaningful action
function parseAge(s : String) {
  try {
    var age = Integer.parseInt(s)
  } catch (e : Exception) {
    throw new RuntimeException("parse failed", e)
  }
}

// PASS: body has a method call
function parseAge(s : String) {
  try {
    var age = Integer.parseInt(s)
  } catch (e : Exception) {
    System.err.println("Parse failed: " + e.getMessage())
  }
}

// FAIL: rethrow-catch - pure rethrow, adds nothing
function pureRethrow() {
  try {
    var x = Integer.parseInt("oops")
  } catch (ex : Exception) {
    throw ex
  }
}

// PASS: logs before rethrowing - body has two statements
function loggedRethrow() {
  try {
    var x = Integer.parseInt("oops")
  } catch (ex : Exception) {
    logger.warning(ex.getMessage())
    throw ex
  }
}

// FAIL: catch-null-reference
function catchesNpe() {
  try {
    var x = getValue().length()
  } catch (e : NullPointerException) {
    print("caught npe")
  }
}

// PASS: guard with null-safe operator
function safeLength() : int {
  return getValue()?.length() ?: 0
}

// FAIL: exception-swallowed-in-finally - return in finally silently discards the IOException
function readFile(path : String) : String {
  try {
    return FileUtil.readFile(path)   // throws IOException
  } finally {
    return ""                         // swallows the exception
  }
}

// PASS: safely handle exceptions in finally
function readFile(path : String) : String {
  try {
    return FileUtil.readFile(path)
  } finally {
    // Finally block can clean up but should not return or throw
    closeResources()
  }
}
```

```properties
# lint.properties
rule.empty-catch.allowCommentedCatch=true
rule.too-many-throws.maxThrows=2
```

```bash
java -jar gslint.jar src \
  --rule-config "empty-catch:allowCommentedCatch=true;too-many-throws:maxThrows=2"
```

</details>

---

### expansion

Gosu's member expansion operator `*.` applies a method call across every element of an array or `Iterable` and returns the results as an array. Misuse tends to fall into three patterns: chaining expansions (creating implicit intermediate arrays), calling void methods (where a plain loop is clearer), and discarding the result entirely.

| Token | Detects |
|-------|---------|
| `chained-expansion` | Two or more `*.` expansions chained in a single expression (e.g., `words*.toUpperCase()*.trim()`). Each link in the chain allocates a full intermediate array that the developer cannot see. Splitting into named intermediate steps makes the allocations explicit and the intent clearer. |
| `expansion-result-unused` | A `*.` expansion whose non-void return value is discarded (neither assigned nor returned). The entire collection is projected and then thrown away, which is almost certainly a bug. |
| `expansion-void-call` | A `*.` expansion that calls a `void`-returning method. Because the expansion discards the result anyway, `items*.process()` is strictly equivalent to `for (item in items) { item.process() }` - the loop form is unambiguous about intent. |

<details>
<summary>expansion examples</summary>

```gosu
// FAIL: chained-expansion - intermediate String[] is invisible
function normalize(words : String[]) : String[] {
  return words*.toUpperCase()*.toLowerCase()
}

// FAIL: three-step chain - two violations
function triple(words : String[]) : String[] {
  return words*.toUpperCase()*.toLowerCase()*.trim()
}

// PASS: single *. per expression
function toUpper(words : String[]) : String[] {
  return words*.toUpperCase()
}

// PASS: split into named steps
function normalize(words : String[]) : String[] {
  var upper = words*.toUpperCase()
  return upper*.toLowerCase()
}

// FAIL: expansion-void-call - process() returns void
function processAll(items : Item[]) {
  items*.process()
}

// PASS: explicit loop
function processAll(items : Item[]) {
  for (item in items) {
    item.process()
  }
}

// FAIL: expansion-result-unused - String[] result is discarded
function discarded(items : Item[]) {
  items*.transform()
}

// PASS: result is returned
function transform(items : Item[]) : String[] {
  return items*.transform()
}

// PASS: result is assigned
function transform(items : Item[]) : String[] {
  var results = items*.transform()
  return results
}
```

</details>

---

### flow

| Token | Detects |
|-------|---------|
| `complex-boolean` | A single boolean expression containing more `and`/`or`/`&&`/`\|\|` operators than the configured threshold. Overly complex conditions are hard to reason about and should be extracted into named predicates or broken into separate `if` statements. **Configurable:** `maxOperators` (default: **3**) |
| `duplicate-condition` | An `if`/`else if` chain where the same boolean condition appears on more than one branch. The second branch can never be reached and is dead code - usually a copy-paste error. |
| `logical-and-style` | Use of the keyword `and` instead of `&&` for logical conjunction. Gosu supports both; this rule enforces the operator form. |
| `logical-or-style` | Use of the keyword `or` instead of `\|\|` for logical disjunction. |
| `missing-braces` | An `if`, `else`, `for`, or `while` body that is written without braces. Single-statement bodies are fragile: the next developer to add a line may not notice the scope boundary. |
| `switch-default` | A `switch` statement that does not have a `default` clause, or whose `default` clause is not the last case. A missing `default` silently ignores unhandled values. |
| `too-many-returns` | A function with more `return` statements than the configured threshold, a signal that the function may be too complex and a candidate for extraction. **Configurable:** `maxReturns` (default: **5**) |

<details>
<summary>flow examples</summary>

```gosu
// FAIL: duplicate-condition - `x > 0` appears in branches 0 and 2
function checkCode(x : int) {
  if (x > 0)       doA()
  else if (x < 0)  doB()
  else if (x > 0)  doC()   // dead branch
}

// PASS: all conditions distinct
function checkCode(x : int) {
  if (x > 0)       doA()
  else if (x < 0)  doB()
  else             doC()
}

// FAIL: complex-boolean - 4 operators (default threshold: 3)
function tooComplex(a : boolean, b : boolean, c : boolean, d : boolean, e : boolean) {
  if (a and b and c or d and e) { print("complex") }
}

// PASS: exactly 3 operators
function atLimit(a : boolean, b : boolean, c : boolean, d : boolean) {
  if (a and b or c and d) { print("ok") }
}

// FAIL: missing-braces
function noBraces(x : int) {
  if (x > 0)
    print("positive")
}

// PASS
function withBraces(x : int) {
  if (x > 0) {
    print("positive")
  }
}

// FAIL: switch-default missing
function noDefault(code : int) {
  switch (code) {
    case 1: print("one")
    case 2: print("two")
  }
}

// PASS: default present and last
function withDefault(code : int) {
  switch (code) {
    case 1: print("one")
    case 2: print("two")
    default: print("other")
  }
}

// FAIL: too-many-returns - 6 returns (threshold: 5)
function classify(code : int) : String {
  if (code == 1) { return "one" }
  if (code == 2) { return "two" }
  if (code == 3) { return "three" }
  if (code == 4) { return "four" }
  if (code == 5) { return "five" }
  return "other"
}
```

```bash
java -jar gslint.jar src \
  --rule-config "complex-boolean:maxOperators=2;too-many-returns:maxReturns=3"
```

</details>

---

### misuse

| Token | Detects |
|-------|---------|
| `bitshift` | Bit-shift operations (`<<`, `>>`, `>>>`). Bit shifts on signed integers produce surprising results and are rarely the intended computation in high-level business logic. |
| `bitwise-operator` | Bitwise `&`, `\|`, and `^` operators. These are distinct from logical `&&` / `\|\|` and do not short-circuit. Mistaking one for the other is a common bug. |
| `concurrenthashmap-contains` | Calls to `ConcurrentHashMap.contains(value)`. This method checks for the *value*, not the key - the correct method is `containsKey(key)` or `containsValue(value)`, depending on intent. |
| `date-time-format` | Date/time format patterns that use lowercase `mm` (minutes) where `MM` (months) is likely intended, or vice versa - one of the most common date-formatting bugs. |
| `duplicate-null-check` | The same variable is checked against `null` more than once in the same scope without any reassignment between the checks. The redundant check adds confusion. |
| `foreach-modify` | Mutations of a collection (`.remove()`, `.add()`, `.clear()`) while iterating over it with a `foreach` loop. This causes `ConcurrentModificationException` at runtime. Collect items to remove into a separate list, then remove after the loop. |
| `immutable-collection-null` | `null` passed to `List.of()`, `Set.of()`, or `Map.of()`. These factory methods throw `NullPointerException`; use `Collections.singletonList(null)` or a mutable collection if null elements are required. |
| `indexed-removal` | `list.remove(int index)` called inside a loop over the same list. After the first removal the indices shift, causing incorrect elements to be removed. Use an `Iterator` or collect indices first. |
| `interval-boundary-confusion` | Inclusive (`..`) range literals whose right-hand side is a `.size()`, `.length()`, or `.Count` call. The intent is almost always exclusive (`...|`) - off-by-one causes `IndexOutOfBoundsException` at runtime. |
| `is-field-changed-string` | String literals passed to `isFieldChanged("FieldName")`. The string can silently refer to a non-existent field after a rename. Use the typed property reference form `Entity#FieldName` instead. |
| `map-returns-collection` | A `.map()` lambda that returns a collection or array type. When the intent is to flatten, use `.flatMap()` instead; `.map()` here produces a `List<List<T>>` rather than `List<T>`. |
| `redundant-size-check` | `collection.size() >= 0` or `collection.size() < 0` - comparisons that are unconditionally true or false because `size()` can never return a negative value. |
| `synchronized-method` | Methods declared `synchronized`. Synchronising on `this` exposes the lock to callers, which can lead to deadlocks. Prefer an explicit private lock object. |
| `system-exit` | `System.exit()` calls outside of a well-defined application entry point. Calling `System.exit()` terminates the entire JVM without giving callers a chance to clean up resources. |
| `thread-sleep` | `Thread.sleep()` calls in non-test production code. Sleeping in production code is almost always a design smell; use scheduled tasks, timers, or reactive primitives instead. |
| `tomap-no-merge` | `Collectors.toMap()` called without a merge function. When duplicate keys exist the call throws `IllegalStateException` at runtime; providing a merge function makes the resolution explicit. |
| `tostring-misuse` | `.toString()` called on types whose `toString()` is unhelpful: arrays (prints a memory address), `Stream` (prints a pipeline reference), `Optional` (rarely the desired value). Use `Arrays.toString()`, collect the stream, or unwrap the optional. |

<details>
<summary>misuse examples</summary>

```gosu
// FAIL: thread-sleep - suspicious in production
function poll() {
  while (!isDone()) {
    Thread.sleep(500)
  }
}

// PASS: no sleep
function poll() {
  while (!isDone()) { check() }
}

// FAIL: system-exit
function run(args : String[]) {
  if (args.length == 0) {
    System.exit(1)
  }
}

// PASS: throw an exception instead
function run(args : String[]) {
  if (args.length == 0) {
    throw new IllegalArgumentException("No arguments provided")
  }
}

// FAIL: foreach-modify - ConcurrentModificationException at runtime
function removeExpired(items : List<String>) {
  for (item in items) {
    if (isExpired(item)) {
      items.remove(item)
    }
  }
}

// PASS: collect first, then remove
function removeExpired(items : List<String>) {
  var expired = items.where(\i -> isExpired(i))
  items.removeAll(expired)
}

// FAIL: is-field-changed-string - string breaks silently after rename
function hasChanged() : boolean {
  return _autoLine.isFieldChanged("YearMakeModel")
}

// PASS: typed property reference - rename-safe
function hasChanged() : boolean {
  return _autoLine.isFieldChanged(AutoLine#YearMakeModel)
}

// FAIL: tostring-misuse - prints memory address
function describe(arr : int[]) : String {
  return arr.toString()
}

// PASS: meaningful output
function describe(arr : int[]) : String {
  return java.util.Arrays.toString(arr)
}

// FAIL: bitwise-operator - & does not short-circuit
function check(a : boolean, b : boolean) : boolean {
  return a & b
}

// PASS: logical operator short-circuits
function check(a : boolean, b : boolean) : boolean {
  return a && b
}

// FAIL: redundant-size-check - always true
function alwaysTrue(list : List<String>) {
  if (list.size() >= 0) { doWork() }
}

// FAIL: interval-boundary-confusion - inclusive range with size() causes off-by-one
function processItems(items : List<String>) {
  for (i in 0..items.size()) {      // off-by-one: iterates 0 through items.size() inclusive
    print(items[i])                 // IndexOutOfBoundsException when i == items.size()
  }
}

// PASS: use exclusive upper bound
function processItems(items : List<String>) {
  for (i in 0..|items.size()) {     // correct: 0 through size()-1
    print(items[i])
  }
}

// PASS: or use direct iteration
function processItems(items : List<String>) {
  for (item in items) {
    print(item)
  }
}
```

</details>

---

### style

| Token | Detects |
|-------|---------|
| `block-parameter-ignored` | Lambda or block expressions with declared parameters that are never referenced in the body. A block parameter that goes unused is misleading - it implies the body depends on each element when it does not. |
| `constant-naming` | `static final` variables whose names do not follow `UPPER_SNAKE_CASE`. Fields that are neither `static` nor `final` are not checked by this rule. |
| `constructor-order` | Constructors (`construct`) that are not grouped together before any non-constructor methods. When constructors are interspersed with regular methods the class structure is harder to navigate. |
| `enhancement-modifies-state` | Enhancement (`.gsx`) methods that mutate the enhanced type's fields via `this.field = ...`. Enhancements are stateless extensions and should not write to the enhanced type's internal state. |
| `excessive-newlines` | More than a configurable number of consecutive blank lines. A single blank line separates logical sections; multiple blank lines in a row add whitespace without structure. **Configurable:** `maxBlankLines` (default: **3**) |
| `foreach-index-arithmetic` | Manual counter variables declared before a for-each loop and incremented inside it. Gosu's `for (item in collection index i)` syntax provides a built-in, zero-based index variable; maintaining a separate counter is redundant boilerplate. |
| `override-grouping` | `override` methods that are not grouped consecutively. A reader scanning for overridden behaviour has to search the whole class when overrides are scattered among other methods. |
| `print-statement` | `print()` or `println()` calls in production code. These are Gosu built-ins that write to stdout and are appropriate in scripts and quick experiments but should not appear in production classes. |
| `property-vs-method` | Trivial properties that simply wrap a backing field with no added logic (validation, transformation, or access-control). The backing field could be exposed as public, or Gosu's `var X : T` shorthand could be used. |
| `string-plus-in-expansion` | String concatenation (`+`) inside a template interpolation expression (`${}`) where one operand is a string literal. The literal can be moved outside the braces to make the template cleaner. |
| `this-qualifier-consistency` | Inconsistent use of `this.` to qualify field accesses within the same class - some accesses use the qualifier and others do not, making the code harder to scan. |

<details>
<summary>style examples</summary>

```gosu
// FAIL: block-parameter-ignored - `item` is declared but never used
function getName(customers : List<Customer>) : List<String> {
  return customers.map(\item -> "Unknown")
}

// PASS: parameter is actually used
function getName(customers : List<Customer>) : List<String> {
  return customers.map(\item -> item.Name)
}

// FAIL: enhancement-modifies-state - enhancing type's field is modified
enhancement AutoEnhancement : Auto {
  function resetName() {
    this._name = ""    // bad: modifies enhanced type's field
  }
}

// PASS: read-only enhancement
enhancement AutoEnhancement : Auto {
  function getName() : String {
    return this._name
  }
}

// FAIL: foreach-index-arithmetic - manual counter is redundant
function printWithIndex(items : List<String>) {
  var counter = 0
  for (item in items) {
    print(counter + ": " + item)
    counter++
  }
}

// PASS: use built-in index clause
function printWithIndex(items : List<String>) {
  for (item in items index i) {
    print(i + ": " + item)
  }
}

// FAIL: property-vs-method - trivial property wrapper
class User {
  property get Name : String {
    return _name
  }
  property set Name(v : String) {
    _name = v
  }
}

// PASS: use Gosu shorthand
class User {
  var _name : String as Name = "default"
}

// FAIL: string-plus-in-expansion - string literal concatenation inside ${}
function greet(name : String) : String {
  return "Hello: ${"Mr. " + name}"
}

// PASS: move literal outside braces
function greet(name : String) : String {
  return "Hello: Mr. ${name}"
}

// FAIL: print-statement
function process(item : Item) {
  print("Processing: " + item)
  item.run()
}

// PASS: use a logger
function process(item : Item) {
  LOG.info("Processing: " + item)
  item.run()
}

// FAIL: constant-naming - not UPPER_SNAKE_CASE
static final var maxRetries : int = 3
static final var appVersion : String = "1.0"
static final var PI_value : double = 3.14

// PASS
static final var MAX_RETRIES : int = 3
static final var APP_VERSION : String = "1.0"
static final var PI : double = 3.14

// FAIL: constructor-order - construct() ... doWork() ... construct(int)
class Example {
  construct() { _x = 0 }
  function doWork() { print("work") }
  construct(x : int) { _x = x }   // second constructor after a method
}

// PASS: all constructors grouped first
class Example {
  construct() { _x = 0 }
  construct(x : int) { _x = x }
  function doWork() { print("work") }
}

// FAIL: override-grouping - override methods interspersed with helpers
class Widget {
  override function toString() : String { return "Widget" }
  function helper() : String { return "help" }
  override function hashCode() : int { return 42 }  // not grouped
}

// PASS
class Widget {
  override function toString() : String { return "Widget" }
  override function hashCode() : int { return 42 }
  function helper() : String { return "help" }
}
```

```bash
java -jar gslint.jar src --rule-config excessive-newlines:maxBlankLines=2
```

</details>

---

### vendor

Rules derived from published secure-coding standards. Token names encode the standard family and rule ID so findings can be traced back to the source document.

| Token | Detects |
|-------|---------|
| `cls01g-public-primitive-field` | Public primitive (`int`, `double`, `boolean`, etc.) instance fields. CLS01-G requires that mutable state not be directly accessible to callers. |
| `cls01g-public-reference-field` | Public reference-type instance fields. Reference fields expose their referent to external mutation even when declared `final`. |
| `cls01g-readonly-array` | `public static final` array fields. Arrays are always mutable - `final` only prevents reassignment of the reference, not modification of the array contents. Callers can still do `MyClass.ITEMS[0] = null`. |
| `cls01g-readonly-reference` | A `private` field exposed as a `readonly` property where the type is a mutable reference (not a primitive, `String`, or array). The `readonly` modifier prevents reassignment but does not prevent the caller from mutating the object's state through the reference. |
| `err01g-throw-generic` | Functions that `throw new RuntimeException(...)`, `throw new Exception(...)`, `throw new Throwable(...)`, or `throw new Error(...)`. ERR01-G requires throwing the most specific exception type available so callers can handle distinct failure modes. |
| `err01g-throw-string` | Gosu allows `throw "message"` which coerces the string to a `RuntimeException`. This hides the exception type and should always be replaced with `throw new SomeException("message")`. |
| `res00g-close-not-in-finally` | A closeable resource (e.g. `Connection`, `InputStream`, `ResultSet`) that is closed in the `try` body rather than in a `finally` block or `using` statement. If an exception is thrown before the `close()` call the resource leaks. |
| `res00g-not-in-using` | A closeable field variable that is not wrapped in a `using` statement. Gosu's `using` block is the idiomatic resource-management construct and is equivalent to Java's try-with-resources. |
| `res00g-unsafe-close-in-finally` | A `close()` call in a `finally` block that is not itself wrapped in a `try`/`catch`. If `close()` throws, the original exception from the `try` body is suppressed and lost. |

<details>
<summary>vendor examples</summary>

```gosu
// FAIL: err01g-throw-generic
function validate(s : String) {
  if (s == null) throw new RuntimeException("Null input")
  if (s.length > 256) throw new Exception("Too long")
}

// PASS: specific exception types
function validate(s : String) {
  if (s == null) throw new NullPointerException("s must not be null")
  if (s.length > 256) throw new IllegalArgumentException("s exceeds 256 chars")
}

// FAIL: err01g-throw-string
function checkAge(age : int) {
  if (age < 21) throw "User is underage"
}

// PASS
function checkAge(age : int) {
  if (age < 21) throw new IllegalArgumentException("User must be 21+")
}

// FAIL: cls01g-readonly-array - array contents are still mutable
public static final var ALLOWED_CODES : String[] = {"A", "B", "C"}

// PASS: immutable list instead
public static final var ALLOWED_CODES : List<String> = java.util.Collections.unmodifiableList({"A", "B", "C"})

// FAIL: res00g-close-not-in-finally - close() in try body leaks on exception
function query() {
  try {
    var conn = getConn()
    var rs = conn.createStatement().executeQuery("SELECT 1")
    processResults(rs)
    rs.close()      // FAIL: not reached if processResults throws
    conn.close()
  } catch (e : Exception) { }
}

// PASS: using statement handles cleanup
function query() {
  using (var conn = getConn()) {
    var rs = conn.createStatement().executeQuery("SELECT 1")
    processResults(rs)
  }
}

// FAIL: res00g-unsafe-close-in-finally - close() can throw and suppress the original exception
function query() {
  var conn : Connection = null
  try {
    conn = getConn()
    processResults(conn.createStatement().executeQuery("SELECT 1"))
  } finally {
    conn?.close()    // FAIL: throws will suppress original exception
  }
}

// PASS: each close wrapped in its own try/catch
function query() {
  var conn : Connection = null
  try {
    conn = getConn()
    processResults(conn.createStatement().executeQuery("SELECT 1"))
  } finally {
    try { conn?.close() } catch (e : Exception) { }
  }
}
```

</details>

---
