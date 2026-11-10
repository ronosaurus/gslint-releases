# Writing a Custom Rule: `comparator-subtraction`

This walkthrough builds one complete custom rule, packages it as its own JAR, and loads it into gslint. For the short version of how custom rules are loaded, see [Extending with Custom Rules](README.md#extending-with-custom-rules).

## The rule

A `Comparator` that returns `a.Priority - b.Priority` looks correct but is not. Subtracting `-1` from `Integer.MAX_VALUE` overflows to a negative number, so the comparator says the larger value is smaller and sorting silently misorders data.

```gosu
// Flag
return a.Priority - b.Priority

// Pass
return Integer.compare(a.Priority, b.Priority)
```

**Scope.** The rule flags a direct `return <int> - <int>` (parentheses allowed) in the `compare(a, b)` method of a named class that implements `java.util.Comparator`. It deliberately leaves out comparator lambdas, anonymous classes, `Comparable.compareTo`, and a subtraction stored in a local variable first. Those make good follow-up exercises.

> **Note:** this example is not trying to be the smallest possible rule. It is deliberately a little bigger than that so it can show a few interesting things you will meet in real rules: checking real type identity, ignoring code inside lambdas, and handling parentheses and unresolved types safely.

## What the rule teaches

1. `@RuleToken` gives the rule its token, description, and whether it needs resolved types.
2. Extending `BaseRule<Violation>` and overriding `processFunction` runs your code once per method.
3. `RuleUtils.isType` checks real type identity (the class really is a `Comparator`) instead of matching on a name.
4. `LocalScopes.collectOutsideLambdas` collects only the `return` statements that belong to `compare` itself, not ones inside a lambda or anonymous class in its body.
5. `new Violation(line, message)` reports a line-numbered finding.

## Project layout

```
my-rules/
  src/main/java/com/example/rules/ComparatorSubtractionRule.java
```

## The Java source

The example uses Java 16+ syntax (pattern matching for `instanceof`) and is compiled for Java 21. gslint itself targets Java 11, so it runs on any JDK from 11 up. Your rule JAR just needs to be compiled for a release no newer than the JDK you run gslint on: if you run gslint on Java 17, compile with `--release 17` instead.

```java
package com.example.rules;

import gw.lang.parser.*;
import gw.lang.parser.expressions.*;
import gw.lang.parser.statements.*;
import gw.lang.reflect.java.JavaTypes;
import org.gslint.parser.*;
import org.gslint.registry.*;
import java.util.List;

/**
 * Flags {@code compare(a, b)} methods of named {@code java.util.Comparator} implementations
 * that return {@code a.X - b.X} for int operands.
 *
 * <p>Subtraction overflows: {@code Integer.MAX_VALUE - (-1)} wraps to a negative number, so the
 * comparator claims the larger value is smaller and sorting silently misorders data.
 */
@RuleToken(value = "comparator-subtraction",
        displayName = "Comparator Subtraction",
        description = "Flags Comparator.compare methods that return the difference of two ints, which can overflow.",
        typeResolution = TypeResolution.REQUIRED)
public class ComparatorSubtractionRule extends BaseRule<Violation> {

    @Override
    protected void processFunction(String name, IDynamicFunctionSymbol function,
                                   IParseTree body, List<Violation> violations) {
        // Cheap checks first, and the type check last: only a two-argument method called
        // "compare" with a body can match. The type check is identity, not name matching: the
        // owning class must really be a Comparator (directly or through a superclass/interface),
        // not just have "Comparator" in its name.
        if (body == null || !"compare".equals(function.getDisplayName())
                || function.getArgs().size() != 2
                || !RuleUtils.isType(function.getGosuClass(), "java.util.Comparator")) {
            return;
        }

        // Skip returns that belong to a lambda or anonymous class inside compare(); they
        // return from that inner callable, not from compare() itself.
        for (IReturnStatement ret : LocalScopes.collectOutsideLambdas(body, IReturnStatement.class)) {
            IExpression value = unwrapParens(ret.getValue());
            if (!(value instanceof IAdditiveExpression sum)) {
                continue;
            }
            if ("-".equals(sum.getOperator()) && isInt(sum.getLHS()) && isInt(sum.getRHS())) {
                violations.add(new Violation(ret.getLineNum(),
                        "compare() returns a subtraction of two ints, which can overflow - use Integer.compare"));
            }
        }
    }

    /** Peels off any number of surrounding parentheses: {@code ((a - b))} becomes {@code a - b}. */
    private static IExpression unwrapParens(IExpression expr) {
        while (expr instanceof IParenthesizedExpression paren) {
            expr = paren.getExpression();
        }
        return expr;
    }

    /** True when the operand resolved to the primitive {@code int} (not long, double or an error type). */
    private static boolean isInt(IExpression operand) {
        return operand != null && JavaTypes.pINT().equals(operand.getType());
    }
}
```

Two details worth knowing:

- `typeResolution = TypeResolution.REQUIRED` declares that the rule largely stops firing when types are missing. Use `NOT_NEEDED` for purely syntactic rules and `HELPFUL` when types improve results but are not essential.
- An unresolved operand has an error type, not `int`, so `isInt` returns `false` and the rule stays quiet instead of guessing.

## Build

The only compile-time dependency is the runnable gslint uber JAR (`gslint-<version>-gosu-<gosu-version>.jar` from `target/` or a release). It bundles the Gosu API classes (`gw.lang.*`) your rule imports; the thin `gslint-<version>.jar` does not. The commands below assume you copied it next to your project and renamed it `gslint.jar`.

```bash
javac --release 21 -cp gslint.jar -d out src/main/java/com/example/rules/*.java
jar cf my-rules.jar -C out .
```

The result is `my-rules.jar`. Rules are discovered by their fully-qualified class name, so no service files or registration are needed.

## Try it on a fixture

`sample/demo/Task.gs`:

```gosu
package demo

class Task {
  var _priority : int as Priority
}
```

`sample/demo/TaskSorters.gs`:

```gosu
package demo

uses java.util.Comparator

class TaskSorters {
  // Flag: overflow-prone subtraction
  static class BySubtraction implements Comparator<Task> {
    override function compare(a : Task, b : Task) : int {
      return a.Priority - b.Priority
    }
  }

  // Flag: parentheses do not hide it
  static class ByParens implements Comparator<Task> {
    override function compare(a : Task, b : Task) : int {
      return (a.Priority - b.Priority)
    }
  }

  // Pass: safe comparison
  static class Safe implements Comparator<Task> {
    override function compare(a : Task, b : Task) : int {
      return Integer.compare(a.Priority, b.Priority)
    }
  }

  // Pass: a lambda's return does not belong to compare()
  static class WithLambda implements Comparator<Task> {
    override function compare(a : Task, b : Task) : int {
      var f = \ x : int, y : int -> { return x - y }
      return Integer.compare(f(a.Priority, 0), f(b.Priority, 0))
    }
  }

  // Pass: not a Comparator, so compare() is just a method
  static class NotAComparator {
    function compare(a : Task, b : Task) : int {
      return a.Priority - b.Priority
    }
  }
}
```

## Run it

`java -jar` ignores `-cp`, so put both JARs on the classpath and name `org.gslint.Main` yourself. Use `;` between entries on Windows and `:` on Linux and macOS.

```bash
# Linux / macOS
java -cp "gslint.jar:my-rules.jar" org.gslint.Main \
  --include sample --init-src-root sample \
  --rules com.example.rules.ComparatorSubtractionRule --rules-only-custom

# Windows
java -cp "gslint.jar;my-rules.jar" org.gslint.Main ^
  --include sample --init-src-root sample ^
  --rules com.example.rules.ComparatorSubtractionRule --rules-only-custom
```

Expected output:

```
-- comparator-subtraction | demo/TaskSorters.gs --
Line 9: compare() returns a subtraction of two ints, which can overflow - use Integer.compare
Line 16: compare() returns a subtraction of two ints, which can overflow - use Integer.compare
```

Drop `--rules-only-custom` and list the rule alongside built-in tokens to run both: `--rules "empty-catch,com.example.rules.ComparatorSubtractionRule"`.

## Shipping the rule with a `LintHook`

Instead of listing the rule on every command line, bundle it with a `LintHook` subclass. A hook registers rules and can also watch or reshape a run. Making your `Main` the hook keeps the whole extension in one class:

```java
package com.example;

import org.gslint.report.LintReport;
import org.gslint.spi.LintHook;
import org.gslint.spi.RuleRegistrar;

import java.io.IOException;
import java.util.Arrays;

/*
 * Order in which gslint calls a hook during one run:
 *
 *   onRegisterRules(registrar)   register extra rule classes (before rules are resolved)
 *   onLintStart(context)         once, before any file is parsed
 *   per file:
 *     onBeforeFile(file)         before parsing; can skip the file
 *     onFileParsed(parsed)       after parsing, before rules run; can skip the file
 *     onFileReport(report)       after the rules ran on that file
 *   onLintComplete(report)       -> LintReport      (may be transformed)
 *   onCreateFormatter(default)   -> OutputFormatter (may be replaced or wrapped)
 *   [formatter.configure() + formatter.format() run here]
 *
 * Key point: onLintComplete runs BEFORE any output is written, and the report it
 * returns is the one the formatter prints. It is not a "done, file already written"
 * callback. Return the report unchanged to just observe; return a modified copy to
 * change the output. Never return null. There is no after-output callback; run that
 * code after the entry point returns.
 *
 * Hooks do not run in turbo-parse mode (--turbo-parse-workers >= 2).
 */
public class Main extends LintHook {

    public static void main(String[] argv) throws IOException {
        // Forward the command line to gslint and name this class as the hook. gslint creates
        // it by name, so it needs a public no-argument constructor.
        String[] full = Arrays.copyOf(argv, argv.length + 2);
        full[argv.length] = "--lint-hook";
        full[argv.length + 1] = Main.class.getName();
        org.gslint.Main.main(full); // fully qualified: both classes are called Main
    }

    @Override
    public void onRegisterRules(RuleRegistrar registrar) {
        // Registered rules obey --rules / --skip-rules like built-ins and need no FQCN on the CLI.
        registrar.register(ComparatorSubtractionRule.class);
    }

    @Override
    public LintReport onLintComplete(LintReport report) {
        System.err.println("Total violations: " + report.totalViolations()); // observe only
        return report;
    }
}
```

Run it with your class as the entry point (`java -jar` ignores `-cp`):

```bash
java -cp "gslint.jar;my-rules.jar" com.example.Main --include sample --init-src-root sample
```

Only override the stages you need; the rest are no-ops. Use this wrapper for `lint` only.

### Advanced: parse `argv` yourself and pass the hook directly

If you would rather not append `--lint-hook`, parse `argv` into a `LintArgs` and hand a hook instance to `LintService.execute`. Replace `main` in the class above with:

`LintArgs` and `LintService` come from gslint (`org.gslint.cli` and `org.gslint.cmd`), so import them as usual. `JCommander` does not come from gslint or from the standard `com.beust` library: it is the Gosu runtime's shaded copy, in `gw.internal.ext.com.beust.jcommander` and bundled in the gslint uber JAR. Use that package, not `com.beust.jcommander`, which is why the snippet writes it fully qualified.

```java
public static void main(String[] argv) {
    LintArgs args = new LintArgs();
    gw.internal.ext.com.beust.jcommander.JCommander jc = new gw.internal.ext.com.beust.jcommander.JCommander(args);
    try {
        jc.parse(argv);
        args.validateFlagDependencies();
    } catch (gw.internal.ext.com.beust.jcommander.ParameterException | IllegalArgumentException e) {
        System.err.println("Error: " + e.getMessage());
        System.exit(1);
    }
    if (args.help) {
        jc.usage();
        return;
    }
    // Hook instance passed directly, so no --lint-hook is needed. If argv also has --lint-hook,
    // both hooks run, composed, and yours runs first.
    System.exit(LintService.execute(args, new Main()));
}
```

This path skips parts of the normal command line that live inside `org.gslint.Main` and are not public:

- "Did you mean `--include`?" suggestions for misspelled flags and the unclosed-quote check.
- Sub-command handling. Only `lint` flags are understood, so a leading `lint` word is rejected.
- The standard logging setup, so log level and format can differ from the normal CLI.

Use the forwarding version above when you want all of that. Use this one when you want full control of the process, for example to call `LintService.execute` from a larger program and keep the exit code.

## Where to go next

- Browse the built-in rules under `src/main/java/org/gslint/rule/` for working patterns. `rule/dataflow/IdentityReturnRule.java` is a good next read: it also walks return statements and skips lambdas.
- [RULES.md](RULES.md) shows how a rule's class JavaDoc becomes its documentation. A `<pre>{@code ... }</pre>` block turns into the Noncompliant/Compliant example.
- Follow-up exercises: cover `Comparable.compareTo`, comparator lambdas and anonymous classes, and a subtraction stored in a local variable before it is returned.
