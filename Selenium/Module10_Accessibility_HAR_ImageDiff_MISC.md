# Chapter 14: Beyond Functional Testing — Accessibility, Visual, Performance, Security & Localization Testing with Selenium

**Level: Architect / Senior Automation Engineer**

> Prerequisite chapters assumed: WebDriver fundamentals, locators & waits, Page Object Model, TestNG/JUnit5 integration, CI/CD pipelines, and Selenium Grid/Cloud execution. This chapter assumes a working, well-architected Selenium 4 + Java 21 + Maven framework already exists and extends it into non-functional quality engineering.

---

## 1. Title & Scope

**Chapter Title:** Non-Functional Quality Engineering with Selenium — Accessibility, Visual Regression, Performance (CDP/Lighthouse/HAR), Security Basics, and Localization/i18n

**Level:** Advanced → Architect

This chapter treats Selenium not merely as a functional-DOM-interaction tool but as an **orchestration layer** sitting on top of the W3C WebDriver Protocol and Chrome DevTools Protocol (CDP), capable of driving specialized non-functional test disciplines that Test Architects are expected to own in mature engineering organizations (Google, Microsoft, Amazon, Netflix-style QE orgs).

---

## 2. Skills Covered

- Automating accessibility audits (axe-core, ARIA, keyboard-only navigation) inside a Selenium suite
- Implementing visual regression testing (pixel-diff vs perceptual/tolerant diff) with Selenium-driven screenshot capture
- Extracting performance metrics via Chrome DevTools Protocol (CDP) through Selenium 4's native `DevTools` class
- Running Lighthouse audits headlessly and correlating results with Selenium test runs
- Capturing and asserting on HAR (HTTP Archive) files for network-level test validation
- Validating CORS and CSP headers programmatically as part of a security smoke suite
- Basic TLS/certificate handling and certificate-pinning validation strategies in automated browsers
- Designing a locale-matrix-driven i18n/l10n Selenium framework (RTL, unicode, date/currency formats)
- Architecting a unified "Quality Gate" framework that merges functional + non-functional signals into one CI report

---

## 3. Learning Objectives

By the end of this chapter, the learner will be able to:

1. Explain the internal communication difference between the W3C WebDriver Protocol and CDP, and justify when each is used.
2. Integrate axe-core with Selenium and assert zero critical/serious violations as a build-blocking gate.
3. Design and implement a visual regression pipeline with baseline management, tolerant diffing, and CI screenshot artifacts.
4. Use Selenium 4's `DevTools` API to capture `Performance`, `Network`, and `Log` domain events without external tools.
5. Trigger a Lighthouse CI run against a Selenium-navigated page and fail a build on a performance-score threshold.
6. Capture a HAR file during a Selenium session and assert on request/response-level details (status codes, headers, timing).
7. Assert CORS and CSP headers using Selenium + CDP `Network.responseReceived` events.
8. Detect certificate/TLS anomalies in an automated Chrome session and explain the limits of Selenium in true cert-pinning validation.
9. Architect a data-driven localization test suite that swaps locale, verifies RTL layout mirroring, and validates translated content without hardcoding strings.
10. Combine all five disciplines into a single Architect-level "Quality Gate" Maven module usable in a CI/CD pipeline.

---

## 4. Architecture Overview — Where This Fits

```
                    ┌────────────────────────────────────────────┐
                    │           CI/CD Pipeline (Jenkins/GHA)      │
                    └───────────────────┬──────────────────────--┘
                                        │
                     ┌──────────────────┴───────────────────┐
                     │         Test Orchestration Layer       │
                     │   (JUnit5 Suite + Maven Surefire)      │
                     └───┬─────────┬─────────┬────────┬──────┘
                         │         │         │        │
              ┌──────────▼──┐ ┌────▼───┐ ┌───▼───┐ ┌──▼─────────┐
              │Functional   │ │A11y    │ │Visual │ │Perf/Sec/i18n│
              │ (existing)  │ │(axe-   │ │(diff  │ │(CDP/HAR/    │
              │ WebDriver   │ │ core)  │ │engine)│ │ Lighthouse) │
              └──────┬──────┘ └───┬────┘ └───┬───┘ └──────┬─────┘
                     │            │          │            │
                     └────────────┴────┬─────┴────────────┘
                                        │
                          ┌─────────────▼─────────────┐
                          │     Selenium WebDriver      │
                          │  (W3C Protocol + CDP layer) │
                          └─────────────┬──────────────┘
                                        │
                          ┌─────────────▼──────────────┐
                          │   Browser (Chrome/Edge)      │
                          │  DevTools Protocol Endpoint   │
                          └───────────────────────────────┘
```

### 4.1 Protocol-Level Distinction

```
  Selenium 3 (JSON Wire Protocol - deprecated)
  ┌────────┐   HTTP/JSON   ┌───────────┐
  │ Client │ ─────────────▶│ Driver EXE │──▶ Browser (custom bridging)
  └────────┘                └───────────┘

  Selenium 4 (W3C WebDriver Protocol - standardized)
  ┌────────┐   HTTP/JSON (W3C spec)   ┌───────────┐
  │ Client │ ────────────────────────▶│ Driver EXE │──▶ Browser (native)
  └────────┘                          └───────────┘
        │
        │  Bidirectional CDP WebSocket (Selenium 4 only)
        ▼
  ┌───────────────┐   ws://localhost:PORT/devtools/...  ┌─────────┐
  │ DevTools class│ ◀────────────────────────────────────▶│ Browser │
  └───────────────┘                                       └─────────┘
```

**What / Why / When / Where / How**

| Aspect | W3C WebDriver Protocol | Chrome DevTools Protocol (CDP) |
|---|---|---|
| What | Standard HTTP/JSON protocol for element interaction, navigation, cookies | Native browser instrumentation protocol (WebSocket-based) |
| Why | Cross-browser interoperability (Chrome, Firefox, Edge, Safari) | Deep browser internals: network, performance, console, security |
| When | All standard functional automation | Non-functional needs: network capture, perf metrics, console logs, emulation |
| Where | `driver.findElement()`, `driver.get()`, `driver.manage()` | `((HasDevTools) driver).getDevTools()` in Selenium 4 |
| How | Driver process translates commands to browser automation hooks | Direct WebSocket session to the browser's debugging port |

Selenium 3 had **no native CDP support** — engineers used third-party libs (`chrome-devtools-java-client`) or Chrome's remote-debugging port directly. Selenium 4 exposes CDP natively via `org.openqa.selenium.devtools.DevTools`, which is the backbone for everything in Sections 6–9 of this chapter.

---

## 5. Accessibility Testing (A11y)

### 5.1 What, Why, When, Where, How

- **What:** Automated verification that a page conforms to WCAG 2.1/2.2 AA rules — ARIA roles, contrast, focus order, alt text, label associations.
- **Why:** Legal compliance (ADA, EN 301 549, WCAG), inclusive design, and it is far cheaper to catch in CI than in a legal audit.
- **When:** On every PR touching UI templates; also as a scheduled full-site crawl.
- **Where:** Component level (unit-like) and page level (E2E) — Selenium handles the E2E layer.
- **How:** Inject `axe-core` JavaScript into the live DOM via Selenium's JavascriptExecutor, run it, and parse the JSON violations.

### 5.2 Architecture — Axe-Core + Selenium Flow

```
 ┌───────────┐   1. driver.get(url)   ┌───────────┐
 │ JUnit Test │ ─────────────────────▶│  Browser   │
 └─────┬─────┘                        └─────┬─────┘
       │ 2. executeScript(axe.min.js)         │
       │──────────────────────────────────────▶│ (axe injected into page context)
       │ 3. executeScript("return axe.run()")  │
       │──────────────────────────────────────▶│
       │◀──────────────────────────────────────│ 4. JSON violations returned
       │ 5. Parse violations → assert count==0 │
       ▼
 ┌───────────┐
 │  Report    │  (JSON/HTML artifact attached to CI build)
 └───────────┘
```

### 5.3 Approaches Compared

| Approach | Description | Pros | Cons | Recommended For |
|---|---|---|---|---|
| Raw JS injection (`executeScript`) | Manually load axe-core.min.js and call `axe.run()` | No extra dependency beyond a JS file; full control | Manual JSON parsing, no Java typing | Lightweight frameworks |
| `com.deque.html.axe-core:selenium` (official Java binding) | Official Deque wrapper (`AxeBuilder`) | Typed Java API, active maintenance, exclude/include rules | Slight version coupling to axe-core releases | **Recommended — enterprise default** |
| Third-party accessibility scanners (Pa11y, Lighthouse a11y category) via CLI | Shell out to Node-based scanner | Very mature ruleset | Breaks single-JVM pipeline, needs Node runtime | Cross-team shared audits |
| Manual keyboard-navigation Selenium scripts (`Keys.TAB` traversal) | Programmatically simulate Tab order and assert focus visibility | Catches keyboard-trap bugs axe misses | Slow, brittle, high maintenance | Supplementary, not primary |

**Recommendation:** Use the official `axe-core-maven` / `axe-selenium-java` binding (`AxeBuilder`) as the default engine, supplemented with a small custom keyboard-navigation suite for focus-trap detection, which static analyzers cannot reliably catch.

### 5.4 Java Implementation (Skeleton)

```java
package com.architect.qe.accessibility;

import com.deque.html.axecore.selenium.AxeBuilder;
import com.deque.html.axecore.results.Results;
import com.deque.html.axecore.results.Rule;
import org.junit.jupiter.api.*;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;

import java.util.List;
import java.util.stream.Collectors;

import static org.junit.jupiter.api.Assertions.assertTrue;

class HomePageAccessibilityTest {

    private WebDriver driver;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver();
        driver.get("https://example.com");
    }

    @Test
    @DisplayName("Home page must have zero critical/serious A11y violations")
    void homePageHasNoSeriousViolations() {
        Results results = new AxeBuilder()
                .withTags(List.of("wcag2a", "wcag2aa"))
                .analyze(driver);

        List<Rule> blocking = results.getViolations().stream()
                .filter(v -> v.getImpact().equals("critical") || v.getImpact().equals("serious"))
                .collect(Collectors.toList());

        assertTrue(blocking.isEmpty(),
                "Blocking A11y violations found: " + blocking.size());
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

### 5.5 Pitfalls & Anti-Patterns

- Running axe **before** dynamic content (SPA) has finished rendering → false negatives. Always wait for a stable DOM signal first.
- Treating axe's "moderate/minor" findings as build-blockers → alert fatigue; gate only on critical/serious.
- Ignoring iframe boundaries — axe-core does not automatically traverse cross-origin iframes; needs explicit frame switching.
- Assuming automated tools catch everything — axe covers ~30-50% of WCAG success criteria; manual review is still required for cognitive/contextual criteria.

### 5.6 Best Practices

- Bake accessibility gates into PR pipelines, not just nightly runs.
- Maintain an allow-list of pre-approved, tracked violations (with ticket links) rather than suppressing silently.
- Pair automated axe scans with a quarterly manual screen-reader (NVDA/JAWS/VoiceOver) audit.

---

## 6. Visual Regression Testing

### 6.1 What, Why, When, Where, How

- **What:** Detecting unintended visual/layout changes by comparing screenshots against an approved baseline.
- **Why:** CSS regressions, font-loading issues, and responsive breakpoints are invisible to DOM-based assertions.
- **When:** After every UI-affecting merge; especially valuable for design-system/component libraries.
- **Where:** Component-level snapshots (Storybook + Chromatic-style) and full-page E2E snapshots (Selenium-driven).
- **How:** Selenium captures a full-page or element screenshot; a diff engine compares pixel or perceptual difference against baseline.

### 6.2 Architecture

```
 ┌─────────────┐        ┌──────────────┐        ┌────────────────┐
 │  Selenium    │ screenshot │  Diff Engine   │ result │  Baseline Store   │
 │  WebDriver   │──────────▶│ (pixelmatch/   │───────▶│ (Git LFS / S3 /    │
 │              │           │  Resemble/     │        │  Applitools cloud) │
 │              │           │  Applitools)   │        └────────────────────┘
 └─────────────┘           └──────────────┘
        ▲                        │
        │                        ▼
        │                 ┌──────────────┐
        └─────────────────│  CI Report    │ (diff % + highlighted image)
                           └──────────────┘
```

### 6.3 Pixel-Diff vs Tolerant/Perceptual Diff

| Criterion | Pixel Diff | Tolerant / Perceptual Diff (AI-based) |
|---|---|---|
| Technique | Byte-level pixel comparison | Structural similarity (SSIM), ML-based visual understanding |
| False positives | High (1px anti-aliasing, font rendering) | Low — ignores sub-visual noise |
| Tooling | `pixelmatch`, `Resemble.js`, ImageMagick `compare` | Applitools Eyes, Percy, Chromatic |
| Cross-OS/browser stability | Poor (font rasterization differs) | Good |
| Cost | Free/open-source | Usually commercial SaaS |
| Best for | Fixed, tightly controlled render environments (Docker+headless Chrome pinned version) | Cross-platform, cross-browser visual testing at scale |

**Recommendation:** For an open-source, self-hosted architecture, pin the Chrome version in Docker and use `pixelmatch` with a configurable threshold (e.g., 0.1% pixel difference tolerance) rather than exact match. For enterprise-scale, multi-browser, multi-device visual testing, adopt a perceptual-diff SaaS (Applitools/Percy) — the ROI on reduced flakiness outweighs licensing cost.

### 6.4 Java Implementation (Skeleton — Selenium + java-pixelmatch style)

```java
package com.architect.qe.visual;

import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.junit.jupiter.api.*;

import java.io.File;
import java.nio.file.*;

import static org.junit.jupiter.api.Assertions.assertTrue;

class VisualRegressionTest {

    private WebDriver driver;
    private static final Path BASELINE_DIR = Paths.get("src/test/resources/baseline");
    private static final Path ACTUAL_DIR = Paths.get("target/visual-actual");

    @BeforeEach
    void setUp() throws Exception {
        driver = new ChromeDriver();
        Files.createDirectories(ACTUAL_DIR);
    }

    @Test
    void homePageVisualMatchesBaseline() throws Exception {
        driver.get("https://example.com");
        File actual = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
        Path actualPath = ACTUAL_DIR.resolve("home.png");
        Files.copy(actual.toPath(), actualPath, StandardCopyOption.REPLACE_EXISTING);

        Path baselinePath = BASELINE_DIR.resolve("home.png");
        double diffPercentage = ImageDiffUtil.compare(baselinePath, actualPath);

        assertTrue(diffPercentage < 0.5,
                "Visual diff exceeded threshold: " + diffPercentage + "%");
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

*(`ImageDiffUtil` is a thin wrapper around a pixel-diff library — implementation detail left to the framework's internal utils module.)*

### 6.5 Pitfalls

- Dynamic content (ads, timestamps, carousels) causes constant false failures — mask or freeze these regions before capture.
- Font rendering differs between CI runner OS and local dev OS — always generate baselines from the **same Docker image** used in CI.
- Full-page screenshots vs viewport screenshots behave differently across drivers — standardize on one and document it.

### 6.6 Best Practices

- Store baselines in version control alongside the code that produced them (baseline drift = code drift).
- Require a mandatory human-reviewed "approve baseline update" PR step — never auto-accept diffs.
- Mask dynamic regions declaratively (CSS selectors) rather than hardcoded pixel coordinates.

---

## 7. Performance Testing Hooks (CDP, Lighthouse, HAR)

### 7.1 What, Why, When, Where, How

- **What:** Capturing real browser performance signals (load time, TTFB, LCP, CLS, network waterfall) during a functional Selenium run, rather than relying only on synthetic load-testing tools (JMeter/Gatling).
- **Why:** Functional correctness is necessary but not sufficient — a page that "works" but takes 12 seconds to render fails the user.
- **When:** Smoke-level performance budgets on every build; deep Lighthouse audits nightly/weekly.
- **Where:** Selenium 4's `DevTools` class (`Performance`, `Network`, `Page` CDP domains).
- **How:** Attach a CDP session, enable the relevant domain, and either poll metrics or capture events into a HAR structure.

### 7.2 Architecture — CDP Performance Capture Flow

```
 ┌────────────┐   1. driver.getDevTools()          ┌────────────┐
 │  Test Code  │───────────────────────────────────▶│  DevTools   │
 └────────────┘                                      │  Session    │
        │  2. devTools.send(Performance.enable())    └─────┬──────┘
        │────────────────────────────────────────────────▶│
        │  3. devTools.send(Network.enable())               │
        │────────────────────────────────────────────────▶│
        │  4. driver.get(url)  (triggers browser events)    │
        │◀───────────────────────────────────────────────  │  (async events streamed)
        │  5. devTools.addListener(Network.responseReceived)│
        │  6. devTools.send(Performance.getMetrics())       │
        │◀───────────────────────────────────────────────  │
        ▼
 ┌─────────────┐
 │ Metrics/HAR  │  → JSON export → assert thresholds (LCP < 2.5s etc.)
 └─────────────┘
```

### 7.3 Approaches Compared

| Approach | Description | Pros | Cons | Recommended For |
|---|---|---|---|---|
| Native Selenium 4 `DevTools` (Performance/Network domains) | Java-native CDP calls | No extra process, tight test integration | Chrome/Edge only (Chromium-based) | **Default for CI perf smoke checks** |
| Lighthouse CI (Node) driven against the same URL | Run `lighthouse-ci` as a separate step, correlate by URL | Industry-standard scoring (0-100), rich audits | Separate toolchain (Node), not tied to live Selenium session state (e.g., post-login pages) | Nightly full audits, marketing/public pages |
| BrowserMob Proxy / HAR capture via proxy | Route Selenium traffic through a MITM proxy that records HAR | Works on any browser, not just Chromium | Extra moving part, proxy setup/cert trust needed | Cross-browser network validation |
| Native `Network.enable` + `Network.getResponseBody` events assembled into HAR in Java | Build HAR programmatically from CDP events | No proxy, no Node dependency, works within logged-in sessions | More code to maintain HAR-spec compliance | **Recommended for authenticated-flow perf/network tests** |

### 7.4 Java Implementation (Skeleton) — CDP Performance Metrics

```java
package com.architect.qe.performance;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v127.performance.Performance;
import org.openqa.selenium.devtools.v127.performance.model.Metric;

import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertTrue;

class PagePerformanceTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver();
        devTools = driver.getDevTools();
        devTools.createSession();
        devTools.send(Performance.enable(Optional.empty()));
    }

    @Test
    void homePageLoadTimeWithinBudget() {
        long start = System.currentTimeMillis();
        driver.get("https://example.com");
        long domContentLoaded = System.currentTimeMillis() - start;

        List<Metric> metrics = devTools.send(Performance.getMetrics());
        double jsHeapUsed = metrics.stream()
                .filter(m -> m.getName().equals("JSHeapUsedSize"))
                .mapToDouble(Metric::getValue)
                .findFirst()
                .orElse(0);

        assertTrue(domContentLoaded < 3000,
                "Page load exceeded 3s budget: " + domContentLoaded + "ms");
        assertTrue(jsHeapUsed < 50_000_000,
                "JS heap usage exceeded 50MB budget: " + jsHeapUsed);
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

### 7.5 HAR Capture (Conceptual Java Skeleton)

```java
package com.architect.qe.performance;

import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v127.network.Network;
import org.openqa.selenium.devtools.v127.network.model.*;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class HarCaptureListener {

    private final ConcurrentHashMap<String, RequestWillBeSentParams> requests = new ConcurrentHashMap<>();

    public void attach(DevTools devTools) {
        devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));

        devTools.addListener(Network.requestWillBeSent(),
                req -> requests.put(req.getRequestId().toString(), req));

        devTools.addListener(Network.responseReceived(), res -> {
            RequestWillBeSentParams reqInfo = requests.get(res.getRequestId().toString());
            if (reqInfo != null) {
                int status = res.getResponse().getStatus();
                String url = res.getResponse().getUrl();
                // Assemble into HAR-spec entry: url, status, timing, headers
                System.out.printf("URL=%s STATUS=%d%n", url, status);
            }
        });
    }
}
```

### 7.6 Lighthouse Integration (Process-Level Skeleton)

```java
package com.architect.qe.performance;

import java.io.*;
import java.util.concurrent.TimeUnit;

public class LighthouseRunner {

    public double runAndGetPerformanceScore(String url) throws IOException, InterruptedException {
        ProcessBuilder pb = new ProcessBuilder(
                "lighthouse", url,
                "--output=json",
                "--output-path=./target/lighthouse-report.json",
                "--chrome-flags=--headless=new");
        pb.redirectErrorStream(true);
        Process process = pb.start();
        boolean finished = process.waitFor(2, TimeUnit.MINUTES);
        if (!finished) process.destroyForcibly();
        // Parse ./target/lighthouse-report.json → categories.performance.score
        return LighthouseReportParser.extractPerformanceScore("./target/lighthouse-report.json");
    }
}
```

### 7.7 Pitfalls

- CDP domain classes are **version-pinned** (`org.openqa.selenium.devtools.v127.*`) — a Chrome upgrade without a matching Selenium/CDP-package upgrade breaks the build. Always align Selenium version ↔ Chrome version ↔ CDP package.
- CDP is Chromium-only — Firefox/Safari require BiDi (Selenium 4's newer WebDriver BiDi protocol) or a proxy-based HAR approach instead.
- Measuring performance on a cold cache vs warm cache gives wildly different, non-comparable numbers — always specify and control cache state.
- Running perf tests on shared, noisy CI runners produces unreliable absolute numbers — prefer **relative regression thresholds** (vs last baseline) over fixed absolute budgets when infra is not perf-isolated.

### 7.8 Best Practices

- Separate "perf smoke" (fast, every PR, CDP-based) from "perf deep audit" (Lighthouse, nightly, dedicated hardware).
- Track performance metrics as a time-series (not just pass/fail) to catch slow regressions.
- Use WebDriver BiDi (`org.openqa.selenium.bidi`) for future cross-browser network/perf capture as CDP-only support becomes a technical debt liability.

---

## 8. Security Testing Basics (CORS, CSP, Certificate Handling)

### 8.1 What, Why, When, Where, How

- **What:** Lightweight, automatable checks that verify security headers and TLS posture are correctly configured — **not** a replacement for penetration testing or DAST/SAST tools.
- **Why:** Misconfigured CORS/CSP is one of the most common production security regressions, and it's cheap to catch via automated header assertions.
- **When:** Every deployment to staging/production-like environments.
- **Where:** CDP `Network.responseReceived` headers, and Java's `HttpsURLConnection`/`SSLContext` for certificate inspection.
- **How:** Capture response headers via CDP or direct HTTP client calls alongside the Selenium session and assert expected header values.

### 8.2 CORS & CSP Validation

```
 Browser Request                     Server Response Headers (asserted)
 ───────────────                     ───────────────────────────────────
 Origin: https://app.example.com     Access-Control-Allow-Origin: https://app.example.com
                                      Access-Control-Allow-Methods: GET, POST
                                      Content-Security-Policy: default-src 'self';
                                                                 script-src 'self' cdn.example.com
                                      Strict-Transport-Security: max-age=63072000
```

```java
package com.architect.qe.security;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v127.network.Network;

import java.util.Optional;
import java.util.concurrent.atomic.AtomicReference;

import static org.junit.jupiter.api.Assertions.*;

class SecurityHeaderTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver();
        devTools = driver.getDevTools();
        devTools.createSession();
        devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));
    }

    @Test
    void mainDocumentHasRequiredSecurityHeaders() {
        AtomicReference<String> csp = new AtomicReference<>();
        AtomicReference<String> hsts = new AtomicReference<>();

        devTools.addListener(Network.responseReceived(), res -> {
            if (res.getResponse().getUrl().equals("https://example.com/")) {
                res.getResponse().getHeaders().toJson().forEach((k, v) -> {
                    if (k.equalsIgnoreCase("content-security-policy")) csp.set(String.valueOf(v));
                    if (k.equalsIgnoreCase("strict-transport-security")) hsts.set(String.valueOf(v));
                });
            }
        });

        driver.get("https://example.com/");

        assertNotNull(csp.get(), "Content-Security-Policy header missing");
        assertNotNull(hsts.get(), "Strict-Transport-Security header missing");
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

### 8.3 Certificate / TLS Basics

| Concept | What Selenium Can Do | What Selenium Cannot Do |
|---|---|---|
| Detect invalid/self-signed cert | `driver.get()` will fail/warn; can assert on Chrome's "Not Secure" interstitial via CDP `Security.certificateError` events | Perform real certificate-pinning validation (that's a mobile-app/native-client concern, not browser-DOM level) |
| Inspect cert chain details | Via CDP `Network.getSecurityDetails` (subject, issuer, valid-from/to, protocol) | Cannot replace a dedicated TLS-scanner (e.g., `testssl.sh`, Qualys SSL Labs) for cipher-suite/vulnerability audits |
| Enforce HTTPS-only navigation in tests | `chromeOptions.addArguments("--ignore-certificate-errors")` for controlled test envs only | Should never be used against production-like security testing — defeats the purpose |

**Certificate Pinning Clarification:** Certificate pinning is fundamentally a **client-application-level control** (mobile apps, native clients binding to a specific cert/public key). A browser automated by Selenium uses the OS/browser trust store, not app-level pinning — so Selenium is the *wrong tool* for validating pinning behavior itself. Its role here is limited to verifying that the **server presents the expected certificate chain** (via CDP `Network.getSecurityDetails`), which is a necessary but not sufficient check.

### 8.4 Pitfalls

- Disabling cert errors globally (`--ignore-certificate-errors`) in a shared test profile silently masks real cert misconfigurations in later environments — scope this flag strictly to local/dev test targets.
- Treating a passing CORS/CSP header check as "secure" — headers can be present but misconfigured (e.g., `Access-Control-Allow-Origin: *` with credentials) — always assert *values*, not just *presence*.
- Security header testing via Selenium is a **smoke layer**, not a substitute for DAST tools (OWASP ZAP, Burp Suite) which should run in parallel in the pipeline.

### 8.5 Best Practices

- Maintain an explicit allow-list of expected header values per environment (staging headers often differ intentionally from prod).
- Fail the build on CSP `unsafe-inline`/`unsafe-eval` directives appearing in production configs.
- Pair this smoke layer with a scheduled OWASP ZAP baseline scan integrated into the same pipeline for deeper coverage.

---

## 9. Localization & Internationalization (i18n/l10n) Testing

### 9.1 What, Why, When, Where, How

- **What:** Verifying that UI text, layout direction, date/number/currency formats, and content correctly adapt per locale.
- **Why:** Hardcoded strings, truncated translations, and broken RTL mirroring are among the highest-frequency bugs in globalized products.
- **When:** Every release for markets currently supported; new-locale rollout requires a dedicated full pass.
- **Where:** Locale switch mechanism (URL param, cookie, `Accept-Language` header, in-app selector).
- **How:** Data-driven Selenium tests parameterized over a locale matrix, asserting against resource-bundle-sourced expected values (never hardcoded translated strings in test code).

### 9.2 Architecture — Locale-Matrix Test Design

```
                     ┌───────────────────────────┐
                     │   locales.csv / JSON        │
                     │  en-US, fr-FR, ar-SA, ja-JP  │
                     └─────────────┬─────────────┘
                                    │  JUnit5 @ParameterizedTest
                     ┌──────────────▼───────────────┐
                     │   LocaleAwareTest (parameterized) │
                     └──────────────┬───────────────┘
                                    │ sets Accept-Language / cookie via CDP
                     ┌──────────────▼───────────────┐
                     │      Selenium WebDriver         │
                     └──────────────┬───────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
     Assert translated text   Assert RTL layout      Assert date/currency
     (from resource bundle,   (dir="rtl", mirrored    format matches
      not hardcoded string)   nav/icons)              locale convention
```

### 9.3 Setting Locale via CDP (`--lang` vs `Emulation.setLocaleOverride`)

| Approach | Mechanism | Pros | Cons |
|---|---|---|---|
| ChromeOptions `--lang=fr-FR` at launch | Sets browser UI + `Accept-Language` at startup | Simple, works for full session | Requires a new driver instance per locale — slower matrix runs |
| CDP `Emulation.setLocaleOverride` (mid-session) | Changes locale on the fly | Fast matrix iteration without restarting driver | Chromium-only; doesn't affect OS-level locale (date pickers using OS locale won't shift) |
| Cookie/URL param locale switch (`?lang=ar`) | App-level i18n mechanism, most apps already support this | Fastest, closest to real user behavior, cross-browser | Only works if the app supports this switching mechanism |
| Full OS-level locale VM/container per test run | Spin up locale-specific containers | Most authentic (fonts, OS date pickers, RTL OS-level rendering) | Heavy infra cost, slow |

**Recommendation:** Use the app's native locale-switching mechanism (cookie/URL param) as the primary driver — it's fastest and most representative of real users — and reserve `--lang` ChromeOptions/CDP override only for verifying the browser's own `Accept-Language` negotiation behavior.

### 9.4 Java Implementation (Skeleton) — Parameterized Locale Matrix

```java
package com.architect.qe.i18n;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.By;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

import java.util.ResourceBundle;
import java.util.Locale;

import static org.junit.jupiter.api.Assertions.assertEquals;

class LocalizationTest {

    @ParameterizedTest
    @CsvSource({
            "en-US, en",
            "fr-FR, fr",
            "ar-SA, ar",
            "ja-JP, ja"
    })
    void welcomeBannerMatchesLocaleBundle(String locale, String bundleKey) {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--lang=" + locale);
        WebDriver driver = new ChromeDriver(options);
        try {
            driver.get("https://example.com/?lang=" + bundleKey);

            ResourceBundle bundle = ResourceBundle.getBundle("messages", Locale.forLanguageTag(locale));
            String expectedText = bundle.getString("welcome.banner");

            String actualText = driver.findElement(By.id("welcome-banner")).getText();
            assertEquals(expectedText, actualText,
                    "Mismatch for locale: " + locale);

            if (bundleKey.equals("ar")) {
                String dir = driver.findElement(By.tagName("html")).getAttribute("dir");
                assertEquals("rtl", dir, "RTL direction not applied for Arabic locale");
            }
        } finally {
            driver.quit();
        }
    }
}
```

### 9.5 Pitfalls

- Hardcoding expected translated strings directly in test code — breaks the moment translators update copy; always source expected values from the **same resource bundles the app uses** (single source of truth).
- Ignoring text-expansion issues — German/Finnish strings can be 30-40% longer than English; layout tests must check for truncation/overflow, not just presence of text.
- Testing RTL only via `dir="rtl"` attribute presence, without verifying visual mirroring (icons, nav order) — pair with a visual regression baseline per locale (Section 6).
- Using Latin-locale test data (e.g., email regex assuming ASCII) against Unicode input fields — validate Unicode/emoji/combining-character handling explicitly.

### 9.6 Best Practices

- Maintain the locale matrix as external data (CSV/JSON/DB), never inline in test code, so QA/localization teams can add locales without touching Java code.
- Combine i18n functional checks with a **visual regression pass per RTL locale** — this is the highest-value combination for catching real localization bugs.
- Automate pseudo-localization (e.g., `[!!! Ẁëlçömê !!!]`) as an early smoke check before real translations exist, to catch hardcoded/non-externalized strings.

---

## 10. Unified Architect-Level "Quality Gate" Framework

### 10.1 Class/Module Architecture

```
 com.architect.qe
 ├── core/
 │   ├── DriverFactory.java          (Selenium Manager-based driver bootstrap)
 │   ├── DevToolsSessionManager.java (wraps DevTools lifecycle)
 │   └── QualityGateConfig.java      (thresholds: perf budget, a11y severity, diff %)
 ├── accessibility/
 │   └── AxeAccessibilityRunner.java
 ├── visual/
 │   ├── ScreenshotCapture.java
 │   └── ImageDiffUtil.java
 ├── performance/
 │   ├── CdpMetricsCollector.java
 │   ├── HarCaptureListener.java
 │   └── LighthouseRunner.java
 ├── security/
 │   └── SecurityHeaderValidator.java
 ├── i18n/
 │   └── LocaleMatrixProvider.java
 └── reporting/
     └── QualityGateReportAggregator.java   (merges all 5 signals → single CI report)
```

### 10.2 Maven `pom.xml` — Key Dependency Additions (Illustrative)

```xml
<dependencies>
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>4.23.0</version>
    </dependency>
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-devtools-v127</artifactId>
        <version>4.23.0</version>
    </dependency>
    <dependency>
        <groupId>com.deque.html.axe-core</groupId>
        <artifactId>selenium</artifactId>
        <version>4.9.1</version>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 10.3 Complete Helper Class Implementations

Every helper class referenced by name in Sections 5–9 (`ImageDiffUtil`, `LighthouseReportParser`, etc.) is a **real, working class**, not a placeholder — implemented below in full so the chapter is copy-paste runnable end-to-end. Add `org.json:json` to your `pom.xml` for the JSON parsing used here (a deliberately lightweight, dependency-light choice over pulling in full Jackson just for this module — swap for Jackson/Gson if your framework already depends on one).

```xml
<dependency>
    <groupId>org.json</groupId>
    <artifactId>json</artifactId>
    <version>20240303</version>
</dependency>
```

#### 10.3.1 `visual/ImageDiffUtil.java`

Performs a per-pixel RGB comparison with a small tolerance band (to absorb anti-aliasing/font-rendering noise — see Section 6.5), resizes the actual image to the baseline's dimensions if they differ, and writes a red-highlighted diff-overlay image next to the actual screenshot for visual triage in CI artifacts.

```java
package com.architect.qe.visual;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
import java.nio.file.Path;

public final class ImageDiffUtil {

    // Per-channel tolerance absorbs anti-aliasing/font-rasterization noise (see Section 6.5)
    private static final int RGB_TOLERANCE = 30;

    private ImageDiffUtil() {
    }

    /**
     * Compares actual screenshot against baseline and returns the percentage
     * of pixels that differ beyond RGB_TOLERANCE. Also writes a
     * "<actual>-diff.png" overlay (differences highlighted in red) alongside
     * the actual image for CI artifact review.
     */
    public static double compare(Path baselinePath, Path actualPath) throws IOException {
        File baselineFile = baselinePath.toFile();
        File actualFile = actualPath.toFile();

        if (!baselineFile.exists()) {
            throw new IOException("Baseline image not found: " + baselinePath
                    + ". Run with -Dvisual.updateBaseline=true to create an initial baseline.");
        }

        BufferedImage baseline = ImageIO.read(baselineFile);
        BufferedImage actual = ImageIO.read(actualFile);
        BufferedImage normalizedActual = resizeIfNeeded(actual, baseline.getWidth(), baseline.getHeight());

        int width = baseline.getWidth();
        int height = baseline.getHeight();
        long diffPixelCount = 0;
        long totalPixels = (long) width * height;

        BufferedImage diffOverlay = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB);

        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                int baselineRgb = baseline.getRGB(x, y);
                int actualRgb = normalizedActual.getRGB(x, y);

                if (isPixelDifferent(baselineRgb, actualRgb)) {
                    diffPixelCount++;
                    diffOverlay.setRGB(x, y, 0xFFFF0000); // highlight diff in red
                } else {
                    diffOverlay.setRGB(x, y, baselineRgb);
                }
            }
        }

        writeDiffOverlay(diffOverlay, actualPath);

        return (diffPixelCount * 100.0) / totalPixels;
    }

    private static boolean isPixelDifferent(int rgb1, int rgb2) {
        int r1 = (rgb1 >> 16) & 0xFF, g1 = (rgb1 >> 8) & 0xFF, b1 = rgb1 & 0xFF;
        int r2 = (rgb2 >> 16) & 0xFF, g2 = (rgb2 >> 8) & 0xFF, b2 = rgb2 & 0xFF;

        return Math.abs(r1 - r2) > RGB_TOLERANCE
                || Math.abs(g1 - g2) > RGB_TOLERANCE
                || Math.abs(b1 - b2) > RGB_TOLERANCE;
    }

    private static BufferedImage resizeIfNeeded(BufferedImage source, int targetWidth, int targetHeight) {
        if (source.getWidth() == targetWidth && source.getHeight() == targetHeight) {
            return source;
        }
        BufferedImage resized = new BufferedImage(targetWidth, targetHeight, BufferedImage.TYPE_INT_ARGB);
        resized.getGraphics().drawImage(source, 0, 0, targetWidth, targetHeight, null);
        return resized;
    }

    private static void writeDiffOverlay(BufferedImage diffOverlay, Path actualPath) throws IOException {
        Path diffPath = actualPath.resolveSibling(
                actualPath.getFileName().toString().replace(".png", "-diff.png"));
        ImageIO.write(diffOverlay, "png", diffPath.toFile());
    }
}
```

#### 10.3.2 `visual/ScreenshotCapture.java`

```java
package com.architect.qe.visual;

import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public final class ScreenshotCapture {

    private ScreenshotCapture() {
    }

    public static Path capture(WebDriver driver, Path targetDir, String fileName) throws IOException {
        Files.createDirectories(targetDir);
        byte[] png = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
        Path target = targetDir.resolve(fileName);
        Files.write(target, png);
        return target;
    }
}
```

#### 10.3.3 `performance/LighthouseReportParser.java`

Parses the JSON report file produced by `LighthouseRunner` (Section 7.6) and extracts the performance category score and any individual numeric audit value (e.g., `largest-contentful-paint`, `cumulative-layout-shift`).

```java
package com.architect.qe.performance;

import org.json.JSONObject;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public final class LighthouseReportParser {

    private LighthouseReportParser() {
    }

    /** Returns performance score on a 0-100 scale (Lighthouse reports 0.0-1.0). */
    public static double extractPerformanceScore(String reportJsonPath) throws IOException {
        JSONObject report = readReport(reportJsonPath);
        double score = report.getJSONObject("categories")
                .getJSONObject("performance")
                .getDouble("score");
        return score * 100;
    }

    /** Returns the raw numericValue for a given Lighthouse audit id, e.g. "largest-contentful-paint". */
    public static double extractMetric(String reportJsonPath, String auditId) throws IOException {
        JSONObject report = readReport(reportJsonPath);
        JSONObject audit = report.getJSONObject("audits").getJSONObject(auditId);
        return audit.optDouble("numericValue", -1);
    }

    private static JSONObject readReport(String reportJsonPath) throws IOException {
        String content = Files.readString(Path.of(reportJsonPath));
        return new JSONObject(content);
    }
}
```

#### 10.3.4 `performance/CdpMetricsCollector.java`

Thin, reusable wrapper around the `Performance` CDP domain used in Section 7.4, so tests don't repeat `Performance.enable(...)` / metric-lookup boilerplate.

```java
package com.architect.qe.performance;

import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v127.performance.Performance;
import org.openqa.selenium.devtools.v127.performance.model.Metric;

import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

public final class CdpMetricsCollector {

    private final DevTools devTools;

    public CdpMetricsCollector(DevTools devTools) {
        this.devTools = devTools;
        this.devTools.send(Performance.enable(Optional.empty()));
    }

    public double getMetric(String metricName) {
        return devTools.send(Performance.getMetrics()).stream()
                .filter(m -> m.getName().equals(metricName))
                .mapToDouble(Metric::getValue)
                .findFirst()
                .orElse(-1);
    }

    public Map<String, Double> getAllMetrics() {
        return devTools.send(Performance.getMetrics()).stream()
                .collect(Collectors.toMap(Metric::getName, Metric::getValue));
    }
}
```

#### 10.3.5 `security/SecurityHeaderValidator.java`

Captures response headers per URL via CDP, used by the CORS/CSP checks in Section 8.2 instead of inlining a listener in every test.

```java
package com.architect.qe.security;

import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v127.network.Network;

import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public final class SecurityHeaderValidator {

    private final Map<String, Map<String, Object>> capturedHeadersByUrl = new ConcurrentHashMap<>();

    public void attach(DevTools devTools) {
        devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));
        devTools.addListener(Network.responseReceived(), res -> {
            String url = res.getResponse().getUrl();
            capturedHeadersByUrl.put(url, res.getResponse().getHeaders().toJson());
        });
    }

    public Optional<String> getHeader(String url, String headerName) {
        Map<String, Object> headers = capturedHeadersByUrl.get(url);
        if (headers == null) return Optional.empty();
        return headers.entrySet().stream()
                .filter(e -> e.getKey().equalsIgnoreCase(headerName))
                .map(e -> String.valueOf(e.getValue()))
                .findFirst();
    }
}
```

#### 10.3.6 `accessibility/AxeAccessibilityRunner.java`

Wraps `AxeBuilder` (Section 5.4) so the "only fail on critical/serious" policy from Section 5.5/5.6 lives in one place instead of being re-implemented per test.

```java
package com.architect.qe.accessibility;

import com.deque.html.axecore.results.Results;
import com.deque.html.axecore.results.Rule;
import com.deque.html.axecore.selenium.AxeBuilder;
import org.openqa.selenium.WebDriver;

import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

public final class AxeAccessibilityRunner {

    private static final Set<String> BLOCKING_IMPACTS = Set.of("critical", "serious");

    public List<Rule> runAndGetBlockingViolations(WebDriver driver, List<String> wcagTags) {
        Results results = new AxeBuilder()
                .withTags(wcagTags)
                .analyze(driver);

        return results.getViolations().stream()
                .filter(v -> BLOCKING_IMPACTS.contains(v.getImpact()))
                .collect(Collectors.toList());
    }
}
```

#### 10.3.7 `i18n/LocaleMatrixProvider.java`

Loads the external locale matrix referenced in Section 9.6 ("maintain the locale matrix as external data, never inline in test code") so `LocalizationTest` can be driven from a CSV instead of a hardcoded `@CsvSource`.

```java
package com.architect.qe.i18n;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

public final class LocaleMatrixProvider {

    public record LocaleEntry(String locale, String bundleKey) {}

    public static List<LocaleEntry> loadFromCsv(Path csvPath) throws IOException {
        List<LocaleEntry> entries = new ArrayList<>();
        for (String line : Files.readAllLines(csvPath)) {
            if (line.isBlank() || line.startsWith("#")) continue;
            String[] parts = line.split(",");
            entries.add(new LocaleEntry(parts[0].trim(), parts[1].trim()));
        }
        return entries;
    }
}
```

*(`locales.csv` example: `en-US, en` / `fr-FR, fr` / `ar-SA, ar` / `ja-JP, ja` — one line per row, no header needed.)*

#### 10.3.8 `core/DriverFactory.java`

Centralizes driver creation (Section 4, referenced throughout) using Selenium Manager for automatic driver-binary resolution — no manual `webdriver.chrome.driver` path management.

```java
package com.architect.qe.core;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public final class DriverFactory {

    private DriverFactory() {
    }

    public static WebDriver createChromeDriver() {
        return createChromeDriver(new ChromeOptions());
    }

    public static WebDriver createChromeDriver(ChromeOptions extraOptions) {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--remote-allow-origins=*");
        options.merge(extraOptions);
        // Selenium Manager resolves the matching chromedriver binary automatically.
        return new ChromeDriver(options);
    }

    public static WebDriver createHeadlessChromeDriver() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--disable-gpu", "--window-size=1920,1080");
        return createChromeDriver(options);
    }
}
```

#### 10.3.9 `core/QualityGateConfig.java`

Externalized thresholds (Section 6.6 / 7.8 / Exercise 14.4 — "configurable via external JSON, no recompilation needed").

```java
package com.architect.qe.core;

import org.json.JSONObject;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public final class QualityGateConfig {

    private final double visualDiffThresholdPercent;
    private final double performanceBudgetMillis;
    private final boolean blockOnCriticalA11y;
    private final boolean blockOnSeriousA11y;

    private QualityGateConfig(double visualDiffThresholdPercent,
                               double performanceBudgetMillis,
                               boolean blockOnCriticalA11y,
                               boolean blockOnSeriousA11y) {
        this.visualDiffThresholdPercent = visualDiffThresholdPercent;
        this.performanceBudgetMillis = performanceBudgetMillis;
        this.blockOnCriticalA11y = blockOnCriticalA11y;
        this.blockOnSeriousA11y = blockOnSeriousA11y;
    }

    public static QualityGateConfig loadFrom(Path configPath) throws IOException {
        JSONObject json = new JSONObject(Files.readString(configPath));
        return new QualityGateConfig(
                json.optDouble("visualDiffThresholdPercent", 0.5),
                json.optDouble("performanceBudgetMillis", 3000),
                json.optBoolean("blockOnCriticalA11y", true),
                json.optBoolean("blockOnSeriousA11y", true));
    }

    public static QualityGateConfig defaults() {
        return new QualityGateConfig(0.5, 3000, true, true);
    }

    public double getVisualDiffThresholdPercent() { return visualDiffThresholdPercent; }
    public double getPerformanceBudgetMillis() { return performanceBudgetMillis; }
    public boolean isBlockOnCriticalA11y() { return blockOnCriticalA11y; }
    public boolean isBlockOnSeriousA11y() { return blockOnSeriousA11y; }
}
```

Example `quality-gate-config.json`:

```json
{
  "visualDiffThresholdPercent": 0.5,
  "performanceBudgetMillis": 3000,
  "blockOnCriticalA11y": true,
  "blockOnSeriousA11y": true
}
```

#### 10.3.10 `reporting/QualityGateReportAggregator.java`

The Challenge Exercise (14.4) deliverable — merges independent discipline reports into one `PASS` / `WARN` / `FAIL` decision, hard-failing on accessibility (per Section 8/interview Q7 policy: functional/security-class issues block, visual/performance-class issues warn).

```java
package com.architect.qe.reporting;

import com.architect.qe.core.QualityGateConfig;
import org.json.JSONArray;
import org.json.JSONObject;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

public final class QualityGateReportAggregator {

    public enum Status { PASS, WARN, FAIL }

    public static final class GateResult {
        public final Status status;
        public final List<String> reasons;

        public GateResult(Status status, List<String> reasons) {
            this.status = status;
            this.reasons = reasons;
        }
    }

    private final QualityGateConfig config;

    public QualityGateReportAggregator(QualityGateConfig config) {
        this.config = config;
    }

    /**
     * a11yReportPath  -> JSON with a top-level "violations" array, each entry having "impact" and "id"
     * visualReportPath -> JSON with a "diffPercentage" numeric field
     * perfReportPath   -> JSON with a "domContentLoadedMillis" numeric field
     * Any path may be null/missing — that discipline is simply skipped.
     */
    public GateResult aggregate(Path a11yReportPath, Path visualReportPath, Path perfReportPath) throws IOException {
        List<String> reasons = new ArrayList<>();
        boolean hardFail = false;
        boolean softWarn = false;

        if (a11yReportPath != null && Files.exists(a11yReportPath)) {
            JSONObject a11y = new JSONObject(Files.readString(a11yReportPath));
            JSONArray violations = a11y.optJSONArray("violations");
            if (violations != null) {
                for (int i = 0; i < violations.length(); i++) {
                    String impact = violations.getJSONObject(i).optString("impact", "minor");
                    if (impact.equals("critical") && config.isBlockOnCriticalA11y()) {
                        hardFail = true;
                        reasons.add("Critical A11y violation: " + violations.getJSONObject(i).optString("id"));
                    } else if (impact.equals("serious") && config.isBlockOnSeriousA11y()) {
                        hardFail = true;
                        reasons.add("Serious A11y violation: " + violations.getJSONObject(i).optString("id"));
                    }
                }
            }
        }

        if (visualReportPath != null && Files.exists(visualReportPath)) {
            JSONObject visual = new JSONObject(Files.readString(visualReportPath));
            double diffPercent = visual.optDouble("diffPercentage", 0);
            if (diffPercent > config.getVisualDiffThresholdPercent()) {
                softWarn = true;
                reasons.add(String.format("Visual diff %.2f%% exceeds threshold %.2f%%",
                        diffPercent, config.getVisualDiffThresholdPercent()));
            }
        }

        if (perfReportPath != null && Files.exists(perfReportPath)) {
            JSONObject perf = new JSONObject(Files.readString(perfReportPath));
            double loadMillis = perf.optDouble("domContentLoadedMillis", 0);
            if (loadMillis > config.getPerformanceBudgetMillis()) {
                softWarn = true;
                reasons.add(String.format("Page load %.0fms exceeds budget %.0fms",
                        loadMillis, config.getPerformanceBudgetMillis()));
            }
        }

        Status status = hardFail ? Status.FAIL : (softWarn ? Status.WARN : Status.PASS);
        return new GateResult(status, reasons);
    }

    public String toMarkdownSummary(GateResult result) {
        StringBuilder sb = new StringBuilder();
        sb.append("## Quality Gate: ").append(result.status).append("\n\n");
        if (result.reasons.isEmpty()) {
            sb.append("All checks passed within configured thresholds.\n");
        } else {
            sb.append("### Findings\n");
            for (String reason : result.reasons) {
                sb.append("- ").append(reason).append("\n");
            }
        }
        return sb.toString();
    }
}
```

**Note on boundary testing (per Exercise 14.4's acceptance criteria):** because `aggregate()` takes file paths and reads plain JSON, you can unit-test it entirely with canned fixture files (no live browser needed) — e.g., a fixture with `"diffPercentage": 0.49` against a `0.5` threshold to prove the boundary doesn't trip, and `0.51` to prove it does.

---

## 11. Technical Validation — Why This Works

- **CDP-based metrics** are sourced directly from the browser's internal instrumentation (the same data DevTools UI displays), so there is no synthetic-agent overhead skewing numbers, unlike third-party JS-injected timers.
- **Axe-core** implements the WAI-ARIA and WCAG rule engine used by Deque's commercial products, giving high confidence in rule accuracy versus a hand-rolled ARIA checker.
- **Perceptual diffing** reduces false positives because it models human visual perception (contrast/structure) rather than raw byte equality, matching how a human reviewer would judge "did this actually change."
- **Locale-bundle-sourced assertions** guarantee the test suite and the application share a single source of truth, eliminating drift between translated content and test expectations.

---

## 12. Debugging Techniques

| Layer | Debugging Technique |
|---|---|
| CDP session issues | Enable `--remote-debugging-port=9222` and inspect `chrome://inspect` manually to confirm the domain events you expect are actually firing |
| Axe-core false negatives | Run `axe.run()` manually in the browser console against the live page before wiring into Java, to isolate JS-injection issues from Java-parsing issues |
| Visual diff noise | Dump both baseline and actual images plus a diff-overlay image as CI artifacts; visually inspect before assuming a real regression |
| HAR/network mismatches | Cross-check captured HAR against the browser's own Network tab (`chrome://net-export`) for the same session |
| Locale rendering issues | Use `driver.getPageSource()` combined with `Locale`-aware `Collator` inspection to rule out encoding (UTF-8 vs Latin-1) issues before blaming layout |
| General CDP version mismatch errors | Check `ChromeDriver` binary version vs `selenium-devtools-vXXX` artifact version — mismatches throw `DevToolsException` at runtime, not compile time |

---

## 13. Interview Preparation

### 13.1 Beginner Level

**Q1: What is the difference between WebDriver Protocol and CDP?**
A: WebDriver Protocol (W3C standard) is used for cross-browser element interaction and navigation. CDP is a Chromium-specific, WebSocket-based protocol used for deeper browser instrumentation — network capture, performance metrics, console logs — available in Selenium 4 via the `DevTools` class.

**Q2: Can Selenium alone do performance testing like JMeter?**
A: No. Selenium captures real-browser, client-side rendering performance (via CDP) for a single simulated user; JMeter/Gatling simulate load at the protocol level across many concurrent virtual users. They are complementary, not substitutes.

### 13.2 Intermediate Level

**Q3: How would you integrate axe-core into an existing Selenium framework without slowing down every test?**
A: Run accessibility scans as a separate, tagged test suite (e.g., JUnit5 `@Tag("a11y")`) executed on a subset of critical pages rather than every functional test, and gate only on critical/serious violations to control CI runtime and noise.

**Q4: How do you handle flaky visual regression tests caused by dynamic content like ads or timestamps?**
A: Mask dynamic regions via CSS selectors before capturing the screenshot, freeze animations (`* { animation: none !important; }` injected via CDP/JS), and use a tolerant/perceptual diff engine with a small allowed threshold rather than exact pixel match.

### 13.3 Advanced Level

**Q5: Explain how you'd capture a HAR file for an authenticated user flow without a proxy.**
A: Use Selenium 4's native CDP `Network.enable`, listen to `requestWillBeSent` and `responseReceived` events, correlate them by `requestId`, and assemble a HAR-spec-compliant JSON structure in Java — avoiding a MITM proxy (which requires trusting a custom CA cert and can break session/cookie behavior in authenticated flows).

**Q6: Why might CDP-based performance numbers differ between CI and local runs, and how do you mitigate this?**
A: CI runners are often shared/noisy-neighbor environments with variable CPU/network throttling, so absolute millisecond budgets are unreliable; mitigate by using relative regression thresholds against a rolling baseline, dedicated perf-test infra, and consistent CDP network/CPU emulation settings (`Network.emulateNetworkConditions`, `Emulation.setCPUThrottlingRate`) to normalize conditions.

### 13.4 Architect Level

**Q7: How would you design a single CI quality gate that merges functional, accessibility, visual, performance, and security signals into one pass/fail decision?**
A: Architect a `QualityGateReportAggregator` that consumes independent JSON reports from each discipline's runner (JUnit XML for functional/a11y, diff-percentage JSON for visual, CDP metrics JSON for performance, header-assertion JSON for security), applies discipline-specific weighted/severity thresholds defined in a central `QualityGateConfig`, and emits a single aggregated build status — while still publishing each discipline's detailed report as a separate CI artifact for triage. Critically, functional and security failures should hard-block the pipeline, while visual/performance regressions beyond a soft threshold should warn without blocking, to avoid pipeline fragility from lower-severity non-functional noise.

**Q8: Certificate pinning validation — can Selenium truly test this, and if not, what can it do?**
A: No — pinning is an application/client-level trust decision (common in native mobile apps) that a browser-driven Selenium session does not exercise the same way. Selenium's role is limited to verifying the server's presented certificate chain via CDP `Network.getSecurityDetails` (issuer, validity, protocol version) as a proxy signal, but true pinning-bypass/enforcement testing belongs in native app test suites or dedicated MITM-proxy security tooling, not browser automation.

### 13.5 Frequently Asked in Indian Product/Service Companies

- **TCS/Infosys/Wipro/Cognizant/Capgemini/LTIMindtree (Service-based, Selenium fundamentals emphasis):** "How do you handle dynamic locators?", "Explain implicit vs explicit wait", "How do you integrate Selenium with Jenkins?", "What is Page Object Model and why use PageFactory or not?"
- **Zoho/Freshworks (Product, pragmatic engineering culture):** "How would you add a lightweight accessibility check to our existing suite without adding a new tool dependency?", "How do you keep visual regression tests from becoming flaky in a fast-releasing SaaS product?"
- **Amazon India/Microsoft India/Oracle (Scale & architecture focus):** "How would you scale non-functional checks (a11y/visual/perf) across thousands of pages without exploding CI time?", "Design a quality gate architecture for a multi-team monorepo."
- **ThoughtWorks/EPAM (Consulting, architecture & best-practice focus):** "How do you decide what belongs in Selenium-driven checks vs a dedicated tool (Lighthouse CI, ZAP, Applitools)?", "Explain trade-offs between build vs buy for visual regression tooling."

---

## 14. Hands-On Practice

### 14.1 Exercise 1 — Accessibility Gate (Guided)

**Task:** Add an axe-core-based JUnit5 test for a login page. Fail the build if any `critical` or `serious` violation exists.

**Acceptance Criteria:**
- Test tagged `@Tag("a11y")`.
- Violations logged with rule ID, impact, and affected selector in the console/report.
- Test fails deterministically when a known violation (e.g., missing `<label>` on an input) is injected via a local test fixture.

**Test Data Suggestion:** Use a local static HTML fixture with one deliberately broken input (`<input type="text">` with no associated label) to prove the gate catches it.

### 14.2 Exercise 2 — Visual Regression Baseline (Guided)

**Task:** Capture a baseline screenshot of a sample page, then intentionally change a CSS margin and demonstrate the diff engine flags it above a 0.5% threshold.

**Acceptance Criteria:**
- Baseline stored under `src/test/resources/baseline/`.
- Diff percentage printed to console and attached as a CI artifact.
- Test passes at 0% diff on unchanged run; fails when CSS margin changed by ≥10px.

### 14.3 Mini Assignment — Performance Budget Gate

**Task:** Implement a CDP-based test asserting `document.readyState === 'complete'` timing and JS heap size stay within budget for three key pages (home, search results, checkout).

**Acceptance Criteria:**
- Budgets externalized in a config file, not hardcoded per test.
- Test reports actual vs budget values even on pass (not just on failure).
- CI run produces a time-series-friendly JSON output (timestamp, page, metric, value).

### 14.4 Challenge Exercise — Unified Quality Gate

**Task:** Build the `QualityGateReportAggregator` described in Section 10 that consumes at least three discipline reports (functional, accessibility, visual) and produces one aggregated JSON summary with an overall `PASS`/`WARN`/`FAIL` status per configurable severity rules.

**Acceptance Criteria:**
- Configurable via an external YAML/JSON (`quality-gate-config.json`) — no recompilation needed to change thresholds.
- Aggregator is unit-tested independently of live browser runs (feed it canned JSON fixtures).
- Produces a human-readable Markdown summary suitable for a PR comment bot.

**Test Data Suggestion:** Provide 3 canned fixture sets — all-pass, one-critical-a11y-failure, borderline-visual-diff (0.49% vs 0.5% threshold) — to validate boundary conditions in the aggregator logic.

---

## 15. Summary

This chapter extended a Selenium framework beyond DOM-level functional checks into five architect-relevant non-functional disciplines: accessibility (axe-core), visual regression (pixel vs perceptual diffing), performance (native CDP metrics, HAR capture, Lighthouse), security basics (CORS/CSP header validation, certificate-chain inspection with explicit limits around pinning), and localization/i18n (locale-matrix-driven, resource-bundle-sourced assertions). The unifying architectural theme is that Selenium 4's native CDP integration (`DevTools` class) is the technical enabler that makes most of this possible without third-party browser-automation add-ons, and that a mature framework aggregates all these independent signals into a single, configurable CI quality gate.

## 16. Revision Notes

- W3C WebDriver Protocol = standard interaction; CDP = deep Chromium instrumentation; Selenium 4 exposes both.
- Axe-core via `AxeBuilder` is the enterprise-default accessibility engine; gate on critical/serious only.
- Visual regression: pixel-diff for controlled/pinned environments, perceptual-diff (Applitools/Percy) for cross-platform scale.
- Performance: use native CDP for perf-smoke on every build; Lighthouse for deep nightly audits; always use relative thresholds on noisy CI infra.
- Security: Selenium checks are a smoke layer only (headers, cert-chain inspection) — not a DAST replacement; certificate pinning is out of scope for browser automation.
- i18n: always source expected strings from the app's own resource bundles; never hardcode translated text in tests; pair with visual regression for RTL.

## 17. Common Mistakes Checklist

- [ ] Gating builds on axe-core "minor/moderate" findings instead of only critical/serious
- [ ] Comparing visual screenshots taken on different OS/Chrome versions than the baseline
- [ ] Hardcoding CDP domain package versions without aligning them to the installed Chrome version
- [ ] Using absolute-millisecond performance budgets on shared, noisy CI infrastructure
- [ ] Treating a present CSP/CORS header as automatically "secure" without asserting its actual value
- [ ] Claiming Selenium validates certificate pinning (it does not — this is a client-app-level control)
- [ ] Hardcoding translated strings inside test code instead of reading from the app's resource bundles
- [ ] Running full non-functional suites (a11y/visual/perf) on every single test instead of tagged, scoped subsets

## 18. Key Takeaways

1. Selenium 4's native CDP support (`DevTools`) is the architectural foundation that unlocks performance, network, and security-header testing without third-party browser hooks.
2. Non-functional testing disciplines each need a *different* diffing/tolerance philosophy — exact-match is almost always wrong (visual diffing, performance budgets, header assertions all need tolerance/threshold design).
3. A Test Architect's job is not to write every check by hand, but to design the **aggregation layer** that turns five independent, noisy non-functional signals into one clear, actionable CI decision.
4. Know the boundaries of Selenium: it is not a load-testing tool, not a DAST scanner, and not a certificate-pinning validator — architects must know when to reach for a complementary tool instead of forcing Selenium to do everything.
