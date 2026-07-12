# MODULE 8 — Page Object Model & Framework Design
### Level: Advanced (Automation Engineer → Senior Automation Engineer → Test Architect)

---

## 1. Skills Covered

- Designing maintainable Page Object Model (POM) hierarchies for large test suites
- Implementing `PageFactory`, `@FindBy`, `@CacheLookup`, and stale-element-resistant locator patterns
- Building Component Object Model (COM) abstractions for reusable UI widgets
- **Architecting a production-grade `DriverFactory` backed by `ThreadLocal<WebDriver>`** for safe parallel execution across Chrome/Firefox/Edge, local and remote
- **Designing a full configuration-management layer**: properties files, YAML, environment variables, system-property overrides, and precedence rules
- **Building a constants layer and a utilities layer** (wait helpers, retry helpers, JS-executor helpers, screenshot helpers) shared across the framework
- **Designing a test-data-management strategy**: POJO-based data models, JSON/CSV-driven data providers, builder patterns, and JUnit 5 `@ParameterizedTest` integration
- **Implementing enterprise logging**: SLF4J + Logback, MDC-based thread correlation for parallel runs, per-test log files
- **Integrating reporting frameworks** (Allure and ExtentReports) and screenshot-on-failure capture via JUnit 5 extensions
- **Configuring JUnit 5 parallel execution** correctly (parallel modes, resource locks, thread-count tuning) so it is safe with a `ThreadLocal` driver
- **Architecting and operating Selenium Grid 4** (Router/Distributor/SessionMap/Node) for distributed execution
- **Running Dockerized Selenium** (standalone and hub-node topologies, video recording, resource sizing)
- **Integrating cloud execution grids** (BrowserStack/Sauce Labs/LambdaTest-style vendors) behind the same `DriverFactory` abstraction

---

## 2. Learning Objectives

By the end of this chapter, the learner will be able to:

1. **Design** a layered POM architecture (Base → Component → Page → Test) that isolates locator changes from test logic.
2. **Implement** three distinct page object strategies and justify which to use for a given team/project size.
3. **Diagnose** the root causes of `StaleElementReferenceException` and apply multiple mitigation strategies.
4. **Build** a complete, thread-safe framework core: `DriverFactory`, `ConfigManager`, `Constants`, a utilities package, and a test-data-management layer — all supporting parallel JUnit 5 execution across Chrome, Firefox, and Edge, locally and on Selenium Grid 4.
5. **Configure logging** with SLF4J/Logback so that concurrent parallel test threads produce clearly attributable, non-interleaved log output.
6. **Integrate** Allure and/or ExtentReports reporting, with screenshot-on-failure capture, via JUnit 5 extensions.
7. **Tune** JUnit 5 parallel execution settings (parallelism, resource locks) to safely scale a suite across CPU cores without race conditions.
8. **Stand up and operate** Selenium Grid 4 — locally, via Docker Compose, and understand how the same test code targets a cloud execution vendor with only configuration changes.
9. **Refactor** a flat, locator-in-test-class script into a full enterprise-grade POM + framework-architecture solution.


---

## 3. Technical Depth

### 3.1 Page Object Model — Principles, Layering, Separation of Concerns

**What.** POM is a design pattern where every web page (or logical screen/component) is represented by a class. The class exposes **business-readable methods** (`login()`, `addToCart()`) and hides **how** those actions are performed (locators, waits, JS execution).

**Why.** Test scripts change for two independent reasons: (a) business/behavioral change and (b) UI/DOM change. Without POM, both reasons force edits to the *same* file. POM decouples these: a locator change touches one page class; a business-flow change touches one test.

**When/Where.** Apply POM once more than a handful of tests share a screen, or more than one contributor writes tests. POM sits between the **test layer** and the **driver layer** — a translation layer, never containing raw `WebDriver` calls in the test, and never containing assertions in the page (discussed in Pitfalls).

**Layering model used in this chapter:**

```
Layer 4:  Test Classes            (JUnit 5, Assertions, business flow)
Layer 3:  Page Objects            (LoginPage, CartPage, CheckoutPage)
Layer 2:  Component Objects       (HeaderComponent, ModalComponent, DataTableComponent)
Layer 1:  BasePage / BaseComponent (shared wait helpers, click(), type(), isVisible())
Layer 0:  DriverFactory / WebDriver (ThreadLocal-scoped Selenium session)
```

**Protocol note.** Every `findElement()` call — raw or wrapped — serializes to a W3C WebDriver Protocol HTTP request (`POST /session/{id}/element`). The browser driver resolves the selector against the live DOM and returns an **opaque element reference**, bound to that DOM node's identity at lookup time — the root cause of stale-element failures (Section 3.3).

### 3.2 PageFactory — Key Mechanics

`PageFactory.initElements(driver, this)` reads `@FindBy`-annotated fields and injects **Java dynamic proxies** in their place, deferring the real `findElement()` call until first use.

```java
public class LoginPage {
    @FindBy(id = "username") private WebElement usernameInput;
    @FindBy(id = "password") private WebElement passwordInput;
    @FindBy(css = "button[type='submit']") private WebElement loginBtn;

    public LoginPage(WebDriver driver) { PageFactory.initElements(driver, this); }

    public void login(String u, String p) {
        usernameInput.sendKeys(u);
        passwordInput.sendKeys(p);
        loginBtn.click();
    }
}
```

- `@CacheLookup` fetches the element **once** and reuses the reference for the object's lifetime — fast, but unsafe on any element that can be re-rendered (grids, SPA-routed content). Reserve it for structurally static elements only (logo, static nav shell).
- `AjaxElementLocatorFactory(driver, timeoutInSeconds)` wraps every proxy with a per-element presence-poll, independent of the driver's global implicit wait — it waits for presence only, not visibility/clickability.
- Selenium 4's own documentation now favors plain `By` fields + explicit waits for new frameworks, since proxy indirection hides *when* the DOM query actually happens, complicating debugging (Section 6.1 compares this in detail).

### 3.3 Strategies to Avoid Stale Elements

A `WebElement` handle is invalidated by: full/partial navigation, JS-framework re-render (React key change, Angular `*ngFor` diff), DOM removal/re-add of an "equivalent" node, or `@CacheLookup` combined with any of the above.

**Strategy 1 — Re-find on every access (default recommendation).** Use `By` constants + fresh `findElement(by)` on every call instead of storing `WebElement` fields — every call performs a fresh lookup, so there is never a stale handle to begin with.

**Strategy 2 — `ExpectedConditions.refreshed(visibilityOfElementLocated(by))`.** Re-runs the locator-based condition on every poll iteration, safely surviving a re-render mid-wait.

**Strategy 3 — Retry wrapper.** Catch `StaleElementReferenceException`, re-locate, and retry the entire find-and-act sequence — useful for legacy code that cannot be fully refactored immediately.

**Strategy 4 — Never cache dynamic containers.** For data grids/lists, re-fetch the **container** fresh each time and search *within* it, rather than caching row/cell elements individually (see `DataTableComponent`, Section 9.5).

| Strategy | Reliability | Performance Cost | Best For |
|---|---|---|---|
| Re-find every access | High | 1 extra round-trip/access | Default for all dynamic pages |
| `refreshed(visibilityOfElementLocated(...))` | High | Poll overhead until met | Waiting through a re-render |
| Retry-on-stale wrapper | Medium-High | Only on failure | Legacy code, incremental refactor |
| `@CacheLookup` | Low (dynamic) / High (static) | Fastest | Static headers/footers/logos only |


### 3.4 Framework Architecture — Deep Dive

This section is the architectural core of the chapter: the scaffolding that supplies every test with a correctly configured, isolated `WebDriver`, centralized configuration, reusable utilities, structured test data, and structured logging — all invisible to the test author.

#### 3.4.1 DriverFactory & ThreadLocal WebDriver

**What.** `DriverFactory` is the single point in the codebase permitted to construct a `WebDriver`. Every page object, component, and test obtains its driver through `DriverFactory.getDriver()` — never via `new ChromeDriver()` scattered through the codebase.

**Why ThreadLocal.** JUnit 5 supports parallel execution (Section 3.5.3). If two test threads share one `static WebDriver` field, both drive the *same browser session* concurrently — clicks and navigations interleave randomly. `ThreadLocal<WebDriver>` gives each thread its **own** driver instance, keyed internally by the calling thread's identity, eliminating cross-thread interference by construction rather than by convention.

**How it works internally.** `ThreadLocal` maintains a hidden `ThreadLocalMap` on each `Thread` object. `DRIVER_THREAD_LOCAL.set(driver)` writes into *that calling thread's* map under a key derived from the `ThreadLocal` instance itself; `DRIVER_THREAD_LOCAL.get()` reads from the *same calling thread's* map. Two different threads calling `get()` on the identical static `ThreadLocal` field always retrieve two independent values — there is no shared mutable state to race on.

**Lifecycle contract (critical):**

```
@BeforeEach  →  DriverFactory.getDriver()   (lazy-inits on first call per thread)
   test body →  page objects call getDriver() repeatedly (same instance, same thread)
@AfterEach   →  DriverFactory.quitDriver()  (driver.quit() THEN ThreadLocal.remove())
```

Omitting `ThreadLocal.remove()` after `quit()` is the single most common parallel-execution defect in real frameworks: JUnit 5's parallel engine executes tests using a **pooled thread executor**, meaning physical OS threads are **reused** across different test classes over the life of a test run. If thread T runs Test A, quits its driver, but never calls `remove()`, the map entry for that `ThreadLocal` key still holds a reference to the now-dead `WebDriver` object (not `null`). When the pool later reuses thread T for Test B, `getDriver()`'s null-check (`if (get() == null)`) is bypassed — it sees a non-null (but dead) driver — and Test B receives a reference to an already-quit session, immediately failing with `NoSuchSessionException` on the first command.

**Multi-browser + local/remote abstraction.** `DriverFactory` should hide *which* browser and *whether local or Grid/cloud* behind a single configuration read, so tests are 100% portable across execution targets without code change — only `-Dbrowser=firefox -Dgrid.enabled=true` changes at invocation time (full implementation in Section 9).

#### 3.4.2 Configuration Management

**What.** A single `ConfigManager` abstraction that resolves every environment-specific value (base URL, browser, timeouts, Grid URL, credentials) from a layered source, with a clear, documented **precedence order** so CI and local runs behave predictably.

**Why layered precedence.** A framework used by 50+ engineers across local dev machines, a shared QA environment, and multiple CI pipelines needs one value (e.g., `base.url`) to differ per context *without* editing checked-in files per run. The standard, battle-tested precedence order (highest wins):

```
1. Command-line system property     (-Dbase.url=https://qa2.example.com)
2. Environment variable             (BASE_URL=https://qa2.example.com)
3. Environment-specific properties  (config-qa.properties, config-prod.properties)
4. Default properties file          (config.properties)
5. Hardcoded fallback in code       (last resort, should rarely be reached)
```

**Multiple approaches to configuration storage** (compare before choosing):

| Approach | Strength | Weakness |
|---|---|---|
| `.properties` (Java `Properties`) | Zero extra dependency, trivial to parse, universally understood | Flat key-value only; no native nesting/lists |
| YAML (via Jackson/SnakeYAML) | Nested structures, lists, multi-environment blocks in one readable file | Extra dependency; indentation-sensitivity can cause silent misconfiguration |
| Environment variables only | Best for containerized/CI-native secrets injection, zero file to leak | Unwieldy for dozens of settings; poor local-dev ergonomics |
| Typed Java config class (e.g., Owner library or a hand-rolled record) | Compile-time safety, autocompletion | Requires a rebuild to add new keys in some implementations |

**Recommendation:** `.properties` as the file-based default (simplicity, zero dependency) layered under environment-variable and system-property overrides — sufficient for the vast majority of frameworks; adopt YAML only if configuration genuinely needs nested structure (e.g., a matrix of per-environment, per-browser capability sets).

#### 3.4.3 Constants Layer

**What.** A `Constants` (or several purpose-scoped constants classes: `Timeouts`, `Endpoints`, `TestUsers`) holding every magic number and magic string used across the framework — timeout durations, default browser name, default Grid URL, retry counts.

**Why.** Magic numbers scattered across dozens of page/test files (`Duration.ofSeconds(15)` repeated 40 times) make a single "increase our default wait" change a multi-file hunt. Centralizing into `Constants.DEFAULT_EXPLICIT_WAIT_SECONDS` makes it a one-line change.

**Approach 1 — Plain constants class (default recommendation for this chapter).**

```java
public final class Timeouts {
    private Timeouts() {}
    public static final Duration EXPLICIT_WAIT = Duration.ofSeconds(15);
    public static final Duration PAGE_LOAD_TIMEOUT = Duration.ofSeconds(30);
    public static final Duration POLLING_INTERVAL = Duration.ofMillis(250);
}
```

`EXPLICIT_WAIT`, `PAGE_LOAD_TIMEOUT`, and `POLLING_INTERVAL` here are independent, unrelated values consumed **by name** at completely different call sites — `WaitUtils` only ever wants `EXPLICIT_WAIT`; `DriverFactory` only ever wants `PAGE_LOAD_TIMEOUT`. Nobody iterates over "all timeouts," switches on one, or resolves one dynamically by name. For that access pattern, `Timeouts.EXPLICIT_WAIT` (direct field access, zero indirection) is simpler than any enum wrapper, and it avoids machinery (`values()`, `ordinal()`, `name()`, `valueOf()`) built for representing a closed, enumerable *set of kinds* — which this isn't.

**Approach 2 — Config-backed enum.** Enum earns its keep the moment the framework needs any of: **runtime lookup by name** (resolving a timeout key read from `config.properties`), **switching/iterating** (behavior that varies per timeout "kind," or logic needing `values()`), or **per-member behavior/data** attached beyond a bare value (a config key, a default override, a unit-conversion rule). Because this chapter's framework already has a layered `ConfigManager` (Section 3.4.2), a config-driven `TimeoutType` enum is a natural next step once per-environment timeout overrides are needed:

```java
public enum TimeoutType {
    EXPLICIT_WAIT("explicit.wait.seconds", 15),
    PAGE_LOAD_TIMEOUT("page.load.timeout.seconds", 30),
    POLLING_INTERVAL_MS("polling.interval.ms", 250);

    private final String configKey;
    private final int defaultValue;

    TimeoutType(String configKey, int defaultValue) {
        this.configKey = configKey;
        this.defaultValue = defaultValue;
    }

    public Duration resolve() {
        int value = ConfigManager.getInt(configKey, defaultValue);
        return this == POLLING_INTERVAL_MS ? Duration.ofMillis(value) : Duration.ofSeconds(value);
    }
}

// usage
WebDriverWait wait = new WebDriverWait(driver, TimeoutType.EXPLICIT_WAIT.resolve());
```

Here each constant is genuinely a "kind of timeout" carrying its own config key and default, and `resolve()` gives config-driven overrides (e.g., a `config-qa.properties` setting `explicit.wait.seconds=20`) for free — something the plain `Duration` constants cannot do without a separate `ConfigManager.getInt(...)` call scattered at every usage site.

**Comparison — Constants Class vs Enum for framework constants:**

| Criterion | Plain Constants Class | Config-Backed Enum |
|---|---|---|
| Access pattern | Direct field access (`Timeouts.EXPLICIT_WAIT`) | Method call to unwrap (`TimeoutType.EXPLICIT_WAIT.resolve()`) |
| Runtime lookup by name | Not supported natively (needs a manual `switch`/`Map`) | Native via `valueOf(String)` |
| Iteration over "all kinds" | Not supported | Native via `values()` |
| Per-member behavior/data | Not supported (bare values only) | Supported (config key, default, custom logic per constant) |
| Config-driven override | Requires a separate `ConfigManager.getInt(...)` call at each site | Built into `resolve()`, one call site |
| Conceptual fit | A bag of unrelated named values | A closed, enumerable set of "kinds" |
| Boilerplate / indirection | Minimal | Higher (constructor, fields, method) |
| Best for | Fixed, unrelated values with no runtime lookup — the common case | Config-overridable values, or genuine switch/iterate needs |

**Recommendation.** Default to the plain constants class for straightforward, independently-named values — it is the more idiomatic choice for a beginner/intermediate teaching example and keeps the constants layer uncluttered. Reach for the config-backed enum only once the framework genuinely needs per-environment overrides, dynamic name-based resolution, or per-constant behavior — at which point it becomes the more enterprise-correct pattern (both versions are implemented side-by-side in Section 9.4).

#### 3.4.4 Utilities Layer

**What.** Small, stateless, framework-wide helper classes that page/component objects (and occasionally tests) call into: wait helpers, retry helpers, JS-executor helpers, screenshot helpers, string/date helpers.

**Why a separate layer, not inside `BasePage`.** `BasePage` is specifically about *element interaction*. Utilities like `ScreenshotUtils` or a generic `RetryUtil<T>` are used by test infrastructure (JUnit 5 extensions, Section 3.5.2) that has no `WebDriver`-per-element context at all — keeping them independent avoids an artificial dependency on `BasePage` from framework-level code.

Utility classes built in this chapter (full code in Section 9.4–9.6): `WaitUtils` (explicit-wait wrapper factory), `RetryUtil` (generic retry-on-exception wrapper), `ScreenshotUtils` (capture + save with a naming convention), `JSExecutorUtils` (scrollIntoView, highlight-for-debugging, readyState checks).

#### 3.4.5 Test Data Management

**What.** A strategy for supplying test input data (user credentials, product SKUs, address forms) that keeps data **external** to both test logic and page objects, and integrates with JUnit 5's data-driven test features.

**Multiple approaches:**

| Approach | Mechanism | Best For |
|---|---|---|
| CSV file | `@CsvFileSource(resources = "/testdata/users.csv")` | Simple flat tabular data, easily editable by non-engineers (BAs, manual QA) |
| JSON + POJO deserialization (Jackson) | Load a `List<UserData>` from a `.json` file via `ObjectMapper` | Structured/nested data (an address object inside a user object) |
| `@MethodSource` with a builder pattern | A Java method returns a `Stream<Arguments>` built via a fluent `TestDataBuilder` | Data that needs computed/derived fields (e.g., a unique email per run via timestamp) |
| Database-backed test data | Fetch rows from a dedicated test-data schema at runtime | Large enterprise suites needing referential integrity with backend state |

**Recommendation:** JSON + POJO for anything beyond trivial flat rows (readability + type safety), CSV only for very simple, spreadsheet-style datasets maintained by non-engineers, and a builder pattern layered on top of either for run-unique fields (timestamps, UUIDs) so parallel test runs never collide on shared test accounts.

#### 3.4.6 Logging Architecture

**What.** SLF4J as the logging façade, Logback as the concrete implementation, configured so that **concurrent parallel test threads produce clearly attributable, non-interleaved-looking output**.

**Why MDC (Mapped Diagnostic Context) matters for parallel runs.** In a sequential suite, a plain log line (`Clicking login button`) is unambiguous. Under JUnit 5 parallel execution, ten threads may log simultaneously, and a flat console stream interleaves their lines with no indication of which test produced which line. Logback's **MDC** lets you inject a per-thread key (e.g., `testId`) into every log line's format automatically:

```java
MDC.put("testId", context.getDisplayName());   // set in a JUnit 5 extension's beforeEach
...
MDC.remove("testId");                          // cleared in afterEach to avoid leaking into pooled-thread reuse
```

```xml
<!-- logback.xml pattern including MDC value -->
<pattern>%d{HH:mm:ss.SSS} [%thread] [%X{testId}] %-5level %logger{36} - %msg%n</pattern>
```

This produces lines like `14:02:11.203 [pool-2-thread-3] [LoginTest.validLogin] DEBUG ... - Clicking login button`, making concurrent logs fully attributable without any change to the log statements themselves.

**Per-test log files (advanced).** For very large parallel suites, route each test's logs to an individually named file (via a Logback `SiftingAppender` keyed on the MDC `testId`) rather than one shared console stream — this makes CI log artifacts trivially attachable per-test-result in reporting tools.


### 3.5 Reporting, Screenshots, Parallel Execution, Selenium Grid 4, Docker, and Cloud Execution

#### 3.5.1 Reporting Frameworks

| Tool | Model | Strength | Weakness |
|---|---|---|---|
| **Allure** | Annotation-driven (`@Step`, `@Epic`, `@Feature`, `@Severity`) + a separate report-generation CLI/plugin that reads result JSON and renders an interactive HTML report | Rich step-level drill-down, attachments (screenshots, logs) auto-linked per step, industry-standard CI dashboard integration (Jenkins Allure plugin) | Requires a separate report-generation step in the pipeline; less control over exact HTML branding |
| **ExtentReports** | Programmatic API (`ExtentTest test = extent.createTest(name)`) called explicitly in test code/listeners | Full control over report structure and branding, single self-contained HTML output, no separate generation step | More boilerplate — every step must be explicitly logged in code; less "free" step capture than Allure's annotations |

**Recommendation:** Allure for most enterprise pipelines (lower boilerplate, best CI-dashboard ecosystem); ExtentReports when a fully self-contained, custom-branded single HTML artifact is a hard requirement (e.g., for a non-technical stakeholder-facing report emailed after every run).

#### 3.5.2 Screenshot Capture Strategies

| Strategy | Trigger | Mechanism |
|---|---|---|
| Screenshot-on-failure only | JUnit 5 `TestWatcher.testFailed()` | Lowest storage cost; sufficient for most CI pipelines |
| Screenshot on every step | Custom `@Step`-wrapping utility or Allure step listener | Highest diagnostic value, highest storage/runtime cost — reserve for flaky-test investigation runs, not routine CI |
| Screenshot on every assertion | Custom `Assertions` wrapper | Middle ground — useful when assertions are sparse but meaningful |

The chapter's reference implementation (Section 9.7) uses **screenshot-on-failure**, attached automatically to both the file system (`target/screenshots/`) and, when Allure is active, to the Allure result set via `Allure.addAttachment(...)`.

#### 3.5.3 Parallel Execution with JUnit 5 — Deep Dive

**Parallel modes.** `junit-platform.properties` controls parallelism at two levels:

```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.config.dynamic.factor=1.0
```

- `mode.default=concurrent` — methods within a class may run concurrently with each other.
- `mode.classes.default=concurrent` — separate test classes may run concurrently with each other.
- `config.strategy=dynamic` with `factor=1.0` — JUnit computes a thread-pool size as `factor × availableProcessors()`; tune the factor down (e.g., `0.5`) on CI agents with limited memory per browser session (each Chrome session is typically 200–400MB RSS).

**Resource locks — the critical safety valve.** Not everything can run concurrently: if two tests both write to the same test-account's cart, or both read/write a shared config file, they need a **resource lock**, not full isolation via `ThreadLocal` alone:

```java
@Test
@ResourceLock(value = "shared-test-account", mode = ResourceAccessMode.READ_WRITE)
void checkoutWithSharedAccount() { ... }
```
JUnit 5's scheduler automatically serializes any tests declaring the same `@ResourceLock` key, even though parallel execution is globally enabled — this is the sanctioned mechanism for "mostly parallel, but these specific tests must not overlap."

**Thread-count tuning.** Rule of thumb for local developer machines: `factor` around `0.5`–`0.75` (leaves headroom for the IDE/OS); for CI agents, benchmark actual memory ceiling ÷ per-browser-session memory footprint, and cap explicitly with a fixed strategy (`config.strategy=fixed`, `config.fixed.parallelism=N`) rather than a factor-of-cores calculation, since CI agent vCPU counts often don't correlate with available memory.

#### 3.5.4 Selenium Grid 4 Architecture — Deep Dive

Grid 4 is a **fully W3C-protocol-native**, distributed system, a major architectural rewrite from Grid 3's Hub/Node model. Four logical components:

```
┌───────────────────────────────────────────────────────────────────┐
│                         SELENIUM GRID 4                            │
│                                                                     │
│   ┌────────────┐     ┌──────────────┐    ┌──────────────────┐     │
│   │   ROUTER   │────►│ DISTRIBUTOR  │───►│   SESSION MAP     │     │
│   │ (entry pt) │     │ (assigns to  │    │ (sessionId → Node)│     │
│   └─────┬──────┘     │  best Node)  │    └──────────────────┘     │
│         │             └──────┬───────┘                            │
│         │  new session req   │  slot request                      │
│         │                    ▼                                    │
│         │            ┌───────────────┐   ┌───────────────┐        │
│         │            │   NODE 1      │   │   NODE 2      │        │
│         │            │ (Chrome x4,   │   │ (Firefox x4)  │        │
│         │            │  Edge x2)     │   │               │        │
│         │            └───────────────┘   └───────────────┘        │
│         │                                                          │
│         ▼  ongoing session commands (routed by sessionId)         │
│   test code talks to Router only — never to a Node directly       │
└───────────────────────────────────────────────────────────────────┘
```

- **Router** — the single entry point every `RemoteWebDriver` call hits; forwards new-session requests to the Distributor and routes ongoing commands (by `sessionId`, looked up via Session Map) directly toward the owning Node.
- **Distributor** — tracks each Node's declared capacity/capabilities and assigns new sessions to the best-fit Node; queues requests when all matching slots are busy (New Session Queue, with a configurable timeout).
- **Session Map** — a lookup table (`sessionId → Node URI`) so the Router can forward subsequent commands for an existing session directly, without re-negotiating capabilities each time.
- **Node** — the process that actually launches and owns local `chromedriver`/`geckodriver`/`msedgedriver` processes and their browsers, advertising its capacity (e.g., 4 Chrome + 2 Firefox slots) to the Distributor.

**Why this matters vs Grid 3.** Grid 3's Hub translated JSON Wire Protocol to whatever the Node's driver expected, adding a serialization/translation layer that was a frequent source of subtle capability-mismatch bugs. Grid 4's Router/Distributor/Node all speak native W3C end-to-end, removing that translation layer entirely.

**Standing up Grid 4 locally (non-Docker):**
```bash
java -jar selenium-server-4.23.0.jar standalone   # single-process, all roles combined — good for local dev/small CI
# or, fully distributed:
java -jar selenium-server-4.23.0.jar hub
java -jar selenium-server-4.23.0.jar node --hub http://localhost:4444
```

#### 3.5.5 Docker Selenium — Standalone vs Hub-Node Topology

**Standalone (simplest, good for small/medium CI):**
```bash
docker run -d -p 4444:4444 -p 7900:7900 --shm-size=2g selenium/standalone-chrome:latest
```
`--shm-size=2g` is not cosmetic — Chrome uses `/dev/shm` for shared memory; Docker's default (64MB) causes frequent, hard-to-diagnose renderer crashes under any real load. Port `7900` exposes a live noVNC viewer for visually debugging a running container session.

**Hub-Node topology via Docker Compose (mirrors Grid 4's real distributed architecture, scales horizontally):**
```yaml
services:
  selenium-hub:
    image: selenium/hub:4.23.0
    ports: ["4442:4442", "4443:4443", "4444:4444"]

  chrome-node:
    image: selenium/node-chrome:4.23.0
    shm_size: 2gb
    depends_on: [selenium-hub]
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=4
    deploy:
      replicas: 3   # scale horizontally: 3 nodes × 4 sessions = 12 concurrent Chrome sessions

  firefox-node:
    image: selenium/node-firefox:4.23.0
    shm_size: 2gb
    depends_on: [selenium-hub]
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443

  video-chrome:
    image: selenium/video:ffmpeg-7.0.2-20240914
    depends_on: [chrome-node]
    environment:
      - DISPLAY_CONTAINER_NAME=chrome-node
      - SE_VIDEO_FILE_NAME=chrome-session.mp4
```
The optional `video` sidecar container records every session on its paired Node — invaluable for diagnosing intermittent CI-only failures after the fact, since the container running the browser is otherwise disposable.

#### 3.5.6 Cloud Execution Integration (BrowserStack / Sauce Labs / LambdaTest-style Vendors)

**Architecture.** Cloud vendors expose the identical `RemoteWebDriver` interface Grid 4 uses — the only difference is the hub URL and a vendor-specific capability block merged onto `ChromeOptions` (e.g., `"bstack:options"`, `"sauce:options"`). This means a well-designed `DriverFactory` (Section 9.3) treats "Grid" and "cloud vendor" as **the same code path** — just a different URL and a different capability-merge function — selected purely by configuration.

```java
public static WebDriver createCloudDriver(String vendorHubUrl, ChromeOptions options,
                                           Map<String, Object> vendorCapabilities) {
    options.setCapability("cloud:options", vendorCapabilities); // vendor-specific namespace key
    try {
        return new RemoteWebDriver(URI.create(vendorHubUrl).toURL(), options);
    } catch (MalformedURLException e) {
        throw new IllegalStateException("Invalid cloud hub URL", e);
    }
}
```

**Why abstract this behind `DriverFactory` rather than branching in tests.** A test suite that runs identically on a developer's laptop (local Chrome), a self-hosted Docker Grid (CI), and a cloud vendor (cross-browser/cross-OS matrix runs) — with zero test-code differences — is only achievable if the local/Grid/cloud decision lives entirely in configuration + `DriverFactory`, never in `@Test` methods.

#### 3.5.7 `@RegisterExtension` vs `@ExtendWith` — Registering the Lifecycle Extension

**What.** JUnit 5 offers two ways to attach an extension (such as the `TestLifecycleExtension` built in Section 9.9) to a test class: the declarative `@ExtendWith(SomeExtension.class)` annotation, and the programmatic `@RegisterExtension` field annotation.

```java
// Declarative — JUnit instantiates the extension itself via a no-arg constructor
@ExtendWith(TestLifecycleExtension.class)
public abstract class BaseTest { ... }
```

```java
// Programmatic — the test class instantiates the extension itself
public abstract class BaseTest {
    @RegisterExtension
    protected final TestLifecycleExtension lifecycleExtension =
            new TestLifecycleExtension(ConfigManager.get("report.dir", "target/reports"));

    @BeforeEach
    void logStart(TestInfo info) {
        // any additional per-test setup that needs the SAME instance the extension used
    }
}
```

**Why it matters — the key architectural difference.** `@ExtendWith` requires JUnit to construct the extension itself, which means the extension class **must have a no-argument constructor** (or be resolved via JUnit's own dependency-injection extension points). `@RegisterExtension` lets *your code* construct the extension instance — meaning you can pass constructor arguments (a screenshot directory, a `ConfigManager`-resolved report path, a feature flag), inject a mock for a unit test of the extension itself, or **share the exact same instance** between the extension's lifecycle callbacks and the test class's own `@BeforeEach`/`@Test` methods via a field reference.

**When to use which:**

| Criterion | `@ExtendWith` | `@RegisterExtension` |
|---|---|---|
| Extension needs constructor parameters (config path, thresholds, mock collaborators) | Not possible directly (no-arg only) | **Native support** — construct with any arguments |
| Extension instance needs to be referenced from within the test body itself | Not possible (JUnit owns the instance, test has no handle to it) | **Native support** — the field holds the exact same instance |
| Simplicity / declarative readability | Preferred — one line, no field | More verbose (a field + `new`) |
| Conditional/environment-based registration (e.g., only register in CI) | Not possible — static annotation | Possible — a static/instance field assigned via a conditional expression |
| Static field for once-per-class setup extensions (e.g., a shared Grid connection check) | N/A | `@RegisterExtension static ...` runs once per class, similar to `@BeforeAll` semantics |

**`beforeEach` usage with `@RegisterExtension` — concrete pattern used in this framework.** The `TestLifecycleExtension` (Section 9.9) implements `BeforeEachCallback`. Whether attached via `@ExtendWith` or `@RegisterExtension`, JUnit invokes `beforeEach(ExtensionContext context)` immediately before every `@Test` method — the *registration style* only changes who constructs the instance, not when the callback fires:

```java
public abstract class BaseTest {

    @RegisterExtension
    protected final TestLifecycleExtension lifecycleExtension =
            new TestLifecycleExtension(ConfigManager.get("screenshot.dir", "target/screenshots"));

    @BeforeEach
    void navigateToBaseUrl() {
        // Runs AFTER lifecycleExtension.beforeEach() has already tagged MDC for this test —
        // so any log statements here are already correctly attributed.
        getDriver().get(ConfigManager.get("base.url", "https://example.com"));
    }
}
```

**Execution order guarantee.** For a given test method, JUnit 5 runs **all registered `beforeEach` callbacks — both extension-provided and the class's own `@BeforeEach` methods — in a single ordered chain**, with extension callbacks registered via `@RegisterExtension` on a field executing in field-declaration order relative to other `@RegisterExtension` fields, and (for most common setups) before the class's own `@BeforeEach` methods. This ordering is precisely why `TestLifecycleExtension.beforeEach()` (which tags MDC) reliably runs before `navigateToBaseUrl()` above — any `log.info(...)` call inside your own `@BeforeEach` is already correctly MDC-tagged.

**Recommendation for this framework:** use `@RegisterExtension` for `TestLifecycleExtension` specifically **because it needs a constructor argument** (the configurable screenshot/report directory) — a case `@ExtendWith` cannot express without resorting to a static/global default inside the extension itself. Reserve `@ExtendWith` for simpler, parameterless, purely declarative extensions (e.g., a plain `TestWatcher` with no configuration).

---

## 4. Architecture — ASCII Diagrams

### 4.1 Framework Component Diagram (Tests → Pages → Driver Factory → Grid/Cloud)

```
┌─────────────────────────────────────────────────────────────────┐
│                        TEST LAYER (JUnit 5)                     │
│   LoginTest.java   CartTest.java   CheckoutTest.java             │
└────────┼─────────────────┼────────────────┼─────────────────────┘
         ▼                 ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│  PAGE OBJECT LAYER  →  COMPONENT OBJECT LAYER (nested widgets)   │
└────────┼─────────────────┼────────────────┼─────────────────────┘
         ▼                 ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│   BASE LAYER — BasePage / BaseComponent (click/type/waitVisible) │
└────────────────────────────┬──────────────────────────────────────┘
                             │ getDriver()
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│   DRIVER FACTORY — ThreadLocal<WebDriver> (per-thread isolation) │
│   reads: ConfigManager (browser, grid.enabled, base.url, ...)    │
└────────────────────────────┬──────────────────────────────────────┘
                 ┌────────────┼────────────┐
                 ▼            ▼             ▼
        LOCAL DRIVER   SELENIUM GRID 4   CLOUD VENDOR
     chromedriver/etc  Router→Distributor  RemoteWebDriver +
                        →Node (Docker)      vendor capabilities
```

### 4.2 ThreadLocal Isolation Under Parallel Execution

```
                    ThreadLocal<WebDriver> DRIVER_THREAD_LOCAL   (one static field)

  Thread Pool Worker 1 ──► ThreadLocalMap[Thread-1] { DRIVER_THREAD_LOCAL → ChromeDriver@A }
  Thread Pool Worker 2 ──► ThreadLocalMap[Thread-2] { DRIVER_THREAD_LOCAL → FirefoxDriver@B }
  Thread Pool Worker 3 ──► ThreadLocalMap[Thread-3] { DRIVER_THREAD_LOCAL → ChromeDriver@C }

  getDriver() on Thread-2 only ever sees @B. No cross-thread visibility, no shared mutable state.

  DANGER (unremoved entry):
  Thread-1 finishes Test A → quit() called, remove() FORGOTTEN
       → ThreadLocalMap[Thread-1] still holds dead ChromeDriver@A reference (not null!)
  Pool reuses Thread-1 for Test B → getDriver() null-check passes (non-null!) → returns dead @A
       → NoSuchSessionException on first command in Test B
```

### 4.3 Configuration Precedence

```
   HIGHEST PRIORITY
   ┌─────────────────────────────────────────┐
   │ 1. -D system property (CLI override)     │  -Dbase.url=https://qa2...
   ├─────────────────────────────────────────┤
   │ 2. Environment variable                   │  BASE_URL=https://qa2...
   ├─────────────────────────────────────────┤
   │ 3. Environment-specific properties file   │  config-qa.properties
   ├─────────────────────────────────────────┤
   │ 4. Default properties file                │  config.properties
   ├─────────────────────────────────────────┤
   │ 5. Hardcoded fallback in code              │  "https://default.example.com"
   └─────────────────────────────────────────┘
   LOWEST PRIORITY (should rarely be reached in a healthy pipeline)
```

### 4.4 Request Flow — Browser Communication (W3C WebDriver Protocol)

```
 Test              PageObject          DriverFactory        chromedriver         Browser
  │  login("a","b")   │                    │                    │                  │
  ├──────────────────►│                    │                    │                  │
  │                   │ getDriver()        │                    │                  │
  │                   ├───────────────────►│                    │                  │
  │                   │◄───────────────────┤ (ThreadLocal driver)│                  │
  │                   │ findElement(By.id) │                    │                  │
  │                   ├─────────────────────────────────────────►│ POST /session/  │
  │                   │                    │                    │  {id}/element    │
  │                   │                    │                    ├─────────────────►│ resolve DOM
  │                   │                    │                    │◄─────────────────┤ node handle
  │                   │◄─────────────────────────────────────────┤ 200 OK {element-id}│
```

### 4.5 Class Hierarchy / Object Relationships

```
                     ┌───────────────────┐
                     │    BasePage       │  (abstract)
                     └─────────┬─────────┘
                               │ extends
        ┌──────────────────────┼───────────────────────┐
        ▼                      ▼                        ▼
 ┌─────────────┐       ┌──────────────┐         ┌───────────────┐
 │ LoginPage   │       │  CartPage    │         │ CheckoutPage  │
 └─────────────┘       └──────┬───────┘         └───────┬───────┘
                               │ has-a                    │ has-a
                               ▼                           ▼
                     ┌──────────────────┐         ┌────────────────────┐
                     │ DataTableComp.   │         │ PaymentFormComp.   │
                     └──────────────────┘         └────────────────────┘
```


---

## 5. Multiple Approaches

### 5.1 Page Object Implementation Strategies

**Approach A — Classic PageFactory** (annotation-driven, proxy-based lazy lookup — concise but obscures lookup timing in stack traces).

**Approach B — Manual `By` Locators + Helper Methods** (explicit, always-fresh lookups; inherently stale-resistant; cleanest stack traces; compatible with `RelativeLocator`).

**Approach C — Component-Based POM** (composition over one large page class; a `DataTableComponent`/`PaymentFormComponent` written once services every page embedding that widget).

| Team/Project Context | Recommended Approach | Justification |
|---|---|---|
| Small team, few pages, quick delivery | A (PageFactory) | Fast to write, sufficient at small scale |
| Mid-size team, dynamic SPA UI | B (Manual `By` + helpers) | Best stale-element resilience, clean debugging |
| Large enterprise app, shared design-system widgets | **C (Component-based POM)** — overall recommended default | Maximizes reuse, isolates blast radius, scales with team size |

### 5.2 DriverFactory Design Approaches

| Approach | Mechanism | Trade-off |
|---|---|---|
| Static single `WebDriver` field | One shared instance | **Unsafe for parallel execution** — never use in a modern framework |
| `ThreadLocal<WebDriver>` (recommended) | One instance per executing thread | Safe parallel default; requires disciplined `remove()` on teardown |
| Driver pool (checkout/checkin pattern) | A bounded pool of pre-warmed driver instances, checked out per test | Reduces cold-start cost for very large suites; adds pool-management complexity and risk of state leakage between tests reusing a "dirty" session |
| Dependency-injected driver (constructor injection via a DI framework) | `WebDriver` passed explicitly into every page/test constructor | Maximizes testability/mockability; heavier setup for small teams without existing DI usage |

**Recommendation:** `ThreadLocal<WebDriver>` as the default for the vast majority of frameworks — it is simple, safe, and battle-tested. Consider a driver pool only once cold-start latency (new browser process per test) is a measured bottleneck at very large scale (thousands of short tests).

### 5.3 Parallel Execution Scaling Approaches

| Approach | Mechanism | Best For |
|---|---|---|
| JUnit 5 native parallel (this chapter's default) | `junit-platform.properties` + `ThreadLocal` driver | Single-JVM scaling, simplest CI setup |
| Multiple CI shards (matrix build) | CI splits the test suite across N independent parallel jobs, each running sequentially or with modest JUnit-level parallelism | Very large suites where a single agent's CPU/memory ceiling is the bottleneck |
| Grid/Docker horizontal scaling (Section 3.5.4–3.5.5) | Many concurrent sessions distributed across multiple Node containers | Cross-browser/cross-OS matrix runs, or when local machine resources cap concurrency regardless of JUnit settings |

**Recommendation:** combine all three at real enterprise scale — CI shards for coarse-grained splitting, JUnit 5 parallel within each shard, and a Grid/Docker backend so no single host's CPU/RAM is the ceiling.

---

## 6. Comparisons

### 6.1 PageFactory vs Manual Page Object (No PageFactory)

| Criterion | PageFactory | Manual `By` + Helper Methods |
|---|---|---|
| Lookup timing | Lazy, on first proxy method call | Explicit, at the line you write `findElement` |
| Stack trace clarity on failure | Obscured by proxy `invoke()` frames | Clean, points directly to your code |
| Stale-element resistance (no caching) | Same as manual (fresh unless `@CacheLookup`) | Same — inherently fresh every call |
| Integration with `RelativeLocator` | Poor (no annotation support) | Native — just another `By` |
| Boilerplate | Lower (annotation + field) | Slightly higher (constant `By` + helper call) |
| Selenium project's current guidance | De-emphasized for new code | Preferred for Selenium 4 tutorials |


---

## 7. Pitfalls & Anti-Patterns

**Page Object level:**
1. **Overuse of `@CacheLookup`** on grids/accordions/SPA-routed content — guarantees intermittent `StaleElementReferenceException`.
2. **Assertions inside Page Objects** — couples pages to a test framework and hides intent; pages should return state, tests should assert.
3. **God Page Objects** — one class with dozens of fields/methods becomes an unmaintainable merge-conflict magnet; decompose into components.
4. **Constructing a `WebDriver` inside a Page Object's constructor** — inverts the dependency direction; pages must receive a driver, never create one.

**Framework architecture level:**
5. **Forgetting `ThreadLocal.remove()` after `driver.quit()`** — the single most common cause of `NoSuchSessionException` under parallel, thread-pool-reused execution (Section 3.4.1 / 4.2).
6. **Static/shared `WebDriver` fields "for convenience"** — silently breaks the moment parallel execution is enabled; often not caught until CI flakes months later.
7. **Hardcoding environment values directly in tests or page objects** instead of routing through `ConfigManager` — breaks portability across dev/QA/staging/CI.
8. **Secrets committed to `config.properties` in source control** — always inject via environment variables or a secrets manager at runtime.
9. **Magic numbers/strings duplicated across dozens of files** instead of a `Constants` class — a single "increase default wait" change becomes a multi-file hunt.
10. **Logging with `System.out.println`** instead of SLF4J — produces unstructured, unlevelled output that cannot be routed to centralized log aggregation and, under parallel execution, is fully unattributable without MDC (Section 3.4.6).
11. **Trying to force `@ExtendWith` onto an extension that needs constructor parameters** — either by adding awkward static/global mutable state inside the extension to work around the no-arg-constructor limitation, or by hardcoding a value that should have been configurable. Use `@RegisterExtension` instead (Section 3.5.7).

**Parallel/Grid/Docker/cloud level:**
12. **Enabling JUnit 5 parallel execution without auditing for shared mutable state** (shared test accounts, shared config files written at runtime) — leads to intermittent, hard-to-reproduce cross-test interference; use `@ResourceLock` for anything genuinely shared.
13. **Setting `config.strategy=dynamic` with a high factor on a memory-constrained CI agent** — spawns more concurrent browser sessions than available RAM supports, causing random OOM-driven session crashes rather than a clean, predictable failure.
14. **Running Dockerized Selenium with default (too-small) `--shm-size`** — causes intermittent, hard-to-diagnose renderer crashes under load; always set `--shm-size=2g` or higher.
15. **Branching test code on "if running on Grid do X, else do Y"** — the local/Grid/cloud decision belongs entirely in `DriverFactory` + configuration, never in `@Test` methods (Section 3.5.6).
16. **No screenshot/log/video correlation strategy for CI-only failures** — without MDC-tagged logs, screenshot-on-failure, and (for Docker) video sidecars, intermittent CI failures become nearly undiagnosable after the fact.

---

## 8. Best Practices (Enterprise-Level)

- **SOLID for framework code:** Single Responsibility for each page/component; `BasePage` extended, not modified, for new capability; Dependency Inversion — pages depend on the `WebDriver` abstraction, never a concrete `ChromeDriver`.
- **One `DriverFactory`, one `ThreadLocal`, always paired `quit()` + `remove()`** in `@AfterEach`, never relying on JVM shutdown hooks alone.
- **Layered configuration with documented precedence** (Section 3.4.2/4.3) so CI overrides never require editing checked-in files.
- **Secrets only via environment variables or a secrets manager**, never in properties files committed to source control.
- **Centralize magic numbers/strings in a `Constants` layer**; centralize reusable, stateless helpers in a `utils` package independent of `BasePage`.
- **Structured, leveled logging (SLF4J/Logback) with MDC thread correlation** as a non-negotiable baseline the moment parallel execution is enabled.
- **Screenshot-on-failure as the default capture strategy**; reserve step-level/every-assertion capture for targeted flaky-test investigation, not routine CI (storage/runtime cost).
- **Tune parallel thread count against measured CI agent memory**, not just core count — cap explicitly (`config.strategy=fixed`) on constrained agents.
- **Use `@ResourceLock` deliberately** for any genuinely shared state instead of disabling parallelism suite-wide.
- **Docker-ize browsers for CI reproducibility**, with `--shm-size=2g`+ and, for hard-to-reproduce failures, a video-recording sidecar.
- **Design `DriverFactory` so local, Grid, and cloud-vendor execution are pure configuration switches**, never test-code branches — this is what makes a suite genuinely portable across a developer laptop, a Docker Grid, and a cross-browser cloud matrix run.


---

## 9. Java Implementation — Full Enterprise Framework

### 9.1 Maven Project Structure

```
selenium-pom-framework/
├── pom.xml
├── docker-compose.yml
├── src
│   ├── main/java/com/enterprise/automation
│   │   ├── config/
│   │   │   └── ConfigManager.java
│   │   ├── constants/
│   │   │   ├── Timeouts.java
│   │   │   └── Endpoints.java
│   │   ├── driver/
│   │   │   ├── DriverFactory.java
│   │   │   ├── ChromeOptionsBuilder.java
│   │   │   ├── FirefoxOptionsBuilder.java
│   │   │   └── EdgeOptionsBuilder.java
│   │   ├── pages/
│   │   │   ├── base/BasePage.java
│   │   │   ├── LoginPage.java
│   │   │   └── HomePage.java
│   │   ├── components/
│   │   │   ├── base/BaseComponent.java
│   │   │   └── DataTableComponent.java
│   │   ├── testdata/
│   │   │   ├── UserData.java
│   │   │   └── TestDataProvider.java
│   │   ├── report/
│   │   │   └── ReportManager.java
│   │   └── utils/
│   │       ├── WaitUtils.java
│   │       ├── RetryUtil.java
│   │       └── ScreenshotUtils.java
│   ├── main/resources/
│   │   ├── config.properties
│   │   ├── config-qa.properties
│   │   └── logback.xml
│   └── test/java/com/enterprise/automation
│       ├── base/BaseTest.java
│       ├── listeners/TestLifecycleExtension.java
│       ├── tests/LoginTest.java
│       └── tests/LoginDataDrivenTest.java
├── src/test/resources/
│   ├── junit-platform.properties
│   └── testdata/users.json
```

### 9.2 `pom.xml` — Key Dependencies

```xml
<properties>
    <maven.compiler.release>21</maven.compiler.release>
    <selenium.version>4.23.0</selenium.version>
    <junit.version>5.10.2</junit.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>${selenium.version}</version>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>2.0.13</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.5.6</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.1</version>
    </dependency>
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit5</artifactId>
        <version>2.27.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.3.0</version>
            <configuration>
                <properties>
                    <configurationParameters>
                        junit.jupiter.execution.parallel.enabled=true
                    </configurationParameters>
                </properties>
            </configuration>
        </plugin>
    </plugins>
</build>
```


### 9.3 `ConfigManager.java` — Layered Configuration Resolution

```java
package com.enterprise.automation.config;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.util.Properties;

/**
 * Resolves configuration in precedence order:
 * 1. -D system property
 * 2. Environment variable (upper-snake-case of the key)
 * 3. Environment-specific properties file (config-<env>.properties)
 * 4. Default config.properties
 * 5. Caller-supplied default value
 */
public final class ConfigManager {

    private static final Properties DEFAULT_PROPS = load("config.properties");
    private static final Properties ENV_PROPS =
            load("config-" + System.getProperty("env", "qa") + ".properties");

    private ConfigManager() {}

    private static Properties load(String fileName) {
        Properties props = new Properties();
        try (InputStream in = ConfigManager.class.getClassLoader().getResourceAsStream(fileName)) {
            if (in != null) {
                props.load(in);
            }
        } catch (IOException e) {
            throw new UncheckedIOException("Unable to load " + fileName, e);
        }
        return props;
    }

    public static String get(String key, String defaultValue) {
        String systemProp = System.getProperty(key);
        if (systemProp != null) return systemProp;

        String envVar = System.getenv(toEnvVarName(key));
        if (envVar != null) return envVar;

        String envSpecific = ENV_PROPS.getProperty(key);
        if (envSpecific != null) return envSpecific;

        return DEFAULT_PROPS.getProperty(key, defaultValue);
    }

    public static boolean getBoolean(String key, boolean defaultValue) {
        return Boolean.parseBoolean(get(key, String.valueOf(defaultValue)));
    }

    public static int getInt(String key, int defaultValue) {
        return Integer.parseInt(get(key, String.valueOf(defaultValue)));
    }

    private static String toEnvVarName(String key) {
        return key.toUpperCase().replace('.', '_');   // base.url -> BASE_URL
    }
}
```

`src/main/resources/config.properties`:
```properties
browser=chrome
base.url=https://example.com
grid.enabled=false
grid.url=http://localhost:4444
explicit.wait.seconds=15
```

`src/main/resources/config-qa.properties`:
```properties
base.url=https://qa.example.com
grid.enabled=true
grid.url=http://qa-grid.internal:4444
```

### 9.4 `Timeouts.java` and `Endpoints.java` — Constants Layer

**Approach 1 — Plain constants class (used by the rest of this chapter's code samples).**

```java
package com.enterprise.automation.constants;

import java.time.Duration;

public final class Timeouts {
    private Timeouts() {}
    public static final Duration EXPLICIT_WAIT = Duration.ofSeconds(15);
    public static final Duration PAGE_LOAD_TIMEOUT = Duration.ofSeconds(30);
    public static final Duration POLLING_INTERVAL = Duration.ofMillis(250);
    public static final int STALE_RETRY_MAX_ATTEMPTS = 3;
}
```

```java
package com.enterprise.automation.constants;

public final class Endpoints {
    private Endpoints() {}
    public static final String LOGIN_PATH = "/login";
    public static final String HOME_PATH = "/home";
    public static final String CHECKOUT_PATH = "/checkout";
}
```

**Approach 2 — Config-backed enum (drop-in alternative once per-environment overrides are needed; see Section 3.4.3 for the full rationale and comparison table).**

```java
package com.enterprise.automation.constants;

import com.enterprise.automation.config.ConfigManager;

import java.time.Duration;

public enum TimeoutType {
    EXPLICIT_WAIT("explicit.wait.seconds", 15),
    PAGE_LOAD_TIMEOUT("page.load.timeout.seconds", 30),
    POLLING_INTERVAL_MS("polling.interval.ms", 250);

    private final String configKey;
    private final int defaultValue;

    TimeoutType(String configKey, int defaultValue) {
        this.configKey = configKey;
        this.defaultValue = defaultValue;
    }

    public Duration resolve() {
        int value = ConfigManager.getInt(configKey, defaultValue);
        return this == POLLING_INTERVAL_MS ? Duration.ofMillis(value) : Duration.ofSeconds(value);
    }
}
```

```java
// Usage — identical call sites, either approach:
WebDriverWait waitPlain = new WebDriverWait(driver, Timeouts.EXPLICIT_WAIT);
WebDriverWait waitEnum  = new WebDriverWait(driver, TimeoutType.EXPLICIT_WAIT.resolve());
```

Both approaches are wired to compile against the same `WaitUtils`/`DriverFactory` call sites shown in Sections 9.5–9.6 — swapping one for the other in a real project is a localized, single-layer change with no ripple into page objects or tests.

### 9.5 Utilities Layer — `WaitUtils`, `RetryUtil`, `ScreenshotUtils`

```java
package com.enterprise.automation.utils;

import com.enterprise.automation.constants.Timeouts;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.support.ui.WebDriverWait;

public final class WaitUtils {
    private WaitUtils() {}

    public static WebDriverWait defaultWait(WebDriver driver) {
        return new WebDriverWait(driver, Timeouts.EXPLICIT_WAIT);
    }
}
```

```java
package com.enterprise.automation.utils;

import org.openqa.selenium.StaleElementReferenceException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class RetryUtil {
    private static final Logger log = LoggerFactory.getLogger(RetryUtil.class);

    private RetryUtil() {}

    public static void withStaleRetry(Runnable action, int maxAttempts) {
        StaleElementReferenceException lastEx = null;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                action.run();
                return;
            } catch (StaleElementReferenceException e) {
                log.warn("Stale element on attempt {}/{}, retrying", attempt, maxAttempts);
                lastEx = e;
            }
        }
        throw lastEx;
    }
}
```

```java
package com.enterprise.automation.utils;

import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;

import java.io.File;
import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;

public final class ScreenshotUtils {
    private ScreenshotUtils() {}

    public static Path capture(WebDriver driver, String directory, String testName) {
        File src = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
        Path dest = Path.of(directory, sanitize(testName) + ".png");
        try {
            Files.createDirectories(dest.getParent());
            Files.copy(src.toPath(), dest, StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
        return dest;
    }

    private static String sanitize(String name) {
        return name.replaceAll("[^a-zA-Z0-9-_]", "_");
    }
}
```


### 9.6 `DriverFactory.java` — Thread-Safe, Multi-Browser, Local/Grid/Cloud

```java
package com.enterprise.automation.driver;

import com.enterprise.automation.config.ConfigManager;
import com.enterprise.automation.constants.Timeouts;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.remote.RemoteWebDriver;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.MalformedURLException;
import java.net.URI;

public final class DriverFactory {

    private static final Logger log = LoggerFactory.getLogger(DriverFactory.class);
    private static final ThreadLocal<WebDriver> DRIVER_THREAD_LOCAL = new ThreadLocal<>();

    private DriverFactory() {}

    public static WebDriver getDriver() {
        if (DRIVER_THREAD_LOCAL.get() == null) {
            initDriver();
        }
        return DRIVER_THREAD_LOCAL.get();
    }

    private static void initDriver() {
        String browser = ConfigManager.get("browser", "chrome").toLowerCase();
        boolean useGrid = ConfigManager.getBoolean("grid.enabled", false);
        log.info("Initializing driver: browser={}, grid={}, thread={}",
                browser, useGrid, Thread.currentThread().getName());

        WebDriver driver = useGrid ? createRemoteDriver(browser) : createLocalDriver(browser);
        driver.manage().timeouts().pageLoadTimeout(Timeouts.PAGE_LOAD_TIMEOUT);
        DRIVER_THREAD_LOCAL.set(driver);
    }

    private static WebDriver createLocalDriver(String browser) {
        return switch (browser) {
            case "chrome" -> new ChromeDriver(ChromeOptionsBuilder.build());
            case "firefox" -> new FirefoxDriver(FirefoxOptionsBuilder.build());
            case "edge" -> new EdgeDriver(EdgeOptionsBuilder.build());
            default -> throw new IllegalArgumentException("Unsupported browser: " + browser);
        };
    }

    private static WebDriver createRemoteDriver(String browser) {
        try {
            String gridUrl = ConfigManager.get("grid.url", "http://localhost:4444");
            return switch (browser) {
                case "chrome" -> new RemoteWebDriver(URI.create(gridUrl).toURL(), ChromeOptionsBuilder.build());
                case "firefox" -> new RemoteWebDriver(URI.create(gridUrl).toURL(), FirefoxOptionsBuilder.build());
                case "edge" -> new RemoteWebDriver(URI.create(gridUrl).toURL(), EdgeOptionsBuilder.build());
                default -> throw new IllegalArgumentException("Unsupported browser: " + browser);
            };
        } catch (MalformedURLException e) {
            throw new IllegalStateException("Invalid grid URL", e);
        }
    }

    public static void quitDriver() {
        WebDriver driver = DRIVER_THREAD_LOCAL.get();
        if (driver != null) {
            log.info("Quitting driver on thread={}", Thread.currentThread().getName());
            driver.quit();
            DRIVER_THREAD_LOCAL.remove();   // CRITICAL — see Section 3.4.1 / 4.2
        }
    }
}
```

```java
package com.enterprise.automation.driver;

import org.openqa.selenium.chrome.ChromeOptions;

public final class ChromeOptionsBuilder {
    private ChromeOptionsBuilder() {}

    public static ChromeOptions build() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--window-size=1920,1080",
                "--no-sandbox", "--disable-dev-shm-usage");
        return options;
    }
}
```
*(`FirefoxOptionsBuilder`/`EdgeOptionsBuilder` follow the identical pattern for their respective `*Options` classes — omitted here for brevity; browser-flag-level depth for CDP/Options is covered in Module 7.)*

### 9.7 Test Data Layer — POJO + JSON + JUnit 5 `@MethodSource`

```java
package com.enterprise.automation.testdata;

public record UserData(String username, String password, String expectedLandingPage) {}
```

```java
package com.enterprise.automation.testdata;

import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.IOException;
import java.io.InputStream;
import java.util.List;
import java.util.stream.Stream;
import org.junit.jupiter.params.provider.Arguments;

public final class TestDataProvider {
    private TestDataProvider() {}

    public static Stream<Arguments> loginUsers() {
        try (InputStream in = TestDataProvider.class.getClassLoader()
                .getResourceAsStream("testdata/users.json")) {
            List<UserData> users = new ObjectMapper().readValue(in, new com.fasterxml.jackson.core.type.TypeReference<>() {});
            return users.stream().map(u -> Arguments.of(u.username(), u.password(), u.expectedLandingPage()));
        } catch (IOException e) {
            throw new RuntimeException("Failed to load test data", e);
        }
    }
}
```

`src/test/resources/testdata/users.json`:
```json
[
  { "username": "standard_user", "password": "secret_sauce", "expectedLandingPage": "/home" },
  { "username": "locked_out_user", "password": "secret_sauce", "expectedLandingPage": "/login-error" }
]
```

```java
@ParameterizedTest
@MethodSource("com.enterprise.automation.testdata.TestDataProvider#loginUsers")
void loginWithMultipleUsers(String username, String password, String expectedPath) {
    loginPage.loginExpectingFailure(username, password);
    // ... assertions using expectedPath
}
```


### 9.8 Logging Configuration — `logback.xml` with MDC Thread Correlation

`src/main/resources/logback.xml`:
```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{testId}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>target/logs/automation.log</file>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{testId}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

Usage inside any page/framework class:
```java
private static final Logger log = LoggerFactory.getLogger(LoginPage.class);
...
log.info("Attempting login for user={}", username);
```

### 9.9 `TestLifecycleExtension.java` — MDC + Screenshot + Allure, All in One JUnit 5 Extension

```java
package com.enterprise.automation.listeners;

import com.enterprise.automation.driver.DriverFactory;
import com.enterprise.automation.utils.ScreenshotUtils;
import io.qameta.allure.Allure;
import org.junit.jupiter.api.extension.*;
import org.slf4j.MDC;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class TestLifecycleExtension implements BeforeEachCallback, AfterEachCallback, TestWatcher {

    private static final String MDC_KEY = "testId";
    private final String screenshotDir;   // constructor-supplied — this is WHY @RegisterExtension is used (Section 3.5.7)

    public TestLifecycleExtension(String screenshotDir) {
        this.screenshotDir = screenshotDir;
    }

    @Override
    public void beforeEach(ExtensionContext context) {
        MDC.put(MDC_KEY, context.getDisplayName());
    }

    @Override
    public void afterEach(ExtensionContext context) {
        MDC.remove(MDC_KEY);   // prevents leaking into a reused pooled thread's next test
    }

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        Path screenshotPath = ScreenshotUtils.capture(DriverFactory.getDriver(), screenshotDir, context.getDisplayName());
        try {
            Allure.addAttachment(
                    "Failure Screenshot",
                    "image/png",
                    new ByteArrayInputStream(Files.readAllBytes(screenshotPath)),
                    "png");
        } catch (IOException e) {
            // attachment failure should never fail the test itself
        }
    }
}
```
**Registration.** Because this extension takes a constructor argument (`screenshotDir`, resolved from `ConfigManager` so it can differ between local runs and CI), it **cannot** be attached via `@ExtendWith(TestLifecycleExtension.class)` — that annotation only works with a no-arg constructor. It must be attached via `@RegisterExtension` on a field, as shown in `BaseTest` (Section 9.11) and explained in Section 3.5.7. This single extension covers logging correlation, screenshot capture, and Allure attachment in one seam, so swapping the reporting tool later only touches this class.

### 9.10 Parallel Execution Configuration

`src/test/resources/junit-platform.properties`:
```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.config.dynamic.factor=0.75
```

Resource-locked test example (Section 3.5.3):
```java
@Test
@ResourceLock(value = "shared-test-account", mode = ResourceAccessMode.READ_WRITE)
void checkoutWithSharedAccount() { /* ... */ }
```


### 9.11 `BasePage.java`, `BaseComponent.java`, and a Complete Test

```java
package com.enterprise.automation.pages.base;

import com.enterprise.automation.utils.WaitUtils;
import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;

public abstract class BasePage {

    protected final WebDriver driver;
    protected final WebDriverWait wait;
    private static final Logger log = LoggerFactory.getLogger(BasePage.class);

    protected BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = WaitUtils.defaultWait(driver);
    }

    protected void click(By by) {
        log.debug("Clicking element: {}", by);
        wait.until(ExpectedConditions.elementToBeClickable(by)).click();
    }

    protected void type(By by, String text) {
        WebElement el = wait.until(ExpectedConditions.visibilityOfElementLocated(by));
        el.clear();
        el.sendKeys(text);
    }

    protected String getText(By by) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(by)).getText();
    }

    protected boolean isDisplayed(By by) {
        List<WebElement> elements = driver.findElements(by);
        return !elements.isEmpty() && elements.get(0).isDisplayed();
    }

    protected void waitForUrlContains(String fragment) {
        wait.until(ExpectedConditions.urlContains(fragment));
    }
}
```

```java
package com.enterprise.automation.base;

import com.enterprise.automation.config.ConfigManager;
import com.enterprise.automation.driver.DriverFactory;
import com.enterprise.automation.listeners.TestLifecycleExtension;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.openqa.selenium.WebDriver;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public abstract class BaseTest {

    private static final Logger log = LoggerFactory.getLogger(BaseTest.class);

    // @RegisterExtension (not @ExtendWith) — TestLifecycleExtension takes a constructor
    // argument, which @ExtendWith cannot supply (Section 3.5.7).
    @RegisterExtension
    protected final TestLifecycleExtension lifecycleExtension =
            new TestLifecycleExtension(ConfigManager.get("screenshot.dir", "target/screenshots"));

    protected WebDriver getDriver() {
        return DriverFactory.getDriver();
    }

    @BeforeEach
    void logTestStart() {
        // Runs AFTER lifecycleExtension.beforeEach() has already tagged MDC for this test,
        // so this log line is already correctly attributed to the right [testId].
        log.info("Starting test");
    }

    @AfterEach
    void tearDown() {
        DriverFactory.quitDriver();
    }
}
```

```java
package com.enterprise.automation.tests;

import com.enterprise.automation.base.BaseTest;
import com.enterprise.automation.config.ConfigManager;
import com.enterprise.automation.constants.Endpoints;
import com.enterprise.automation.pages.HomePage;
import com.enterprise.automation.pages.LoginPage;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class LoginTest extends BaseTest {

    private LoginPage loginPage;

    @BeforeEach
    void navigateToLogin() {
        getDriver().get(ConfigManager.get("base.url", "https://example.com") + Endpoints.LOGIN_PATH);
        loginPage = new LoginPage(getDriver());
    }

    @Test
    void validCredentialsLandOnHomePage() {
        HomePage homePage = loginPage.login("standard_user", "secret_sauce");
        Assertions.assertTrue(homePage.isWelcomeBannerVisible(),
                "Expected welcome banner to be visible after valid login");
    }
}
```

### 9.12 Running Against Grid 4 / Docker / Cloud — Configuration-Only Switch

```bash
# Local Chrome (default)
mvn test

# Local Firefox
mvn test -Dbrowser=firefox

# Against a Dockerized Selenium Grid 4 (see docker-compose.yml, Section 3.5.5)
docker compose up -d
mvn test -Dgrid.enabled=true -Dgrid.url=http://localhost:4444

# Against a cloud vendor (hub URL + vendor capabilities supplied via env/config, same DriverFactory path)
mvn test -Dgrid.enabled=true -Dgrid.url=https://hub-cloud.example.com/wd/hub -Denv=cloud
```
No test class, page object, or component changes across any of these four invocations — only `ConfigManager`-resolved values differ, exactly as designed in Section 3.5.6.


---

## 10. Technical Validation — Why This Works

- **`ThreadLocal` isolation is verified at the HTTP layer.** Each `WebDriver` corresponds to a distinct `sessionId` in its command path (`/session/{sessionId}/...`). Because `ThreadLocal` guarantees each thread reads/writes its own map slot, two parallel JUnit 5 threads never share a `sessionId`, even when the underlying thread pool later reuses the same OS thread for a different test — **provided** `remove()` is called on teardown (Section 4.2).
- **Configuration precedence is deterministic and testable.** Because `ConfigManager.get()` checks system property → env var → env-specific file → default file in a fixed order, the same codebase produces predictable, reproducible values in any environment — a QA engineer can always answer "why is `base.url` X here?" by walking exactly that five-step list.
- **MDC-tagged logging is thread-safe by construction.** SLF4J's `MDC` is itself backed by a `ThreadLocal`-like mechanism, so tagging `testId` in `beforeEach` and clearing it in `afterEach` guarantees log lines from concurrently executing tests are never cross-attributed, without any change to individual `log.info(...)` call sites.
- **Resource locks compose correctly with global parallelism.** JUnit 5's scheduler treats `@ResourceLock` declarations as a constraint solved independently of the global parallel/concurrent setting — tests without a matching lock run fully concurrently, while tests sharing a lock key are automatically serialized relative to each other only, not to the whole suite.
- **Grid 4's native W3C path removes a translation layer** (vs Grid 3's Hub), so a session negotiated with a given `ChromeOptions` payload lands on a Node with matching capabilities without any JSON Wire Protocol re-shaping step that could silently drop or reinterpret a capability.

---

## 11. Debugging

- **IDE breakpoints:** set a breakpoint inside `DriverFactory.getDriver()`/`quitDriver()` to inspect exactly which thread is initializing/tearing down a session when diagnosing parallel-execution flakiness.
- **MDC-tagged log files:** grep `target/logs/automation.log` for a specific `[testId]` tag to isolate one test's full log trail out of an otherwise-interleaved parallel run.
- **Driver logs:** enable verbose chromedriver logs (`webdriver.chrome.logfile` / `webdriver.chrome.verboseLogging` system properties) when a session fails to *start* — most useful for local/CI driver-launch failures, distinct from in-test element failures.
- **Grid 4 console:** `http://<router-host>:4444/ui` exposes live session queues, per-Node capacity, and (with Docker video enabled) per-session recordings — check this first when Grid-based tests hang rather than fail cleanly, since a hang often means the New Session Queue timed out waiting for a free matching slot.
- **Docker container logs:** `docker logs <node-container>` surfaces browser-crash or `/dev/shm`-exhaustion errors that never reach the Selenium/JUnit log at all.
- **Screenshot-on-failure + Allure attachment (Section 9.9):** the first artifact to check for any CI-only failure — captures DOM state at the exact failure moment, often revealing an unexpected modal or loading spinner.
- **Reproducing a "CI-only" flake locally:** replicate the CI's parallel factor (`-Djunit.jupiter.execution.parallel.config.dynamic.factor=...`) and Docker resource limits (`--shm-size`, CPU/memory caps) locally before assuming the failure is environment-specific noise.

---

## 12. Interview Preparation

### 12.1 Beginner Level

**Q1. Why use `ThreadLocal<WebDriver>` instead of a single shared `WebDriver` field?**
A: A shared field means every thread drives the *same* browser session concurrently, causing random interleaved commands. `ThreadLocal` gives each executing thread its own isolated driver instance, keyed by thread identity, eliminating cross-thread interference by construction.

**Q2. What is the purpose of a `ConfigManager`?**
A: It centralizes every environment-specific value (URLs, browser, timeouts, Grid URL) behind one API, resolved from a layered source (system property → env var → properties file → default), so the same test code runs unchanged across local, QA, and CI environments.

**Q3. Why is `System.out.println` discouraged in a test framework?**
A: It produces unstructured, unlevelled output that cannot be filtered by severity or routed to centralized log aggregation (Splunk/ELK); SLF4J + Logback gives leveled, structured, and (with MDC) thread-attributable logs instead.

**Q4. For the framework's `Timeouts` class, why use a plain constants class instead of an enum?**
A: `EXPLICIT_WAIT`, `PAGE_LOAD_TIMEOUT`, and `POLLING_INTERVAL` are independent, unrelated values each consumed by name at its own call site — nobody iterates over "all timeouts" or switches on one. A plain `public static final` field (`Timeouts.EXPLICIT_WAIT`) is direct, zero-indirection access; an enum would add unused machinery (`values()`, `ordinal()`, `valueOf()`) built for representing a closed, enumerable set of "kinds," which this isn't. Enum becomes the better choice only once the framework needs runtime lookup by name, iteration/switching, or per-constant behavior — e.g., a config-backed `TimeoutType` enum resolving per-environment overrides (Section 3.4.3).

### 12.2 Intermediate Level

**Q5. Walk through what happens if you forget to call `ThreadLocal.remove()` after `driver.quit()` in a parallel suite.**
A: JUnit 5's parallel engine reuses pooled OS threads across test classes. The `ThreadLocalMap` entry for that thread still holds a reference to the now-dead `WebDriver` (not `null`). The next test scheduled on that same pooled thread calls `getDriver()`, its null-check passes because the reference is non-null, and it receives a dead session — failing immediately with `NoSuchSessionException`.

**Q6. How would you correlate log lines from ten concurrently running tests in a single console stream?**
A: Use SLF4J's MDC to tag a `testId` key per thread in a `BeforeEachCallback`, include `%X{testId}` in the Logback pattern, and clear it in `AfterEachCallback` — this makes every log line automatically attributable to its originating test without changing any individual log statement.

**Q7. What's the difference between JUnit 5's `mode.default` and `mode.classes.default` parallel settings?**
A: `mode.default` controls whether **methods within the same class** may run concurrently with each other; `mode.classes.default` controls whether **separate test classes** may run concurrently with each other — both must be set to `concurrent` for full-suite parallelism.

**Q8. When would you use `@ResourceLock` in an otherwise fully parallel suite?**
A: When two or more tests touch genuinely shared, mutable state — a shared test account's cart, a shared config file written at runtime — declaring the same `@ResourceLock` key on those tests serializes them relative to each other while leaving the rest of the suite fully concurrent.

**Q9. What is the difference between `@ExtendWith` and `@RegisterExtension`, and when is `@RegisterExtension` required rather than optional?**
A: `@ExtendWith(SomeExtension.class)` is declarative — JUnit constructs the extension itself, which requires a no-argument constructor. `@RegisterExtension` is programmatic — the test class instantiates the extension on a field, so it can be given constructor arguments (a config-resolved directory, a threshold, a mock) and the test class retains a direct handle to that exact instance. `@RegisterExtension` is *required*, not just preferred, whenever the extension's constructor takes parameters — `@ExtendWith` has no mechanism to pass them (Section 3.5.7). Both styles invoke the same lifecycle callbacks (`beforeEach`, `afterEach`, etc.) at the same points; only the *construction/registration* mechanism differs.

**Q10. If a `@RegisterExtension` field implements `BeforeEachCallback` and the test class also declares its own `@BeforeEach` method, what is the execution order, and why does it matter?**
A: JUnit 5 runs the extension's `beforeEach()` callback before the class's own `@BeforeEach` method executes for that test. This matters whenever the class's `@BeforeEach` logic depends on state the extension sets up first — in this framework's `BaseTest` (Section 9.11), `TestLifecycleExtension.beforeEach()` tags the SLF4J MDC `testId` before `logTestStart()`'s own `@BeforeEach` runs, guaranteeing that log line is already correctly attributed to the right test.

### 12.3 Advanced Level

**Q11. Design the precedence order for a configuration system used across local dev, QA, and multiple CI pipelines. Justify the order.**
A: System property (CLI override) highest, then environment variable, then an environment-specific properties file (`config-qa.properties`), then a default properties file, then a hardcoded fallback. This order lets CI override any value at invocation time without editing files, while still giving a sensible, versioned default when nothing is overridden — matching how most CI systems (Jenkins parameters, GitHub Actions env/secrets) already inject configuration.

**Q12. Explain the architectural difference between Selenium Grid 3's Hub/Node model and Grid 4's Router/Distributor/SessionMap/Node model.**
A: Grid 3's Hub translated JSON Wire Protocol calls to whatever the target Node's driver expected, adding a serialization/translation layer. Grid 4 is fully W3C-native end-to-end: the Router forwards new-session requests to the Distributor (which assigns based on advertised Node capacity/capabilities), the Session Map tracks `sessionId → Node` for routing ongoing commands, and Nodes speak native W3C directly — removing the translation layer and its associated capability-mismatch bugs.

**Q13. How do you decide the right JUnit 5 parallel thread count for a CI agent?**
A: Benchmark the CI agent's available memory divided by the measured per-browser-session memory footprint (typically 200–400MB for headless Chrome), rather than relying purely on `availableProcessors()` — vCPU count often doesn't correlate with memory ceiling on shared CI infrastructure. Use `config.strategy=fixed` with an explicit `config.fixed.parallelism` derived from that calculation rather than a factor-of-cores dynamic strategy on constrained agents.

### 12.4 Architect Level

**Q14. Design a framework where the same test suite runs against local Chrome, a self-hosted Docker Grid, and a cloud vendor with zero test-code changes. What are the key seams?**
A: (1) `DriverFactory` is the only class that knows how to construct a driver, branching purely on `ConfigManager`-resolved flags (`grid.enabled`, `grid.url`, `env`); (2) `*OptionsBuilder` classes isolate browser-specific flags from `DriverFactory`; (3) vendor-specific capabilities are merged onto the same `ChromeOptions`/`FirefoxOptions` object via `setCapability(vendorNamespaceKey, map)`, never via a parallel code path; (4) `BasePage`/`BaseComponent` are the only classes touching `WebDriver` directly, so nothing above the driver layer needs to know whether it's local, Grid, or cloud.

**Q15. How would you architect logging, screenshots, and reporting so that swapping Allure for ExtentReports later touches the minimum number of files?**
A: Centralize all three concerns inside a single JUnit 5 extension (`TestLifecycleExtension`, Section 9.9) implementing `BeforeEachCallback`/`AfterEachCallback`/`TestWatcher`. Page objects and tests never call the reporting API directly — they only produce log statements and let the extension's `testFailed()` hook capture the screenshot and attach it. Swapping the reporting backend means editing the attachment call inside this one extension class, with zero changes to any page object or test.

**Q16. A 200-engineer org runs a 6,000-test Selenium suite. What combination of scaling techniques would you apply, and in what order would you diagnose a sudden increase in flaky failures after enabling more parallelism?**
A: Scaling: CI matrix sharding (coarse split across many agents) + JUnit 5 native parallel within each shard + a horizontally-scaled Docker/Grid 4 backend so no single host's CPU/RAM caps concurrency. Diagnosis order: (1) check whether new flakes cluster around specific tests sharing state — apply `@ResourceLock`; (2) check CI agent memory pressure against the new session count — the `dynamic.factor` may now be spawning more sessions than RAM supports; (3) check for `NoSuchSessionException` specifically, which almost always indicates a `ThreadLocal.remove()` gap surfaced only now that thread-pool reuse is happening at higher volume; (4) check Docker Node `--shm-size` if failures correlate with the Docker-backed shard specifically.

### 12.5 Frequently Asked in Indian Product & Service Companies

*(TCS, Infosys, Cognizant, Accenture, Capgemini, Wipro, LTIMindtree, Zoho, Freshworks, Amazon India, Microsoft India, Oracle, ThoughtWorks, EPAM)*

**Q17 (Service companies — TCS/Infosys/Cognizant/Wipro/Capgemini, framework-design staple).** *"How do you manage test data for a data-driven Selenium framework used by both automation engineers and manual QA?"*
A: JSON + POJO deserialization for structured data engineers maintain, with a CSV option for simple flat datasets manual QA can edit directly in a spreadsheet tool — both feeding JUnit 5's `@ParameterizedTest` via `@MethodSource`/`@CsvFileSource`, keeping data fully external to test and page-object code (Section 3.4.5).

**Q18 (Accenture/LTIMindtree, common scenario question).** *"Your suite runs fine sequentially but fails intermittently once you enable parallel execution. How do you triage?"*
A: First check for shared mutable state (shared accounts, shared files) and add `@ResourceLock`; second, check for `NoSuchSessionException`, which points to a `ThreadLocal.remove()` gap; third, check CI agent memory against the configured parallel factor, since over-provisioned concurrency can crash sessions under memory pressure rather than failing predictably.

**Q19 (Zoho/Freshworks — product companies, infra-depth focus).** *"Why would you choose Selenium Grid 4 with Docker over just scaling cloud vendor usage, or vice versa?"*
A: Self-hosted Docker Grid gives full control over Node count/capability mix, no per-session vendor cost, and keeps traffic inside the corporate network — ideal for high-volume CI regression runs. Cloud vendors are preferable for genuine cross-browser/cross-OS/real-device matrix coverage that would be expensive or impractical to self-host. Many enterprise setups use both: Docker Grid for the bulk of CI regression, cloud vendor for a smaller, scheduled cross-browser compatibility pass — both reachable through the identical `DriverFactory` abstraction.

**Q20 (Amazon India/Microsoft India — scale and reliability focus).** *"How do you ensure log output remains diagnosable when running thousands of tests in parallel?"*
A: MDC-based thread correlation (`testId` tagged per test, included in the Logback pattern) so every log line is attributable regardless of interleaving, combined with per-test log files (via a `SiftingAppender`) for the largest suites, so a CI artifact can attach exactly the right log slice to exactly the right failed test result.

**Q21 (Oracle/ThoughtWorks/EPAM — architecture and design-quality focus).** *"Critique a framework where `@Test` methods contain `if (gridEnabled) {...} else {...}` branches to choose local vs remote execution."*
A: This is an architectural smell — the local/Grid/cloud decision leaks into test code, meaning every test author must understand and maintain execution-target branching logic that has nothing to do with the business flow under test. It belongs entirely inside `DriverFactory`, driven by configuration, so test code is 100% portable across execution targets with zero changes (Section 3.5.6, Q14 above).


---

## 13. Practice

### 13.1 Exercise Set

1. **Refactor to a full framework core.** Given a flat script with inline `driver.findElement()` calls and a hardcoded base URL, refactor it into `LoginPage`/`HomePage` extending `BasePage`, a `ConfigManager`-backed base URL, and a `ThreadLocal`-backed `DriverFactory`.
   - **Acceptance criteria:** no `@Test` method contains a `By` locator, a raw `driver.findElement()` call, or a hardcoded URL; running `mvn test -Dbase.url=https://staging.example.com` changes the target environment with no code edits.
   - **Test data:** any public demo login page (e.g., a Saucedemo-style app); credentials `standard_user` / `secret_sauce`.

2. **Add MDC-correlated logging.** Instrument `BasePage` methods with `log.debug(...)` calls, wire up `logback.xml` with the `%X{testId}` pattern, and implement the `beforeEach`/`afterEach` MDC tagging extension.
   - **Acceptance criteria:** running two tests in parallel produces a log file where every line is unambiguously attributable to its originating test by the `[testId]` tag.

### 13.2 Mini Assignment — Build a `DriverFactory` with Config-Driven Local/Grid Switching

Build a `DriverFactory` supporting Chrome, Firefox, and Edge, switchable via `-Dbrowser`, with a `-Dgrid.enabled=true` flag routing to a `RemoteWebDriver` pointed at a locally-run `selenium/standalone-chrome` Docker container instead of a local browser — with **zero test-code differences** between the two modes.

- **Acceptance criteria:** `mvn test` runs local Chrome; `docker run -d -p 4444:4444 --shm-size=2g selenium/standalone-chrome` followed by `mvn test -Dgrid.enabled=true` runs the identical test suite against the container with no code changes; a unit test verifies that calling `getDriver()` twice after `quitDriver()` creates a genuinely new session rather than reusing a dead reference.
- **Test data:** N/A (infrastructure-only); validate against any publicly reachable demo site.

### 13.3 Challenge Exercise — Parallelize Safely with Resource Locks, Reporting, and a Docker Grid

Take the refactored suite from 13.1–13.2 and: (a) enable JUnit 5 parallel execution across at least 4 test classes; (b) identify one test that must not run concurrently with another (simulate shared state via a static counter file) and protect it with `@ResourceLock`; (c) wire in the `TestLifecycleExtension` for MDC logging + screenshot-on-failure + Allure attachment; (d) point the run at a Docker Compose Hub-Node Grid (Section 3.5.5) instead of local browsers.

- **Acceptance criteria:** all 4+ classes pass consistently across 5 consecutive full-suite runs against the Docker Grid; the `@ResourceLock`-protected tests never interleave (verified by log timestamps showing no overlap); a deliberately broken locator in one page class produces exactly one failing test with a correctly attributed screenshot and Allure attachment; no `NoSuchSessionException` appears in any run.
- **Test data:** 4 distinct user accounts/flows, one per parallel test class, to avoid shared-state collisions.


---

## 14. Summary

This chapter established the Page Object Model as a layered architecture (Test → Page → Component → Base → Driver), then concentrated most of its depth on the **framework architecture surrounding those page objects**: a `DriverFactory` backed by `ThreadLocal<WebDriver>` for safe parallel execution, a layered `ConfigManager` with a documented precedence order, a `Constants` layer, a stateless `utils` package, a JSON/POJO-based test-data-management strategy integrated with JUnit 5's `@ParameterizedTest`, and SLF4J/Logback logging with MDC thread correlation so concurrent test output remains fully attributable. On top of that core, we covered reporting (Allure vs ExtentReports), screenshot-on-failure via a single consolidated JUnit 5 extension, JUnit 5 parallel-execution tuning (modes, resource locks, thread-count sizing), Selenium Grid 4's Router/Distributor/SessionMap/Node architecture, Dockerized Selenium (standalone and Hub-Node topologies with video recording), and cloud-vendor execution — all unified behind the same `DriverFactory` so local, Grid, and cloud execution are pure configuration switches with zero test-code branching.

## 15. Revision Notes

- POM layering: Test → Page → Component → BasePage/BaseComponent → DriverFactory.
- `ThreadLocal<WebDriver>` isolates per-thread driver instances; always pair `quit()` with `remove()` to avoid `NoSuchSessionException` on reused pooled threads.
- Configuration precedence: system property → env var → env-specific properties file → default properties file → hardcoded fallback.
- Constants and utilities live in dedicated, stateless layers — never duplicated magic numbers, never coupled to `BasePage`.
- Constants layer choice: plain constants class for independent, by-name values (the default); config-backed enum only once runtime lookup, iteration/switching, or per-constant behavior (e.g., per-environment overrides) is genuinely needed.
- Test data: JSON+POJO for structured data, CSV for simple flat data, both feeding `@ParameterizedTest`.
- MDC (`%X{testId}`) makes concurrent parallel log output fully attributable per test.
- Allure vs ExtentReports: annotation-driven + separate report generation vs programmatic API + self-contained HTML.
- JUnit 5 parallel: `mode.default` (methods) and `mode.classes.default` (classes) must both be `concurrent`; use `@ResourceLock` for genuinely shared state; size thread count against CI memory, not just core count.
- Grid 4 = Router + Distributor + Session Map + Node, fully W3C-native, no Hub-style translation layer.
- Docker Selenium: always set `--shm-size=2g`+; Hub-Node Compose topology scales horizontally; video sidecars aid CI-only failure diagnosis.
- Cloud vendors reuse the identical `RemoteWebDriver` path as Grid — only the hub URL and a vendor capability namespace differ.
- `@ExtendWith` = JUnit constructs the extension (no-arg only); `@RegisterExtension` = your code constructs it (constructor args allowed, and the test class keeps a handle to that instance). Both fire the same lifecycle callbacks (`beforeEach`, `afterEach`, etc.) at the same points — only construction/registration differs.

## 16. Common Mistakes Checklist

- [ ] Sharing a `static WebDriver` field across parallel test threads
- [ ] Forgetting `ThreadLocal.remove()` after `driver.quit()`, causing `NoSuchSessionException` on reused pooled threads
- [ ] Hardcoding environment URLs/credentials directly in tests or page objects instead of `ConfigManager`
- [ ] Committing secrets into a properties file in source control
- [ ] Duplicating magic numbers/timeouts across dozens of files instead of a `Constants` class
- [ ] Reaching for an enum by default for framework constants when a plain constants class would be simpler (or the reverse — hand-rolling name-based lookup/switch logic instead of using an enum once that's genuinely needed)
- [ ] Using `System.out.println` instead of SLF4J/Logback, losing structured/leveled output
- [ ] Enabling parallel execution without auditing for shared mutable state, instead of using `@ResourceLock`
- [ ] Sizing JUnit 5 parallel thread count off core count alone rather than measured CI memory per browser session
- [ ] Running Dockerized Selenium with the default (too-small) `--shm-size`, causing intermittent renderer crashes
- [ ] Branching test code on "if Grid then X else Y" instead of hiding execution target entirely behind `DriverFactory` + configuration
- [ ] No screenshot/log/video correlation strategy, making CI-only flakes nearly undiagnosable after the fact
- [ ] Trying to use `@ExtendWith` for an extension that needs constructor parameters instead of `@RegisterExtension`

## 17. Key Takeaways

1. **A `ThreadLocal`-backed `DriverFactory`, paired with disciplined `remove()` on teardown, is the non-negotiable foundation for safe parallel Selenium execution.**
2. **Configuration, constants, utilities, test data, and logging deserve their own dedicated, independently testable layers — never folded into page objects or scattered across the codebase.**
3. **MDC-based logging correlation is what makes parallel test output diagnosable; without it, concurrency and debuggability are in direct tension.**
4. **Selenium Grid 4's native W3C architecture and Dockerized Node topologies let a framework scale horizontally without any test-code changes, as long as `DriverFactory` — not the tests — owns the local/Grid/cloud decision.**
5. **The mark of an enterprise-grade framework is that local, Docker Grid, and cloud-vendor execution are all reachable through identical test code, differing only in configuration.**

