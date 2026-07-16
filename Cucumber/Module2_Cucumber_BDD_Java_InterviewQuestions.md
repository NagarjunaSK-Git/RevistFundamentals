# SDET Interview Q&A Bank — Cucumber BDD with Java (Real-Project Aligned)

> 61 questions with concise model answers and code snippets, consistent with a
> `cucumber-java` / `cucumber-junit-platform-engine` / `cucumber-testng` / `cucumber-picocontainer` **7.20.x**,
> `testng` **7.10.x**, `allure-cucumber7-jvm` **2.35.x** stack on OpenJDK 17/21.

---

## 1. Gherkin & BDD Fundamentals

**Q1. What is Cucumber, and is it a testing framework itself?**
Cucumber is a BDD **automation tool**, not an assertion library. It parses Gherkin `.feature` files into
Pickles, matches each step's text to glue code, executes that code (which typically delegates to
JUnit/TestNG/AssertJ assertions or Selenium/REST-assured calls), and reports the result. The actual pass/fail
logic always lives in the step definition, not in Cucumber.

**Q2. What are the three purposes of a `.feature` file?**
It is simultaneously an unambiguous executable specification, an automated test, and living documentation of
system behaviour that stays versioned alongside the code it describes. This is the core BDD value proposition —
one artifact serves business, QA, and engineering.

**Q3. What's the difference between `Feature`, `Rule`, and `Scenario`/`Example`?**
`Feature` is the single top-level container per file. `Rule` (Gherkin 6+) groups scenarios that illustrate one
business rule and is optional. `Scenario`/`Example` are true synonyms for one concrete, executable example —
`Example` was introduced later to align with "specification by example" phrasing.

```gherkin
Feature: User Login

  Rule: Account is locked after 3 consecutive failed attempts

    Example: Third consecutive wrong password locks the account
      Given the user "alice" has failed to log in 2 times already
      When "alice" attempts to log in with an incorrect password
      Then the account "alice" should be locked
```

**Q4. Does the `Given`/`When`/`Then` keyword affect step matching?**
No. Cucumber strips the keyword and matches only the remaining text against the registered pattern. Reusing
identical step text under two different keywords produces a **duplicate/ambiguous step**:

```gherkin
Given there is money in my account
Then there is money in my account        # duplicate — identical match text, ambiguous
```

Fix by making the wording state-specific (`Given my account has a balance of £430` / `Then my account should
have a balance of £430`).

**Q5. `Background` vs a `@Before` hook — when do you use which?**
`Background` is **visible in the feature file**, limited to `Given` steps, and readable by non-technical
stakeholders — use it for business-meaningful shared context (e.g., "the catalog contains these products").
`@Before` hooks are invisible to the spec and are for low-level plumbing like browser startup or DB cleanup that
a business reader shouldn't need to see.

**Q6. How many `Background` sections can one `Feature` have, and what's the hygiene rule?**
Exactly one per `Feature` (and one per `Rule`, if used). Keep it to 3–5 steps and only for state genuinely
needed by every scenario in the file — if less than roughly 80% of scenarios need a given `Background` step, it
doesn't belong there; split the file or move it into the scenarios that need it.

---

## 2. Step Definition Mapping

**Q7. Cucumber Expressions vs Regex — when do you pick each?**
Cucumber Expressions are the default/recommended style: readable, typed (`{int}`, `{string}`, `{word}`,
`{float}`), and IDE-friendly via the official plugin. Reach for regex only when you need alternation or
lookarounds Expressions can't express, e.g. `^I have (?:a|an) (.*)$`.

```java
@Given("{int} cucumbers are in the basket")
public void cucumbersInBasket(int count) { /* ... */ }

@Given("^(\\d+) cucumbers are in the basket$")   // regex form, equivalent
public void cucumbersInBasketRegex(int count) { /* ... */ }
```

**Q8. What happens when two step definitions match the same Gherkin step?**
Cucumber throws an **ambiguous step definitions** error at runtime for that Pickle and fails the scenario before
any step logic executes — it does not guess or silently pick one. Fix by tightening one pattern's specificity
(anchoring regex with `^...$`) or removing the overlapping definition.

**Q9. What's the parameter-count rule for a step definition method?**
The method's parameter count must exactly equal the number of capture groups/placeholders in the pattern. A
trailing `DocString` or `DataTable` adds exactly **one** extra parameter. A mismatch throws an error at
**glue-loading time**, before any scenario runs — a fast-fail rather than a mid-suite surprise.

**Q10. How do you create a custom `@ParameterType`?**
Register a regex-backed transformer method annotated `@ParameterType`; Cucumber then lets you reference it by
name in Expression steps, validating and typing the value before your step method even runs.

```java
@ParameterType("small|medium|large")
public String size(String raw) { return raw; }

@Given("the user selects a {size} coffee")
public void selectSize(String size) { /* already validated */ }
```

**Q11. How do you map a `DataTable` to a POJO with `@DataTableType`?**
Define a transformer method that converts one row (`Map<String,String>`) into your domain object; Cucumber then
auto-converts a `List<Product>` parameter without any manual parsing in the step body. See §3 for the full
data-driven example.

**Q12. Why should step definitions avoid scenario-specific literals baked into the method/regex?**
Hard-coded literals make a step non-reusable across scenarios, forcing near-duplicate step definitions.
Parameterizing with `{string}`, `{int}`, or a custom `ParameterType` keeps one method reusable across dozens of
Gherkin lines, which is central to keeping the glue-code layer maintainable as a suite grows.

---

## 3. Data-Driven Testing

**Q13. `Scenario Outline` + `Examples` vs `DataTable` — when do you use which?**
`Scenario Outline` re-runs the **whole scenario** once per row, and each row is reported as an independent,
separately-passable/failable scenario — use it for genuinely distinct test cases. `DataTable` feeds **one step**
a block of related rows within a single scenario — use it for bulk setup/assertion data belonging to one test
case (e.g., seeding a catalog).

**Q14. Data-driven mapping example: `Scenario Outline` with typed substitution.**

```gherkin
Scenario Outline: Applying a discount code reduces the total correctly
  Given "carol" has "SKU100" in the cart with quantity <qty>
  When "carol" applies the discount code "<code>"
  Then the cart total should be "<expectedTotal>"

  Examples: Percentage-based codes
    | qty | code   | expectedTotal |
    | 1   | SAVE10 | 22.50         |
    | 2   | SAVE10 | 45.00         |
```

Each row is matched to typed step parameters (`int`, `String`) exactly as a single `Scenario` would be — no
special-casing needed in the glue code.

**Q15. Data-driven mapping example: `DataTable` → `List<Map<String,String>>` and `DataTable` → POJO.**

```java
// Raw form — no custom type needed
@Given("the catalog contains the following products:")
public void catalog(io.cucumber.datatable.DataTable table) {
    List<Map<String, String>> rows = table.asMaps();
}
```

```java
// POJO form — cleaner step bodies via @DataTableType
public class Product {
    public String sku;
    public String name;
    public double price;
}

public class TypeRegistryConfig {
    @DataTableType
    public Product productEntry(Map<String, String> entry) {
        Product p = new Product();
        p.sku = entry.get("sku");
        p.name = entry.get("name");
        p.price = Double.parseDouble(entry.get("price"));
        return p;
    }
}

@Given("the catalog contains the following products:")
public void catalog(List<Product> products) { /* auto-mapped, fully typed */ }
```

```gherkin
Given the catalog contains the following products:
  | sku    | name           | price |
  | SKU100 | Wireless Mouse | 25.00 |
  | SKU200 | USB-C Cable    | 9.50  |
```

**Q16. Data-driven mapping example: loading data from an external CSV/JSON file.**
For large or environment-specific data sets, load the file inside a `@Before` hook or a step-definition helper
instead of inflating the `.feature` file — keep Gherkin readable and treat bulky fixtures as an implementation
detail.

```java
@Given("the discount codes from {string} are loaded")
public void loadDiscountCodes(String csvFileName) {
    try (Reader reader = Files.newBufferedReader(Paths.get("src/test/resources/data", csvFileName))) {
        List<DiscountCode> codes = new CsvToBeanBuilder<DiscountCode>(reader)
                .withType(DiscountCode.class)
                .build()
                .parse();
        testContext.setDiscountCodes(codes);
    } catch (IOException e) {
        throw new RuntimeException("Failed to load discount codes from " + csvFileName, e);
    }
}
```

```gherkin
Given the discount codes from "discount_codes.csv" are loaded
```

**Q17. What's the parallel-execution implication of choosing `Scenario Outline` over `DataTable`?**
Each `Scenario Outline` row is its own independent Pickle and therefore its own unit of parallel work under both
the TestNG DataProvider model and the JUnit Platform engine. A `DataTable`'s rows are not independent units —
they're part of one scenario's single Pickle, so they can never be split across threads.

---

## 4. Hooks & Lifecycle

**Q18. List the hook types and their scope.**
`@BeforeAll`/`@AfterAll` run once per JVM and must be `static` (no DI/instance state available). `@Before`/
`@After` run once per scenario. `@BeforeStep`/`@AfterStep` run once per step, wrapping every step regardless of
whether it came from `Background` or the scenario body.

**Q19. What is the precise execution order for a scenario tagged `@api`?**

```
@BeforeAll (once per JVM, static)
  -> @Before hooks, ASCENDING order (unspecified order defaults to 10000, i.e. runs last)
       -> conditional @Before(value="@api") runs here because the scenario carries @api
  -> Background steps (top to bottom), each wrapped by @BeforeStep/@AfterStep
  -> Scenario's own steps, each wrapped by @BeforeStep -> [step] -> @AfterStep
  -> @After hooks, DESCENDING order (reverse of @Before — last resource acquired is first torn down)
@AfterAll (once per JVM, static, after the last scenario)
```

**Q20. How do you write a conditional (tagged) hook?**

```java
@Before("@smoke and not @flaky")
public void setupForStableSmoke() { /* ... */ }

@After("@checkout or @payment")
public void captureNetworkHar(Scenario scenario) { /* ... */ }
```

Tag expressions in hook annotations use the exact same grammar as `@CucumberOptions(tags = ...)` /
`cucumber.filter.tags`.

**Q21. Why does `@After(order = 1)` run before `@After(order = 0)`?**
`@After` hooks execute in **descending** order — the reverse of `@Before`'s ascending order — so teardown mirrors
a stack: the last resource acquired (typically the highest-`order` `@Before`) is the first one torn down. In the
project's `Hooks` class, `tearDownApiClient` (`order = 1`) runs before `quitDriver` (`order = 0`).

**Q22. Why should you avoid `@ClassRule`/`@BeforeClass`/`@AfterClass` with the JUnit 4 runner?**
Cucumber supports them but the docs recommend against them because they hurt portability across CLI/IDE test
runners. Prefer Cucumber's own `@BeforeAll`/`@AfterAll` hooks, which behave consistently regardless of how the
suite is invoked.

**Q23. What's a real screenshot-on-failure `@AfterStep` hook look like?**

```java
@AfterStep
public void screenshotOnFailure(Scenario scenario) {
    if (scenario.isFailed() && testContext.getDriver() != null) {
        byte[] shot = ((TakesScreenshot) testContext.getDriver())
                .getScreenshotAs(OutputType.BYTES);
        scenario.attach(shot, "image/png", scenario.getName());
    }
}
```

`@AfterStep` (not `@After`) is used here because it needs to fire after *each* step to catch the exact step that
failed, not just once at the end of the scenario.

---

## 5. Tags & Tag Expressions

**Q24. What operators does a tag expression support?**
`and`, `or`, `not`, and parenthetical grouping — e.g. `(@smoke or @regression) and not @flaky`. These combine
with standard boolean precedence rules, and grouping is required whenever OR and AND are mixed.

**Q25. Where can tags legally be placed in Gherkin?**
Only above `Feature`, `Rule`, `Scenario`/`Example`, `Scenario Outline`, and `Examples`. Tags **cannot** be placed
above an individual step or a `Background`. A tag above `Feature`/`Rule` is inherited by every nested scenario.

**Q26. How do `cucumber.filter.tags` and `cucumber.filter.name` combine?**
With logical **AND** — a scenario must satisfy both the tag expression and the name regex to be selected. This
is easy to get wrong in interviews: people often assume OR.

**Q27. How would you exclude flaky and work-in-progress scenarios from a CI regression run?**

```bash
mvn test -Dcucumber.filter.tags="@regression and not (@flaky or @wip)"
```

```java
@CucumberOptions(tags = "@regression and not (@flaky or @wip)")
```

---

## 6. Dependency Injection & State Management

**Q28. Why is PicoContainer the recommended default DI module?**
It requires zero framework code in the application — pure constructor injection based on the types glue classes
declare — and creates exactly one shared object graph per scenario, with no annotations required (unlike
Spring's `@Autowired`). This gives shared Page Objects across step-definition classes without state leaking
between scenarios, and it composes safely with parallel execution since each thread gets an independent graph.

**Q29. What does the `TestContext` ("World object") pattern look like?**

```java
public class TestContext {
    // NOTHING here is static — PicoContainer creates ONE instance per scenario
    // and injects that SAME instance into every glue class constructor that asks for it.
    private WebDriver driver;
    private Scenario scenario;

    public WebDriver getDriver() { return driver; }
    public void setDriver(WebDriver driver) { this.driver = driver; }
}

public class LoginSteps {
    private final TestContext testContext;
    public LoginSteps(TestContext testContext) { this.testContext = testContext; }  // injected

    @Given("the user navigates to the login page")
    public void navigateToLogin() {
        testContext.getDriver().get("https://app.example.com/login");
    }
}
```

**Q30. Why is a `static WebDriver` field dangerous, and what's the fix?**
A `static` field is shared across every concurrently-running scenario under parallel execution, so multiple
threads drive the same browser session simultaneously — causing `StaleElementReferenceException`s and
cross-scenario navigation interference. Fix with a DI-scoped instance created fresh per scenario (preferred,
via PicoContainer/`TestContext`) or an explicit `ThreadLocal<WebDriver>`.

```java
// WRONG
private static WebDriver driver;   // one instance shared by every thread

// RIGHT (if not using DI)
private static final ThreadLocal<WebDriver> DRIVER = ThreadLocal.withInitial(ChromeDriver::new);
```

**Q31. What is the single rule that governs state management across scenarios?**
Never share state between scenarios — no static/global mutable fields — because scenarios must be able to run
independently, in any order, including in parallel. State sharing **between steps within one scenario** is fine
via DI or instance fields on the glue class, since Cucumber gives every scenario a fresh object graph anyway.

**Q32. How does DI compose with Guice or Spring if the app under test already uses one of them?**
Implement `io.cucumber.core.backend.ObjectFactory` to delegate Cucumber's injector into the app's own DI
container (e.g. `CucumberModules.createScenarioModule()` for Guice), then register it via SPI or
`@CucumberOptions(objectFactory = ...)` / the `cucumber.object-factory` property. This avoids running two
unrelated DI containers side by side.

---

## 7. Parallel Execution

**Q33. How do you enable scenario-level parallelism with TestNG?**

```java
public class ParallelCucumberTestRunner extends AbstractTestNGCucumberTests {
    @Override
    @DataProvider(parallel = true)
    public Object[][] scenarios() {
        return super.scenarios();
    }
}
```

```xml
<suite name="ParallelSuite" parallel="methods" data-provider-thread-count="4">
    <test name="RegressionParallel">
        <classes><class name="com.example.runners.ParallelCucumberTestRunner"/></classes>
    </test>
</suite>
```

Both the `@DataProvider(parallel = true)` override **and** `data-provider-thread-count` (or Surefire's
`dataproviderthreadcount` property) are required together — the annotation alone is necessary but not
sufficient. If no thread count is specified anywhere, it defaults to **10**.

**Q34. Scenario-level vs feature-level parallelism — when do you pick feature-level?**
Scenario-level (§9.1 of the guide) treats every scenario, and every `Scenario Outline` row, as an independent
parallel unit — the default choice for stateless, fully-isolated scenarios. Feature-level parallelism (one thin
runner per feature, `parallel="tests"`) is reserved for features that share an expensive/exclusive fixture
internally but are safe to run alongside *other* features concurrently.

**Q35. Why does JUnit 4's Cucumber runner behave differently from TestNG/JUnit Platform under parallelism?**
`cucumber-junit` can only parallelize at the **feature** level — every scenario inside one `.feature` file
always runs on the same thread. Scenario-level parallelism requires TestNG's DataProvider model or the JUnit
Platform engine's native parallel execution.

**Q36. How do you enable and tune parallelism under the JUnit Platform engine?**

```properties
# junit-platform.properties
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=fixed
cucumber.execution.parallel.config.fixed.parallelism=4
cucumber.execution.parallel.config.fixed.max-pool-size=4
```

It's off by default. `dynamic` (default strategy once enabled) computes parallelism as `<cores> * factor`
(factor defaults to `1`); `fixed` lets you set explicit parallelism and pool-size ceilings. Restrict to
feature-level granularity with `cucumber.execution.execution-mode.feature=same_thread`.

**Q37. What does `AbstractTestNGCucumberTests` actually do under the hood?**
It exposes every Gherkin Pickle (a scenario, or one `Scenario Outline` row) as a row of a TestNG
`@DataProvider`-backed method called `scenarios()`. Left un-overridden, TestNG runs those rows sequentially on
one thread; overriding it with `@DataProvider(parallel = true)` is what turns each row into an independently
schedulable unit of work.

---

## 8. Selenium & Page Object Model

**Q38. Explicit vs implicit waits — which do you use, and why?**
Explicit waits (`WebDriverWait`/`FluentWait` polling for a specific condition) are strongly preferred over
implicit waits, which apply a blanket polling timeout to every `findElement` call and can mask real timing bugs
or slow down every lookup uniformly. `Thread.sleep()` should never appear in step definitions — it's the
single most common source of flaky, slow-running suites.

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
wait.until(ExpectedConditions.elementToBeClickable(By.id("submit-btn"))).click();
```

**Q39. What makes a locator "stable" versus fragile?**
Stable locators target attributes the application team controls for testing (`data-testid`, `id`, `name`) rather
than brittle structural paths (deep XPath chains, nth-child CSS) or generated/auto-incrementing class names that
change with every UI refactor. Stable locators are the single biggest lever for reducing UI-suite maintenance
cost.

**Q40. Sketch a minimal Page Object for a login page.**

```java
public class LoginPage {
    private final WebDriver driver;
    private final By usernameField = By.id("username");
    private final By passwordField = By.id("password");
    private final By submitButton  = By.id("submit-btn");

    public LoginPage(WebDriver driver) { this.driver = driver; }

    public DashboardPage loginAs(String username, String password) {
        driver.findElement(usernameField).sendKeys(username);
        driver.findElement(passwordField).sendKeys(password);
        driver.findElement(submitButton).click();
        return new DashboardPage(driver);
    }
}
```

Locators and low-level interactions live entirely inside the Page Object; step definitions only call
business-meaningful methods (`loginAs(...)`), which is what keeps Gherkin steps readable and glue code thin.

**Q41. What are the most common root causes of flaky UI tests, beyond `Thread.sleep()`?**
Shared mutable state across parallel threads (a `static WebDriver`), asserting against a UI state before an
async operation (AJAX call, animation) has actually completed, test-order dependency (a scenario relying on
leftover data from a previous one), and locators tied to volatile DOM structure rather than stable attributes.

**Q42. How do Page Objects interact with the DI/`TestContext` pattern from §6?**
Page Objects are typically lazily constructed and cached inside the `TestContext`, taking the scenario-scoped
`WebDriver` in their constructor. Because `TestContext` is instantiated fresh per scenario by PicoContainer,
each Page Object instance is automatically scoped correctly under parallel execution with no manual locking.

---

## 9. Reporting

**Q43. How do you wire ExtentReports (or a similar reporter) alongside Cucumber?**
Implement a Cucumber `EventListener`/plugin (or use the community `cucumber-extentsreport` adapter) that
subscribes to `TestStepFinished`/`TestCaseFinished` events, then map each event to an ExtentTest node so pass/
fail status, logs, and attachments line up per scenario. It's typically registered in the `plugin = {...}` array
alongside `pretty`/`html`/`json`, the same way Allure's plugin class is registered.

**Q44. What's the standard pattern for attaching a screenshot to a report on failure?**

```java
@AfterStep
public void screenshotOnFailure(Scenario scenario) {
    if (scenario.isFailed()) {
        byte[] shot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
        scenario.attach(shot, "image/png", scenario.getName());       // Cucumber's own attach()
        // Allure equivalent: Allure.getLifecycle().addAttachment("Screenshot", "image/png", "png", shot);
    }
}
```

Cucumber's `scenario.attach(...)` surfaces into most report plugins (Allure, ExtentReports adapters) without
needing a separate attachment call per reporter.

**Q45. How do you keep report history across CI runs instead of losing it on every job?**
Persist the raw results directory (`allure-results`, or the reporter's equivalent) as a CI artifact with a
retention policy, and — for Allure specifically — merge the previous run's history folder back into the new
`allure-results` before generating the report, so trend/retry graphs carry forward instead of resetting every
build.

```yaml
- name: Upload Allure results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: allure-results
    path: target/allure-results
    retention-days: 30
```

**Q46. Does report generation need special handling when tests run in parallel across CI shards?**
Yes — each shard produces its own results directory (Allure writes each scenario's result to a UUID-named file,
so no cross-shard collision occurs), but you must merge every shard's results directory into one location before
running the report-generation step, or you'll end up with N separate reports instead of one combined view.

---

## 10. Design Patterns & Code Quality

**Q47. What design pattern underlies the Page Object Model, and why does it matter for SDETs?**
POM is an application of the **Facade** pattern: it hides low-level driver/locator interactions behind a
business-readable API. This matters because it isolates UI-structure churn to one class per page, so a locator
change requires editing one Page Object instead of every step definition that touches that page.

**Q48. How would you apply the Factory pattern to `WebDriver` creation?**

```java
public class DriverFactory {
    public static WebDriver create(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome"  -> new ChromeDriver(new ChromeOptions());
            case "firefox" -> new FirefoxDriver(new FirefoxOptions());
            case "edge"    -> new EdgeDriver(new EdgeOptions());
            default -> throw new IllegalArgumentException("Unsupported browser: " + browser);
        };
    }
}
```

A factory centralizes browser-specific setup (capabilities, headless flags, remote grid URLs) behind one method,
so hooks and `TestContext` never need to know which concrete `WebDriver` implementation is in use.

**Q49. How do SOLID principles show up specifically in step-definition design?**
Single Responsibility: one step-definition class per business area (`LoginSteps`, `CartSteps`), not one giant
class. Open/Closed: new scenarios extend behaviour via new steps/Page Object methods rather than editing
existing step logic. Dependency Inversion: step classes depend on `TestContext`/Page Object abstractions injected
by the DI container, not on concretely `new`-ing up a `WebDriver` themselves.

**Q50. Why is explicit version-pinning of Cucumber/TestNG/PicoContainer/Allure coordinates a code-quality issue, not just a build detail?**
Cucumber-JVM has had breaking changes across major versions (e.g. plugin discovery and `TypeRegistryConfigurer`
defaults changing between v6 and v7). An unpinned transitive dependency silently upgrading in CI can change
step-matching or DI behaviour without any code change on your side — an expensive class of failure to diagnose,
since the symptom shows up in test results, not a compile error.

---

## 11. Performance / Non-Functional Awareness for SDETs

**Q51. What's an SDET's role regarding non-functional requirements, at a conceptual level?**
SDETs typically don't own deep performance-engineering work (that's usually a dedicated perf team), but they're
expected to recognize when a functional test is masking a latency, throughput, or resource-leak issue — e.g. a
`Then` step that times out only under load — and to route that signal to the right owner rather than just
retrying the scenario.

**Q52. How does parallel test execution itself relate to non-functional thinking?**
Choosing scenario- vs feature-level parallelism (§7) is itself a performance/resource trade-off: more threads
reduce wall-clock suite time but increase contention on shared infrastructure (DB connections, browser grid
capacity, rate-limited APIs) — an SDET is expected to reason about that ceiling, not just maximize thread count.

**Q53. What's a lightweight way to catch an API response-time regression inside a functional BDD suite?**
Assert a response-time budget directly in the `Then` step alongside the functional assertion, using the
timestamp already captured by REST-assured/HTTP client — not a substitute for dedicated load testing, but cheap
early-warning coverage.

```java
@Then("the API should respond within {int} ms")
public void assertResponseTime(int maxMillis) {
    assertThat(testContext.getLastResponseTime()).isLessThanOrEqualTo(maxMillis);
}
```

---

## 12. Tricky Scenario-Based Questions

**Q54. Two step definitions match the same Gherkin step text. What happens, and how do you fix it?**
Cucumber throws an ambiguous-step-definitions error at that Pickle's execution and fails the scenario before any
step logic runs. Diagnose by checking whether one pattern is an unintentionally broader regex/Expression that
overlaps another (a common cause is an un-anchored regex like `I have (.*) items` matching text meant for a more
specific step). Fix by anchoring the regex (`^...$`), narrowing the Expression's parameter type, or merging the
two into one parameterized method.

**Q55. A scenario passes locally (sequential run) but fails only in parallel CI runs. How do you debug it?**
First suspect shared mutable state: search for `static` fields (drivers, singletons, counters) in glue classes
or utility classes — these are safe under sequential execution but become race conditions once N scenarios run
concurrently. Reproduce locally by running the same TestNG/JUnit Platform parallel configuration (same thread
count) instead of the default sequential run, then confirm the fix by wrapping the offending state in the
`TestContext`/DI pattern (§6) or a `ThreadLocal`.

**Q56. A `Scenario Outline` row intermittently fails, but only the second `Examples:` block, only under load.**
This pattern suggests the two `Examples:` blocks aren't as independent as they look — check whether they share
a code (`SAVE10` reused with different expected totals) or hit a resource with a fixed capacity (a discount-code
usage counter, a rate-limited endpoint) that the first block's rows are exhausting before the second block runs
under parallel scheduling. The fix is usually to make each row use uniquely-scoped test data rather than
depend on interaction ordering.

**Q57. A step is marked "undefined" but you're certain you wrote the step definition. What are the likely causes?**
Most commonly: the glue package isn't included in `glue`/`cucumber.glue` (or the runner's classpath scan), the
step text has a subtle whitespace/typo mismatch, or the class containing the step definition lacks a no-arg
constructor compatible with the DI module in use (PicoContainer needs a constructor whose parameters it can all
resolve). Confirm by running with `dryRun = true`, which surfaces all matching failures without executing real
logic.

**Q58. Your suite's Allure report shows only one shard's results after a sharded CI run. Why, and how do you fix it?**
Each CI shard writes to its own `allure-results` directory in isolation; if `allure:report`/`allure generate`
runs against only one shard's directory (or the shards were never merged), you get a partial report. Fix by
downloading every shard's `allure-results` artifact, merging them into one directory (concatenating, not
overwriting, since each scenario's result file is uniquely UUID-named), and generating the report from the
merged directory.

**Q59. A `@Before` hook that provisions a REST client only runs for some scenarios in the suite. Is that a bug?**
Not necessarily — check whether the hook is conditional (`@Before("@api")`). If so, this is expected: it only
fires for scenarios carrying the `@api` tag, by design, so non-API scenarios don't pay the cost of provisioning
a client they never use. Confirm the tag is present on the failing scenario before treating it as a defect.

**Q60. A retry-enabled TestNG suite keeps "passing" a scenario that's actually broken. What's the risk, and how do you govern it?**
Blanket retries (`IRetryAnalyzer` + `IAnnotationTransformer`, since Cucumber-TestNG scenarios run through a
`@DataProvider` rather than individual `@Test` methods) mask genuine flakiness rather than fixing it. Cap
`MAX_RETRY` at 1–2, and treat any scenario that regularly needs its retry as a `@flaky`-tagged item routed to
triage — not something silently green-washed forever by the retry mechanism.

**Q61. A scenario needs to assert something only a database check can confirm — is asserting against the DB in `Then` ever acceptable?**
Prefer asserting an **observable** outcome (UI, API response, queued message) in `Then`, since that's what a
real consumer perceives and it decouples the spec from implementation details that could change independently
of behaviour. If no observable surface exists yet (e.g. an async batch job with no callback/API), a DB check is
a pragmatic stopgap — but it should be flagged as technical debt and reserved for `Given` setup wherever
possible, not treated as the long-term pattern for `Then`.

