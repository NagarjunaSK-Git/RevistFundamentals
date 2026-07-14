# TestNG Annotations & Attributes — The Complete Beginner-Friendly Guide

**Source of truth:** Official TestNG Documentation — `https://testng.org/#_testng_documentation` (doc revision 7.9.0; API surface for annotations is stable across 7.x).
**Version note (important, often trips people up):** TestNG ≤ 7.5.x requires JDK 8. **TestNG 7.6.0 and above requires JDK 11+.** If your project is stuck on JDK 8, you must stay on 7.5.x or lower. This is a very common interview trap and a common cause of "why won't my CI pipeline build?" errors.

> This guide keeps **every** annotation, attribute, table, code example, interview question, and mistake from the original reference — nothing has been removed. What's different is that every topic now starts with a plain-English explanation before the technical details, so a beginner can follow along without prior TestNG experience.

## Table of Contents
0. What Is TestNG, and Why Does It Use Annotations?
1. Module 1 — Configuration Annotations (`@Before*` / `@After*`) and Their Shared Attributes
2. Module 2 — The `@Test` Annotation: Full Attribute Reference
3. Module 3 — Data-Driven & Object-Creation Annotations (`@DataProvider`, `@Factory`, `@Parameters`, `@Optional`)
4. Module 4 — Class-Level, Injection & Wiring Annotations (`@Listeners`, `@Ignore`, `@Guice`, `@NoInjection`, `@CustomAttribute`)
5. Interview Prep — API-Level Q&A
6. Cheat Sheet & Common Mistakes

---

# 0. What Is TestNG, and Why Does It Use Annotations?

**TestNG** ("Testing, the Next Generation") is a testing framework for Java. It's used to write and run automated tests — for example, checking that a login page works, that a calculator function returns the right answer, or that an API responds correctly. It was inspired by JUnit but was built to add features JUnit didn't originally have, like flexible test grouping, parallel execution, data-driven testing, and dependency-based test ordering.

If you've never used a testing framework before, think of TestNG as a **project manager for your tests**. You write the actual test logic (the "what to check"), and TestNG handles the "when to run it, how many times, in what order, and with what setup/cleanup around it."

**Why annotations?** In plain Java, if you want a method to be treated specially (like "run this before every test"), you'd normally have to write extra boilerplate code to register it. TestNG instead lets you just put a small label — an **annotation** — directly above a method, like `@Test` or `@BeforeMethod`. TestNG scans your code, finds these labels, and automatically wires everything together. You focus on writing test logic; TestNG figures out the orchestration.

```java
@Test
public void verifyLoginPageLoads() {
    // your test logic here
}
```

That `@Test` line is all it takes to tell TestNG "this method is a test — please run it." The rest of this guide walks through every annotation TestNG offers, and every attribute (configuration option) you can add inside the parentheses, e.g. `@Test(priority = 1)`.

---

# Module 1: Configuration Annotations

**Beginner explanation:** Most tests need some setup before they run (e.g., opening a browser, connecting to a database) and some cleanup after (e.g., closing the browser, disconnecting). Instead of repeating that setup/cleanup code inside every single test method, TestNG lets you pull it out into separate methods and label them with "configuration annotations." TestNG then calls these methods automatically at the right moment — you never call them yourself.

## 1.1 The Nine Configuration Annotations

Think of these as hooks that fire at different "zoom levels" — some run once for your entire test suite, some run once per class, and some run before/after every single test method.

| Annotation | Fires | Scope |
|---|---|---|
| `@BeforeSuite` | Once, before **any** test in the `<suite>` runs | Whole suite |
| `@AfterSuite` | Once, after **all** tests in the `<suite>` have run | Whole suite |
| `@BeforeTest` | Once, before any method inside classes under the current `<test>` tag runs | `<test>` tag |
| `@AfterTest` | Once, after all methods inside classes under the current `<test>` tag have run | `<test>` tag |
| `@BeforeGroups` | Shortly before the **first** method of the listed group(s) is invoked | Group |
| `@AfterGroups` | Shortly after the **last** method of the listed group(s) is invoked | Group |
| `@BeforeClass` | Once, before the first `@Test` method in the current class | Class |
| `@AfterClass` | Once, after all `@Test` methods in the current class have run | Class |
| `@BeforeMethod` | Before **every** `@Test` method | Method |
| `@AfterMethod` | After **every** `@Test` method | Method |

**A simple mental model for beginners:** imagine nested boxes — Suite → Test tag → Class → Method. The "Before" annotations fire as you walk **into** each box (outermost first), and the "After" annotations fire as you walk **out** of each box (innermost first). `@BeforeGroups`/`@AfterGroups` are the exception — they're tied to *groups* (tags you assign to tests) rather than to the class/suite structure.

**Inheritance rule (frequently missed):** these annotations are honored on superclasses. `@Before*` methods run **top-down** (highest superclass first); `@After*` methods run in **reverse** (bottom-up). This lets you centralize setup in a `BaseTest` without any explicit `super.setUp()` call — TestNG walks the hierarchy for you.

```java
public class BaseTest {
    @BeforeClass
    public void baseSetup() { System.out.println("1: BaseTest setup"); }
}

public class LoginTest extends BaseTest {
    @BeforeClass
    public void loginSetup() { System.out.println("2: LoginTest setup"); }

    @Test
    public void verifyLogin() { System.out.println("3: test runs"); }
}
// Output order: 1, 2, 3 — guaranteed, not incidental.
```

## 1.2 Shared Attributes on Configuration Annotations

**Beginner explanation:** Just like `@Test` can take extra options in parentheses, so can the `@Before*`/`@After*` annotations. These options control things like "should this setup step run even if it's not in the group being tested?" or "should this cleanup step run even if something before it failed?"

These attributes apply to every `@Before*`/`@After*` annotation (with the noted exceptions):

| Attribute | Type | Applies to | Meaning |
|---|---|---|---|
| `alwaysRun` | `boolean` | All `@Before*` (except `@BeforeGroups`) and `@After*` | On a **before** method: if `true`, runs regardless of what groups it belongs to. On an **after** method: if `true`, runs even if an earlier method in the chain failed or was skipped. |
| `dependsOnGroups` | `String[]` | All | The groups this configuration method depends on. |
| `dependsOnMethods` | `String[]` | All | The specific methods this configuration method depends on. |
| `enabled` | `boolean` | All | If `false`, this method is skipped entirely (default `true`). |
| `groups` | `String[]` | All | Groups this configuration method belongs to. |
| `inheritGroups` | `boolean` | All | If `true`, inherits the groups declared on the class-level `@Test`. |
| `onlyForGroups` | `String[]` | **Only** `@BeforeMethod` / `@AfterMethod` | Restricts this setup/teardown to run only when the target `@Test` method belongs to one of the listed groups. |

```java
@BeforeMethod(onlyForGroups = "smoke")
public void smokeOnlySetup() {
    System.out.println("Runs only before @Test methods tagged group=smoke");
}

@AfterClass(alwaysRun = true)
public void teardownDriver() {
    // Guarantees driver.quit() runs even if an earlier @Test/@BeforeMethod in the class failed
    if (driver != null) driver.quit();
}
```

**Critical distinction — `alwaysRun` means two different things depending on direction:**
- On `@BeforeX`: "ignore group filtering, run me regardless."
- On `@AfterX`: "ignore upstream failures/skips, run me for cleanup."

This asymmetry is a classic trick interview question. A simple way to remember it: on the "Before" side it's about **which tests I set up for**, and on the "After" side it's about **whether I still clean up even after a disaster**.

## 1.3 Java-Level Method Overriding (Not the Same as Listener-Based Overriding)

**Beginner explanation:** This section isn't a TestNG feature at all — it's how normal Java works, but it surprises a lot of people the first time they see it in a TestNG project. If a child class (subclass) has a method with the exact same name and parameters as one in its parent class (superclass), the child's version "wins" and the parent's version simply never runs on its own.

If a subclass declares a method with the **same signature** as a superclass `@Test`/`@Before*`/`@After*` method, standard Java overriding rules apply — only the subclass version executes; the superclass version is silently not invoked separately.

```java
public class SuperTestNgClass {
    @Test
    public void superTestNgMethod() { System.out.println("Super version"); }
    @Test
    public void anotherMethod() { System.out.println("Another super method"); }
}

public class SubTestNgClass extends SuperTestNgClass {
    @Test
    public void superTestNgMethod() { System.out.println("Overridden in sub class"); } // wins
    @Test
    public void subOnlyMethod() { System.out.println("Sub-only method"); }
}
// Running SubTestNgClass invokes: "Overridden in sub class", "Sub-only method", "Another super method"
// — never "Super version". Total tests run: 3, not 4.
```

This is plain Java polymorphism, not a TestNG feature — but it's a common source of "why did my superclass test disappear?" confusion, since TestNG reports show 3 tests run instead of the 4 you might expect by counting `@Test` annotations across both classes.

> For **programmatic, conditional** overriding/skipping of a test or configuration method at runtime (not compile-time inheritance), see `IHookable` and `IConfigurable` in the companion file `TestNG-XML-CLI-Core-API-Reference.md` §8.8.

## 1.4 Execution Nesting Order (single class, one `@Test`)

**Beginner explanation:** Here's the full "zoom in, zoom out" order for a single test method, all in one line, so you can see exactly where each hook fires relative to your actual test.

```
@BeforeSuite → @BeforeTest → @BeforeGroups → @BeforeClass → @BeforeMethod
    → @Test → @AfterMethod → @AfterGroups → @AfterClass → @AfterTest → @AfterSuite
```

---

# Module 2: The `@Test` Annotation — Full Attribute Reference

**Beginner explanation:** `@Test` is the most important annotation in TestNG — it's the one that actually marks a method as a test TestNG should run and report on. But `@Test` isn't just an on/off switch; it accepts many options that control *how* that test behaves: how many times to run it, whether it depends on another test, how long it's allowed to take, and more. This module goes through every one of those options.

`@Test` can be placed on a **method** or on a **class** (class-level `@Test` marks every public method as a test method; see Module 4).

## 2.1 Complete Attribute Table

| Attribute | Type | Default | Purpose |
|---|---|---|---|
| `alwaysRun` | `boolean` | `false` | Runs this test even if a method it `dependsOnMethods` failed. |
| `dataProvider` | `String` | — | Name of the `@DataProvider` method supplying arguments. |
| `dataProviderClass` | `Class<?>` | — | Class to look for the named data provider in (must be a **static** method if specified). Defaults to the current class/superclass. |
| `dependsOnGroups` | `String[]` | — | Groups this method depends on. |
| `dependsOnMethods` | `String[]` | — | Specific methods this method depends on. |
| `description` | `String` | — | Free-text description, shown in reports. |
| `enabled` | `boolean` | `true` | If `false`, method is skipped (does not appear as SKIP — it's excluded). |
| `expectedExceptions` | `Class<?>[]` | — | Exception types this method must throw to **pass**. Any other exception (or none) is a failure. |
| `groups` | `String[]` | — | Groups this method belongs to. |
| `invocationCount` | `int` | `1` | Number of times to invoke this method. |
| `invocationTimeOut` | `long` (ms) | — | Cumulative time budget across all `invocationCount` runs; ignored if `invocationCount` is unset. |
| `priority` | `int` | `0` | Lower number scheduled first among methods with no dependency relationship to each other. |
| `retryAnalyzer` | `Class<? extends IRetryAnalyzer>` | — | Class implementing retry-on-failure logic. |
| `singleThreaded` | `boolean` | `false` | **Class-level only.** Forces all methods on this class to run on the same thread even under `parallel="methods"`. (Formerly `sequential`, now deprecated.) |
| `successPercentage` | `int` | `100` | Percentage of `invocationCount` runs expected to pass for the method to be reported as a pass. |
| `threadPoolSize` | `int` | — | Runs this single method from N threads concurrently. **Ignored unless `invocationCount` is also set.** |
| `timeOut` | `long` (ms) | — | Max time for a single invocation; throws `TestTimeoutException` on breach (works in both parallel and non-parallel mode). |

**New to some of these terms?**
- **"Groups"** are just labels/tags you attach to tests (like `"smoke"`, `"regression"`) so you can run subsets of your suite selectively.
- **"Dependency"** means "don't run this test until that other one has finished" — useful for workflows like "log in" → "add item to cart" → "checkout."
- **"Invocation"** just means "one run of the method."

## 2.2 Worked Examples Per Attribute

**`expectedExceptions` — negative testing done right:**
```java
@Test(expectedExceptions = IllegalArgumentException.class)
public void divideByZero_shouldThrow() {
    Calculator calc = new Calculator();
    calc.divide(10, 0); // must throw IllegalArgumentException to PASS
}
```
*Beginner note:* this is how you test that your code correctly rejects bad input. The test **passes** only if the exception you named is thrown — if nothing is thrown, or a different exception is thrown, the test fails.

**`invocationCount` + `threadPoolSize` + `timeOut` — concurrent load-style repetition:**
```java
@Test(invocationCount = 10, threadPoolSize = 3, timeOut = 10000)
public void hitEndpointConcurrently() {
    // invoked 10 times total, spread across 3 threads,
    // each individual invocation must finish inside 10s
}
```

**`successPercentage` — flaky-tolerant assertions:**
```java
@Test(invocationCount = 20, successPercentage = 90)
public void flakyThirdPartyApiCall() {
    // Passes overall if at least 18 of 20 invocations succeed
}
```

**`priority` vs `dependsOnMethods` — ordering:**
```java
@Test(priority = 2)
public void step2() { }

@Test(priority = 1)
public void step1() { }
// step1 dispatched before step2 — but priority gives NO guarantee under
// parallel="methods" that step1 will *finish* first. Use dependsOnMethods
// for a hard completion-order guarantee.
```
*Beginner note:* `priority` only controls the **order TestNG starts** methods, not the order they **finish** — those can differ under parallel execution. If you truly need "B must not run until A is fully done," use `dependsOnMethods`, not `priority`.

**`retryAnalyzer`:**
```java
public class FlakyRetry implements IRetryAnalyzer {
    private int count = 0;
    private static final int MAX = 2;
    @Override
    public boolean retry(ITestResult result) {
        return count++ < MAX;
    }
}

@Test(retryAnalyzer = FlakyRetry.class)
public void unstableNetworkCall() {
    Assert.assertTrue(callService());
}
```
> For a production-grade pattern — auto-attaching this to every `@Test` via `IAnnotationTransformer`, plus surfacing retry counts in reports via `ITestListener` — see **§4.3 "Full Working Listener Implementations"** in the companion file `TestNG-XML-CLI-Core-API-Reference.md`.

**`dataProviderClass` — cross-class static provider:**
```java
public class SharedProviders {
    @DataProvider(name = "loginCreds")
    public static Object[][] loginCreds() {
        return new Object[][]{ {"admin", "Admin@123"}, {"user1", "User@123"} };
    }
}

public class LoginTest {
    @Test(dataProvider = "loginCreds", dataProviderClass = SharedProviders.class)
    public void login(String user, String pass) { /* ... */ }
}
```

## 2.3 Dependency Semantics: Hard vs Soft

**Beginner explanation:** When Test B "depends on" Test A, what happens if A fails? By default, B is politely **skipped** rather than run and failed — TestNG assumes if the prerequisite failed, running the dependent test wouldn't tell you anything useful. This is called a "hard" dependency. But sometimes you *do* want B to run anyway (e.g., cleanup steps) — that's a "soft" dependency, turned on with `alwaysRun = true`.

| Type | How | Behavior on upstream failure |
|---|---|---|
| **Hard dependency** | `dependsOnMethods`/`dependsOnGroups`, `alwaysRun` left `false` (default) | Downstream method is **SKIPPED**, not failed |
| **Soft dependency** | Same, plus `alwaysRun = true` | Downstream method **still runs** even if upstream failed |

```java
@Test
public void serverStartedOk() { }

@Test(dependsOnMethods = "serverStartedOk")
public void hardDependent() { }          // SKIPPED if serverStartedOk() fails

@Test(dependsOnMethods = "serverStartedOk", alwaysRun = true)
public void softDependent() { }          // runs regardless
```

**Partial groups — class-level `groups` combine with method-level `groups`:**
```java
@Test(groups = "checkin-test")
public class AllChecks {
    @Test(groups = "func-test")
    public void method1() { }   // belongs to BOTH "checkin-test" AND "func-test"

    public void method2() { }   // belongs ONLY to "checkin-test" (inherited from class level)
}
```
Class-level `groups` acts as a default that every method inherits; a method-level `@Test(groups=...)` **adds to**, rather than replaces, that default.

**Factory/DataProvider instance nuance:** when `b()` depends on `a()` and multiple instances exist (via `@Factory`), TestNG groups dependents **by class** by default — it waits for *every* instance's `a()` before running *any* instance's `b()`. To force per-instance ordering (e.g., sign-in/sign-out pairs per country), set `group-by-instances="true"` on `<suite>` or `<test>`.

---

# Module 3: Data-Driven & Object-Creation Annotations

**Beginner explanation:** So far every test example ran with fixed, hardcoded values. But often you want to run the **same test logic** with many different sets of input data — for example, testing login with 10 different username/password combinations. That's called "data-driven testing," and TestNG has two main tools for it: `@DataProvider` (feeds different *arguments* into the same test instance) and `@Factory` (creates multiple, fully separate *instances* of a test class). `@Parameters`/`@Optional` are a simpler, XML-driven way to pass in values like environment URLs or browser names.

## 3.1 `@DataProvider`

**Beginner explanation:** A `@DataProvider` is a method that returns a table of test data (rows and columns). You link it to a `@Test` method, and TestNG automatically calls that test once per row, passing each row's values in as method arguments.

| Attribute | Type | Default | Purpose |
|---|---|---|---|
| `name` | `String` | method name | Identifier referenced by `@Test(dataProvider = "...")`. |
| `parallel` | `boolean` | `false` | If `true`, the data-driven test invocations run in parallel (default pool size 10, override via `data-provider-thread-count` on `<suite>`). |
| `retryUsing` | `Class<? extends IRetryDataProvider>` | — | Retry the data provider call itself if it throws (since 7.x). |

```java
@DataProvider(name = "accountTiers")
public Object[][] accountTiers() {
    return new Object[][] {
        { "ACC-1001", "Gold" },
        { "ACC-1002", "Standard" }
    };
}

@Test(dataProvider = "accountTiers")
public void verifyTier(String accountId, String tier) {
    Assert.assertNotNull(accountId);
}
```

**Return-type flexibility (all valid):** `Object[][]`, `Iterator<Object[]>` (lazy), `Object[]` (one arg per invocation), `Iterator<Object>` (lazy single-arg), or any custom array type like `MyData[][]`.

**Method-name-aware provider** — inject `java.lang.reflect.Method` as the first parameter to branch data by which test is calling:
```java
@DataProvider(name = "dp")
public Object[][] createData(Method m) {
    System.out.println("Supplying data for: " + m.getName());
    return new Object[][]{ { "Cedric" } };
}
```

**Retry-on-provider-failure:**
```java
public class RetryDataProvider implements IRetryDataProvider {
    private final AtomicInteger counter = new AtomicInteger(1);
    @Override
    public boolean retry(IDataProviderMethod dp) {
        return counter.getAndIncrement() <= 2;
    }
}

@DataProvider(retryUsing = RetryDataProvider.class, name = "test-data")
public Object[][] testDataSupplier() { /* may throw once, retried up to 2x */ return new Object[][]{{1},{2}}; }
```

## 3.2 `@Factory`

**Beginner explanation:** Where `@DataProvider` reuses the *same* test object and just feeds it different arguments, `@Factory` creates completely separate, independent objects of your test class — each with its own constructor arguments and its own state. Use this when tests genuinely need to be isolated from each other (not just fed different numbers).

Marks a method (or constructor) that returns `Object[]` — instances TestNG treats as fully independent test classes, each with isolated constructor-scoped state.

| Attribute | Type | Purpose |
|---|---|---|
| `dataProvider` | `String` | Feed factory constructor args from a `@DataProvider`. |
| `dataProviderClass` | `Class<?>` | Class holding that provider if not co-located. |
| `indices` | `int[]` | Only pass the data-provider elements at these indices to the constructor. |

**Method-based factory:**
```java
public class WebTestFactory {
    @Factory
    public Object[] createInstances() {
        Object[] result = new Object[10];
        for (int i = 0; i < 10; i++) result[i] = new WebTest(i * 10);
        return result;
    }
}
```

**Constructor-based factory + `@DataProvider`:**
```java
public class AccountDashboardTest {
    private final int accountNumber;
    @Factory(dataProvider = "dp")
    public AccountDashboardTest(int accountNumber) { this.accountNumber = accountNumber; }

    @DataProvider(name = "dp")
    public static Object[][] dp() {
        return new Object[][]{ {41}, {42} };
    }
    @Test
    public void verify() { Assert.assertTrue(accountNumber > 0); }
}
// Produces two fully independent AccountDashboardTest instances.
```

**`indices` — selective factory instantiation:**
```java
@Factory(dataProvider = "dp", indices = {1, 3})
public ExampleTestCase(String text) { this.text = text; }

@DataProvider(name = "dp")
public static Object[] getData() { return new Object[]{"Java", "Kotlin", "Golang", "Rust"}; }
// Only "Kotlin" (index 1) and "Rust" (index 3) become instances — 2 total, not 4.
```

**`@DataProvider` vs `@Factory` — the interview-defining distinction:**

| Aspect | `@DataProvider` | `@Factory` |
|---|---|---|
| Varies | Method **arguments**, within one instance | Constructor **arguments**, across many instances |
| Object identity | Same instance across all data rows | New, isolated instance per row |
| Use when | Same logic, different input values | Genuinely isolated state/reporting per record |

## 3.3 `@Parameters` and `@Optional`

**Beginner explanation:** `@Parameters` is the simplest way to feed values into your tests from outside your Java code — typically from your `testng.xml` suite file, or from a system property. It's commonly used for things like which browser to test on, or which environment URL to hit. `@Optional` lets you set a fallback value in case the XML doesn't provide one, so your test doesn't crash if a parameter is missing.

`@Parameters` maps `testng.xml`-defined values (or system properties) onto Java method/constructor parameters, in declaration order.

```java
@Parameters({ "browser", "envUrl" })
@BeforeClass
public void setUp(String browser, String envUrl) { /* ... */ }
```

```xml
<suite name="Suite">
  <parameter name="browser" value="chrome"/>
  <parameter name="envUrl" value="https://staging.example.com"/>
  <test name="Regression">...</test>
</suite>
```

**Resolution scope precedence (lowest → highest):** `<suite>` → `<test>` → `<class>` → `<methods>`. A same-named parameter defined at `<methods>` wins over one at `<suite>` — useful for a global default overridden for one specific test.

**`@Optional` — default when the XML parameter is missing:**
```java
@Parameters("db")
@Test
public void testConnection(@Optional("mysql") String db) { /* uses "mysql" if <parameter name="db"> is absent */ }
```

`@Parameters` is valid on: any `@Test`/`@Before*`/`@After*`/`@Factory` method, and **at most one constructor** of the test class (used to initialize instance fields from XML).

**Parameters in reports:** every value TestNG resolves for a `@Parameters`-annotated method is echoed in the generated HTML report next to that test invocation — a built-in audit trail of exactly what input produced a given pass/fail, with zero extra logging code required.

---

# Module 4: Class-Level, Injection & Wiring Annotations

**Beginner explanation:** This last group of annotations covers things that are less about "what to test" and more about "how to wire your test project together" — plugging in custom reporting logic, disabling old tests without deleting them, using dependency-injection frameworks like Guice, attaching custom metadata, or applying `@Test` to a whole class at once.

## 4.1 `@Listeners`

**Beginner explanation:** A "listener" is a class that TestNG calls automatically when certain events happen — like a test starting, passing, or failing. This is how people build custom behavior such as "take a screenshot automatically whenever a test fails" or "generate a custom HTML report." `@Listeners` is how you plug those classes into a specific test class.

```java
@Listeners({ ScreenshotListener.class, CustomHtmlReporter.class })
public class BaseTest { }
```

| Attribute | Type | Purpose |
|---|---|---|
| `value` | `Class<? extends ITestNGListener>[]` | Listener implementations to wire in. |

**Hard constraint:** `@Listeners` accepts any `ITestNGListener` **except `IAnnotationTransformer`**. Annotation transformers must be registered via `testng.xml`'s `<listeners>`, the `-listener` CLI flag, or `TestNG.addListener()` in code — because annotation rewriting has to happen before TestNG even finishes reading `@Listeners` itself.

## 4.2 `@Ignore`

**Beginner explanation:** Sometimes you want to temporarily "turn off" a test or a whole batch of tests without deleting the code — maybe the feature is being rebuilt, or the test is known to be broken and you don't want it cluttering your reports. `@Ignore` does exactly that.

Disables all `@Test` methods in a class, or an entire package (via `package-info.java`), without deleting code.

```java
@Ignore
public class LegacyCheckoutTests {
    @Test public void oldFlow1() { }
    @Test public void oldFlow2() { }
}
```
```java
// package-info.java — disables every @Test in this package and sub-packages
@Ignore
package com.qa.legacy;
import org.testng.annotations.Ignore;
```
At the method level, `@Ignore` is functionally identical to `@Test(enabled = false)`. Class-level `@Ignore` overrides any method-level `@Test(enabled = true)` beneath it — `@Ignore` always wins.

## 4.3 `@Guice`

**Beginner explanation:** Guice is a popular Java dependency-injection framework (a way of automatically supplying objects your code needs, instead of manually constructing them yourself). If your test project already uses Guice to manage its objects/services, `@Guice` lets TestNG hook into that same system so your test classes can have dependencies auto-injected too. If you don't use Guice in your project, you can safely skip this section for now.

```java
@Guice(modules = GuiceExampleModule.class)
public class GuiceTest {
    @Inject ISingleton service;
    @Test
    public void singletonShouldWork() { service.doSomething(); }
}
```

| Attribute | Type | Purpose |
|---|---|---|
| `modules` | `Class<? extends Module>[]` | Guice modules to bind for this test class. |
| `moduleFactory` | `Class<? extends IModuleFactory>` | Dynamic module selection based on `ITestContext`, when static `modules` isn't flexible enough. |

Suite-wide bindings can be layered in via `<suite parent-module="com.example.ParentModule" guice-stage="PRODUCTION">` — `guice-stage` accepts `DEVELOPMENT` (default), `PRODUCTION`, `TOOL`.

## 4.4 `@NoInjection`

**Beginner explanation:** TestNG is sometimes "helpful" in a way that can surprise you: if one of your method parameters matches a known type (like `Method` or `ITestContext`), TestNG will automatically fill it in for you rather than pulling it from your data provider. `@NoInjection` tells TestNG "no, don't auto-fill this one — I want the actual value from my data provider instead."

TestNG auto-injects certain parameter types (see the injection table below). `@NoInjection` opts a parameter out so it's treated as ordinary `@DataProvider` data instead.

```java
@Test(dataProvider = "provider")
public void withoutInjection(@NoInjection Method m) {
    Assert.assertEquals(m.getName(), "f"); // m came from the data provider, not TestNG injection
}
```

## 4.5 `@CustomAttribute`

**Beginner explanation:** This lets you attach your own arbitrary label/value pairs to a test method — kind of like sticking a custom sticky note on it — which your own code can read later at runtime. It's an advanced feature, typically used to build custom filtering logic for data providers.

Attaches arbitrary name/value metadata to a `@Test` method, readable at runtime via `ITestNGMethod.getAttributes()` — commonly used to drive custom `IDataProviderInterceptor` filtering logic.

```java
@Test(dataProvider = "numbers",
      attributes = { @CustomAttribute(name = "filter", values = {"com.qa.filters.EvenOnly"}) })
public void passingTest(int i) { System.out.println("Value = " + i); }
```

## 4.6 Class-Level `@Test`

**Beginner explanation:** Instead of writing `@Test` above every single method, you can put `@Test` once at the top of the class. TestNG then treats every public method in that class as a test method automatically. You can still add `@Test(...)` on individual methods if you want to give a specific method extra options (like its own group or priority) — it won't lose the class-level default.

```java
@Test
public class SmokeSuite {
    public void test1() { }              // becomes a test method automatically

    @Test(groups = "g1")
    public void test2() { }              // still a test method, PLUS now in group "g1"
}
```
Putting `@Test` on the class makes **every public method** a test method by default. You can still annotate individual methods to layer on extra attributes (groups, priority, etc.) without losing the class-level default.

---

# 5. Interview Prep — API-Level Q&A

**Q1. A `@BeforeMethod` needs to run only before methods tagged `"smoke"`, but must also run unconditionally regardless of the currently included groups. Can you combine `onlyForGroups` and `alwaysRun` on the same method — what happens?**
A: `onlyForGroups` restricts the setup to fire only when the target `@Test`'s groups intersect the listed set; `alwaysRun` on a before-method separately overrides *group-inclusion filtering* for the setup method itself. Combining them is legal but the semantics stack: the setup still only fires for smoke-tagged tests (per `onlyForGroups`), and within that subset it additionally ignores whether "smoke" itself was excluded from the run's group filter.

**Q2. Why does `@NoInjection` exist, and what breaks if you forget it?**
A: TestNG auto-injects known types (`Method`, `ITestContext`, `Object[]`, etc.) into `@BeforeMethod`/`@DataProvider` parameters by type-matching. If your data provider legitimately wants to *supply* a `Method`-typed value as test data (not have TestNG inject the currently running method), TestNG's injection will silently override your data unless the parameter is annotated `@NoInjection`.

**Q3. `dataProviderClass` requires the target method to be `static` — why?**
A: Because TestNG resolves a cross-class data provider before it has necessarily instantiated (or has any reason to instantiate) that other class. A static method needs no instance to invoke; requiring `static` avoids TestNG having to guess a constructor strategy for an unrelated provider class.

**Q4. `threadPoolSize` is set on a `@Test` method but nothing changes — single-threaded execution observed. Likely cause?**
A: `threadPoolSize` is explicitly documented as ignored unless `invocationCount` is also specified — it defines *how many threads* the invocations are spread across, not a standalone concurrency switch.

**Q5. Two `@Factory`-produced instances of the same class both have a method `b()` depending on `a()`. Execution log shows `a(1) a(2) b(2) b(2)` instead of interleaved sign-in/sign-out order. Fix?**
A: This is default TestNG behavior — dependents are grouped by class, so all `a()` instances run before any `b()`. Add `group-by-instances="true"` on `<suite>` or `<test>` to force TestNG to respect per-instance ordering instead.

---

# 6. Cheat Sheet & Common Mistakes

**Cheat Sheet — Attribute → Annotation Matrix**

| Attribute | `@Test` | `@Before*/@After*` | `@DataProvider` | `@Factory` |
|---|---|---|---|---|
| `groups` | ✅ | ✅ | ❌ | ❌ |
| `dependsOnMethods` | ✅ | ✅ | ❌ | ❌ |
| `alwaysRun` | ✅ | ✅ | ❌ | ❌ |
| `enabled` | ✅ | ✅ | ❌ | ❌ |
| `invocationCount` | ✅ | ❌ | ❌ | ❌ |
| `dataProvider` | ✅ | ❌ | — | ✅ |
| `parallel` | ❌ | ❌ | ✅ | ❌ |
| `indices` | ❌ | ❌ | ❌ | ✅ |

**Common Mistakes Checklist**
- [ ] Assuming `priority` guarantees completion order under `parallel="methods"` — it only guarantees dispatch order.
- [ ] Forgetting `alwaysRun = true` on `@AfterMethod`/`@AfterClass` cleanup, leaving resources leaked when an earlier step fails.
- [ ] Setting `threadPoolSize` without `invocationCount` and expecting concurrency.
- [ ] Registering `IAnnotationTransformer` via `@Listeners` (silently ignored) instead of `testng.xml`/CLI/code.
- [ ] Using a non-`static` method for `dataProviderClass` cross-class lookups.
- [ ] Expecting `@DataProvider` to isolate state the way `@Factory` does — it reuses one instance across all rows.
- [ ] Missing `group-by-instances="true"` when factory-produced instances need per-instance dependency ordering.
