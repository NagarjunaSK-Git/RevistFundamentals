# Cucumber BDD with Java — Complete Architect-Grade Guide
### Basics → Advanced | JUnit 4 · JUnit 5 (Platform) · TestNG | Runners, Parallelism & Allure Reporting

> Validated against `cucumber.io/docs` (Gherkin Reference, Cucumber-JVM Installation, Parallel Execution guide,
> `cucumber-junit-platform-engine` README) and `allurereport.org/docs` (Cucumber-JVM integration, Configuration,
> Reference), as of **July 2026**. Code targets `cucumber-java` / `cucumber-junit` /
> `cucumber-junit-platform-engine` / `cucumber-testng` / `cucumber-picocontainer` **7.20.x**, `testng` **7.10.x**,
> `allure-cucumber7-jvm` **2.35.x**, on **OpenJDK 17/21**. Parallelism defaults and Allure plugin class names were
> cross-checked against source docs; footnotes at the end of each section cite the relevant page.

---

## Table of Contents

1. [What is Cucumber & BDD](#1-what-is-cucumber--bdd)
2. [Toolchain & Maven/Gradle Coordinates](#2-toolchain)
3. [Gherkin — Every Keyword Explained](#3-gherkin)
4. [Step Definitions — Cucumber Expressions, Regex, Custom Types](#4-step-definitions)
5. [Data-Driven Testing — DataTable, Scenario Outline, POJOs](#5-data-driven-testing)
6. [Hooks — Full Lifecycle & Exact Execution Order](#6-hooks)
7. [Tags & Tag Expressions](#7-tags)
8. [Runners — JUnit 4, JUnit 5 (Platform), TestNG](#8-runners)
9. [Parallel Execution — Deep Dive](#9-parallel-execution)
10. [Dependency Injection — PicoContainer](#10-di)
11. [State Management Rules](#11-state-management)
12. [Reporting — Allure (Full Configuration)](#12-allure)
13. [Reporting — Other Plugins (HTML/JSON/JUnit/Timeline)](#13-other-reporting)
14. [CI Integration](#14-ci)
15. [Best Practices](#15-best-practices)
16. [Interview Q&A Bank](#16-qa)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. What is Cucumber & BDD

Cucumber is a **test automation tool that supports Behaviour-Driven Development (BDD)**. It is not itself an
assertion framework — it reads plain-text executable specifications (**Gherkin**), matches each line to a piece
of glue code (a **step definition**), executes that code (typically delegating to JUnit/TestNG assertions,
Selenium, REST-assured, etc.), and reports pass/fail per scenario.

```
┌────────────┐                 ┌──────────────┐                 ┌───────────┐
│   Steps    │   matched with  │     Step     │   manipulates   │  System   │
│ in Gherkin ├────────────────>│ Definitions  ├────────────────>│  Under    │
│ (.feature) │                 │  (glue code) │                 │   Test    │
└────────────┘                 └──────────────┘                 └───────────┘
```

**Core BDD idea:** discovery, collaboration, and *executable examples* — business, QA, and Dev agree on
behaviour using concrete examples written in a shared "ubiquitous language" (Gherkin), which doubles as living
documentation and an automated regression suite.

**A `.feature` file serves three purposes:**
- An unambiguous, executable specification
- An automated test (via the glue code)
- Living documentation of actual system behaviour, versioned alongside the code it specifies

**How a run works, end to end (relevant to later sections):**
1. Cucumber discovers `.feature` files (from a `features` path/classpath resource).
2. It parses each `Scenario`/`Example`/`Scenario Outline` row into an internal unit called a **Pickle**.
3. For each Pickle, it matches every step's text against the registered step-definition patterns on the **glue**
   path.
4. It instantiates a **fresh object graph of every glue class** for that Pickle — the fact that makes parallel
   execution safe or unsafe (§9, §10).
5. It executes hooks and steps in order, records the result, and feeds it to every configured **plugin** (pretty
   console printer, HTML, JSON, Allure, etc. — §12–13).
6. A **runner** (JUnit 4 `@RunWith(Cucumber.class)`, JUnit Platform `@Suite`, or TestNG
   `AbstractTestNGCucumberTests`) invokes steps 1–5; Cucumber-JVM has no standalone "run" without one of these
   three (or the bare CLI, `io.cucumber.core.cli.Main`).

---

## 2. Toolchain & Maven/Gradle Coordinates <a name="2-toolchain"></a>

```xml
<properties>
    <java.version>17</java.version>
    <cucumber.version>7.20.1</cucumber.version>
    <junit.jupiter.version>5.11.0</junit.jupiter.version>
    <junit.platform.version>1.11.0</junit.platform.version>
    <testng.version>7.10.2</testng.version>
    <allure.version>2.35.3</allure.version>
    <aspectj.version>1.9.25</aspectj.version>
</properties>

<dependencies>
    <!-- Core: step definitions, Gherkin parsing, plugin engine -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-java</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Pick ONE runner family (or more than one, if you truly need to) -->
    <dependency> <!-- JUnit 4 -->
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency> <!-- JUnit 5 / JUnit Platform (recommended by Cucumber docs) -->
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit-platform-engine</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.platform</groupId>
        <artifactId>junit-platform-suite</artifactId>
        <version>${junit.platform.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency> <!-- TestNG -->
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-testng</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>${testng.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Dependency injection (recommended default for shared Page Objects) -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-picocontainer</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Which runner?** The Cucumber docs recommend **JUnit Platform** (`cucumber-junit-platform-engine`) as the
default for new projects — JUnit 4's `cucumber-junit` is functionally frozen/deprecated in its favour. **TestNG
remains the mainstream choice** for teams needing TestNG's native `@DataProvider`-driven parallelism, XML suite
orchestration, and retry/listener ecosystem (`IRetryAnalyzer`, `IAnnotationTransformer`) — both are covered in
full in §8–9.

---

## 3. Gherkin — Every Keyword Explained <a name="3-gherkin"></a>

Gherkin has **primary** and **secondary** keywords. Indentation is convention only (2 spaces or tabs — Gherkin
doesn't enforce it). Comments start a line with `#`; there is **no block comment** syntax.

### 3.1 Primary keywords

| Keyword | Colon required? | Purpose |
|---|---|---|
| `Feature` | Yes | Top-level container; exactly one per `.feature` file |
| `Rule` | Yes | Groups scenarios illustrating one business rule (Gherkin 6+) |
| `Example` / `Scenario` | Yes | A single concrete, executable example (true synonyms) |
| `Given` | No | Establish initial context (past state) |
| `When` | No | Describe the action/event under test |
| `Then` | No | Assert the expected, *observable* outcome |
| `And`, `But` | No | Continuation of the previous step's type, for readability |
| `*` | No | Generic bullet-style step keyword |
| `Background` | Yes | Steps that run before every scenario in a Feature/Rule |
| `Scenario Outline` / `Scenario Template` | Yes | A templated scenario, run once per data row |
| `Examples` / `Scenarios` | Yes | The data table that feeds a `Scenario Outline` |

> **Common pitfall:** a colon after a step keyword (`Given:`) silently breaks the parse — only the structural
> keywords above (`Feature`, `Rule`, `Example`/`Scenario`, `Background`, `Scenario Outline`, `Examples`) take one.

### 3.2 Secondary keywords

| Symbol | Purpose |
|---|---|
| `"""` or ` ``` ` | Doc Strings (multi-line text argument to a step) |
| `\|` | Data Tables |
| `@` | Tags |
| `#` | Comments (line-start only) |

### 3.3 `Feature`

```gherkin
Feature: User Login
  As a registered user
  I want to log into the application
  So that I can access my personalized dashboard
```

- Free-text lines under `Feature:` (the "As a / I want / So that" narrative) are **not parsed** — pure
  documentation, preserved verbatim in HTML/Allure reports.
- The description ends the moment a line starts with `Background`, `Rule`, `Example`/`Scenario`, or
  `Scenario Outline`.
- Exactly **one `Feature` per file**. Tags above `Feature` propagate to every scenario inside it.

### 3.4 `Rule` (Gherkin 6+)

Groups the examples that illustrate one business rule — Cucumber's HTML/Allure reports nest scenarios under
their `Rule` when present.

```gherkin
Feature: User Login

  Rule: Account is locked after 3 consecutive failed attempts

    Example: Third consecutive wrong password locks the account
      Given the user "alice" has failed to log in 2 times already
      When "alice" attempts to log in with an incorrect password
      Then the account "alice" should be locked
      And a lockout notification email should be queued

    Example: A successful login before the 3rd attempt resets the counter
      Given the user "bob" has failed to log in 1 time already
      When "bob" logs in with the correct password
      Then the failed-attempt counter for "bob" should reset to 0
```

`Background` is valid **inside a `Rule`** too — scoped to that rule's examples, running after any Feature-level
`Background`.

### 3.5 `Background`

Runs **before every `Scenario`** in its enclosing `Feature`/`Rule`, immediately after any `@Before` hooks and
before the scenario's own steps (exact ordering in §6.1).

```gherkin
Feature: Shopping Cart

  Background:
    Given the catalog contains the following products:
      | sku    | name           | price |
      | SKU100 | Wireless Mouse | 25.00 |
      | SKU200 | USB-C Cable    | 9.50  |
    And the user "carol" is logged in

  Scenario: Adding a single item updates the cart total
    When "carol" adds "SKU100" to the cart
    Then the cart total should be "25.00"
```

**Hygiene rule:** keep `Background` to 3–5 steps, and only for state genuinely needed by *every* scenario in the
file. A `Background` that only 60% of scenarios need is a smell — split the feature file, or move that setup
into the specific scenarios that need it.

### 3.6 `Scenario` (a.k.a. `Example`)

A concrete list of steps that is simultaneously specification, documentation, and an executable test. Keep to
roughly **3–5 steps**; more erodes expressive power and usually signals the scenario tests more than one
behaviour.

```gherkin
  Scenario: Removing the only item empties the cart
    Given "carol" has "SKU200" in the cart
    When "carol" removes "SKU200" from the cart
    Then the cart should be empty
    And the cart total should be "0.00"
```

### 3.7 `Scenario Outline` / `Examples`

Runs the same steps once per data row, substituting `<placeholder>` tokens. **Each row becomes a separate,
independently reportable scenario at runtime** — this matters for parallelism (§9), because each Outline row is
its own unit of parallel work, exactly like a standalone `Scenario`.

```gherkin
  Scenario Outline: Applying a discount code reduces the total correctly
    Given "carol" has "SKU100" in the cart with quantity <qty>
    When "carol" applies the discount code "<code>"
    Then the cart total should be "<expectedTotal>"

    Examples: Percentage-based codes
      | qty | code   | expectedTotal |
      | 1   | SAVE10 | 22.50         |
      | 2   | SAVE10 | 45.00         |

    Examples: Flat-amount codes
      | qty | code     | expectedTotal |
      | 1   | FLAT5OFF | 20.00         |
```

Multiple `Examples:` blocks (with distinct titles) under one Outline are legal — useful for grouping
semantically different data sets, each carrying its own tags.

### 3.8 Steps: `Given`, `When`, `Then`, `And`, `But`, `*`

**Critical rule:** the keyword has **zero significance to step matching**. Cucumber strips it and matches only
the remaining text against step-definition patterns. So the following are **duplicate/ambiguous** at
glue-loading time:

```gherkin
Given there is money in my account
Then there is money in my account        # duplicate! identical match text
```

Fix by making the wording state-specific:

```gherkin
Given my account has a balance of £430
Then my account should have a balance of £430
```

- **`Given`** — put the system into a known state (seed a DB, create objects). Avoid describing UI interaction.
- **`When`** — the single action under test. Rule of thumb: *"imagine it's 1922"* — describe the action without
  assuming a particular UI/technology (`the user submits the order`, not `the user clicks the blue Submit
  button`).
- **`Then`** — assert an **observable** outcome (UI element, API response body, queued message, report) —
  **never** assert directly against internal DB state; that couples the spec to an implementation detail
  (§15.2).
- **`And` / `But`** — stylistic continuation of the preceding step's type; improves readability with multiple
  `Given`s/`Then`s.
- **`*`** — generic bullet keyword, useful for list-like sequences of `Given`s where "And" reads awkwardly.

---

## 4. Step Definitions — Cucumber Expressions, Regex, Custom Types <a name="4-step-definitions"></a>

```java
package com.example.steps;

import io.cucumber.java.en.*;
import io.cucumber.java.ParameterType;
import io.cucumber.java.DataTableType;

public class OrderSteps {

    // Cucumber Expression — {int}, {string}, {word}, {float}, {double}, {biginteger},
    // {bigdecimal}, {byte}, {short}, {long} are built-in parameter types.
    @Given("{int} cucumbers are in the basket")
    public void cucumbersInBasket(int count) { /* ... */ }

    // Regex step (must start with ^ and end with $ by convention, though not enforced)
    @Given("^(\\d+) cucumbers are in the basket$")
    public void cucumbersInBasketRegex(int count) { /* ... */ }

    // {string} captures the content INSIDE the quotes, quotes are stripped automatically
    @When("the user searches for {string}")
    public void search(String term) { /* ... */ }

    // Custom ParameterType — maps a domain word straight to a typed/enum value
    @ParameterType("small|medium|large")
    public String size(String raw) { return raw; }

    @Given("the user selects a {size} coffee")
    public void selectSize(String size) { /* size is already validated by the regex above */ }

    // DocString — always the LAST method parameter, typically String (or io.cucumber.docstring.DocString)
    @Given("the following JSON payload:")
    public void jsonPayload(String json) { /* ... */ }

    // DataTable — last parameter can be DataTable, List<Map<String,String>>, List<List<String>>,
    // or a @DataTableType-mapped POJO/List<POJO>
    @Given("the catalog contains the following products:")
    public void catalog(io.cucumber.datatable.DataTable table) {
        java.util.List<java.util.Map<String, String>> rows = table.asMaps();
    }
}
```

**Argument-count rule:** the method's parameter count must exactly equal the number of capture
groups/placeholders in the pattern; a DocString or DataTable adds exactly **one** extra trailing parameter.
Mismatches throw an error at **glue-loading time**, before any scenario runs — a fast-fail, not a runtime
surprise mid-suite.

**Cucumber Expressions vs Regex:** Cucumber Expressions are the default/recommended style (readable, typed,
IDE-friendly via the official plugin). Reach for regex only when you need alternation or lookarounds Cucumber
Expressions can't express (`^I have (?:a|an) (.*)$`).

---

## 5. Data-Driven Testing — DataTable, Scenario Outline, POJOs <a name="5-data-driven-testing"></a>

### 5.1 `Scenario Outline` vs `DataTable`

| | `Scenario Outline` | `DataTable` |
|---|---|---|
| Re-runs | The **whole scenario**, once per row | **One step** gets a block of rows |
| Reported as | N separately reported scenario results | One scenario result |
| Best for | Genuinely distinct, independently-reportable test cases | Bulk setup/assertion data belonging to one test case |
| Parallel unit | Each row is its own parallel unit (§9) | N/A — the row block is part of one scenario's single unit |

### 5.2 Mapping a `DataTable` to a POJO

```java
public class Product {
    public String sku;
    public String name;
    public double price;
}
```

```java
package com.example.config;

import io.cucumber.java.DataTableType;

public class TypeRegistryConfig {
    @DataTableType
    public Product productEntry(java.util.Map<String, String> entry) {
        Product p = new Product();
        p.sku = entry.get("sku");
        p.name = entry.get("name");
        p.price = Double.parseDouble(entry.get("price"));
        return p;
    }
}
```

```java
@Given("the catalog contains the following products:")
public void catalog(java.util.List<Product> products) { /* auto-mapped via @DataTableType */ }
```

### 5.3 External data sources

For large or environment-specific data sets, load from CSV/JSON/a database inside a `@Before` hook or a
step-definition helper rather than inflating the `.feature` file — keep the Gherkin readable and treat bulky
fixtures as an implementation detail behind a step like
`Given the discount codes from "discount_codes.csv" are loaded`.

---

## 6. Hooks — Full Lifecycle & Exact Execution Order <a name="6-hooks"></a>

```java
package com.example.hooks;

import io.cucumber.java.*;
import com.example.context.TestContext;

public class Hooks {

    private final TestContext testContext;

    // PicoContainer injects the SAME TestContext instance step-def classes receive — see §10
    public Hooks(TestContext testContext) {
        this.testContext = testContext;
    }

    @BeforeAll  // static, runs ONCE per JVM before any scenario — no DI/instance state available here
    public static void globalSetup() {
        System.setProperty("webdriver.http.factory", "jdk-http-client");
    }

    @Before(order = 0)  // lower order runs first among @Before hooks
    public void initDriver(Scenario scenario) {
        testContext.setDriver(DriverFactory.create(System.getProperty("browser", "chrome")));
        testContext.setScenario(scenario);
    }

    @Before(value = "@api", order = 1)  // conditional/tagged hook — only for @api-tagged scenarios
    public void initApiClient() {
        testContext.setApiClient(new RestAssuredClientWrapper());
    }

    @BeforeStep
    public void logStepStart() {
        testContext.getScenario().log("Starting step at " + java.time.Instant.now());
    }

    @AfterStep
    public void screenshotOnFailure(Scenario scenario) {
        if (scenario.isFailed() && testContext.getDriver() != null) {
            byte[] shot = ((org.openqa.selenium.TakesScreenshot) testContext.getDriver())
                    .getScreenshotAs(org.openqa.selenium.OutputType.BYTES);
            scenario.attach(shot, "image/png", scenario.getName());
        }
    }

    @After(order = 1)  // @After hooks run in REVERSE numeric order — this runs BEFORE order=0
    public void tearDownApiClient(Scenario scenario) {
        testContext.closeApiClientIfPresent();
    }

    @After(order = 0)
    public void quitDriver(Scenario scenario) {
        if (testContext.getDriver() != null) testContext.getDriver().quit();
    }

    @AfterAll  // static, runs ONCE per JVM, after the LAST scenario
    public static void globalTeardown() { /* flush a shared metrics/reporting sink */ }
}
```

### 6.1 Execution order, precisely (for a scenario tagged `@api`)

```
@BeforeAll (once per JVM, static)
  -> @Before hooks, ASCENDING `order` (an unspecified order defaults to 10000, i.e. runs last
     among Befores — set order explicitly whenever sequencing matters)
       -> conditional @Before(value="@api") runs here only because the scenario carries @api
  -> Background steps (Gherkin, top to bottom) — each wrapped by @BeforeStep / @AfterStep too
  -> Scenario's own steps, each wrapped by: @BeforeStep -> [step] -> @AfterStep
  -> @After hooks, DESCENDING `order` (reverse of @Before — mirrors a stack: last resource
     acquired is the first torn down)
@AfterAll (once per JVM, static, after the LAST scenario in the run)
```

**Commonly missed detail:** `Background` steps run **after** `@Before` hooks, but are themselves ordinary
Gherkin steps — `@BeforeStep`/`@AfterStep` wrap them too, since those hooks fire for *every* step regardless of
whether it came from `Background` or the scenario body. `@Before`/`@After` are scenario-level (run once);
`@BeforeStep`/`@AfterStep` are step-level (run once per step).

### 6.2 Conditional (tagged) hooks

```java
@Before("@smoke and not @flaky")
public void setupForStableSmoke() { /* ... */ }

@After("@checkout or @payment")
public void captureNetworkHar(Scenario scenario) { /* ... */ }
```

Tag expressions in hook annotations use the same grammar as `@CucumberOptions(tags = ...)` /
`cucumber.filter.tags` — see §7.

---

## 7. Tags & Tag Expressions <a name="7-tags"></a>

| Operator | Symbol | Example | Meaning |
|---|---|---|---|
| AND | `and` | `@smoke and @checkout` | both tags present |
| OR | `or` | `@smoke or @regression` | either tag present |
| NOT | `not` | `not @wip` | tag absent |
| Grouping | `( )` | `(@smoke or @regression) and not @flaky` | precedence control |

```gherkin
@login @regression
Feature: User Login

  @smoke @fast
  Scenario: Successful login with valid credentials
    ...
```

```bash
mvn test -Dcucumber.filter.tags="@smoke and not @wip"
```

```java
@CucumberOptions(tags = "@smoke and not @wip")
```

**Tags may only be placed above:** `Feature`, `Rule`, `Scenario`/`Example`, `Scenario Outline`, and `Examples`.
Tags **cannot** be placed above an individual step or a `Background`. A tag above `Feature`/`Rule` is inherited
by every nested scenario.

`cucumber.filter.tags` and `cucumber.filter.name` (a regex on the scenario name) combine with **logical AND** —
a scenario must satisfy both to be selected.

---

## 8. Runners — JUnit 4, JUnit 5 (Platform), TestNG <a name="8-runners"></a>

A runner is what actually *drives* Cucumber — Cucumber-JVM cannot execute on its own. This section covers the
three officially supported runners; §9 builds on it for parallelism.

### 8.1 JUnit 4 runner (`cucumber-junit`)

```java
package com.example;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
    features   = "src/test/resources/features",
    glue       = {"com.example.steps", "com.example.hooks"},
    tags       = "@smoke and not @wip",
    plugin     = {"pretty", "html:target/cucumber-report.html", "json:target/cucumber.json"},
    monochrome = true,
    dryRun     = false,
    snippets   = io.cucumber.junit.CucumberOptions.SnippetType.CAMELCASE
)
public class RunCucumberTest {
    // intentionally empty — acts as a JUnit "suite" wrapper; no test methods of its own
}
```

- `dryRun = true` verifies every step has exactly one matching definition **without executing real logic** — a
  fast CI "spec completeness" gate before the slow suite runs.
- `snippets` controls the style of auto-generated stubs Cucumber prints for undefined steps (`UNDERSCORE` is
  default; `CAMELCASE` also available).
- Cucumber supports JUnit's `@ClassRule`/`@BeforeClass`/`@AfterClass` on the runner class, but the docs
  **recommend against** them — they hurt portability across CLI/IDE test runners; prefer `@BeforeAll`/`@AfterAll`
  Cucumber hooks instead.
- **JUnit 4 parallelism limitation (§9.2):** only **features** can run in parallel, never individual scenarios
  within one feature file — all scenarios in a given `.feature` are always executed by the same thread.

### 8.2 JUnit 5 / JUnit Platform runner (`cucumber-junit-platform-engine`) — recommended

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <version>7.20.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-suite</artifactId>
    <version>1.11.0</version>
    <scope>test</scope>
</dependency>
```

**Two equivalent ways to define the runner class:**

```java
// Option A — @Suite (works with JUnit Platform Surefire provider)
package com.example;

import org.junit.platform.suite.api.*;
import static io.cucumber.junit.platform.engine.Constants.*;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.steps")
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@smoke and not @wip")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, value = "pretty, html:target/cucumber-report.html")
public class RunCucumberTest {
}
```

```java
// Option B — the newer @Cucumber marker annotation (equivalent, less boilerplate)
package com.example;

import io.cucumber.junit.platform.engine.Cucumber;

@Cucumber
public class RunCucumberTest {
    // Cucumber discovers features/glue by convention: features on the classpath resource
    // matching this package, glue = this package. Override via junit-platform.properties.
}
```

Configuration is driven by **JUnit Platform's own mechanisms** — `@ConfigurationParameter` annotations or a
`src/test/resources/junit-platform.properties` file — **not** `@CucumberOptions` (that annotation doesn't exist
for this runner). Common properties:

```properties
# junit-platform.properties
cucumber.glue=com.example.steps,com.example.hooks
cucumber.plugin=pretty, html:target/cucumber-report.html, io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm
cucumber.filter.tags=@smoke and not @wip
cucumber.execution.order=random
```

**Maven Surefire provider requirement:** running through Maven `test`/`verify` needs either the JUnit Platform
provider (bundled from Surefire 2.22+) or an explicit `junit-platform-surefire-provider` — with a recent
`maven-surefire-plugin` (3.x) and `junit-platform-suite` on the classpath, no extra provider config is normally
required.

### 8.3 TestNG runner (`cucumber-testng`) — sequential baseline

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-testng</artifactId>
    <version>7.20.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testng</groupId>
    <artifactId>testng</artifactId>
    <version>7.10.2</version>
    <scope>test</scope>
</dependency>
```

```java
package com.example.runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;

@CucumberOptions(
    features = "src/test/resources/features",
    glue     = {"com.example.steps", "com.example.hooks"},
    tags     = "@regression",
    plugin   = {"pretty", "html:target/cucumber-reports/report.html",
                "json:target/cucumber-reports/report.json"},
    monochrome = true
)
public class RunCucumberTest extends AbstractTestNGCucumberTests {
    // No method overrides needed for sequential execution — see §9.1 for parallel.
}
```

`AbstractTestNGCucumberTests` exposes every Gherkin Pickle (scenario, or one row of a Scenario Outline) as a row
of a TestNG `@DataProvider`-backed method called `scenarios()`. Left un-overridden, TestNG runs those rows
**sequentially**, one at a time, on a single thread.

**Version compatibility:** Cucumber-JVM 7 needs **TestNG 7.8 or higher**, which in turn requires **Java 11+**.

---

## 9. Parallel Execution — Deep Dive <a name="9-parallel-execution"></a>

Cucumber-JVM supports three independent parallel-execution mechanisms — one per runner family — plus a
CLI-only mode. They are **not interchangeable**: granularity and configuration surface differ meaningfully.

### 9.1 TestNG — parallel BY SCENARIO (the idiomatic Cucumber-TestNG approach)

```java
package com.example.runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;
import org.testng.annotations.DataProvider;

@CucumberOptions(
    features = "classpath:features",
    glue     = {"com.example.steps", "com.example.hooks", "com.example.di"},
    tags     = "@regression",
    plugin   = {
        "pretty",
        "json:target/cucumber-reports/cucumber.json",
        "html:target/cucumber-reports/cucumber.html",
        "timeline:target/cucumber-reports/timeline",        // visualises thread usage
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm"   // Allure — see §12
    },
    monochrome = true
)
public class ParallelCucumberTestRunner extends AbstractTestNGCucumberTests {

    // Overriding scenarios() with @DataProvider(parallel = true) is what actually
    // enables scenario-level parallelism under TestNG's DataProvider mechanism.
    @Override
    @DataProvider(parallel = true)
    public Object[][] scenarios() {
        return super.scenarios();
    }
}
```

**`pom.xml` — Surefire configuration:**

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.5.2</version>
      <configuration>
        <suiteXmlFiles>
          <suiteXmlFile>testng.xml</suiteXmlFile>
        </suiteXmlFiles>
        <properties>
          <property>
            <name>dataproviderthreadcount</name>
            <value>4</value>   <!-- default is 10 if omitted -->
          </property>
        </properties>
      </configuration>
    </plugin>
  </plugins>
</build>
```

**`testng.xml`:**

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="ParallelSuite" parallel="methods" data-provider-thread-count="4">
    <test name="RegressionParallel">
        <classes>
            <class name="com.example.runners.ParallelCucumberTestRunner"/>
        </classes>
    </test>
</suite>
```

Both `@DataProvider(parallel = true)` **and** the suite-level `data-provider-thread-count` (or the Surefire
`dataproviderthreadcount` property) are required together — the annotation alone is necessary but not
sufficient. **Every `@DataProvider(parallel = true)` in a suite defaults to 10 threads if no thread-count is
specified anywhere.**

Under this model, **each Gherkin scenario — and each individual row of a `Scenario Outline`** — is one
independent unit of parallel work, dispatched to TestNG's own thread pool.

### 9.2 Feature-level parallelism (TestNG, coarser granularity)

Where scenario-level parallelism is too fine-grained (e.g. all scenarios in one feature share an exclusive
sandbox/tenant that can't be hit concurrently, but other features can run alongside it safely), give each
feature file its own thin runner class and its own `<test>` block, and parallelize at the **feature** level via
`parallel="tests"` instead of `parallel="methods"`:

```java
// LoginFeatureRunner.java — drives ONLY login.feature
@CucumberOptions(
    features = "src/test/resources/features/login.feature",
    glue = {"com.example.steps", "com.example.hooks"},
    plugin = {"pretty", "json:target/cucumber-reports/login.json"},
    monochrome = true
)
public class LoginFeatureRunner extends AbstractTestNGCucumberTests {
    // NOTE: scenarios() is NOT overridden with parallel=true here — scenarios inside
    // this one feature run sequentially; only the *feature* itself is a parallel unit.
}
```

```xml
<suite name="Feature-Parallel Suite" parallel="tests" thread-count="3">
    <test name="Login">
        <classes><class name="com.example.runners.LoginFeatureRunner"/></classes>
    </test>
    <test name="ShoppingCart">
        <classes><class name="com.example.runners.CartFeatureRunner"/></classes>
    </test>
    <test name="OrderManagement">
        <classes><class name="com.example.runners.OrderFeatureRunner"/></classes>
    </test>
</suite>
```

| | Scenario-level (§9.1) | Feature-level (§9.2) |
|---|---|---|
| Unit of parallelism | Every scenario (and Outline row) independently | Every feature file; scenarios inside it run sequentially |
| Best for | Stateless, fully-isolated scenarios (own test data, own DB rows) | Features sharing an expensive/exclusive fixture internally, but safe across features |
| Config | One runner, `@DataProvider(parallel=true)`, `data-provider-thread-count` | N thin runners (one per feature/group), `parallel="tests"` |
| Thread-safety bar | Every hook/page-object/context must be thread-safe **per scenario** | Only cross-feature state must be thread-safe; within-feature state can assume single-thread |

Most mature suites default to **scenario-level (9.1)** and reach for feature-level runners only for the small
set of features with real cross-scenario shared-resource constraints.

### 9.3 JUnit Platform — parallel execution (opt-in, off by default)

Under the JUnit Platform engine, **Cucumber runs sequentially in a single thread by default.** Parallel
execution is an explicit opt-in, configured entirely through JUnit Platform configuration parameters — most
conveniently a `src/test/resources/junit-platform.properties` file:

```properties
# junit-platform.properties
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=fixed
cucumber.execution.parallel.config.fixed.parallelism=4
cucumber.execution.parallel.config.fixed.max-pool-size=4
```

Or via `@ConfigurationParameter` on the `@Suite` runner class:

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = "cucumber.execution.parallel.enabled", value = "true")
@ConfigurationParameter(key = "cucumber.execution.parallel.config.strategy", value = "fixed")
@ConfigurationParameter(key = "cucumber.execution.parallel.config.fixed.parallelism", value = "4")
public class RunCucumberTest {
}
```

**Strategy options** (`cucumber.execution.parallel.config.strategy`):

| Strategy | Behaviour |
|---|---|
| `dynamic` (default) | Desired parallelism = `<available CPU cores> * cucumber.execution.parallel.config.dynamic.factor` (factor defaults to `1`) |
| `fixed` | Set `cucumber.execution.parallel.config.fixed.parallelism` (desired) and `.fixed.max-pool-size` (ceiling on the underlying ForkJoinPool) explicitly |
| `custom` | Point `cucumber.execution.parallel.config.custom.class` at your own `ParallelExecutionConfigurationStrategy` implementation |

**Granularity:** by default, once parallel execution is enabled, **both scenarios and individual Scenario
Outline rows** run in parallel (the JUnit 5/6 engine's native scenario-level model). Restrict this back to
feature-level granularity with `cucumber.execution.execution-mode.feature=same_thread`, which forces every
scenario within one feature file onto the same thread while still parallelizing across different feature
files.

**Known caveat (documented upstream):** `fixed.max-pool-size` caps the underlying ForkJoin pool's size, but
does **not** strictly guarantee the number of concurrently *executing* scenarios never exceeds that number —
treat it as an upper-bound tuning knob, not an absolute hard ceiling.

**Synchronizing on a shared/exclusive resource:** tag a scenario, map that tag to a lock name via
configuration, and Cucumber will serialize scenarios sharing the same read-write or read-only lock — the
JUnit-Platform-native equivalent of §9.2's feature-level-runner workaround.

### 9.4 CLI-only parallel execution (no runner class at all)

```bash
java -cp "target/classes:target/test-classes:lib/*" \
     io.cucumber.core.cli.Main --threads 4 -g com.example.steps src/test/resources/features
```

`--threads` set above `1` activates the CLI's own parallel engine, independent of both TestNG and JUnit
Platform. Scenarios and individual Scenario Outline rows run on different threads, same as the JUnit Platform
model in §9.3.

### 9.5 Thread-safety requirements parallelism introduces

Whichever mechanism you use, **N scenarios now run concurrently on N threads**, each needing its **own**
`WebDriver`/HTTP client/mutable state:

```java
// WRONG under parallel execution — shared mutable static field, race condition
public class DriverFactory {
    private static WebDriver driver;   // ❌ one instance shared by every thread
}

// RIGHT — ThreadLocal, OR (preferred) a DI-scoped instance created fresh per scenario (§10)
public class DriverFactory {
    private static final ThreadLocal<WebDriver> DRIVER = ThreadLocal.withInitial(ChromeDriver::new);
    public static WebDriver get()   { return DRIVER.get(); }
    public static void remove()     { DRIVER.remove(); }
}
```

Cucumber itself already **instantiates a brand-new instance of every glue class per scenario/Pickle** — the
risk lies only in code you make `static` yourself (drivers, singletons, caches, counters). This is why §10's
PicoContainer-scoped `TestContext`/`DriverManager` pattern is the architecturally correct pairing with any of
the parallel models above: PicoContainer's per-scenario object graph *is* the isolation boundary, eliminating
manual `ThreadLocal` bookkeeping.

---

## 10. Dependency Injection — PicoContainer <a name="10-di"></a>

### 10.1 Why DI at all

Step-definition classes are stateless collaborators — Cucumber instantiates a **new object graph for every
scenario/Pickle**, and different Gherkin steps of one scenario are typically implemented across *multiple*
step-definition classes (`LoginSteps`, `CartSteps`, `Hooks`). Those classes need to share a `WebDriver`, Page
Objects, and scenario-scoped state — exactly what a DI module solves.

Cucumber officially supports: **PicoContainer** (the recommended default), Spring, Guice, OpenEJB, CDI
(Weld/Jakarta), Quarkus. PicoContainer requires the least setup and adds zero framework dependency to
application code.

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-picocontainer</artifactId>
    <version>7.20.1</version>
    <scope>test</scope>
</dependency>
```

**How it works:** PicoContainer inspects the **constructors** of your glue classes. Any parameter of a type
PicoContainer already manages gets injected automatically — **no annotations required**, unlike Spring's
`@Autowired`. It creates and destroys this object graph **once per scenario**, exactly the lifetime you want for
a shared `WebDriver`.

### 10.2 A shared per-scenario `TestContext`

```java
package com.example.context;

import io.cucumber.java.Scenario;
import org.openqa.selenium.WebDriver;

public class TestContext {
    // NOTHING here is static. PicoContainer creates ONE TestContext per scenario and
    // injects that SAME instance into every glue class constructor that asks for it.
    // A static field would be shared across ALL concurrently-running scenarios under
    // §9's parallel models — causing cross-thread state corruption.
    private WebDriver driver;
    private Scenario scenario;

    public WebDriver getDriver() { return driver; }
    public void setDriver(WebDriver driver) { this.driver = driver; }
    public Scenario getScenario() { return scenario; }
    public void setScenario(Scenario scenario) { this.scenario = scenario; }
}
```

```java
package com.example.steps;

import com.example.context.TestContext;
import io.cucumber.java.en.*;

public class LoginSteps {
    private final TestContext testContext;

    public LoginSteps(TestContext testContext) {   // <-- PicoContainer injects it here
        this.testContext = testContext;
    }

    @Given("the user navigates to the login page")
    public void navigateToLogin() {
        testContext.getDriver().get("https://app.example.com/login");
    }
}
```

`Hooks.java` (§6) also takes `TestContext` in its constructor — the driver initialized in `@Before` is the exact
same `WebDriver` every step-definition class retrieves via `testContext.getDriver()`, with zero manual wiring
code and zero static fields.

### 10.3 Why this is safe under parallel execution

Because `TestContext` is instantiated **fresh, per scenario, by PicoContainer**, every parallel thread — TestNG
(§9.1/9.2), JUnit Platform (§9.3), or CLI (§9.4) — gets its own independent object graph automatically. No
`ThreadLocal` is needed at this level; PicoContainer's per-scenario container *is* the isolation boundary:

```
Parallel runner (TestNG DataProvider / JUnit Platform ForkJoinPool / CLI --threads)
        →  N scenarios execute concurrently
        +
cucumber-picocontainer
        →  N independent object graphs, one per scenario
        =
Thread-safe parallel execution with shared Page Objects per scenario, zero manual locking
```

### 10.4 Custom object factory (integrating your app's own DI framework)

If your application under test already runs on Guice/Spring, make Cucumber's injector delegate into that same
framework:

```java
package com.example.app;

import io.cucumber.core.backend.ObjectFactory;
import io.cucumber.guice.CucumberModules;
import io.cucumber.guice.ScenarioScope;
import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.Stage;

public final class CustomObjectFactory implements ObjectFactory {
    private final Injector injector = Guice.createInjector(
        Stage.PRODUCTION, CucumberModules.createScenarioModule(), new ServiceModule());

    @Override public boolean addClass(Class<?> clazz) { return true; }
    @Override public void start() { injector.getInstance(ScenarioScope.class).enterScope(); }
    @Override public void stop()  { injector.getInstance(ScenarioScope.class).exitScope(); }
    @Override public <T> T getInstance(Class<T> clazz) { return injector.getInstance(clazz); }
}
```

Registered via SPI (`META-INF/services/io.cucumber.core.backend.ObjectFactory`) or explicitly with
`@CucumberOptions(objectFactory = CustomObjectFactory.class)` (TestNG/JUnit4) or the equivalent
`cucumber.object-factory` property (JUnit Platform).

---

## 11. State Management Rules <a name="11-state-management"></a>

| Rule | Why |
|---|---|
| **Never share state between scenarios.** No static/global mutable fields. | Scenarios must run independently and in any order, including in parallel (§9). |
| Clean the database in a `@Before` hook. | Prevents leakage from a previous scenario's leftover rows. |
| Delete cookies / reset the browser in `@Before` if reusing a driver. | Prevents session bleed-through. |
| Sharing state **between steps within one scenario** is fine, via DI (§10) or instance fields on the glue class. | This is the intended use of a `TestContext`/"World" object — Cucumber gives every scenario a fresh instance anyway. |
| Spring users: annotate shared beans `@ScenarioScope`. Guice users: annotate step classes `@ScenarioScoped`. | Ensures the DI container itself does not leak state across scenarios. |

---

## 12. Reporting — Allure (Full Configuration) <a name="12-allure"></a>

Allure Report generates rich, interactive HTML reports (timeline view, retries/history, step trees, attachments,
categorized failures) from Cucumber's run results.

### 12.1 Integration steps overview

1. Add Allure dependencies (via `allure-bom` for version alignment).
2. Activate the Allure Cucumber-JVM plugin — the **plugin class name is the same regardless of runner**:
   `io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm`.
3. Configure AspectJ (needed for `@Step`/`@Attachment` annotations to work).
4. Point Allure at a results directory via `allure.properties`.
5. Run the tests, then generate/serve the HTML report from the produced `allure-results` folder.

> **Version-name gotcha:** the artifact and plugin class are literally named `cucumber7jvm` — this covers
> Cucumber-JVM 7.x. For Cucumber-JVM 6 use `allure-cucumber6-jvm` and
> `io.qameta.allure.cucumber6jvm.AllureCucumber6Jvm`; the "7" is **not** an Allure major version, it tracks the
> Cucumber-JVM major version it targets.

### 12.2 Maven dependencies (same for all three runners)

```xml
<properties>
    <allure.version>2.35.3</allure.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-bom</artifactId>
            <version>${allure.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-cucumber7-jvm</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Pick the ONE matching your runner: -->
    <dependency> <!-- JUnit Platform (recommended) -->
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit-platform</artifactId>
        <scope>test</scope>
    </dependency>
    <!--
    <dependency>  TestNG
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-testng</artifactId>
        <scope>test</scope>
    </dependency>
    -->
    <!--
    <dependency>  JUnit 4
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit4</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit4-aspect</artifactId>
        <scope>test</scope>
    </dependency>
    -->
</dependencies>
```

> **TestNG dependency-order caveat (official docs):** if you also use `allure-testng` alongside Cucumber-TestNG,
> declare `io.qameta.allure:allure-testng` **after** `io.cucumber:cucumber-testng` (or any explicit TestNG
> dependency) in your POM — otherwise Maven can transitively resolve a conflicting/older TestNG version, since
> `allure-testng` also supports TestNG 6.

### 12.3 Activating the plugin per runner

**JUnit Platform** — `src/test/resources/junit-platform.properties`:
```properties
cucumber.plugin=io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm
```
or via `@ConfigurationParameter`:
```java
import org.junit.platform.suite.api.*;
import static io.cucumber.core.options.Constants.PLUGIN_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, value = "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm")
public class RunCucumberTest {
}
```

**TestNG:**
```java
@CucumberOptions(
    plugin = { "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm" }
)
public class RunCucumberTest extends AbstractTestNGCucumberTests {
}
```

**JUnit 4:**
```java
@RunWith(Cucumber.class)
@CucumberOptions(
    plugin = { "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm" }
)
public class RunCucumberTest {
}
```

Combine it with other plugins in the same array:
```java
plugin = {
    "pretty",
    "json:target/cucumber-reports/cucumber.json",
    "timeline:target/cucumber-reports/timeline",
    "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm"
}
```

### 12.4 AspectJ configuration (required for `@Step` / `@Attachment`)

Allure's `@Step` and `@Attachment` annotations (used for §12.7's sub-step breakdowns) rely on AspectJ weaving.
Configure the AspectJ weaver as a Surefire Java agent:

```xml
<properties>
    <aspectj.version>1.9.25</aspectj.version>
</properties>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.5.2</version>
    <configuration>
        <argLine>
            -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
        </argLine>
    </configuration>
    <dependencies>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>${aspectj.version}</version>
        </dependency>
    </dependencies>
</plugin>
```

> If your Surefire `argLine` is already used for something else (e.g. JaCoCo coverage), **append** the
> `-javaagent` flag rather than overwriting it — a common CI regression is one plugin's `argLine` silently
> clobbering another's.

### 12.5 Results directory — `allure.properties`

By default Allure writes results to the project root — rarely what you want alongside a `target`/`build`
convention. Override via `src/test/resources/allure.properties`:

```properties
# allure.properties (Maven layout)
allure.results.directory=target/allure-results
```
```properties
# allure.properties (Gradle layout)
allure.results.directory=build/allure-results
```

### 12.6 Running tests & generating the HTML report

```bash
# Maven — 'verify' (not just 'test') ensures post-integration-test-phase plugins can react too
./mvnw verify

# Gradle
./gradlew test
```

This produces raw result files (`*-result.json`, attachments) under `allure-results`. Turn that into a browsable
HTML report with the **Allure commandline** (installed separately, or via the `allure-maven` plugin):

```xml
<plugin>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-maven</artifactId>
    <version>2.15.2</version>
    <configuration>
        <reportVersion>${allure.version}</reportVersion>
        <resultsDirectory>${project.build.directory}/allure-results</resultsDirectory>
        <reportDirectory>${project.build.directory}/allure-report</reportDirectory>
    </configuration>
</plugin>
```

```bash
mvn allure:report   # generates a static HTML report at target/allure-report
mvn allure:serve     # generates AND opens a temporary local report in the browser (dev use)
```

(Gradle equivalent: the `io.qameta.allure` Gradle plugin exposes `allureReport` / `allureServe` tasks.)

### 12.7 Enriching reports — metadata, hierarchy, sub-steps, parameters, attachments

**Metadata via Gherkin tags** (requires Allure Report **2.27+**) or the Runtime API:

```gherkin
@allure.label.layer:web
@allure.label.owner:eroshenkoam
Feature: Labels

  @critical
  @allure.label.jira:AE-2
  Scenario: Create new label for authorized user
    When I open labels page
```

```java
import io.qameta.allure.Allure;

@When("^I open labels page$")
public void openLabelsPage() {
    Allure.label("severity", "critical");
    Allure.label("jira", "AE-2");
}
```

> Setting metadata via the Runtime API mid-step risks an incomplete report if the scenario fails before the
> call executes — set it as early in the step/hook as practical.

**Report navigation hierarchy** — behaviour-based (`epic`/`feature`/`story`), suite-based
(`parentSuite`/`suite`/`subSuite`), and package-based (derived automatically from the `.feature` file's location
relative to the test-resources root):

```gherkin
@allure.label.epic:Web
@allure.label.parentSuite:Cucumber
@allure.label.suite:Labels
Feature: Labels
```

**Sub-steps** — break one Gherkin step into a visible tree of finer-grained actions in the report, useful when
one step reads well as a single line but has multiple internal failure points:

```java
import io.qameta.allure.Step;
import io.qameta.allure.Allure;

@When("^I open labels page$")
public void openLabelsPage() {
    subStep1();
    subStep2();
}

@Step("Sub-step 1")
public void subStep1() { /* ... */ }

// or inline, without a dedicated method:
public void subStep2() {
    Allure.step("Sub-step 2", (step) -> { /* ... */ });
}
```

**Scenario Outline parameters** are supported automatically with no extra setup; you can also surface extra
parameters explicitly:

```java
@Then("authorize as {string}")
public void testAuthentication(String login) {
    Allure.parameter("Login", login);
}
```

**Attachments (screenshots, logs, payloads)** — the pattern every Selenium/Playwright suite needs:

```java
import io.qameta.allure.Allure;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Paths;

@AfterStep
public void screenshotOnFailure(Scenario scenario) {
    if (scenario.isFailed()) {
        byte[] shot = ((org.openqa.selenium.TakesScreenshot) driver)
                .getScreenshotAs(org.openqa.selenium.OutputType.BYTES);
        Allure.getLifecycle().addAttachment("Screenshot", "image/png", "png", shot);
        // OR, equivalently, Cucumber's own attach() also surfaces into Allure:
        // scenario.attach(shot, "image/png", scenario.getName());
    }
}
```

### 12.8 Environment information on the report's main page

Put an `environment.properties` file directly into the `allure-results` directory **after** the test run (e.g.
a post-test Maven/Gradle step, or a CI script step) to have OS/Java/browser/environment details show on the
report's landing page — useful when a failure only reproduces in one environment:

```properties
# allure-results/environment.properties
OS=Ubuntu 24.04
Java.Version=21.0.4
Browser=Chrome 127
Environment=staging
```

This is for properties **constant across the whole report**; for per-scenario values, use labels/parameters
(§12.7) instead.

### 12.9 Failure categories

Place a `categories.json` in `allure-results` to bucket failures into custom categories (e.g. "Product defects"
vs "Test defects" vs "Timeouts") shown as a dedicated tab in the report, matched by message/trace regex:

```json
[
  {
    "name": "Ignored tests",
    "matchedStatuses": ["skipped"]
  },
  {
    "name": "Infrastructure problems",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*(TimeoutException|ConnectException).*"
  },
  {
    "name": "Product defects",
    "matchedStatuses": ["failed"],
    "traceRegex": ".*AssertionError.*"
  }
]
```

### 12.10 Allure with parallel execution (§9)

Allure's Cucumber plugin is inherently thread-aware — each scenario's results are written to its own
UUID-identified result file, so **no extra configuration is needed** to make it safe under TestNG or JUnit
Platform parallel execution (§9.1–9.3). The one thing to watch: if you generate/aggregate results from multiple
parallel CI shards, merge all shards' `allure-results` directories into one before running
`allure:report`/`allure generate`, or you'll get one report per shard instead of a combined one.

### 12.11 Selective re-run (Allure TestOps integration)

If you use Allure TestOps (the commercial companion product) to re-run only a failed subset through the same CI
job: the JUnit Platform runner supports this natively; the JUnit 4 runner supports it out of the box under
Gradle (Maven Surefire has known limitations here); TestNG needs an extra `IDataProviderInterceptor` filter
class registered as a `META-INF/services/org.testng.ITestNGListener`. See the official reference docs
(`allurereport.org/docs/cucumberjvm-reference`) for this — it's specific to TestOps, not open-source Allure
Report.

---

## 13. Reporting — Other Plugins (HTML/JSON/JUnit/Timeline) <a name="13-other-reporting"></a>

Allure isn't the only option, and it's common to run several plugins together in the same array:

```java
plugin = {
    "pretty",                                    // human-readable console output
    "summary",                                   // concise pass/fail summary + snippets for undefined steps
    "html:target/cucumber-reports/report.html",  // static built-in HTML report
    "json:target/cucumber-reports/report.json",  // machine-readable, also feeds Masterthought/other 3rd-party reporters
    "junit:target/cucumber-reports/report.xml",  // JUnit XML — consumed by CI dashboards (Jenkins, GitLab, Azure DevOps)
    "timeline:target/cucumber-reports/timeline",  // visualises parallel THREAD usage over time — invaluable for tuning §9's thread counts
    "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm" // Allure — §12
}
```

### `cucumber.properties` / configuration parameters

Precedence (highest wins): **CLI args > `@CucumberOptions`/`@ConfigurationParameter` > environment variables >
`cucumber.properties`/`junit-platform.properties`**.

```properties
cucumber.filter.tags=@smoke and not @wip
cucumber.glue=com.example.steps,com.example.hooks
cucumber.plugin=pretty, html:target/cucumber-reports/report.html
cucumber.execution.dry-run=false
cucumber.execution.order=random          # lexical | reverse | random | random:[seed]
cucumber.snippet-type=camelcase
cucumber.object-factory=com.example.app.CustomObjectFactory
cucumber.publish.enabled=true            # publishes a shareable report link to reports.cucumber.io
```

### Dry run — CI "spec completeness" gate

```java
@CucumberOptions(dryRun = true)
```

Verifies every step in every feature file has exactly one matching step definition, **without executing any
real logic** — a fast pre-merge check before running the full (slow) Selenium/API suite.

---

## 14. CI Integration <a name="14-ci"></a>

### 14.1 GitHub Actions — run + publish Allure artifacts

```yaml
- name: Run regression suite
  run: mvn -B verify -Pregression

- name: Upload Allure results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: allure-results
    path: target/allure-results
    retention-days: 30

# Generate & publish an HTML report from the results, e.g. with a community action,
# or with `mvn allure:report` followed by an upload-artifact / GitHub Pages deploy step.
```

### 14.2 Flaky-test / retry strategy — `IRetryAnalyzer` (TestNG)

```java
package com.example.listeners;

import org.testng.IRetryAnalyzer;
import org.testng.ITestResult;

public class RetryAnalyzer implements IRetryAnalyzer {
    private int retryCount = 0;
    private static final int MAX_RETRY = 2;

    @Override
    public boolean retry(ITestResult result) {
        if (retryCount < MAX_RETRY) { retryCount++; return true; }
        return false;
    }
}
```

Because Cucumber-TestNG scenarios run through the `scenarios()` `@DataProvider`, there's no `@Test` method to
attach `retryAnalyzer =` to directly — wire the retry listener at the **suite level** via an
`IAnnotationTransformer`:

```java
package com.example.listeners;

import org.testng.IAnnotationTransformer;
import org.testng.annotations.ITestAnnotation;
import java.lang.reflect.Method;
import java.lang.reflect.Constructor;

public class RetryTransformer implements IAnnotationTransformer {
    @Override
    public void transform(ITestAnnotation annotation, Class testClass,
                           Constructor testConstructor, Method testMethod) {
        annotation.setRetryAnalyzer(RetryAnalyzer.class);
    }
}
```

```xml
<suite name="Regression Suite" parallel="methods" thread-count="4" data-provider-thread-count="4">
    <listeners>
        <listener class-name="com.example.listeners.RetryTransformer"/>
    </listeners>
    <test name="Cucumber Scenario-Parallel Run">
        <classes><class name="com.example.runners.ParallelCucumberTestRunner"/></classes>
    </test>
</suite>
```

**Governance note:** blanket retries mask genuine flakiness instead of fixing it. Cap `MAX_RETRY` at 1–2, and
treat any scenario that regularly needs its retry as a `@flaky`-tagged item to triage, not a permanently
green-washed pass.

---

## 15. Best Practices <a name="15-best-practices"></a>

1. **State-specific `Given`/`Then` wording, not implementation wording.** Write
   `Given the account "alice" is locked`, not `Given the "locked" flag is set to true in the accounts table` —
   Gherkin describes business state; the step definition hides *how* that state is achieved.
2. **Never assert against the database directly inside a `Then` step.** Assert what a real user/external
   consumer would observe (UI, API response, queued message). Reserve direct DB access for `Given` setup, not
   `Then` verification.
3. **`Background` hygiene.** If fewer than roughly 80% of scenarios in a file need a given `Background` step,
   it doesn't belong there — move it into the scenarios that need it, or split the file.
4. **Tag hygiene.** Maintain a short, documented tag vocabulary (`@smoke`, `@regression`, `@wip`, `@flaky`,
   `@api`, `@ui`) rather than letting tags proliferate ad hoc.
5. **One `When` per scenario where feasible.** Multiple `When`s usually mean the scenario is a multi-step
   workflow, not one unit of behaviour — sometimes correct for a deliberate end-to-end smoke test, but a smell
   in a feature-level regression scenario.
6. **Step definitions must be reusable, not scenario-specific.** Parameterize (`{string}`, `{int}`, custom
   `ParameterType`s) instead of baking scenario-specific literals into the method/regex.
7. **Keep `Scenario Outline` `Examples` tables focused on genuinely equivalent variations** — rows exercising
   materially different logic paths belong in separate `Scenario`s.
8. **Avoid `Thread.sleep()` in step definitions; use explicit waits** (`WebDriverWait`/`FluentWait`) — this is
   what actually eliminates the flaky scenarios §14.2's retry mechanism exists to paper over.
9. **Don't let `TestContext` become a god object.** Hold only cross-cutting per-scenario state (driver, scenario
   handle, lazily-built Page Objects, API client); domain-specific data used by one or two step classes is often
   better modeled as a small dedicated object passed explicitly.
10. **Version-pin all Cucumber/TestNG/PicoContainer/Allure coordinates explicitly** rather than relying on
    transitive resolution — Cucumber-JVM has had breaking changes across major versions (notably around plugin
    discovery and `TypeRegistryConfigurer` defaults between v6 and v7), and an unpinned transitive upgrade
    silently changing step-matching or DI behaviour in CI is expensive to diagnose after the fact.
11. **Choose your parallel granularity deliberately (§9).** Default to scenario-level parallelism; only drop to
    feature-level when a genuine shared-fixture constraint demands it — fix the underlying thread-safety issue
    via DI (§10) rather than reaching for feature-level runners just because scenario-level failures are harder
    to debug.

---

## 16. Interview Q&A Bank <a name="16-qa"></a>

**Q1. Does the keyword (`Given`/`When`/`Then`) affect step-definition matching?**
No. Cucumber strips the keyword and matches only the remaining text against the pattern. Using the same
sentence with two different keywords produces a **duplicate step**, causing ambiguity.

**Q2. Difference between `Background` and a `@Before` hook?**
`Background` is **visible in the feature file** — readable by non-technical stakeholders, limited to `Given`
steps. `@Before` hooks are **invisible to the spec**, used for low-level plumbing (browser startup, DB cleanup)
a business reader shouldn't need to see.

**Q3. How many `Background` sections can a `Feature` have?**
Exactly one (also one per `Rule`, if used). Different setups for different scenario groups → split via
`Rule`s/Features, or use conditional hooks.

**Q4. `Scenario Outline` vs. `DataTable` — when to use which?**
`Scenario Outline` re-runs the **whole scenario** per row, producing N separately reported results — best for
genuinely distinct test cases. `DataTable` feeds **one step** a block of related rows within a single scenario —
best for bulk setup/assertion data belonging to one test case.

**Q5. What happens when Cucumber finds no matching step definition?**
The step is marked **undefined**, and **all subsequent steps in that scenario are skipped** (not failed) —
Cucumber also prints a code snippet stub for the missing definition.

**Q6. How do you run scenarios in parallel with TestNG?**
Extend `AbstractTestNGCucumberTests`, override the inherited `scenarios()` `@DataProvider` method with
`@DataProvider(parallel = true)`, call `super.scenarios()`, and set `data-provider-thread-count` in `testng.xml`
(or `dataproviderthreadcount` in Surefire's `<properties>`). Default thread count if unset is **10**.

**Q7. How do you run scenarios in parallel with the JUnit Platform runner?**
Set `cucumber.execution.parallel.enabled=true` in `junit-platform.properties` (or via
`@ConfigurationParameter`), and choose a strategy: `dynamic` (default, `<cores> * factor`) or `fixed` (explicit
`parallelism`/`max-pool-size`). It's **off by default** and, once enabled, parallelizes both scenarios and
Scenario Outline rows unless restricted to feature-level via
`cucumber.execution.execution-mode.feature=same_thread`.

**Q8. Why does JUnit 4's Cucumber runner behave differently from JUnit Platform/TestNG under parallelism?**
JUnit 4's `cucumber-junit` can only parallelize at the **feature** level — all scenarios inside one `.feature`
file always run on the same thread. Scenario-level parallelism requires either TestNG's DataProvider model or
the JUnit Platform engine's native parallel execution.

**Q9. Why is PicoContainer the recommended default DI module?**
It requires **zero framework code** in the application — purely constructor injection based on the types
Cucumber's glue classes declare, creating exactly one shared object graph per scenario. This gives shared Page
Objects across step-definition classes without state leaking between scenarios, and composes safely with
parallel execution because each scenario/thread gets an independent graph.

**Q10. How do hooks interact with `Background`?**
Execution order per scenario: `@Before` hooks → `Background` steps → scenario's own steps (each wrapped by
`@BeforeStep`/`@AfterStep`) → `@After` hooks (in reverse `order`).

**Q11. Why should `Then` steps avoid asserting directly against the database?**
`Then` should assert an **observable** outcome (UI, API response, message) — what a real user/external system
perceives. Asserting a raw DB row couples the spec to an implementation detail that could change without the
observable behaviour changing, and vice versa.

**Q12. What is the Allure plugin class for Cucumber-JVM 7, and does it differ per runner?**
`io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm` — the **same class name regardless of whether you run under
JUnit Platform, TestNG, or JUnit 4**; only the surrounding adapter dependency
(`allure-junit-platform`/`allure-testng`/`allure-junit4`) differs per runner.

**Q13. Where does Allure look for test results by default, and how do you change it?**
The project root, by default — override it by placing an `allure.properties` file in `src/test/resources` with
`allure.results.directory=target/allure-results` (Maven) or `build/allure-results` (Gradle).

**Q14. Does Allure need any special configuration to work correctly under parallel Cucumber execution?**
No — each scenario's results are written to its own UUID-named result file, so Allure's Cucumber-JVM integration
is inherently safe under TestNG or JUnit Platform parallelism (§9). The one caveat is CI sharding: merge every
shard's `allure-results` directory before generating a single combined report.

**Q15. What's the risk of running Selenium tests in parallel without DI or ThreadLocal for the WebDriver?**
A `static` shared `WebDriver` field becomes a race condition — multiple threads drive the same browser session
simultaneously, causing `StaleElementReferenceException`s, cross-scenario navigation interference, and
non-deterministic failures. Fix with PicoContainer-scoped instances (recommended, one graph per scenario) or
explicit `ThreadLocal<WebDriver>`.

**Q16. `cucumber.filter.tags` vs `cucumber.filter.name` — how do they combine?**
With a logical **AND** — a scenario must satisfy both the tag expression and the name regex to be selected.

**Q17. How do you make AspectJ work with Allure's `@Step`/`@Attachment` annotations?**
Add `-javaagent:.../aspectjweaver-<version>.jar` to the Surefire `argLine` (Maven) or configure a `javaagent` JVM
arg on the `test` task (Gradle), with `org.aspectj:aspectjweaver` on the classpath — without this, `@Step`
sub-step breakdowns in the Allure report silently don't appear.

**Q18. Is `Scenario` a synonym of `Example`, or the reverse?**
They're true synonyms — `Example` was introduced later to better reflect BDD's "specification by example"
philosophy, but both keywords are fully interchangeable and behave identically.

---

## 17. Cheat Sheet <a name="17-cheat-sheet"></a>

```gherkin
# Full syntax skeleton
@feature-tag
Feature: <name>
  <optional free-text description>

  Background:
    Given <shared precondition>

  Rule: <business rule name>

    @scenario-tag
    Example: <name>              # a.k.a. Scenario
      Given <context>
      And <more context>
      When <action>
      Then <observable outcome>
      But <not this other outcome>

    Scenario Outline: <name>
      Given <context with <param>>
      When <action>
      Then <outcome with <param>>

      @tag-a
      Examples:
        | param |
        | val1  |
```

| Concept | Annotation / Syntax |
|---|---|
| Cucumber Expression step | `@Given("{int} cucumbers")` |
| Regex step | `@Given("^(\\d+) cucumbers$")` |
| Before scenario | `@Before` / `@Before(order = n)` |
| After scenario | `@After` (reverse order) |
| Before/after each step | `@BeforeStep` / `@AfterStep` |
| Once per whole JVM | `@BeforeAll` / `@AfterAll` (must be `static`) |
| Conditional hook | `@After("@tag and not @tag2")` |
| JUnit 4 runner | `@RunWith(Cucumber.class)` + `@CucumberOptions` |
| JUnit 5 runner | `@Suite` + `@IncludeEngines("cucumber")` + `@ConfigurationParameter`, or `@Cucumber` |
| TestNG runner | `extends AbstractTestNGCucumberTests` |
| TestNG parallel (by scenario) | `@Override @DataProvider(parallel = true) public Object[][] scenarios()` + `data-provider-thread-count` |
| TestNG parallel (by feature) | One runner per feature, `testng.xml` `parallel="tests"` |
| JUnit Platform parallel | `cucumber.execution.parallel.enabled=true` + `.strategy=fixed\|dynamic` in `junit-platform.properties` |
| JUnit Platform: force feature-level | `cucumber.execution.execution-mode.feature=same_thread` |
| CLI parallel | `io.cucumber.core.cli.Main --threads N` |
| DI (recommended default) | `cucumber-picocontainer` — constructor injection, one graph per scenario |
| Dry run (CI gate) | `@CucumberOptions(dryRun = true)` / `cucumber.execution.dry-run=true` |
| Allure plugin (Cucumber-JVM 7) | `io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm` in `plugin = {...}` |
| Allure results directory | `allure.properties` → `allure.results.directory=target/allure-results` |
| Generate Allure HTML report | `mvn allure:report` (static) / `mvn allure:serve` (dev, opens browser) |
| HTML/JSON/JUnit report | `plugin = {"html:...", "json:...", "junit:..."}` |
| Timeline (thread visualisation) | `plugin = {"timeline:target/cucumber-reports/timeline"}` |

---

*Guide compiled and validated against the official Cucumber documentation at cucumber.io/docs (Gherkin
Reference, Cucumber Reference/API, Parallel Execution guide, `cucumber-junit-platform-engine` README) and the
official Allure Report documentation at allurereport.org/docs/cucumberjvm (Getting Started, Configuration,
Reference pages), as of July 2026. Java examples target `cucumber-java`/`cucumber-picocontainer`/
`cucumber-testng`/`cucumber-junit-platform-engine` 7.20.x, `testng` 7.10.x, and `allure-cucumber7-jvm` 2.35.x on
OpenJDK 17/21. Where an upstream source flagged a known limitation or gotcha (e.g. JUnit 4's feature-only
parallelism, `fixed.max-pool-size`'s non-strict guarantee, TestNG dependency ordering with `allure-testng`),
that caveat is called out explicitly above rather than smoothed over.*
