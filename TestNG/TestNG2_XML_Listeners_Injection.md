# TestNG XML, CLI & Core API Masterclass — The Complete Beginner-Friendly Guide

**Source of truth:** Official TestNG Documentation — `https://testng.org/#_testng_documentation`. DTD: `https://testng.org/testng-1.0.dtd` (use `testng-1.1.dtd` for the newer thread-pool-control attributes documented in §1.4).

> This guide keeps **every** table, XML snippet, Java example, interview question, and mistake from the original reference — nothing has been removed. What's different is that every topic now opens with a plain-English explanation before the technical details, so someone who has never touched `testng.xml` or a TestNG listener before can follow along step by step.

## Table of Contents
0. What This Guide Covers, and How TestNG "Configuration" Actually Works
1. `testng.xml` Schema — Suite, Test, Groups, Parallelism Attributes
2. Command-Line Parameters — Full Reference Table
3. System Properties
4. Core Listener APIs (the "TestNG Listeners" family)
   - 4.3 Full Working Listener Implementations — Real-World Use Cases (13 complete, wired examples: `IAnnotationTransformer`, `IRetryAnalyzer`, `ITestListener`, `ISuiteListener`, `IInvokedMethodListener`, `IConfigurationListener`, `IMethodInterceptor`, `IDataProviderInterceptor`, `IHookable`, `IReporter`, `IExecutionListener`, `IAlterSuiteListener`)
5. Dependency Injection — Native Injection Type Matrix
6. Programmatic API — Running TestNG from Java
7. Reporter API, Exit Codes & Retry Mechanics
8. Test Results, Assertions & Alternate Run Modes (Assertions, XML/JUnit Reports, JUnit Interop, YAML Suites, Dry Run, JVM Arguments, `IConfigurable`, Test-Jar Execution)
9. Interview Prep & Cheat Sheet

---

# 0. What This Guide Covers, and How TestNG "Configuration" Actually Works

If you've already met `@Test`, `@BeforeMethod`, and the other annotations, you know how to label *individual methods*. But real projects don't run one method at a time — they run entire test suites, often across many classes, sometimes in parallel, sometimes filtered down to just a handful of "smoke" tests before a deploy. That orchestration layer — "which classes run, in what order, with how many threads, and what happens when a test fails" — is controlled from **outside** your Java code, mainly through a file called `testng.xml`, plus command-line flags and a family of "listener" interfaces that let you hook into what TestNG is doing.

Think of it this way:
- **Annotations** (`@Test`, `@BeforeMethod`, etc.) live *inside* your test classes and describe individual methods.
- **`testng.xml`** lives *outside* your code and describes how to run a whole collection of those classes — which ones, in what groups, how many threads, what listeners to attach.
- **The command line** (`java org.testng.TestNG testng.xml`) is how you actually kick off a run, and it has its own flags that can override or supplement the XML.
- **Listeners** are Java classes you write that "plug in" to TestNG's execution pipeline — they get notified when a test starts, fails, when a suite finishes, etc., so you can add behavior like screenshots-on-failure or custom reports without touching the test methods themselves.

This guide walks through all four of those pieces, in the order you're most likely to need them: first the XML file that defines what runs, then the CLI that launches it, then system properties, then the listener APIs that let you customize behavior, then dependency injection, the programmatic (pure-Java) way of doing all of this without an XML file at all, the reporting/retry/exit-code mechanics, and finally assertions and some less-common run modes. It closes with interview-style Q&A and a cheat sheet you can use for quick lookups later.

---

# 1. `testng.xml` Schema

**Beginner explanation:** `testng.xml` is the "control panel" for a TestNG run. Instead of hardcoding which classes to run inside Java, you list them in this XML file, along with rules like "only run tests tagged `checkinTests`," "use 4 threads," or "load this custom listener before starting." TestNG reads this file, builds an execution plan from it, and then runs your tests according to that plan. You don't need to memorize the whole schema on day one — but knowing it exists, and roughly what each block controls, is the foundation for everything else in this guide.

## 1.1 Minimal Structure

Here's a small but realistic `testng.xml`. Read it top-to-bottom: it names a suite, sets some suite-wide options, registers a listener, and then defines two separate `<test>` blocks — one that picks specific classes/methods and filters by group, and one that just scans an entire package.

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="RegressionSuite" verbose="1" parallel="methods" thread-count="4" preserve-order="true">
    <parameter name="browser" value="chrome"/>
    <listeners>
        <listener class-name="com.qa.listeners.ScreenshotListener"/>
    </listeners>
    <test name="LoginModuleTests">
        <groups>
            <run>
                <include name="checkinTests"/>
                <exclude name="brokenTests"/>
            </run>
        </groups>
        <classes>
            <class name="com.qa.tests.LoginTest">
                <methods>
                    <include name="testValidLogin"/>
                    <exclude name="testDeprecatedFlow"/>
                </methods>
            </class>
        </classes>
    </test>
    <test name="PackageScan">
        <packages>
            <package name="com.qa.tests.dashboard"/>
        </packages>
    </test>
</suite>
```

## 1.2 `<suite>` Attributes

**Beginner explanation:** Everything in a `testng.xml` file lives inside one `<suite>` tag — it's the outermost container, and its attributes set the *defaults* for everything nested inside it (individual `<test>` blocks can override most of these locally, which you'll see in §1.3).

| Attribute | Values | Purpose |
|---|---|---|
| `name` | string | Mandatory; unique identifier used in reports. |
| `verbose` | `0`–`10` | Console diagnostic detail; `10` = maximum internal TestNG logging. |
| `parallel` | `methods` \| `tests` \| `classes` \| `instances` | Default parallelism mechanism for this suite (see §1.5). |
| `thread-count` | int | Threads allocated for the chosen `parallel` mode. |
| `preserve-order` | `true`/`false` | `true` (default) runs classes/methods in XML-declared order; `false` allows unpredictable order. |
| `allow-return-values` | `true`/`false` | Permits `@Test` methods that return a value (ignored otherwise). |
| `group-by-instances` | `true`/`false` | Forces per-instance (not per-class) dependency ordering for `@Factory`-produced instances. |
| `data-provider-thread-count` | int | Thread-pool size for `parallel = true` data providers (default 10). |
| `parent-module` | FQCN | Guice parent module shared across the whole suite. |
| `guice-stage` | `DEVELOPMENT` \| `PRODUCTION` \| `TOOL` | Guice injector stage (default `DEVELOPMENT`). |

**Newer, `testng-1.1.dtd`-only suite attributes (TestNG 7.9.0+):** these two are recent additions, so they only work if you switch your `DOCTYPE` line to the newer `testng-1.1.dtd`, shown right after the table.

| Attribute | Purpose |
|---|---|
| `share-thread-pool-for-data-providers` | If `true`, all data-driven tests in the suite share one thread pool sized by `data-provider-thread-count`. Default `false`. |
| `use-global-thread-pool` | If `true`, regular **and** data-driven tests share one common thread pool sized by `thread-count`. Default `false`. |

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.1.dtd">
<suite name="sample" use-global-thread-pool="true" thread-count="8">
```

## 1.3 `<test>`, `<classes>`, `<packages>`, `<methods>`

**Beginner explanation:** A `<suite>` can contain multiple `<test>` blocks — think of a `<test>` as a named sub-run inside the suite. This is the level `@BeforeTest`/`@AfterTest` annotations are tied to (from the annotations guide). Inside a `<test>` block, you tell TestNG *which* classes/packages/methods to actually run, using one of the tags below.

- `<test name="...">` — smallest scope `@BeforeTest`/`@AfterTest` binds to; can also carry its own `parallel`/`thread-count`, `preserve-order`, `junit="true"`, `group-by-instances`, and `allow-return-values`, overriding the suite defaults for just this block.
- `<classes><class name="fully.qualified.Name"/></classes>` — explicit class listing.
- `<packages><package name="com.qa.tests.dashboard"/></packages>` — scans one package level for annotated classes; **does not recurse sub-packages** by default.
- `<methods><include name="testX"/><exclude name="testY"/></methods>` — nested under `<class>`; regex-capable, **exclude always wins** when both match.

**Common trap for beginners:** people assume `<packages>` will automatically pull in every sub-package too — it won't. If you have `com.qa.tests` and `com.qa.tests.dashboard`, and you only list `com.qa.tests`, the `dashboard` classes are silently skipped. You'd need to list `com.qa.tests.dashboard` as its own `<package>` entry.

## 1.4 Groups: Definition, Meta-Groups, Exclusion, Dependencies

**Beginner explanation:** "Groups" are just labels (tags) you attach to `@Test` methods (e.g., `groups = "smoke"`), and `testng.xml` is where you decide which of those labels actually get run in a given execution. This section covers three related ideas: bundling multiple groups into a bigger "meta-group" with `<define>`, filtering with `<run>`, and expressing that one group can't run until another group has (`<dependencies>`).

```xml
<test name="Regression1">
  <groups>
    <define name="functest">
      <include name="windows"/>
      <include name="linux"/>
    </define>
    <define name="all">
      <include name="functest"/>
      <include name="checkintest"/>
    </define>
    <run>
      <include name="all"/>
      <exclude name="broken"/>
    </run>
    <dependencies>
      <group name="c" depends-on="a b"/>
      <group name="z" depends-on="c"/>
    </dependencies>
  </groups>
  <classes><class name="test.sample.Test1"/></classes>
</test>
```
- `<define>` builds "meta-groups" (groups of groups) — resolved before `<run>` filtering.
- Group names support **regular expressions** (not wildmats) — `"windows.*"` matches `windows.checkintest`, `windows.functest`, etc.
- `<dependencies><group depends-on="...">` — an XML-only alternative to Java-side `dependsOnGroups`, using a space-separated group list.

## 1.5 Parallelism Modes — Precise Semantics

**Beginner explanation:** `parallel` controls *what unit* of work gets its own thread. Getting this wrong is a very common source of flaky tests — for example, running `parallel="methods"` on a class that shares one non-thread-safe object across all its test methods. The table below spells out exactly what each mode isolates.

| Mode | Behavior |
|---|---|
| `parallel="methods"` | Every test method runs in its own thread. Dependent methods still run in separate threads but respect declared order. |
| `parallel="tests"` | All methods within one `<test>` tag share a thread; each `<test>` tag gets its own thread. Use to isolate non-thread-safe classes together while still parallelizing across `<test>` blocks. |
| `parallel="classes"` | All methods in one class share a thread; each class gets its own thread. |
| `parallel="instances"` | All methods on one factory-produced instance share a thread; different instances run on different threads. |

Suite-level parallelism (running multiple `.xml` files concurrently) is separate — controlled via `-suitethreadpoolsize` on the CLI, not the `parallel` suite attribute.

## 1.6 BeanShell Method Selectors (advanced filtering)

**Beginner explanation:** This is an advanced, less commonly used feature — a way to write a tiny script that decides, method by method, whether it should run. Most projects never need this (plain `<include>`/`<exclude>` and groups cover 95% of cases), but it's worth knowing it exists for the rare case where filtering logic is too dynamic to express as a static list.

```xml
<test name="BeanShell test">
  <method-selectors>
    <method-selector>
      <script language="beanshell"><![CDATA[
        groups.containsKey("test1")
      ]]></script>
    </method-selector>
  </method-selectors>
</test>
```
Must return a `boolean`. TestNG exposes `method` (`java.lang.reflect.Method`), `testngMethod` (`ITestNGMethod`), and `groups` (`Map<String,String>`) inside the script scope. **When a `<script>` tag is present, TestNG ignores any `<include>`/`<exclude>` in that `<test>` block entirely** — the script becomes the sole filter. Requires an explicit BeanShell dependency since TestNG 7.5 (no longer bundled).

---

# 2. Command-Line Parameters — Full Reference

**Beginner explanation:** Once you have a `testng.xml`, you launch it from the command line (or via your build tool, which does this under the hood). The command-line flags below let you tweak or override behavior without editing the XML — handy for CI pipelines where you might want to pass a different thread count or group filter per environment, without maintaining separate XML files.

Invocation: `java org.testng.TestNG testng1.xml [testng2.xml ...]`

| Option | Argument | Purpose |
|---|---|---|
| `-configfailurepolicy` | `skip`,`continue` | Continue or skip remaining suite tests when an `@Before*` fails. Default `skip`. |
| `-d` | directory | Report output directory (default `test-output`). |
| `-dataproviderthreadcount` | int | Default thread count for parallel data providers. |
| `-excludegroups` | comma list | Groups to exclude. |
| `-groups` | comma list | Groups to run, e.g. `windows,regression`. |
| `-listener` | comma list of FQCNs | Custom `ITestListener`(and other listener) implementations. |
| `-usedefaultlisteners` | `true`,`false` | Toggle TestNG's built-in listeners. |
| `-methods` | `pkg.Class.method,...` | Run specific methods only. |
| `-methodselectors` | `pkg.Selector:priority,...` | Custom method selectors on the CLI. |
| `-parallel` | `methods,tests,classes,instances` | Default parallel mechanism (overridable in suite XML). |
| `-reporter` | `pkg.Reporter:key=val` | Configure a custom `IReporter` with JavaBean-style properties; repeatable. |
| `-sourcedir` | semicolon-separated dirs | Location of javadoc-style annotated sources (legacy). |
| `-suitename` | string | Suite name for a suite defined purely on the CLI. |
| `-testclass` | comma list of FQCNs | Test classes to run without an XML file. |
| `-testjar` | jar path | Run tests packaged inside a jar. |
| `-testname` | string | Name for a CLI-defined `<test>`. |
| `-testnames` | comma list | Restrict to `<test>` tags matching these names. |
| `-testrunfactory` | FQCN | Custom `ITestRunnerFactory`. |
| `-threadcount` | int | Default thread count for `-parallel`. |
| `-xmlpathinjar` | path | Path to the suite XML inside a `-testjar`; defaults to `testng.xml`. |
| `-shareThreadPoolForDataProviders` | `true`,`false` | Global shared thread pool for data-driven tests, sized by `-dataproviderthreadcount`. |
| `-useGlobalThreadPool` | `true`,`false` | Global shared thread pool for regular + data-driven tests, sized by `-threadcount`. |
| `-log` / `-verbose` | level | Logging verbosity. |
| `-junit` | `true`,`false` | Run in JUnit mode. |
| `-mixed` | `true`,`false` | Auto-detect JUnit vs TestNG per class. |
| `-objectfactory` | FQCN | Custom `ITestObjectFactory` for instance creation. |
| `-ignoreMissedTestNames` | `true`,`false` | Don't fail when `-testnames` references a non-existent `<test>`. |
| `-skipfailedinvocationcounts` | `true`,`false` | Skip (not fail) remaining `invocationCount` iterations after a failure. |
| `-suitethreadpoolsize` | int | Thread pool size for running **multiple suite files** concurrently (default 1). |
| `-randomizesuites` | `true`,`false` | Randomize suite execution order. |
| `-alwaysrunlisteners` | `true`,`false` | Run method-invocation listeners even for skipped methods. |
| `-dependencyinjectorfactory` | FQCN | Custom `IInjectorFactory`. |
| `-failwheneverythingskipped` | `true`,`false` | Fail the run if literally everything was skipped. |
| `-spilistenerstoskip` | comma list of FQCNs | Exclude specific listeners from ServiceLoader auto-wiring. |
| `-overrideincludedmethods` | `true`,`false` | Explicitly-included methods still get excluded if their group is excluded. |
| `-includeAllDataDrivenTestsWhenSkipping` | `true`,`false` | Report every data-driven iteration as an individual SKIP on upstream failure. |
| `-propagateDataProviderFailureAsTestFailure` | `true`,`false` | Treat a `@DataProvider` exception as a test failure rather than an error. |
| `-generateResultsPerSuite` | `true`,`false` | Write results into a per-suite subdirectory. |

**Precedence rule (worth memorizing):** CLI flags that specify *what to run* are ignored once a `testng.xml` is also supplied — **except** `-groups`/`-excludegroups`, which always override the XML's group filters. In other words: if you pass both a `testng.xml` and `-methods`, the `-methods` flag is silently ignored; but `-groups` on the CLI always wins over `<groups>` in the XML.

---

# 3. System Properties

**Beginner explanation:** System properties are values you pass to the JVM itself with `-D`, before TestNG even starts. TestNG uses one of its own (`testng.test.classpath`) to narrow where it looks for test classes, and — more commonly — lets you feed any `-D` value straight into your `@Parameters`-annotated methods, which is a quick way to override a config value per environment without touching the XML file.

| Property | Purpose |
|---|---|
| `testng.test.classpath` | Semicolon-separated dirs to search for test classes instead of the full classpath — useful with `<packages>` scanning to avoid noise from non-test classes. |
| `-D<name>=<value>` (JVM `-D` flags) | Any JVM system property is usable as a `@Parameters` source; overrides same-named `testng.xml` `<parameter>` values (since TestNG 7.x — not true in 6.x). |

```
java -Dfirst-name=Cedrick -Dlast-name="von Braun" org.testng.TestNG testng.xml
```

---

# 4. Core Listener APIs

**Beginner explanation:** Listeners are the plug-in system of TestNG. Instead of TestNG trying to guess every possible thing you might want to do (take a screenshot on failure, send a Slack message when a suite finishes, retry a flaky test, rewrite an annotation at runtime), it exposes a family of Java interfaces. You implement one, register it, and TestNG calls your methods at the right moment in its execution lifecycle. The table below is a map of "which listener do I reach for" — §4.1–4.3 then show exactly how to register them and give complete, working code for every single one.

| Listener | Fires on | Typical use |
|---|---|---|
| `ITestListener` | `onTestStart/Success/Failure/Skipped/FailedButWithinSuccessPercentage`, `onStart/onFinish` (per `<test>`) | Logging, screenshot-on-failure at result level. |
| `ISuiteListener` | `onStart(ISuite)/onFinish(ISuite)` | Suite-wide setup/teardown, broadest scope. |
| `IInvokedMethodListener` | `beforeInvocation`/`afterInvocation` around **every** method, config or test | Global timing/profiling, cross-cutting hooks. |
| `IConfigurationListener` | Configuration method success/failure specifically | Alerting when environment setup itself breaks, distinct from a test assertion failing. |
| `IClassListener` | `onBeforeClass`/`onAfterClass` | Class-scoped lifecycle hooks. |
| `IAnnotationTransformer` | During suite compilation, before any method runs | Mutating `@Test`/config/`@DataProvider`/`@Factory` annotation values at runtime. **Cannot** be wired via `@Listeners`. |
| `IMethodInterceptor` | After ordering is calculated, for methods with no explicit dependency relationship | Reordering/filtering the "free order" method bucket (e.g., always run "fast"-grouped tests first). |
| `IDataProviderInterceptor` | After a `@DataProvider` produces its iterator, before feeding the `@Test` | Mutating/filtering data-provider output (e.g., even-numbers-only filter). |
| `IHookable` | Wraps a single `@Test` invocation | Full control to skip/replace test invocation logic. |
| `IReporter` | Once, after the entire suite completes | Custom HTML/Extent/Allure report generation from `List<ISuite>`. |
| `IExecutionListener` | `onExecutionStart`/`onExecutionFinish` for the whole TestNG run | Global run-level bracketing across all suites. |
| `IAlterSuiteListener` | Before suite XML is finalized | Programmatically rewrite `XmlSuite` objects (e.g., inject classes at runtime). |

## 4.1 Registration Mechanisms

**Beginner explanation:** There are four different ways to "plug in" a listener — which one you use usually comes down to how broad the effect should be (one test class vs. the whole project) and whether the listener is one of the two special ones that *require* XML/CLI/code registration.

```xml
<suite>
  <listeners>
    <listener class-name="com.example.MyListener"/>
  </listeners>
</suite>
```
```java
@Listeners({ com.example.MyListener.class, com.example.MyMethodInterceptor.class })
public class MyTest { }
```
```
java org.testng.TestNG -listener com.example.MyTransformer testng.xml
```
```java
TestNG tng = new TestNG();
tng.addListener(new MyTransformer());
```
**Rule:** `@Listeners` accepts any `ITestNGListener` **except `IAnnotationTransformer`**, which must go through XML, CLI, or code — annotation rewriting must complete before TestNG finishes parsing `@Listeners` itself.

**ServiceLoader auto-wiring:** drop a jar on the classpath containing `META-INF/services/org.testng.ITestNGListener` (one FQCN per line) and TestNG wires it automatically — org-wide listener enforcement with zero XML/annotation boilerplate per project.

## 4.2 Ordering Listeners (TestNG 7.10.0+)

**Beginner explanation:** If you register several listeners, you sometimes need them to fire in a specific order (e.g., a logging listener should start before a metrics listener). This is a newer TestNG feature for exactly that.

```java
public class AnnotationBackedListenerComparator implements ListenerComparator {
    @Override
    public int compare(ITestNGListener l1, ITestNGListener l2) {
        return Integer.compare(getRunOrder(l1), getRunOrder(l2));
    }
}
```
Wire via `-listenercomparator <FQCN>` or `testng.setListenerComparator(...)`. Teardown order is symmetrical: the listener whose `onExecutionStart` ran **last** has its `onExecutionFinish` run **first**. Exclude preferred-order exemptions (e.g., IDE listeners) via `-Dtestng.preferential.listeners.package`.

---

## 4.3 Full Working Listener Implementations — Real-World Use Cases

**Beginner explanation:** Reading an interface signature rarely tells you *why* you'd use it. Each of the thirteen examples below is a complete, compilable class — imports included — plus its `testng.xml`/`@Listeners` wiring, so you can see the whole picture: the problem it solves, the code, and how it gets plugged in. These mirror patterns used in real production automation frameworks.

### 4.3.1 `IAnnotationTransformer` — Auto-Attach a Global Retry Analyzer

**Use case:** every `@Test` in the codebase should retry twice on failure by default, without every engineer remembering to add `retryAnalyzer = ...` by hand. A transformer rewrites the annotation at suite-compile time, before any method runs.

```java
package com.qa.listeners;

import org.testng.IAnnotationTransformer;
import org.testng.IRetryAnalyzer;
import org.testng.annotations.ITestAnnotation;
import org.testng.internal.annotations.DisabledRetryAnalyzer;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

public class GlobalRetryTransformer implements IAnnotationTransformer {

    @Override
    public void transform(ITestAnnotation annotation, Class testClass,
                           Constructor testConstructor, Method testMethod) {

        // Only attach a default retry if the developer didn't already specify one
        Class<? extends IRetryAnalyzer> current = annotation.getRetryAnalyzerClass();
        boolean noneSet = current == null || current == DisabledRetryAnalyzer.class;

        if (noneSet) {
            annotation.setRetryAnalyzer(DefaultRetryAnalyzer.class);
        }

        // Bonus: bump invocationCount for methods explicitly tagged "smoke" during a
        // canary run, driven by a system property set in the CI pipeline
        if (testMethod != null && annotation.getGroups() != null) {
            boolean isSmoke = java.util.Arrays.asList(annotation.getGroups()).contains("smoke");
            if (isSmoke && "true".equals(System.getProperty("canaryRun"))) {
                annotation.setInvocationCount(3);
            }
        }
    }
}
```

```java
package com.qa.listeners;

import org.testng.IRetryAnalyzer;
import org.testng.ITestResult;

public class DefaultRetryAnalyzer implements IRetryAnalyzer {
    private int count = 0;
    private static final int MAX_RETRY = 2;

    @Override
    public boolean retry(ITestResult result) {
        if (count < MAX_RETRY) {
            count++;
            return true;
        }
        return false;
    }
}
```

**Wiring — must be XML, CLI, or code, never `@Listeners`:**
```xml
<suite name="Suite">
    <listeners>
        <listener class-name="com.qa.listeners.GlobalRetryTransformer"/>
    </listeners>
    <test name="Regression">
        <classes><class name="com.qa.tests.CheckoutFlowTest"/></classes>
    </test>
</suite>
```
Console on a real failure-then-pass run:
```
FAILED: verifyCheckout (attempt 1)
FAILED: verifyCheckout (attempt 2)
PASSED: verifyCheckout (attempt 3, final)
```

### 4.3.2 `IRetryAnalyzer` + `ITestListener` — Retry With Retry-Count Visibility in Reports

**Use case:** raw retry hides *how many* retries actually fired — flaky-test dashboards need that count. Pair a per-method retry analyzer with a listener that stamps the final result.

```java
package com.qa.listeners;

import org.testng.IRetryAnalyzer;
import org.testng.ITestResult;

public class SmartRetryAnalyzer implements IRetryAnalyzer {
    private int attempt = 1;
    private static final int MAX_ATTEMPTS = 3;

    @Override
    public boolean retry(ITestResult result) {
        if (attempt < MAX_ATTEMPTS) {
            result.setAttribute("retryAttempt", attempt);
            attempt++;
            return true;
        }
        return false;
    }
}
```

```java
package com.qa.listeners;

import org.testng.ITestListener;
import org.testng.ITestResult;

public class RetryCountListener implements ITestListener {

    @Override
    public void onTestSuccess(ITestResult result) {
        Object retries = result.getAttribute("retryAttempt");
        if (retries != null) {
            System.out.printf("%s ultimately PASSED after %s retr%s%n",
                    result.getMethod().getMethodName(), retries,
                    ((int) retries == 1 ? "y" : "ies"));
        }
    }

    @Override
    public void onTestFailure(ITestResult result) {
        System.out.printf("%s FAILED permanently after exhausting retries%n",
                result.getMethod().getMethodName());
    }
}
```

```java
package com.qa.tests;

import com.qa.listeners.RetryCountListener;
import com.qa.listeners.SmartRetryAnalyzer;
import org.testng.Assert;
import org.testng.annotations.Listeners;
import org.testng.annotations.Test;

@Listeners(RetryCountListener.class)
public class PaymentGatewayTest {

    @Test(retryAnalyzer = SmartRetryAnalyzer.class)
    public void verifyPaymentConfirmation() {
        // A real-world flaky call: third-party sandbox gateway is intermittently slow
        Assert.assertTrue(PaymentGatewayClient.confirm("TXN-9001"));
    }
}
```

### 4.3.3 `ITestListener` — Screenshot-on-Failure for Selenium

**Use case:** the single most common listener beginners write first — automatically capture a screenshot the moment any `@Test` fails, without adding try/catch boilerplate to every test method.

```java
package com.qa.listeners;

import com.qa.factory.DriverFactory;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;
import org.testng.ITestListener;
import org.testng.ITestResult;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Paths;

public class ScreenshotListener implements ITestListener {

    @Override
    public void onTestFailure(ITestResult result) {
        WebDriver driver = DriverFactory.getDriver(); // same ThreadLocal the test used
        if (driver instanceof TakesScreenshot) {
            try {
                File src = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
                String fileName = result.getMethod().getMethodName()
                        + "_" + Thread.currentThread().getId() + ".png";
                Files.createDirectories(Paths.get("test-output/screenshots"));
                Files.copy(src.toPath(), Paths.get("test-output/screenshots", fileName));
                System.out.println("Screenshot saved: " + fileName);
            } catch (Exception e) {
                System.err.println("Could not capture screenshot: " + e.getMessage());
            }
        }
    }
}
```
Wire via `@Listeners(ScreenshotListener.class)` on a `BaseTest`, or `<listener class-name="com.qa.listeners.ScreenshotListener"/>` in XML.

### 4.3.4 `ISuiteListener` — Selenium Grid Session Bootstrap & Slack Notification

**Use case:** something that needs to happen exactly once for the whole suite — checking a Selenium Grid hub is reachable before anything starts, and posting a pass/fail summary once everything finishes.

```java
package com.qa.listeners;

import org.testng.ISuite;
import org.testng.ISuiteListener;
import org.testng.ISuiteResult;

import java.util.Map;

public class GridSuiteListener implements ISuiteListener {

    @Override
    public void onStart(ISuite suite) {
        System.out.println("Verifying Selenium Grid hub is reachable before: " + suite.getName());
        // e.g., HttpClient ping to http://grid-hub:4444/status here
    }

    @Override
    public void onFinish(ISuite suite) {
        int passed = 0, failed = 0, skipped = 0;
        for (Map.Entry<String, ISuiteResult> entry : suite.getResults().entrySet()) {
            passed += entry.getValue().getTestContext().getPassedTests().size();
            failed += entry.getValue().getTestContext().getFailedTests().size();
            skipped += entry.getValue().getTestContext().getSkippedTests().size();
        }
        String summary = String.format("Suite [%s] finished — Pass: %d, Fail: %d, Skip: %d",
                suite.getName(), passed, failed, skipped);
        System.out.println(summary);
        // postToSlack(summary); // real webhook call would go here
    }
}
```

### 4.3.5 `IInvokedMethodListener` — Per-Method Execution Time Profiling

**Use case:** you want timing data for *every* method TestNG invokes — not just `@Test` methods, but `@Before*`/`@After*` too — to find out whether a slow suite is actually a slow `@BeforeMethod`.

```java
package com.qa.listeners;

import org.testng.IInvokedMethod;
import org.testng.IInvokedMethodListener;
import org.testng.ITestResult;

import java.util.concurrent.ConcurrentHashMap;

public class ExecutionTimingListener implements IInvokedMethodListener {

    private final ConcurrentHashMap<String, Long> startTimes = new ConcurrentHashMap<>();

    @Override
    public void beforeInvocation(IInvokedMethod method, ITestResult testResult) {
        String key = method.getTestMethod().getMethodName() + "#" + Thread.currentThread().getId();
        startTimes.put(key, System.currentTimeMillis());
    }

    @Override
    public void afterInvocation(IInvokedMethod method, ITestResult testResult) {
        String key = method.getTestMethod().getMethodName() + "#" + Thread.currentThread().getId();
        Long start = startTimes.remove(key);
        if (start != null) {
            long durationMs = System.currentTimeMillis() - start;
            String type = method.isTestMethod() ? "@Test" : "@Config";
            System.out.printf("[%s] %s took %dms%n", type, method.getTestMethod().getMethodName(), durationMs);
        }
    }
}
```
This fires for **every** invoked method — `@BeforeMethod`/`@AfterMethod` included — because `IInvokedMethodListener` wraps configuration and test methods alike, unlike `ITestListener` which only reports `@Test` outcomes.

### 4.3.6 `IConfigurationListener` — Alert When Setup Itself Breaks

**Use case:** distinguish "the environment is broken" from "the test assertion failed" — a broken `@BeforeClass` (e.g., driver can't start) is an infra incident, not a product bug.

```java
package com.qa.listeners;

import org.testng.IConfigurationListener;
import org.testng.ITestResult;

public class ConfigFailureAlertListener implements IConfigurationListener {

    @Override
    public void onConfigurationFailure(ITestResult itr) {
        String failedMethod = itr.getMethod().getMethodName();
        System.err.println("ENVIRONMENT ALERT: configuration method '" + failedMethod
                + "' failed — likely infra issue, not a product bug: " + itr.getThrowable());
        // pageDutyClient.triggerIncident("TestNG config failure: " + failedMethod);
    }

    @Override
    public void onConfigurationSuccess(ITestResult itr) {
        // Optional: no-op, but available for audit logging
    }
}
```

### 4.3.7 `IMethodInterceptor` — Run "smoke" Group First, Always

**Use case:** force every test tagged `smoke` to run before everything else, computed dynamically rather than hand-ordered in the XML file.

```java
package com.qa.listeners;

import org.testng.IMethodInstance;
import org.testng.IMethodInterceptor;
import org.testng.ITestContext;
import org.testng.annotations.Test;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

public class SmokeFirstInterceptor implements IMethodInterceptor {

    @Override
    public List<IMethodInstance> intercept(List<IMethodInstance> methods, ITestContext context) {
        List<IMethodInstance> reordered = new ArrayList<>();
        for (IMethodInstance mi : methods) {
            Test test = mi.getMethod().getConstructorOrMethod().getMethod().getAnnotation(Test.class);
            Set<String> groups = test != null ? new HashSet<>(Arrays.asList(test.groups())) : new HashSet<>();
            if (groups.contains("smoke")) {
                reordered.add(0, mi);  // push smoke-tagged methods to the front
            } else {
                reordered.add(mi);
            }
        }
        return reordered;
    }
}
```
Only affects methods with **no** `dependsOnMethods`/`dependsOnGroups` relationship — dependency-chained methods already have a fixed order that this interceptor cannot violate.

### 4.3.8 `IDataProviderInterceptor` — Environment-Based Data Filtering

**Use case:** a data provider returns 500 production-scale records, but local dev runs should only exercise a fast 10-record subset.

```java
package com.qa.listeners;

import org.testng.IDataProviderInterceptor;
import org.testng.IDataProviderMethod;
import org.testng.ITestContext;
import org.testng.ITestNGMethod;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class DevModeDataLimiter implements IDataProviderInterceptor {

    @Override
    public Iterator<Object[]> intercept(Iterator<Object[]> original,
                                         IDataProviderMethod dataProviderMethod,
                                         ITestNGMethod method,
                                         ITestContext context) {
        if (!"dev".equalsIgnoreCase(System.getProperty("env", "prod"))) {
            return original; // no limiting outside dev
        }
        List<Object[]> limited = new ArrayList<>();
        int count = 0;
        while (original.hasNext() && count < 10) {
            limited.add(original.next());
            count++;
        }
        return limited.iterator();
    }
}
```
```java
@Listeners(DevModeDataLimiter.class)
public class BulkAccountValidationTest {
    @Test(dataProvider = "allAccounts")
    public void validate(String accountId) { /* ... */ }

    @DataProvider(name = "allAccounts")
    public Object[][] allAccounts() { /* returns 500 rows in prod */ return loadFromCsv(); }
}
```

### 4.3.9 `IHookable` — Conditional Test Skipping With Full Control

**Use case:** skip an entire category of tests at invocation time based on a feature flag service, while still reporting them distinctly from a normal `enabled=false` exclusion.

```java
package com.qa.listeners;

import org.testng.IHookCallBack;
import org.testng.IHookable;
import org.testng.ITestResult;
import org.testng.SkipException;

public class FeatureFlagHook implements IHookable {

    @Override
    public void run(IHookCallBack callBack, ITestResult testResult) {
        String requiredFlag = testResult.getMethod().getConstructorOrMethod()
                .getMethod().getAnnotation(RequiresFeatureFlag.class) != null
                ? testResult.getMethod().getConstructorOrMethod()
                    .getMethod().getAnnotation(RequiresFeatureFlag.class).value()
                : null;

        if (requiredFlag != null && !FeatureFlagClient.isEnabled(requiredFlag)) {
            throw new SkipException("Skipped: feature flag '" + requiredFlag + "' is OFF");
        }
        callBack.runTestMethod(testResult); // proceed normally
    }
}
```
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RequiresFeatureFlag { String value(); }
```
```java
@Listeners(FeatureFlagHook.class)
public class NewCheckoutUiTest {
    @Test
    @RequiresFeatureFlag("new-checkout-ui")
    public void verifyNewLayout() { /* ... */ }
}
```

### 4.3.10 `IReporter` — Minimal Custom Summary Reporter

**Use case:** you want a plain-text or Markdown summary of the whole run, generated exactly once after everything finishes — the entry point most custom HTML/Extent/Allure-style reports plug into.

```java
package com.qa.listeners;

import org.testng.ISuite;
import org.testng.ISuiteResult;
import org.testng.IReporter;
import org.testng.xml.XmlSuite;

import java.io.FileWriter;
import java.io.IOException;
import java.util.List;

public class SlackStyleSummaryReporter implements IReporter {

    @Override
    public void generateReport(List<XmlSuite> xmlSuites, List<ISuite> suites, String outputDirectory) {
        StringBuilder sb = new StringBuilder("# Execution Summary\n\n");
        for (ISuite suite : suites) {
            for (ISuiteResult result : suite.getResults().values()) {
                var ctx = result.getTestContext();
                sb.append(String.format("**%s** — Pass: %d | Fail: %d | Skip: %d%n",
                        ctx.getName(), ctx.getPassedTests().size(),
                        ctx.getFailedTests().size(), ctx.getSkippedTests().size()));
            }
        }
        try (FileWriter fw = new FileWriter(outputDirectory + "/summary.md")) {
            fw.write(sb.toString());
        } catch (IOException e) {
            System.err.println("Failed writing summary report: " + e.getMessage());
        }
    }
}
```
Fires exactly once, after **all** `<test>` tags in **all** suites finish — the correct place to aggregate a cross-suite dashboard rather than per-test logging.

### 4.3.11 `IExecutionListener` — Global Start/Stop for an External Process

**Use case:** start an Appium server once before the entire run (across every suite file), and stop it once at the very end — broader scope than even `ISuiteListener`.

```java
package com.qa.listeners;

import org.testng.IExecutionListener;

public class AppiumServerLifecycleListener implements IExecutionListener {

    private static Process appiumProcess;

    @Override
    public void onExecutionStart() {
        try {
            appiumProcess = new ProcessBuilder("appium", "--port", "4723").start();
            Thread.sleep(3000); // crude wait for server boot; prefer polling a health endpoint
            System.out.println("Appium server started for the entire run");
        } catch (Exception e) {
            throw new RuntimeException("Could not start Appium server", e);
        }
    }

    @Override
    public void onExecutionFinish() {
        if (appiumProcess != null) {
            appiumProcess.destroy();
            System.out.println("Appium server stopped after the entire run");
        }
    }
}
```

### 4.3.12 `IAlterSuiteListener` — Inject Classes Into the Suite at Runtime

**Use case:** scan a directory of compiled smoke-test classes at runtime and add them to the suite, without hand-maintaining a `<classes>` list.

```java
package com.qa.listeners;

import org.testng.IAlterSuiteListener;
import org.testng.xml.XmlClass;
import org.testng.xml.XmlSuite;
import org.testng.xml.XmlTest;

import java.util.List;

public class DynamicSmokeInjector implements IAlterSuiteListener {

    @Override
    public void alter(List<XmlSuite> suites) {
        for (XmlSuite suite : suites) {
            XmlTest smokeTest = new XmlTest(suite);
            smokeTest.setName("InjectedSmokeRun");
            smokeTest.setXmlClasses(List.of(
                    new XmlClass("com.qa.smoke.LoginSmokeTest"),
                    new XmlClass("com.qa.smoke.CheckoutSmokeTest")
            ));
        }
    }
}
```
Runs **before** the suite XML is finalized for execution — the earliest point at which the object graph can still be mutated, which is why it can only be wired via `<listeners>` in a *separate*, already-parsed suite XML or via `-listener` on the CLI (never `@Listeners`, same restriction as `IAnnotationTransformer`).

### 4.3.13 Wiring All of Them Together

**Beginner explanation:** here's what a "real" framework's suite XML tends to look like once several listeners are combined — some are suite-wide (registered in `<listeners>`), and some are scoped to individual classes (registered via `@Listeners` on the class itself), matching the pattern each listener's use case calls for.

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="FullFrameworkSuite" parallel="classes" thread-count="4">
    <listeners>
        <listener class-name="com.qa.listeners.GlobalRetryTransformer"/>
        <listener class-name="com.qa.listeners.GridSuiteListener"/>
        <listener class-name="com.qa.listeners.ExecutionTimingListener"/>
        <listener class-name="com.qa.listeners.ConfigFailureAlertListener"/>
        <listener class-name="com.qa.listeners.SmokeFirstInterceptor"/>
        <listener class-name="com.qa.listeners.SlackStyleSummaryReporter"/>
    </listeners>
    <test name="Regression">
        <classes>
            <class name="com.qa.tests.PaymentGatewayTest"/>
            <class name="com.qa.tests.NewCheckoutUiTest"/>
        </classes>
    </test>
</suite>
```
`ScreenshotListener`, `RetryCountListener`, `DevModeDataLimiter`, and `FeatureFlagHook` are applied via `@Listeners` on the individual base/test classes that need them (Section 4.1) rather than suite-wide, since they're scoped to Selenium/data-provider/feature-flag concerns specific to those classes.

---

# 5. Dependency Injection — Native Injection Type Matrix

**Beginner explanation:** "Dependency injection" here doesn't mean Guice or Spring — it means TestNG automatically passing certain useful objects as method parameters, purely based on the parameter's *type*, with no `@Inject` annotation needed. For example, if a `@BeforeMethod` declares a parameter of type `Method`, TestNG will hand it the currently-executing test method automatically. The table below tells you exactly which annotation types support which injected parameter types — a quick "can I ask TestNG for this?" lookup. `@NoInjection` opts a parameter out if you don't want this automatic behavior.

| Annotation | `ITestContext` | `XmlTest` | `Method` | `Object[]` | `ITestResult` | `ConstructorOrMethod` |
|---|---|---|---|---|---|---|
| `@BeforeSuite` | No | No | No | No | No | No |
| `@BeforeTest` | Yes | Yes | No | No | No | No |
| `@BeforeGroups` | Yes | Yes | No | No | No | No |
| `@BeforeClass` | Yes | Yes | No | No | No | No |
| `@BeforeMethod` | Yes | Yes | Yes | Yes | Yes | No |
| `@Test` | Yes | No | No | No | No | No |
| `@DataProvider` | Yes | No | Yes | No | No | Yes |
| `@AfterMethod` | Yes | Yes | Yes | Yes | Yes | No |
| `@AfterClass` | Yes | Yes | No | No | No | No |
| `@AfterGroups` | Yes | Yes | No | No | No | No |
| `@AfterTest` | Yes | Yes | No | No | No | No |
| `@AfterSuite` | No | No | No | No | No | No |
| `@Factory` | Yes | Yes | No | No | No | No |

```java
@AfterMethod
public void logResult(ITestResult result, Method testMethod) {
    System.out.println(testMethod.getName() + " -> " + result.getStatus());
}
```

---

# 6. Programmatic API — Running TestNG from Java

**Beginner explanation:** Everything so far assumed a `testng.xml` file. But TestNG can also be configured and launched **entirely in Java**, with no XML file at all — useful for tools that need to build a test run dynamically (e.g., an internal test-selection UI, or a plugin that decides what to run based on a code diff). The class hierarchy mirrors the XML structure you already know: `XmlSuite` → `XmlTest` → `XmlClass`.

```java
import org.testng.TestNG;
import org.testng.xml.XmlClass;
import org.testng.xml.XmlSuite;
import org.testng.xml.XmlTest;

public class ProgrammaticRunner {
    public static void main(String[] args) {
        XmlSuite suite = new XmlSuite();
        suite.setName("ProgrammaticSuite");
        suite.setParallel(XmlSuite.ParallelMode.CLASSES);
        suite.setThreadCount(3);

        XmlTest test = new XmlTest(suite);        // parent-binds test to suite
        test.setName("DynamicRun");
        test.setXmlClasses(java.util.List.of(new XmlClass("com.qa.tests.LoginTest")));

        TestNG tng = new TestNG();
        tng.setXmlSuites(java.util.List.of(suite));
        tng.setOutputDirectory("test-output/programmatic");
        tng.run();
    }
}
```
**Class hierarchy:** `XmlSuite` → `XmlTest` (must be constructed with the suite as parent, or via `new XmlTest(suite)`) → `XmlClass`. Forgetting to attach `XmlTest` to `XmlSuite`, or forgetting `test.setXmlClasses(...)`, produces **zero tests run with no exception** — always the first thing to check when a programmatic runner silently does nothing.

Other useful `TestNG` object setters: `setTestClasses(Class<?>[])`, `setGroups(String)`, `setExcludedGroups(String)`, `addListener(ITestNGListener)`, `setVerbose(int)`, `setSuiteThreadPoolSize(int)`.

---

# 7. Reporter API, Exit Codes & Retry Mechanics

## 7.1 `IReporter`

**Beginner explanation:** This is the interface behind §4.3.10's example — worth seeing on its own since it's the integration point most custom reporting tools (Extent Reports, Allure, homegrown dashboards) build against.

```java
public interface IReporter {
    void generateReport(java.util.List<org.testng.xml.XmlSuite> xmlSuites,
                         java.util.List<org.testng.ISuite> suites,
                         String outputDirectory);
}
```
Fires once, after the whole run. `ISuite.getResults()` → `Map<String, ISuiteResult>` → `ITestContext` exposes pass/fail/skip method sets — the integration point for Extent Reports/Allure/custom dashboards.

## 7.2 `Reporter` (log utility class)

**Beginner explanation:** Don't confuse this with `IReporter` above — `Reporter` (no "I" prefix) is a small static utility you call *from inside* a test method to add a custom log line that shows up in the generated HTML report, next to that specific test's result.

```java
Reporter.log("Custom step-level log line", true); // true = also echo to console
```
Appends to the current test's HTML report output — commonly used inside `@Test`/`@Before*` methods for step-level diagnostics that show up per-method in `test-output/index.html`.

## 7.3 Retry Analyzer (test-level, distinct from `IRetryDataProvider`)

**Beginner explanation:** You've already seen `IRetryAnalyzer` in action in §4.3.1 and §4.3.2 — here's the interface in isolation, plus a very useful side effect: TestNG automatically writes out a "failures-only" XML file after every run, which you can replay to rerun just what broke.

```java
public class MyRetry implements IRetryAnalyzer {
    private int retryCount = 0;
    private static final int maxRetryCount = 3;
    @Override
    public boolean retry(ITestResult result) {
        return retryCount++ < maxRetryCount;
    }
}
```
```java
@Test(retryAnalyzer = MyRetry.class)
public void test2() { Assert.fail(); }
```
Every failed suite also auto-generates `test-output/testng-failed.xml`, containing just the failed methods **plus their required dependencies** — rerun with `java org.testng.TestNG test-output/testng-failed.xml` to reproduce failures without a full-suite run.

## 7.4 TestNG Exit Codes

**Beginner explanation:** When TestNG runs from the command line (or in CI), it returns a numeric exit code that your build tool/CI system checks to decide "did this build pass or fail?" `0` is the only code that means a completely clean run.

| Code | Meaning |
|---|---|
| `0` | All tests passed |
| `1` | Test failure(s) occurred |
| `2` | Test failure(s) **and** skip(s) occurred |
| `-1` (or similar negative) | No tests were run at all, or a command-line/parse error occurred |

(Exact non-zero skip/failure bit-flag combinations are documented in `org.testng.TestNG` Javadoc — treat `0` as the only "clean" CI signal.)

---

# 8. Test Results, Assertions & Alternate Run Modes

## 8.1 Assertions — Hard vs Soft

**Beginner explanation:** An assertion is the actual "check" inside your test — comparing an actual value against an expected one. TestNG gives you two flavors, and mixing them up is a classic beginner mistake: hard assertions stop the method the instant something's wrong, soft assertions keep going and report everything at the end.

**Hard assertions** (`org.testng.Assert`, static methods): throw `AssertionError` **immediately** on first failure — the rest of the method body does not execute.

```java
import org.testng.Assert;

@Test
public void verifyProfile_hardAssert() {
    Assert.assertEquals(profile.getName(), "John Doe", "Name mismatch");
    Assert.assertEquals(profile.getEmail(), "john.doe@example.com", "Email mismatch");
    // If the first assertEquals fails, this second line NEVER runs
}
```

**Soft assertions** (`org.testng.asserts.SoftAssert`, instance-based): collect every failure internally instead of throwing; call `assertAll()` once at the end to report everything collected as one aggregated `AssertionError`.

```java
import org.testng.asserts.SoftAssert;

@Test
public void verifyProfile_softAssert() {
    SoftAssert softAssert = new SoftAssert();  // fresh instance every method — never a shared/static field
    softAssert.assertEquals(profile.getName(), "John Doe", "Name mismatch");
    softAssert.assertEquals(profile.getEmail(), "john.doe@example.com", "Email mismatch");
    softAssert.assertEquals(profile.getRole(), "Admin", "Role mismatch");
    softAssert.assertAll(); // reports ALL captured failures together, or passes silently if none
}
```
**Critical rule:** `SoftAssert` is not thread-safe and not reusable across test methods — always a fresh instance per method, never a shared/static field. Forgetting `assertAll()` at the end silently discards every collected failure — the method reports as PASSED even with real assertion mismatches inside it.

## 8.2 XML Reports (`testng-results.xml`)

**Beginner explanation:** Every time you run TestNG, it writes a machine-readable file alongside the human-readable HTML report. This is what most tooling (CI dashboards, Allure/ReportPortal adapters) actually parses, rather than scraping HTML.

Every run writes `test-output/testng-results.xml` — a machine-readable summary of every suite/test/class/method result, including parameters used, timestamps, and exceptions. This is the file most CI dashboards and custom parsers (Allure, ReportPortal adapters) ingest directly rather than scraping the HTML report.

## 8.3 JUnit Reports

**Beginner explanation:** Many CI systems (Jenkins, GitLab CI) were originally built expecting JUnit's report format specifically. TestNG accommodates this by also generating that format alongside its own, so you don't need to convert anything manually.

TestNG can additionally emit JUnit-XML-format results (`test-output/junitreports/`) for CI systems (Jenkins, GitLab CI) that expect the classic `TEST-*.xml` JUnit format rather than TestNG's native `testng-results.xml`. This is enabled by default alongside the native reports — most CI plugins for "JUnit test results" point directly at this directory.

## 8.4 JUnit Interop — Running JUnit Tests Under TestNG

**Beginner explanation:** If a codebase is mid-migration from JUnit to TestNG (a very common real-world scenario), TestNG can run the *old* JUnit tests too, without rewriting them first.

```xml
<test name="LegacyJUnitSuite" junit="true">
    <classes>
        <class name="com.qa.legacy.OldJUnit4Test"/>
    </classes>
</test>
```
- **JUnit 3** classpath detected: methods named `test*` run automatically; `setUp()`/`tearDown()` are treated as before/after each method; a `suite()` method's returned tests are all invoked.
- **JUnit 4** classpath detected: TestNG delegates to `org.junit.runner.JUnitCore` internally.
- `-mixed true` (CLI) auto-detects per-class whether to run it as JUnit or TestNG — useful mid-migration, when a codebase has both test styles side by side.

## 8.5 YAML Suite Files — Alternative to `testng.xml`

**Beginner explanation:** If your XML suite files get large, TestNG lets you write the same configuration in YAML instead — same capabilities, just a more compact syntax that's often easier to review in a pull request.

TestNG accepts YAML as a drop-in alternative to XML suite files — same capabilities, more compact syntax. Requires an explicit `snakeyaml` dependency (not bundled by default).

```xml
<!-- Maven -->
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>2.2</version>
</dependency>
```

Equivalent configs:
```xml
<suite name="SingleSuite" verbose="2" thread-count="4">
  <parameter name="n" value="42"/>
  <test name="Regression2">
    <groups><run><exclude name="broken"/></run></groups>
    <classes><class name="com.qa.tests.ResultEndMillisTest"/></classes>
  </test>
</suite>
```
```yaml
name: SingleSuite
verbose: 2
threadCount: 4
parameters: { n: 42 }
tests:
  - name: Regression2
    excludedGroups: [ broken ]
    classes:
      - com.qa.tests.ResultEndMillisTest
```
Run it exactly like an XML suite: `java org.testng.TestNG testng.yaml`. Useful when suite files grow large and YAML's terser syntax meaningfully improves readability/diff-review in code review.

## 8.6 Dry Run Mode

**Beginner explanation:** Before committing to a long real run — especially in CI, where minutes are expensive — you can ask TestNG to just *list* what it would run, without actually running any of it. This is the fastest way to sanity-check a group/include/exclude filter.

```
java -Dtestng.mode.dryrun=true org.testng.TestNG testng.xml
```
Lists every `@Test` method that **would** be invoked, in the order it would run, **without actually executing any of them** — configuration methods (`@Before*`/`@After*`) are not invoked either. Invaluable for validating a large suite's group/include/exclude filtering before committing to a long real run, especially in CI pipelines where you want to sanity-check "did my group filter actually select what I think it selected?" before burning minutes of real execution time.

## 8.7 JVM Arguments in TestNG

**Beginner explanation:** A quick-reference list of the `-D` flags introduced throughout this guide, gathered in one place.

| JVM Argument | Effect |
|---|---|
| `-Dtestng.mode.dryrun=true` | Enables dry-run mode (§8.6). |
| `-Dtestng.test.classpath=...` | Restricts class scanning to specific directories (useful with `<packages>`). |
| `-D<param-name>=<value>` | Supplies/overrides any `@Parameters`-resolved value (§1 of the companion Annotations file). |
| `-Dtestng.preferential.listeners.package=...` | Excludes listener packages (e.g. IDE-injected ones) from `-listenercomparator` ordering. |

## 8.8 `IConfigurable` — Overriding/Skipping Configuration Method Invocation

**Use case:** conditionally skip a specific `@BeforeMethod`/`@AfterMethod` at runtime based on a custom attribute on the target `@Test`, without deleting or `enabled=false`-ing the configuration method for everyone.

**Beginner explanation:** Think of `IConfigurable` as the setup/teardown counterpart to `IHookable` (§4.3.9). `IHookable` wraps the `@Test` method itself; `IConfigurable` wraps the `@Before*`/`@After*` methods around it. Both work the same way — you get a "callback" object, and your code decides whether to actually invoke it.

```java
package com.qa.listeners;

import org.testng.IConfigurable;
import org.testng.IConfigureCallBack;
import org.testng.ITestResult;
import org.testng.SkipException;
import org.testng.annotations.Test;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.Optional;

public abstract class ConfigurableBaseTest implements IConfigurable {

    protected static final String OMIT_CONFIG = "omit-config";

    @Override
    public void run(IConfigureCallBack callBack, ITestResult testResult) {
        Optional<Method> targetTestMethod = Arrays.stream(testResult.getParameters())
                .filter(p -> p instanceof Method)
                .map(p -> (Method) p)
                .findFirst();

        boolean shouldSkip = targetTestMethod
                .map(m -> m.getAnnotation(Test.class))
                .map(t -> Arrays.stream(t.attributes())
                        .anyMatch(a -> OMIT_CONFIG.equalsIgnoreCase(a.name())))
                .orElse(false);

        if (shouldSkip) {
            throw new SkipException("Skipping config method " + testResult.getMethod().getQualifiedName());
        }
        callBack.runConfigurationMethod(testResult); // proceed with the real @Before/@After logic
    }
}
```

```java
package com.qa.tests;

import com.qa.listeners.ConfigurableBaseTest;
import org.testng.annotations.BeforeMethod;
import org.testng.annotations.CustomAttribute;
import org.testng.annotations.Test;

import java.lang.reflect.Method;

public class LoginTest extends ConfigurableBaseTest {

    @BeforeMethod
    public void setupSession(Method method) {
        System.out.println("Setting up session for: " + method.getName());
    }

    @Test
    public void normalLogin() { /* setupSession runs as usual */ }

    @Test(attributes = { @CustomAttribute(name = OMIT_CONFIG, values = "true") })
    public void loginWithoutSessionSetup() { /* setupSession is SKIPPED for this one test */ }
}
```
`IConfigurable` is the configuration-method counterpart to `IHookable` (§4.3.9) — `IHookable` wraps `@Test` invocation itself, `IConfigurable` wraps `@Before*`/`@After*` invocation. Both are wired the same way: implement the interface directly on the test class or a shared base class — **not** via `@Listeners`.

## 8.9 Running Tests From a Test Jar

**Beginner explanation:** In some environments — like a dedicated performance/soak-test machine — you don't ship your whole source repo, just a built jar containing compiled test classes. `-testjar` is how TestNG runs directly against that jar.

```
java -jar uber-testjar-with-deps.jar -testjar tests.jar
```
`-testjar` tells TestNG to look for test classes **inside the jar**, not the classpath; if a `testng.xml` exists at the jar root it's used automatically, otherwise every discovered test class runs. To target a specific suite file inside the jar:
```
java -jar uber-testjar-with-deps.jar -testjar tests.jar -xmlpathinjar suites/regression.xml
```
Common in environments that ship a self-contained "test artifact" to a separate execution host (e.g., a performance/soak-test box that only receives the built jar, not the source repo).

---

# 9. Interview Prep & Cheat Sheet

**Beginner explanation:** These are framed exactly as they might come up in a real Senior SDET/Architect interview — each one targets a specific, easy-to-miss detail from the sections above. Even if you're not interviewing, working through the "why" of each answer is a great way to test whether the earlier sections actually stuck.

**Q1. Your `testng.xml` includes `<packages><package name="com.qa.tests"/></packages>` but classes in `com.qa.tests.dashboard` never run. Why?**
A: Package scanning in a `<packages>` block is **not recursive** by default — it scans exactly the declared package level. List `com.qa.tests.dashboard` explicitly (or as its own `<package>` entry) if sub-package classes must be included.

**Q2. When would you reach for `IMethodInterceptor` instead of just reordering methods in the XML file?**
A: `IMethodInterceptor` only affects the "no explicit ordering" bucket — methods with no `dependsOnMethods`/`dependsOnGroups` relationships, whose order is otherwise unspecified/random. It's the right tool when ordering needs to be computed dynamically (e.g., "always run group=fast first") rather than hand-maintained in static XML.

**Q3. `-groups` is passed on the CLI, but you also supply a `testng.xml` that has its own `<groups>` filter. Which wins?**
A: `-groups`/`-excludegroups` are the sole CLI flags that **override** XML group filtering even when a suite XML is present; every other "what to run" CLI flag (`-methods`, `-testclass`, etc.) is ignored once a `testng.xml` is supplied.

**Q4. What's the practical difference between `IRetryAnalyzer` and `IRetryDataProvider`?**
A: `IRetryAnalyzer` retries a **failed `@Test` invocation** itself. `IRetryDataProvider` retries the **`@DataProvider` method call** if *it* throws before ever supplying data to the test — they operate at different stages of the pipeline and are configured independently (`retryAnalyzer` on `@Test`, `retryUsing` on `@DataProvider`).

**Q5. A programmatic `XmlSuite`/`XmlTest` runner reports "Total tests run: 0" with no exception. Fastest triage?**
A: Check `test.getXmlClasses().size()` and confirm `XmlTest` was actually constructed against the suite (`new XmlTest(suite)`, not a bare `new XmlTest()`) — TestNG does not throw on an empty scope, it just runs nothing.

**Q6. A `SoftAssert` is declared as a `static` field on a base test class to "save an allocation per method." What breaks under `parallel="methods"`?**
A: Multiple threads share the same `SoftAssert` instance's internal failure map concurrently, so one method's captured failures can leak into another method's `assertAll()` report (or get lost entirely to a race). `SoftAssert` must be a fresh, method-local instance every time — never shared, never static.

**Q7. What's the actual difference between `IHookable` and `IConfigurable`?**
A: `IHookable.run()` wraps the invocation of the `@Test` method itself — the classic use case is injecting a security context or conditionally replacing the test body. `IConfigurable.run()` wraps invocation of a `@Before*`/`@After*` configuration method instead — used to conditionally skip setup/teardown logic per test without touching `enabled` globally. Both intercept via a callback (`IHookCallBack`/`IConfigureCallBack`) that you must explicitly invoke to let the real method run.

**Q8. CI reports "0 methods executed, filters matched nothing" but the team isn't sure whether the group filter or the class list is the problem. Fastest non-destructive way to check?**
A: Run with `-Dtestng.mode.dryrun=true` — TestNG lists every `@Test` method it *would* invoke, honoring all group/include/exclude filtering, without executing anything. If the dry-run list is empty, the filter is the problem; if it lists the expected methods, the issue is elsewhere (e.g., a broken `@BeforeSuite`).

**Cheat Sheet — Which File Configures What**

| Need | Where |
|---|---|
| Group include/exclude for a run | `testng.xml` `<groups><run>` or `-groups`/`-excludegroups` CLI |
| Parallel execution strategy | `<suite parallel="..." thread-count="...">` or `-parallel`/`-threadcount` |
| Register a custom listener | `<listeners>`, `@Listeners`, `-listener`, or ServiceLoader jar |
| Register an `IAnnotationTransformer` | `<listeners>`, `-listener`, or `addListener()` — **never** `@Listeners` |
| Pass simple config values | `<parameter>` + `@Parameters`, or `-D` system properties |
| Pass complex/object data | `@DataProvider` |
| Rerun only failures | `test-output/testng-failed.xml` |

**Common Mistakes Checklist**
- [ ] Assuming `<packages>` scans sub-packages recursively.
- [ ] Wiring `IAnnotationTransformer` via `@Listeners` and wondering why it's silently ignored.
- [ ] Forgetting CLI "what to run" flags are ignored once a `testng.xml` is passed (except `-groups`/`-excludegroups`).
- [ ] Setting `thread-count` without also setting `parallel` at the `<suite>` or `<test>` level.
- [ ] Confusing `IRetryAnalyzer` (test retry) with `IRetryDataProvider` (data-provider-call retry).
- [ ] Building `XmlTest` without binding it to `XmlSuite`, then debugging a "0 tests run" mystery.
- [ ] Pinning TestNG 7.6.0+ on a JDK 8 build — requires JDK 11 or higher.
- [ ] Declaring `SoftAssert` as a shared/static field instead of a fresh per-method instance.
- [ ] Forgetting to call `assertAll()` on a `SoftAssert` — collected failures are silently discarded and the method reports PASSED.
- [ ] Adding a YAML suite file without the `snakeyaml` dependency, then wondering why TestNG can't parse it.
- [ ] Implementing `IConfigurable`/`IHookable` but forgetting to call the callback (`runConfigurationMethod`/`callback`) — the wrapped method never actually runs.
