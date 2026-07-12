# Module 7 — Selenium 4 Advanced Browser Capabilities (CDP & BiDi)
### Level: Advanced | Track: Automation Engineer → Senior Automation Engineer → Test Architect

---

## 1. Skills Covered

- Establishing and managing Chrome DevTools Protocol (CDP) sessions from Selenium 4 Java bindings
- Understanding and using the emerging W3C **WebDriver BiDi** (Bidirectional) protocol
- Network traffic interception: request blocking, response mocking, header injection, HTTP Basic Auth automation
- Geolocation override for location-based testing (multi-region simulation)
- Network throttling (2G/3G/4G/offline) and CPU throttling for performance/resilience testing
- Device emulation (viewport, user-agent, touch, pixel ratio) via CDP `Emulation` domain
- Browser permission automation (camera, microphone, notifications, clipboard)
- Configuring `ChromeOptions`, `EdgeOptions`, `FirefoxOptions` for headless, extensions, profiles, and certificates
- Migrating legacy `DesiredCapabilities`-based code to Selenium 4 `*Options` classes
- Choosing between CDP, BiDi, proxy tools (BrowserMob/Zap), and server-side stubs (WireMock) for network control
- Debugging via DevTools Network panel, CDP command/event logs, and Selenium tracing

---

## 2. Learning Objectives

By the end of this chapter, the learner will be able to:

1. **Explain** the internal architecture of CDP and BiDi, and articulate why Selenium is migrating from one to the other.
2. **Implement** network request interception in Java to mock API responses, block resources (ads/images/fonts), and inject custom headers — using at least two different Selenium APIs.
3. **Override geolocation coordinates** to test location-aware application behavior for multiple simulated regions in a single test run.
4. **Simulate degraded network conditions** (3G/2G, high latency, packet loss) and **CPU throttling** to validate application resilience and performance budgets.
5. **Configure** `ChromeOptions`/`EdgeOptions`/`FirefoxOptions` correctly for headless execution, custom profiles, extensions, and self-signed certificate handling.
6. **Migrate** a codebase using deprecated `DesiredCapabilities` patterns to Selenium 4 idiomatic `Options` + `Capabilities` merge patterns.
7. **Compare and justify** the choice between CDP-based interception, a local proxy (BrowserMob Proxy), and a server-side mock (WireMock) for a given testing scenario.
8. **Debug** failed interception/emulation scenarios using DevTools Network panel, Selenium `LogEntries`, and CDP event traces.
9. **Answer** architecture and India-market interview questions on CDP, BiDi, network virtualization, and browser capability configuration with technically accurate depth.


---

## 3. Technical Depth

### 3.1 Chrome DevTools Protocol (CDP) — Architecture & Internals

**What is CDP?**
CDP is a protocol Chromium exposes over a **WebSocket** connection that allows external tools to instrument, inspect, and control the browser at a level far deeper than the W3C WebDriver spec allows. It is organized into **domains** (e.g., `Network`, `Page`, `Emulation`, `Security`, `Log`, `Performance`, `DOM`), each domain exposing:

- **Commands** — request/response calls the client sends to the browser (e.g., `Network.enable`, `Emulation.setGeolocationOverride`).
- **Events** — asynchronous notifications the browser pushes to the client (e.g., `Network.requestWillBeSent`, `Network.responseReceived`).
- **Types** — structured data objects used by commands/events.

**Why does Selenium expose CDP?**
The W3C WebDriver spec intentionally standardizes only a common denominator of browser automation actions (navigation, element interaction, cookies, basic alerts) so that any conforming driver — ChromeDriver, GeckoDriver, msedgedriver, SafariDriver — behaves identically. But real-world testing frequently needs *browser-specific* power: intercepting network calls, forcing geolocation, throttling network/CPU, reading console logs, and capturing performance metrics. Since Chromium browsers already speak CDP internally (DevTools itself is a CDP client), Selenium 4 added a **CDP bridge** so Java code can tunnel CDP commands over the *same* WebDriver session, using the browser's already-open debugging port.

**When to use CDP**
- You need Chrome/Edge-specific power features (network mocking, geolocation, performance metrics, console log capture).
- You are testing exclusively on Chromium-based browsers (Chrome, Edge, Chromium-based Brave/Opera).
- You need low-level control not exposed by W3C WebDriver (e.g., forcing colorScheme, setting device metrics override).

**Where CDP fits in the stack**

```
 Test Code (Java)
      │
      │  HasCdp / DevTools API (org.openqa.selenium.devtools)
      ▼
 Selenium Java Bindings  ──────────────►  ChromeDriver / EdgeDriver process
      │                                          │
      │        WebSocket (CDP JSON-RPC)          │
      └──────────────────────────────────────────┘
                                                   │
                                          Chromium Browser Process
                                          (Renderer + Network + GPU processes)
```

**How CDP session negotiation happens internally**
1. Selenium launches `chromedriver` with a `--remote-debugging-port` (managed internally).
2. `chromedriver` starts Chrome with `--remote-debugging-pipe`/port enabled.
3. When Java code calls `((HasDevTools) driver).getDevTools().createSession()`, Selenium's Java bindings open a **second** WebSocket directly to the browser's debugger endpoint (separate from the WebDriver HTTP/W3C channel).
4. Domain-specific classes (auto-generated from the CDP JSON schema, versioned e.g. `v135`) are used to send commands (`Network.enable()`) and register event listeners (`devTools.addListener(Network.responseReceived(), ...)`).
5. CDP is **versioned per Chrome release**; Selenium ships multiple CDP version packages (`org.openqa.selenium.devtools.v135`, `v136`, etc.) and picks the closest match to the running browser — a major source of breakage when Chrome auto-updates ahead of the Selenium CDP snapshot.

**Selenium 3 vs Selenium 4 — CDP-relevant differences**

| Capability | Selenium 3 | Selenium 4 |
|---|---|---|
| CDP access | Not available in core API; required 3rd-party libs (chrome-devtools-java, or raw WebSocket clients) | Native `DevTools` class, `HasDevTools` interface |
| Network interception | Not possible without external proxy | Native `Network.setRequestInterception` / `Network.continueRequest` wrappers |
| Capabilities API | `DesiredCapabilities` (mutable map, no type safety) | `ChromeOptions`/`FirefoxOptions`/`EdgeOptions` (typed, per-browser, merge-friendly with `Capabilities.merge()`) |
| Driver executable management | Manual download & PATH management | Selenium Manager (auto-resolves matching driver binary) |
| Relative locators | Not available | `RelativeLocator` (`with(tagName).above/near/below/toLeftOf/toRightOf`) |
| Windows/Tabs | `getWindowHandles()` only | `NewWindow` API: `driver.switchTo().newWindow(WindowType.TAB/WINDOW)` |

**Deprecated APIs and modern alternatives**

| Deprecated (Selenium 3-era) | Modern Alternative (Selenium 4) |
|---|---|
| `DesiredCapabilities caps = new DesiredCapabilities();` | `ChromeOptions options = new ChromeOptions();` (implements `Capabilities`) |
| `new ChromeDriver(caps)` mixed with options | `new ChromeDriver(options)` |
| Manual proxy via `org.openqa.selenium.Proxy` only | `Proxy` still valid, but CDP `Network` domain now preferred for mocking |
| `FirefoxDriver.SystemProperty` manual binary path | Selenium Manager auto-resolution (fallback to explicit `setBinary()` when needed) |

### 3.2 WebDriver BiDi — Architecture, Event-Driven Automation & Limitations

**What is BiDi?**
WebDriver BiDi ("Bidirectional") is a **W3C standard** (unlike CDP, which is Chromium-proprietary) that adds a persistent, bidirectional WebSocket channel to the classic WebDriver session. It standardizes cross-browser access to capabilities that used to require vendor-specific protocols: network events, console/log events, script execution events, and (increasingly) browsing-context and emulation control.

**Why BiDi exists**
CDP works only on Chromium browsers. Firefox and Safari never implemented CDP (Firefox has its own low-level protocol; Safari has none exposed). Prior to BiDi, achieving equivalent capabilities cross-browser was impossible through Selenium — teams needed browser-specific branches of their framework. BiDi is being co-developed by Google, Mozilla, and the W3C WebDriver working group specifically to give **one standardized API surface** across Chrome, Edge, and Firefox (Safari support is still catching up as of this writing).

**Architecture**

```
        ┌────────────────────────── Test Code (Java) ───────────────────────────┐
        │                                                                        │
        │   driver.script().addConsoleMessageHandler(...)                       │
        │   driver.network().addRequestHandler(...)                             │
        │                                                                        │
        └───────────────┬───────────────────────────────┬──────────────────────┘
                         │  Classic HTTP/W3C commands     │  BiDi WebSocket (JSON)
                         ▼                                 ▼
                 ┌───────────────┐                 ┌──────────────────┐
                 │  WebDriver    │◄───same session─►│  BiDi Endpoint   │
                 │  HTTP Server  │                 │  (same driver)   │
                 └───────┬───────┘                 └─────────┬────────┘
                         │                                    │
                         ▼                                    ▼
                 ┌─────────────────────────────────────────────────┐
                 │            Browser (Chrome / Firefox / Edge)     │
                 └─────────────────────────────────────────────────┘
```

Unlike CDP (a *second*, separate WebSocket the Java client opens directly to the browser), BiDi is negotiated **as part of the same WebDriver session** — the driver advertises a `webSocketUrl` capability at session creation, and Selenium's Java bindings (`org.openqa.selenium.bidi.*`) connect to it.

**Event-driven automation model**
BiDi is fundamentally event/subscription based:

```java
try (Network network = new Network(driver)) {
    network.onResponseCompleted(responseDetails -> {
        System.out.println(responseDetails.getResponseData().getUrl());
    });
}
```

This is conceptually similar to CDP listeners, but portable across Firefox and Chrome without rewriting the domain classes.

**BiDi Modules relevant to this chapter**

| Module | Purpose |
|---|---|
| `browsingContext` | Create/close tabs, navigate, capture screenshots |
| `network` | Intercept/modify requests & responses, auth handling, blocking |
| `log` | Console and JS error/log entries |
| `script` | Inject/evaluate scripts, realm/channel management |
| `session` | Subscribe/unsubscribe to event streams |

**Limitations (as of Selenium 4.2x / current stable)**
- Firefox support is broader than Chrome's for some modules; Chrome's BiDi implementation still routes some functionality through CDP internally.
- Not all CDP capabilities have a BiDi equivalent yet — **CPU throttling and full device emulation are still CDP-only** in most current implementations; BiDi network interception is functional but has narrower body-modification support than CDP's `Fetch` domain.
- Safari (WebKit) BiDi support is minimal/experimental.
- API surface in Selenium Java (`org.openqa.selenium.bidi`) is newer and less battle-tested than the CDP wrapper — expect breaking changes across minor Selenium versions.

**CDP vs BiDi decision guidance (summary; full comparison table in Section 6)**
Prefer BiDi when you need **cross-browser** event handling (console logs, basic network events) and your Selenium version's BiDi support covers the feature. Prefer CDP when you need **Chromium-specific deep control** (CPU throttling, full device emulation, `Fetch.continueWithAuth`/`Fetch.fulfillRequest` fine-grained mocking) and are only targeting Chrome/Edge.

### 3.3 Network Interception — Request/Response Mocking, Blocking, Header Injection, Auth

**What**
Network interception intercepts outgoing HTTP(S) requests from the browser (or incoming responses) before they hit the wire, allowing the test to: block a request outright (e.g., ads, analytics beacons, images — to speed up tests), fulfill it with a fabricated response (mocking a backend API for a frontend-only test), modify request headers before sending, or modify the response body/headers before it reaches the renderer.

**Why**
- Deterministic tests: mock a flaky/slow third-party API so the UI test is not coupled to backend availability.
- Negative-path testing: force a 500/timeout response to verify UI error handling without needing the real backend to fail.
- Performance: block heavy resources (images, fonts, trackers) to speed up CI runs.
- Security testing: inject/verify headers (CSP, auth tokens) without modifying application code.

**When**
Use interception when the scenario under test is about **frontend behavior in response to specific network conditions**, not about verifying the real backend contract. If you need to verify actual backend integration, interception is the wrong tool — use a real (or realistic staging) backend instead.

**Where — three architectural options** (detailed comparison in Section 6):
1. **CDP `Network`/`Fetch` domains** (Selenium native, Chromium-only)
2. **Local proxy** (BrowserMob Proxy / mitmproxy) sitting between browser and network
3. **Server-side stub** (WireMock/MockServer) that the *application* is configured to call instead of the real service

**How — CDP interception pipeline**

```
Browser Renderer                         ChromeDriver/Browser Process
      │  1. issues request                        │
      ▼                                            │
 Network.requestWillBeSent  ─────event────────────►│  (if Network.setRequestInterception
                                                    │   or Fetch.enable is active)
      │                                            │
      │        Fetch.requestPaused  event          │
      │◄───────────────────────────────────────────┤
      │                                            │
 Test code inspects intercepted request            │
      │                                            │
      │  Fetch.continueRequest()                   │
      │  Fetch.fulfillRequest()  (mock response)   │
      │  Fetch.failRequest()     (block)           │
      ├───────────────────────────────────────────►│
      │                                            │
      ▼                                            ▼
  Response delivered to renderer (real, mocked, or failed)
```

Selenium's Java `DevTools` wrapper exposes this through the `Network` (older, coarse) and `Fetch` (newer, fine-grained interception with pause/resume semantics) domains. The **`Fetch` domain is the recommended modern approach** for request modification; the older `Network.setRequestInterception` is effectively superseded.

**Basic authentication strategies**
Two common patterns:
1. **URL-embedded credentials**: `https://user:pass@example.com` — simplest, but fails on some Chromium versions/redirect chains and leaks credentials into logs/history.
2. **CDP `Fetch.authRequired` event + `Fetch.continueWithAuth`** — intercept the browser's native basic-auth challenge and supply credentials programmatically, without ever touching a JS/UI dialog. This is the robust, production-grade approach and is covered in the code section.

### 3.4 Browser Emulation — Geolocation, Throttling, Device Emulation, Permissions

**Geolocation override**
CDP's `Emulation.setGeolocationOverride(latitude, longitude, accuracy)` forces `navigator.geolocation` calls in the page to resolve to fixed coordinates — essential for testing location-gated features (store locators, region-locked content, currency/locale switching) without needing a VPN or GPS spoofing hardware. Selenium 4 Java also now offers `((JavascriptExecutor) driver)`-independent, W3C-native `Location` via `driver.manage()` in newer versions on top of CDP, but the CDP path remains the most reliable cross-version approach today.

**Network throttling**
`Network.emulateNetworkConditions(offline, latencyMs, downloadThroughput, uploadThroughput)` simulates 2G/3G/4G profiles by capping throughput and injecting latency at the network stack level *inside* the browser process — this is different from a proxy-based throttle, which throttles at the OS/socket level outside the browser. CDP throttling is more realistic because it reflects exactly what DevTools' own "Network throttling" dropdown does.

**CPU throttling**
`Emulation.setCPUThrottlingRate(rate)` (rate = slowdown multiplier, e.g., `4` = 4x slower) simulates low-end devices — critical for performance testing (e.g., verifying a page doesn't become unresponsive on a throttled CPU during a heavy JS operation). There is **no BiDi equivalent yet** — this remains CDP-only.

**Device emulation**
`Emulation.setDeviceMetricsOverride(width, height, deviceScaleFactor, mobile)` plus `Network.setUserAgentOverride(...)` together emulate a mobile device's viewport, pixel density, touch events, and user-agent string — used for responsive-design and mobile-web testing without needing a real device farm for every scenario.

**Permissions**
CDP's `Browser.setPermission` (or the newer `Browser.grantPermissions`) programmatically grants/denies permissions (`geolocation`, `notifications`, `camera`, `microphone`, `clipboard-read`) so tests never have to click a native browser permission prompt — those prompts are OS-level UI outside the DOM and are otherwise unautomatable by WebDriver directly.

### 3.5 Browser Configuration — Options, Headless, Extensions, Certificates, Profiles

**`ChromeOptions` vs `DesiredCapabilities` — why the migration matters**
`DesiredCapabilities` was a loosely-typed `Map<String, Object>` shared across all browsers — nothing stopped a developer from setting a Firefox-only capability on a Chrome session (silently ignored, or worse, causing obscure driver errors). Selenium 4's per-browser `Options` classes (`ChromeOptions`, `FirefoxOptions`, `EdgeOptions`, `SafariOptions`) are strongly typed, IDE-autocompletable, and each implements the `Capabilities` interface so they can still be merged/passed wherever a generic `Capabilities` object is expected (e.g., in Selenium Grid node matching).

**Headless modes**
Chrome/Edge have had two distinct headless implementations:
- Legacy headless: `--headless` (old rendering path, some feature gaps vs headed Chrome — e.g., inconsistent font rendering, some GPU-accelerated CSS features unsupported).
- **New headless** (`--headless=new`, Chrome 109+): shares the same rendering pipeline as headed Chrome — recommended default today for parity between local headed debugging and CI headless runs.

**Extensions**
`.crx` extension files or unpacked extension directories can be loaded via `ChromeOptions.addExtensions(File...)` or `addArguments("--load-extension=<path>")`. Useful for ad-blockers in perf tests, or testing an in-house browser extension.

**Certificate handling**
`ChromeOptions.setAcceptInsecureCerts(true)` (a W3C-standard capability, works across browsers) allows navigation to self-signed/invalid-cert staging environments without a manual "Proceed anyway" click — critical for internal QA environments that don't have production-grade certs.

**Browser profiles**
`--user-data-dir=<path>` (Chrome) or `FirefoxOptions.setProfile(new FirefoxProfile())` load a persistent profile — useful for tests that require pre-authenticated sessions, saved extensions, or specific browser settings, avoiding repeated login flows per test.

---

## 4. Architecture Diagrams

### 4.1 Full CDP Session Establishment & Network Interception Pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         JUnit 5 Test (Java)                              │
│  ChromeDriver driver = new ChromeDriver(options);                        │
│  DevTools devTools = ((HasDevTools) driver).getDevTools();               │
│  devTools.createSession();                                                │
└───────────────────────────────┬────────────────────────────────────────-─┘
                                 │ 1. HTTP  POST /session   (W3C new session)
                                 ▼
                    ┌─────────────────────────┐
                    │      chromedriver         │
                    │  - spawns chrome binary   │
                    │  - opens remote-debug port│
                    └────────────┬──────────────┘
                                 │ 2. WebSocket handshake (CDP)
                                 ▼
                    ┌─────────────────────────┐
                    │     Chrome Browser        │
                    │  Browser Process          │
                    │   ├── Renderer Process     │
                    │   ├── Network Service       │
                    │   └── GPU Process            │
                    └────────────┬──────────────┘
                                 │ 3. devTools.send(Network.enable())
                                 │ 4. devTools.send(Fetch.enable(patterns))
                                 ▼
                    Browser pauses matching requests
                                 │
                                 │ 5. Event: Fetch.requestPaused
                                 ▼
                    ┌─────────────────────────┐
                    │   Java Listener Callback   │
                    │  inspect request/headers  │
                    │  decide: continue/fulfill/ │
                    │          fail               │
                    └────────────┬──────────────┘
                                 │ 6. devTools.send(Fetch.continueRequest /
                                 │                   fulfillRequest / failRequest)
                                 ▼
                        Request proceeds, mocked, or blocked
```

### 4.2 Class Hierarchy — Options, Capabilities, DevTools

```
                        ┌─────────────────────┐
                        │     Capabilities      │  (interface)
                        └──────────▲────────────┘
                                    │ implements
              ┌─────────────────────┼─────────────────────┐
              │                     │                       │
     ┌────────┴───────┐   ┌─────────┴────────┐   ┌──────────┴────────┐
     │  ChromeOptions   │   │  FirefoxOptions   │   │   EdgeOptions      │
     └────────┬───────┘   └─────────┬────────┘   └──────────┬────────┘
              │                     │                       │
   passed to constructor  passed to constructor    passed to constructor
              │                     │                       │
     ┌────────▼───────┐   ┌─────────▼────────┐   ┌──────────▼────────┐
     │   ChromeDriver   │   │   FirefoxDriver    │   │    EdgeDriver       │
     │ implements       │   │ implements         │   │ implements          │
     │  HasDevTools     │   │  (no CDP; BiDi)     │   │  HasDevTools         │
     └────────┬───────┘   └────────────────────┘   └──────────┬────────┘
              │                                                │
              └───────────────────┬────────────────────────────┘
                                   ▼
                        ┌─────────────────────┐
                        │     DevTools object    │
                        │  .send(Command<T>)       │
                        │  .addListener(Event<T>)   │
                        └─────────────────────┘
```

### 4.3 BiDi Event Subscription Flow

```
Test Code                     Selenium BiDi Java Layer            Browser
   │  new FirefoxDriver / ChromeDriver(options)                     │
   │──────────────────────────────────────────────────────────────►│
   │           session capabilities include webSocketUrl            │
   │◄──────────────────────────────────────────────────────────────│
   │  Network network = new Network(driver)                         │
   │  network.onResponseCompleted(handler)                            │
   │─────────► opens BiDi WebSocket, sends session.subscribe ───────►│
   │                                                                  │
   │                    ... user navigates page ...                 │
   │                                                                  │
   │◄──── network.responseCompleted event pushed asynchronously ────│
   │  handler invoked in Java with ResponseDetails                   │
```

---

## 5. Multiple Approaches — Network Control Strategies

For any given need to control network behavior in a test, there are at least three architecturally distinct approaches. This section compares them; complete code for the recommended (CDP) approach follows in Section 8.

### Approach 1: CDP-based Interception (Selenium native, in-process)
The test itself, via Selenium's `DevTools` object, enables `Fetch`/`Network` domains and handles events inline.

### Approach 2: Local Proxy (BrowserMob Proxy / mitmproxy / Selenium `Proxy` class)
The browser is configured (via `ChromeOptions.setProxy(seleniumProxy)`) to route all traffic through a local proxy server process. The proxy (a separate JVM process or Python process) intercepts and rewrites traffic *outside* the browser, and Selenium never talks CDP for this purpose.

### Approach 3: Server-Side Stub / Contract Mock (WireMock, MockServer, Mock Service Worker)
The application under test is configured (via environment variable or config file) to call a **mock server** instead of the real backend. Selenium does nothing special — it just drives the browser normally against an app that happens to be wired to a stub backend.

### Approach 4 (Hybrid): Service Worker–based Mocking (MSW - Mock Service Worker) injected via CDP `Page.addScriptToEvaluateOnNewDocument`
Selenium injects a JS service worker at page-load time (via CDP) that intercepts `fetch`/`XHR` calls **inside the browser's JS runtime**, before they even reach network internals. Useful for SPA-heavy apps where you want mocking to survive client-side routing without a real network round trip at all.

### Comparison

| Criteria | CDP Interception | Local Proxy | Server-Side Stub | Service Worker Injection |
|---|---|---|---|---|
| Setup complexity | Medium (Java-only, no extra process) | High (extra process, port management, cert trust for HTTPS MITM) | Low-Medium (needs a stub server + app config) | Medium-High (needs injected JS bundle) |
| Cross-browser support | Chromium only | Yes (any browser, OS-level) | Yes (browser-agnostic) | Yes (if app loads the worker) |
| HTTPS handling | Native (no cert trust issues) | Requires trusting proxy's root CA | Native | Native |
| Granularity | Very high (per-request, per-header, per-body) | High | Medium (predefined stub scenarios) | Very high (JS-level) |
| Performance overhead | Low | Medium (extra hop, decrypt/re-encrypt) | Low | Very low |
| Best for | Chrome/Edge-only suites needing precise per-test mocking | Multi-browser suites, HAR capture/replay | Full E2E suites where backend team owns stubs | SPA testing without any network layer dependency |
| Maintainability | High (co-located in test code) | Medium (proxy config drifts from tests) | High (stub definitions versioned with API contracts, e.g. OpenAPI) | Medium (JS injection logic separate from Java) |

**Recommendation:** For Chromium-only automation suites needing fine per-test control (which is the majority of enterprise Selenium suites today), **CDP `Fetch` domain interception is the recommended default** — it has the lowest infra overhead, the highest fidelity to real HTTPS behavior, and keeps mock definitions co-located with the test. Reach for a **local proxy** only when true cross-browser (Firefox/Safari) mocking is required and BiDi's network module doesn't yet cover the needed granularity. Reach for **server-side stubs** when the mocking need is shared across teams (e.g., contract testing owned by backend engineers) rather than owned by the UI test suite.

---

## 6. Comparison Tables

### 6.1 ChromeOptions vs DesiredCapabilities

| Aspect | `DesiredCapabilities` (Selenium 3 legacy) | `ChromeOptions` (Selenium 4) |
|---|---|---|
| Type safety | None — `Map<String,Object>` | Strong — dedicated setter methods |
| Browser scoping | Shared across all browsers (error-prone) | Browser-specific class |
| IDE autocomplete | No | Yes |
| Merge behavior | Manual map merging | `Capabilities.merge()` built-in, plus `options.merge()` |
| CDP/BiDi awareness | None | Integrates with `HasDevTools`, BiDi session negotiation |
| Recommended today | Deprecated; avoid in new code | Use exclusively |

### 6.2 CDP vs BiDi

| Aspect | CDP | BiDi |
|---|---|---|
| Standard | Chromium proprietary (Google-controlled) | W3C standard |
| Browser support | Chrome, Edge, other Chromium browsers | Chrome, Edge, Firefox (growing); Safari minimal |
| Stability across versions | Fragile — versioned per Chrome release, breaking changes common | Stable — spec-governed, backward-compatible intent |
| Feature completeness (today) | Very high — full domain set (Network, Fetch, Emulation, Performance, Security...) | Growing — network, log, script, browsingContext solid; emulation/performance still maturing |
| CPU throttling | Supported (`Emulation.setCPUThrottlingRate`) | Not yet standardized |
| Fine-grained request mocking | Supported (`Fetch` domain) | Supported (`network` module) but narrower body-mutation support |
| Selenium Java API maturity | Mature (`org.openqa.selenium.devtools`) | Newer (`org.openqa.selenium.bidi`), evolving |
| Long-term direction | Being phased toward parity-then-secondary role | Selenium's strategic long-term direction for cross-browser advanced control |

### 6.3 Selenium Manager vs WebDriverManager

| Aspect | Selenium Manager (built-in, Selenium 4.6+) | WebDriverManager (Boni Garcia, 3rd-party) |
|---|---|---|
| Installation | Bundled with Selenium — zero extra dependency | Requires adding a Maven dependency |
| Driver resolution | Automatic at `new ChromeDriver()` call time | Automatic via explicit `WebDriverManager.chromedriver().setup();` call |
| Version pinning | Limited built-in control | Rich API: pin exact versions, use local cache, proxy support |
| CI offline/air-gapped support | Weaker (needs internet for driver metadata) | Better caching/versioning control for restricted networks |
| When to prefer | Simple projects, latest Selenium, minimal deps | Enterprise pipelines needing fine version control, caching, proxy-aware downloads |

### 6.4 get() vs navigate()

| Aspect | `driver.get(url)` | `driver.navigate().to(url)` |
|---|---|---|
| Underlying call | Internally calls `navigate().to()` | Direct API |
| History awareness | N/A (fresh navigation) | Part of a `Navigation` interface offering `.back()`, `.forward()`, `.refresh()` |
| Practical difference | None functionally for a single navigation | Preferred when you need to also use back/forward/refresh in the same flow |

### 6.5 findElement() vs findElements()

| Aspect | `findElement()` | `findElements()` |
|---|---|---|
| Return type | Single `WebElement` | `List<WebElement>` (possibly empty) |
| No match behavior | Throws `NoSuchElementException` | Returns empty list — safer for existence checks |
| Typical use | You expect exactly one guaranteed element | Checking presence/count, or iterating multiple matches |

### 6.6 Implicit vs Explicit Wait

| Aspect | Implicit Wait | Explicit Wait (`WebDriverWait`) |
|---|---|---|
| Scope | Applies globally to every `findElement` call for the driver's lifetime | Applies to a specific condition at a specific point in the test |
| Condition granularity | Only "element present in DOM" | Any `ExpectedCondition` (visible, clickable, text present, custom lambda) |
| Mixing with Fluent/explicit waits | **Not recommended** — mixing causes unpredictable total wait times | Composable via `FluentWait` for custom polling intervals |
| Best practice | Avoid or set to 0; prefer explicit waits exclusively | Preferred modern pattern |

### 6.7 PageFactory vs Page Object (plain)

| Aspect | PageFactory (`@FindBy` + `initElements`) | Plain Page Object (constructor-based `findElement` calls) |
|---|---|---|
| Element lookup timing | Lazy proxy — elements located on first interaction | Eager or on-demand, developer-controlled |
| Stale element handling | Proxy can re-locate transparently in some cases, but has known caching quirks | Full manual control, easier to reason about with dynamic DOM |
| Selenium team guidance (current) | De-emphasized in recent Selenium docs due to StaleElementReference issues with complex SPAs | Increasingly recommended, especially combined with explicit waits per action |
| Recommended for new frameworks | Optional | **Preferred** for architect-level frameworks targeting dynamic, JS-heavy apps |

---

## 7. Pitfalls & Anti-Patterns

1. **CDP version mismatch crashes.** Selenium ships CDP classes per Chrome major version (`v135`, `v136`...). If your Selenium jar predates the installed Chrome, `getDevTools()` may pick a slightly incompatible version — usually still works (Selenium logs a warning and falls back to the nearest version) but can silently drop new fields. **Pin your Chrome version in CI** or upgrade Selenium in lockstep with Chrome's auto-update cadence.

2. **Forgetting to enable the domain before listening.** Registering `devTools.addListener(Network.responseReceived(), ...)` without first calling `devTools.send(Network.enable())` results in the listener never firing — commonly mistaken for a Selenium bug.

3. **Leaving `Fetch.enable()` active with no matching pattern → all requests hang.** If you enable `Fetch` domain interception but your listener never calls `continueRequest`/`fulfillRequest`/`failRequest` for a paused request, the browser will hang waiting indefinitely, causing the whole test to time out mysteriously. Always ensure every paused request path has a resolution branch (including a default "just continue" case).

4. **Headless rendering differences causing false failures.** Legacy `--headless` Chrome sometimes fails to load certain fonts or WebGL-based UI, producing layout differences that break screenshot-based or pixel-based assertions that pass in headed mode. Migrate to `--headless=new` and, ideally, avoid pixel-perfect assertions in favor of DOM-state assertions.

5. **Cross-origin interception limits.** `Fetch.enable` patterns are matched by URL glob and resource type, but some cross-origin requests (especially `no-cors` prefetches, service-worker-initiated requests, or requests from iframes on a different origin) may not be interceptable the same way — verify interception actually fired via the `Network.requestWillBeSent` event log rather than assuming success.

6. **Certificate pinning / HSTS breaking proxy-based interception.** If you choose the local-proxy approach (Approach 2) against an app using certificate pinning or strict HSTS, the browser will refuse the proxy's re-signed certificate even after adding it to the trust store — CDP interception avoids this entirely since it doesn't MITM the TLS layer.

7. **Geolocation override without granting the permission first.** Calling `Emulation.setGeolocationOverride` while the page's own permission prompt is still pending can result in the override being ignored by the page's `navigator.geolocation.getCurrentPosition` callback. Always pair it with `Browser.grantPermissions(["geolocation"])` for the origin.

8. **Throttling values that don't match real-world profiles.** Passing arbitrary latency/throughput numbers ("looks about right") produces non-reproducible performance test results. Use the well-known standard profiles (see code section) so results are comparable across runs and teams.

9. **Mixing implicit and explicit waits under CDP interception.** Slow network emulation combined with a global implicit wait frequently causes total wait times to become the *sum* rather than the *max* of the two waits, causing intermittent, hard-to-diagnose timeouts. Keep implicit wait at 0 in any suite using network throttling.

10. **DesiredCapabilities lingering in shared utility code.** Legacy internal libraries often still expose a `DesiredCapabilities`-typed helper method; when merged into a Selenium 4 `ChromeOptions.merge(caps)` call, browser-specific keys can silently collide or be dropped. Audit and remove `DesiredCapabilities` from shared frameworks entirely.

---

## 8. Best Practices (Enterprise-Grade)

- **Isolate CDP/BiDi logic behind a dedicated abstraction** (`NetworkController`, `EmulationController` interfaces) rather than sprinkling `devTools.send(...)` calls through step definitions — this is what distinguishes a maintainable framework from a script collection, and is the pattern used in mature internal frameworks at large tech companies.
- **Always pair `Fetch.enable` with a catch-all continuation.** Register the interception handler so any unmatched/unexpected request defaults to `continueRequest` — never leave a code path where a paused request has no resolution.
- **Centralize standard network profiles** (2G/3G/4G/offline) as named constants/enums rather than magic numbers scattered across tests — mirrors what Chrome DevTools itself offers as presets.
- **Prefer `acceptInsecureCerts(true)` over disabling TLS validation at the OS/JVM level** — keep certificate trust scoped to the browser session, not the whole test-runner machine.
- **Treat CDP session lifecycle as scoped to a test**, not shared globally — create and close CDP sessions/listeners per test (or use `try-with-resources` on `Network`/`DevTools` where the API supports `AutoCloseable`) to avoid listener leakage across tests in the same JVM (common in parallel execution with a shared driver pool).
- **Version-pin your CDP domain import** (`org.openqa.selenium.devtools.v135.network.Network`) to the Chrome version used in CI, and re-evaluate on every Chrome major bump — track this in your dependency-upgrade runbook, not as an afterthought.
- **Log every intercepted request/response pair** at DEBUG level (URL, status, timing) — this becomes essential troubleshooting data when a mocked scenario doesn't behave as expected, mirroring what a proper API gateway audit log provides in production systems.
- **Favor BiDi over CDP going forward for new capabilities that support both**, to reduce long-term coupling to Chromium internals — an architectural decision that pays off when the org later needs Firefox coverage.

---

## 9. Java Implementation — Production-Grade Code Examples

> **Environment**: Java 21, Selenium 4.2x, JUnit 5, Maven. CDP domain classes are versioned (`org.openqa.selenium.devtools.v135.*` in these examples) — **replace `v135` with the package matching your installed Selenium/Chrome pair**; check `org.openqa.selenium.devtools` on your classpath for available versions. Some CDP method overloads include additional `Optional<T>` parameters not shown here for readability — consult your IDE's autocomplete for the exact signature on your version.

### 9.1 Maven Dependencies (`pom.xml` excerpt)

```xml
<properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <selenium.version>4.24.0</selenium.version>
    <junit.version>5.11.0</junit.version>
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
</dependencies>
```

Recommended package layout:

```
src/test/java/com/company/automation/
 ├── config/          -> DriverFactory, BrowserOptionsBuilder
 ├── cdp/              -> NetworkController, EmulationController (abstractions)
 ├── tests/            -> JUnit 5 test classes
 └── util/             -> NetworkProfile enum, TestConstants
```

---

### 9.2 Scenario: Mocking an API Response via CDP `Fetch` Domain

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.fetch.Fetch;
import org.openqa.selenium.devtools.v135.fetch.model.HeaderEntry;
import org.openqa.selenium.devtools.v135.fetch.model.RequestPattern;
import org.openqa.selenium.devtools.v135.network.model.RequestId;
import org.openqa.selenium.devtools.v135.network.model.ResourceType;

import java.util.Base64;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertTrue;

class NetworkMockingTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new");
        driver = new ChromeDriver(options);
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();
    }

    @Test
    @DisplayName("Product list API is mocked with an empty-state response")
    void shouldRenderEmptyStateWhenApiReturnsNoProducts() {
        RequestPattern pattern = new RequestPattern(
                Optional.of("*/api/products*"),
                Optional.of(ResourceType.XHR),
                Optional.empty());

        devTools.send(Fetch.enable(Optional.of(List.of(pattern)), Optional.of(false)));

        devTools.addListener(Fetch.requestPaused(), requestPaused -> {
            RequestId requestId = requestPaused.getRequestId();
            String mockJsonBody = "{\"products\": [], \"total\": 0}";
            String base64Body = Base64.getEncoder()
                    .encodeToString(mockJsonBody.getBytes());

            List<HeaderEntry> headers = List.of(
                    new HeaderEntry("Content-Type", "application/json"));

            devTools.send(Fetch.fulfillRequest(
                    requestId,
                    200,
                    Optional.of(headers),
                    Optional.empty(),
                    Optional.of(base64Body),
                    Optional.empty()));
        });

        driver.get("https://example-shop.test/catalog");

        assertTrue(driver.findElement(org.openqa.selenium.By.cssSelector(".empty-state"))
                .isDisplayed(), "Empty-state UI should render when API returns zero products");
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

**Why this works:** `Fetch.enable` with a `RequestPattern` scoped to `*/api/products*` and `ResourceType.XHR` pauses *only* matching XHR calls; every other request (HTML, CSS, JS, images) passes through untouched because it never matches the pattern and is therefore never paused. The listener fabricates a 200 response with a base64-encoded JSON body via `Fetch.fulfillRequest`, which the renderer treats exactly as if the real backend had returned it.

---

### 9.3 Scenario: Blocking Resource Types for Faster CI Runs

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.fetch.Fetch;
import org.openqa.selenium.devtools.v135.fetch.model.RequestPattern;
import org.openqa.selenium.devtools.v135.network.model.ResourceType;

import java.util.List;
import java.util.Optional;
import java.util.Set;

class RequestBlockingTest {

    private static final Set<ResourceType> BLOCKED_TYPES =
            Set.of(ResourceType.IMAGE, ResourceType.FONT, ResourceType.MEDIA);

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();

        List<RequestPattern> patterns = BLOCKED_TYPES.stream()
                .map(type -> new RequestPattern(Optional.of("*"), Optional.of(type), Optional.empty()))
                .toList();

        devTools.send(Fetch.enable(Optional.of(patterns), Optional.of(false)));

        devTools.addListener(Fetch.requestPaused(), paused -> {
            if (BLOCKED_TYPES.contains(paused.getResourceType())) {
                devTools.send(Fetch.failRequest(paused.getRequestId(),
                        org.openqa.selenium.devtools.v135.network.model.ErrorReason.BLOCKEDBYCLIENT));
            } else {
                devTools.send(Fetch.continueRequest(paused.getRequestId(),
                        Optional.empty(), Optional.empty(), Optional.empty(),
                        Optional.empty(), Optional.empty()));
            }
        });
    }

    @Test
    void pageLoadsWithoutImagesFontsOrMedia() {
        driver.get("https://example-shop.test/catalog");
        // Assert on functional DOM state, not on visual/pixel output, since
        // images/fonts are intentionally blocked in this run.
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

**Pitfall reminder applied here:** note the `else` branch always resolves unmatched-but-paused requests with `continueRequest` — this is the "catch-all continuation" best practice from Section 7/8, preventing indefinite hangs.

---

### 9.4 Scenario: Automating HTTP Basic Authentication via `Fetch.authRequired`

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.fetch.Fetch;
import org.openqa.selenium.devtools.v135.fetch.model.AuthChallengeResponse;

import java.util.Optional;

class BasicAuthInterceptionTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();

        // handleAuthRequests = true is the second argument to Fetch.enable
        devTools.send(Fetch.enable(Optional.empty(), Optional.of(true)));

        devTools.addListener(Fetch.authRequired(), authRequired ->
                devTools.send(Fetch.continueWithAuth(
                        authRequired.getRequestId(),
                        new AuthChallengeResponse(
                                AuthChallengeResponse.Response.PROVIDECREDENTIALS,
                                Optional.of("qa-service-account"),
                                Optional.of("s3cure-test-pass")))));

        devTools.addListener(Fetch.requestPaused(), paused ->
                devTools.send(Fetch.continueRequest(paused.getRequestId(),
                        Optional.empty(), Optional.empty(), Optional.empty(),
                        Optional.empty(), Optional.empty())));
    }

    @Test
    void authenticatesTransparentlyWithoutNativeDialog() {
        driver.get("https://staging.example.test/protected-report");
        // No native basic-auth dialog appears; assert on post-auth page content.
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

**Why the URL-embedded-credential alternative is weaker:** `https://user:pass@staging.example.test` works for a simple first navigation, but breaks across redirect chains on some Chromium versions, and the credentials are visible in browser history/logs and in `driver.getCurrentUrl()` assertions — the CDP approach never exposes the credential in the URL at all.

---

### 9.5 Scenario: Geolocation Override for Multi-Region Testing

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.browser.Browser;
import org.openqa.selenium.devtools.v135.browser.model.PermissionType;
import org.openqa.selenium.devtools.v135.emulation.Emulation;

import java.util.List;
import java.util.Optional;
import java.util.stream.Stream;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

import static org.junit.jupiter.api.Assertions.assertEquals;

class GeolocationOverrideTest {

    private ChromeDriver driver;
    private DevTools devTools;

    record Region(String label, double lat, double lon, String expectedLocale) {}

    static Stream<Region> regions() {
        return Stream.of(
                new Region("Mumbai, IN", 19.0760, 72.8777, "IN"),
                new Region("London, UK", 51.5072, -0.1276, "UK"),
                new Region("New York, US", 40.7128, -74.0060, "US"));
    }

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();

        devTools.send(Browser.grantPermissions(
                List.of(PermissionType.GEOLOCATION),
                Optional.of("https://example-shop.test"),
                Optional.empty()));
    }

    @ParameterizedTest(name = "Locale switches correctly for {0}")
    @MethodSource("regions")
    void appliesCorrectLocaleForGeolocation(Region region) {
        devTools.send(Emulation.setGeolocationOverride(
                Optional.of(region.lat()), Optional.of(region.lon()), Optional.of(1)));

        driver.get("https://example-shop.test/");

        String detectedLocale = driver.findElement(
                org.openqa.selenium.By.id("locale-indicator")).getText();

        assertEquals(region.expectedLocale(), detectedLocale,
                "Storefront should auto-detect locale from overridden geolocation");
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

**Technical validation:** `Browser.grantPermissions` must run *before* the page requests geolocation, otherwise the page's own permission prompt logic may reject the override (Pitfall #7). Using JUnit 5 `@ParameterizedTest` with a `record`-based data provider lets one test method validate multiple regions without duplicating driver setup — an architect-level pattern for data-driven CDP scenarios.

---

### 9.6 Scenario: Network Throttling with Named Profiles (2G/3G/4G)

```java
package com.company.automation.util;

public enum NetworkProfile {
    // latencyMs, downloadThroughputBytesPerSec, uploadThroughputBytesPerSec
    OFFLINE(0, 0, 0, true),
    GPRS_2G(500, 50 * 1024 / 8, 20 * 1024 / 8, false),
    REGULAR_3G(300, 750 * 1024 / 8, 250 * 1024 / 8, false),
    GOOD_3G(150, 1_500 * 1024 / 8, 750 * 1024 / 8, false),
    REGULAR_4G(70, 4_000 * 1024 / 8, 3_000 * 1024 / 8, false),
    NO_THROTTLING(0, -1, -1, false);

    public final int latencyMs;
    public final int downloadThroughput;
    public final int uploadThroughput;
    public final boolean offline;

    NetworkProfile(int latencyMs, int downloadThroughput, int uploadThroughput, boolean offline) {
        this.latencyMs = latencyMs;
        this.downloadThroughput = downloadThroughput;
        this.uploadThroughput = uploadThroughput;
        this.offline = offline;
    }
}
```

```java
package com.company.automation.tests;

import com.company.automation.util.NetworkProfile;
import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.network.Network;

import java.time.Duration;
import java.util.Optional;

class NetworkThrottlingTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();
        devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));
    }

    void applyProfile(NetworkProfile profile) {
        devTools.send(Network.emulateNetworkConditions(
                profile.offline,
                profile.latencyMs,
                profile.downloadThroughput,
                profile.uploadThroughput,
                Optional.empty()));
    }

    @Test
    void appShowsOfflineBannerUnderRegular3G() {
        applyProfile(NetworkProfile.REGULAR_3G);

        long start = System.currentTimeMillis();
        driver.get("https://example-shop.test/catalog");
        long durationMs = System.currentTimeMillis() - start;

        Assertions.assertTrue(durationMs > 500,
                "Page load under Regular 3G should be measurably slower than an unthrottled load");
    }

    @Test
    void appShowsOfflineBannerWhenNetworkDrops() {
        driver.get("https://example-shop.test/catalog");
        applyProfile(NetworkProfile.OFFLINE);

        driver.findElement(org.openqa.selenium.By.id("refresh-data")).click();

        Assertions.assertTrue(driver.findElement(
                org.openqa.selenium.By.cssSelector(".offline-banner")).isDisplayed());
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

---

### 9.7 Scenario: CPU Throttling for Low-End Device Simulation

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.emulation.Emulation;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

class CpuThrottlingTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();
    }

    @Test
    void heavyJsOperationRemainsResponsiveUnder4xCpuThrottle() {
        devTools.send(Emulation.setCPUThrottlingRate(4));

        driver.get("https://example-shop.test/reports/heavy-chart");

        new WebDriverWait(driver, Duration.ofSeconds(15)).until(d ->
                d.findElement(org.openqa.selenium.By.id("chart-rendered-flag")).isDisplayed());

        // Assert UI remained interactive (e.g., a spinner did not block > N seconds)
    }

    @AfterEach
    void tearDown() {
        devTools.send(Emulation.setCPUThrottlingRate(1)); // always reset to baseline
        if (driver != null) driver.quit();
    }
}
```

**Best practice applied:** the throttle is reset (`setCPUThrottlingRate(1)`) in `@AfterEach` even before quitting the driver — relevant when the driver instance is pooled/reused across tests rather than freshly created each time.

---

### 9.8 Scenario: Device Emulation (Mobile Viewport + Touch + User-Agent)

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.emulation.Emulation;
import org.openqa.selenium.devtools.v135.network.Network;

import java.util.Optional;

class DeviceEmulationTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();
    }

    @Test
    void rendersMobileLayoutForIphoneStyleViewport() {
        devTools.send(Emulation.setDeviceMetricsOverride(
                390, 844, 3, true,
                Optional.empty(), Optional.empty(), Optional.empty(),
                Optional.empty(), Optional.empty(), Optional.empty(),
                Optional.empty(), Optional.empty(), Optional.empty()));

        devTools.send(Network.setUserAgentOverride(
                "Mozilla/5.0 (iPhone; CPU iPhone OS 17_5 like Mac OS X) "
                        + "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1",
                Optional.empty(), Optional.empty(), Optional.empty()));

        driver.get("https://example-shop.test/");

        Assertions.assertTrue(driver.findElement(
                org.openqa.selenium.By.cssSelector(".mobile-nav-hamburger")).isDisplayed(),
                "Mobile navigation should render at emulated iPhone viewport width");
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

---

### 9.9 Scenario: Cookie Management + Permission Automation

```java
package com.company.automation.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.Cookie;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.HasDevTools;
import org.openqa.selenium.devtools.v135.browser.Browser;
import org.openqa.selenium.devtools.v135.browser.model.PermissionType;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Optional;

class CookieAndPermissionsTest {

    private ChromeDriver driver;
    private DevTools devTools;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(new ChromeOptions().addArguments("--headless=new"));
        devTools = ((HasDevTools) driver).getDevTools();
        devTools.createSession();
    }

    @Test
    void skipsLoginWithPreAuthenticatedSessionCookie() {
        driver.get("https://example-shop.test/"); // must navigate first to set cookie domain

        Cookie sessionCookie = new Cookie.Builder("SESSION_ID", "qa-fixture-session-token")
                .domain("example-shop.test")
                .path("/")
                .expiresOn(java.util.Date.from(Instant.now().plus(Duration.ofHours(2))))
                .isSecure(true)
                .isHttpOnly(true)
                .build();

        driver.manage().addCookie(sessionCookie);
        driver.navigate().refresh();

        Assertions.assertTrue(driver.findElement(
                org.openqa.selenium.By.cssSelector(".account-menu")).isDisplayed(),
                "Pre-authenticated cookie should skip the login screen");
    }

    @Test
    void autoGrantsNotificationPermissionWithoutNativePrompt() {
        devTools.send(Browser.grantPermissions(
                List.of(PermissionType.NOTIFICATIONS),
                Optional.of("https://example-shop.test"),
                Optional.empty()));

        driver.get("https://example-shop.test/alerts-opt-in");

        Assertions.assertEquals("granted",
                ((org.openqa.selenium.JavascriptExecutor) driver)
                        .executeScript("return Notification.permission;"));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

---

### 9.10 Scenario: Browser Configuration — Headless, Extensions, Certs, Profiles

```java
package com.company.automation.config;

import org.openqa.selenium.chrome.ChromeOptions;

import java.io.File;
import java.nio.file.Path;

public final class BrowserOptionsBuilder {

    private BrowserOptionsBuilder() {}

    public static ChromeOptions headlessCiOptions() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments(
                "--headless=new",          // modern headless: same render path as headed
                "--window-size=1920,1080",
                "--disable-gpu",            // stabilizes CI VMs without a real GPU
                "--no-sandbox",              // required in many containerized CI runners
                "--disable-dev-shm-usage"    // avoids /dev/shm size issues in Docker
        );
        options.setAcceptInsecureCerts(true); // trust staging self-signed certs
        return options;
    }

    public static ChromeOptions withUnpackedExtension(Path extensionDir) {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--load-extension=" + extensionDir.toAbsolutePath());
        return options;
    }

    public static ChromeOptions withPackedExtension(File crxFile) {
        ChromeOptions options = new ChromeOptions();
        options.addExtensions(crxFile);
        return options;
    }

    public static ChromeOptions withPersistentProfile(Path profileDir) {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--user-data-dir=" + profileDir.toAbsolutePath());
        options.addArguments("--profile-directory=Default");
        return options;
    }
}
```

---

### 9.11 Migration Example: `DesiredCapabilities` (Legacy) → `ChromeOptions` (Modern)

```java
// ============ BEFORE (Selenium 3 legacy pattern — DO NOT USE) ============
DesiredCapabilities caps = new DesiredCapabilities();
caps.setCapability("browserName", "chrome");
caps.setCapability("acceptInsecureCerts", true);
caps.setCapability("goog:chromeOptions",
        java.util.Map.of("args", java.util.List.of("--headless")));
WebDriver legacyDriver = new ChromeDriver(caps); // fragile, untyped, no IDE help

// ============ AFTER (Selenium 4 idiomatic pattern) ============
ChromeOptions options = new ChromeOptions();
options.setAcceptInsecureCerts(true);
options.addArguments("--headless=new");
WebDriver driver = new ChromeDriver(options);

// If a legacy Capabilities object must still be merged (e.g., from a shared
// Grid-matching utility), merge it into the typed Options object rather than
// constructing the driver from the raw map directly:
Capabilities legacyExtras = caps; // implements Capabilities
ChromeOptions merged = options.merge(legacyExtras);
WebDriver mergedDriver = new ChromeDriver(merged);
```

---

### 9.12 Architect-Level Abstraction: `NetworkController`

```java
package com.company.automation.cdp;

import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v135.fetch.Fetch;
import org.openqa.selenium.devtools.v135.fetch.model.HeaderEntry;
import org.openqa.selenium.devtools.v135.network.model.RequestId;

import java.util.Base64;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * Centralizes all Fetch-domain mocking logic so step definitions / test
 * classes never call devTools.send(Fetch...) directly. This is the
 * "isolate CDP logic behind an abstraction" best practice from Section 8.
 */
public class NetworkController implements AutoCloseable {

    private final DevTools devTools;

    public NetworkController(DevTools devTools) {
        this.devTools = devTools;
        this.devTools.send(Fetch.enable(Optional.empty(), Optional.of(false)));
    }

    public void mockJsonResponse(String urlSubstring, int status, String jsonBody) {
        devTools.addListener(Fetch.requestPaused(), paused -> {
            if (paused.getRequest().getUrl().contains(urlSubstring)) {
                String encoded = Base64.getEncoder().encodeToString(jsonBody.getBytes());
                devTools.send(Fetch.fulfillRequest(
                        paused.getRequestId(), status,
                        Optional.of(List.of(new HeaderEntry("Content-Type", "application/json"))),
                        Optional.empty(), Optional.of(encoded), Optional.empty()));
            } else {
                continueUnmatched(paused.getRequestId());
            }
        });
    }

    private void continueUnmatched(RequestId requestId) {
        devTools.send(Fetch.continueRequest(requestId,
                Optional.empty(), Optional.empty(), Optional.empty(),
                Optional.empty(), Optional.empty()));
    }

    @Override
    public void close() {
        devTools.send(Fetch.disable());
    }
}
```

Usage in a test:

```java
try (NetworkController network = new NetworkController(devTools)) {
    network.mockJsonResponse("/api/inventory", 503, "{\"error\":\"service unavailable\"}");
    driver.get("https://example-shop.test/inventory");
    // assert on the UI's error-handling path
}
```

---

## 10. Technical Validation — Why These Solutions Work

- **Mocking (9.2, 9.12):** The renderer has no way to distinguish a `Fetch.fulfillRequest` response from a genuine network response — both arrive through the same internal network-service-to-renderer IPC path. This is why CDP mocking is high-fidelity: the JS `fetch()`/`XMLHttpRequest` promise resolves identically either way.
- **Blocking (9.3):** `Fetch.failRequest` with `ErrorReason.BLOCKEDBYCLIENT` causes the browser to surface the failure to JS exactly as a `net::ERR_BLOCKED_BY_CLIENT` — the same error class real ad-blockers produce — so application error-handling code paths are exercised authentically, not bypassed.
- **Basic Auth (9.4):** `Fetch.continueWithAuth` intervenes at the exact point Chromium's network stack would otherwise pop the native credentials dialog — since that dialog is OS chrome (not DOM), it is fundamentally unautomatable via `driver.switchTo().alert()`; CDP is the only reliable path (aside from URL-embedded credentials, which have the redirect/logging weaknesses noted in 9.4).
- **Geolocation (9.5):** `Emulation.setGeolocationOverride` patches the value returned by the browser's internal geolocation provider that backs `navigator.geolocation`; JS code cannot distinguish an override from a real GPS/network-based fix.
- **Throttling (9.6, 9.7):** `Network.emulateNetworkConditions` and `Emulation.setCPUThrottlingRate` operate inside the browser process itself (not an external shaper), so they throttle exactly what DevTools' own UI throttling dropdown throttles — verified reproducible results across machines regardless of the host's actual network/CPU speed.
- **Device emulation (9.8):** `Emulation.setDeviceMetricsOverride` changes the values `window.innerWidth/innerHeight`, `devicePixelRatio`, and CSS media-query evaluation see — so responsive breakpoints genuinely fire as they would on the physical device, not just visually scaled.

---

## 11. Debugging

- **DevTools Network panel (manual cross-check):** Run the same scenario once with a visible (non-headless) browser and the DevTools Network panel open to visually confirm which requests are being intercepted/mocked/blocked before trusting the automated assertion.
- **CDP command/event tracing:** Temporarily add a catch-all listener that logs every `Fetch.requestPaused` URL and resource type to stdout/logger — this quickly reveals pattern-matching mistakes (e.g., a glob pattern that's too broad or too narrow).
- **Selenium/ChromeDriver logs:** Launch with `--verbose` on `chromedriver` (or set `ChromeDriverService` log level) to see the raw WebSocket traffic between Selenium and the browser when a CDP command silently fails.
- **Browser console logs via `driver.manage().logs()` or BiDi `log` module:** Surface JS exceptions caused by an unexpected mocked response shape (e.g., the app's JS expected a field the mock omitted).
- **Network HAR capture:** For proxy-based approaches, exporting a HAR file from BrowserMob Proxy after a failing run gives a full request/response timeline to compare against the mock definitions.
- **Breakpoint strategy:** Place breakpoints inside the `Fetch.requestPaused` lambda itself (not just in the `@Test` method) — since it executes asynchronously on a Selenium-managed callback thread, breakpoints in the main test thread will not trigger when the interception logic misbehaves.
- **Timeout diagnosis:** If a test hangs specifically when `Fetch.enable` is active, suspect an unresolved paused request (Pitfall #3) before suspecting a locator or explicit-wait issue.

---

## 12. Interview Preparation

### 12.1 Beginner-Level

**Q1. What is the difference between Selenium's WebDriver protocol and Chrome DevTools Protocol (CDP)?**
A: WebDriver (W3C) is a vendor-neutral standard supported by every browser vendor for basic automation (navigation, element interaction, cookies). CDP is Chromium's own, much richer, proprietary protocol that Selenium 4 additionally tunnels into for Chrome/Edge-specific power features like network interception and emulation that W3C WebDriver doesn't standardize.

**Q2. How do you make Selenium wait for a page to load with poor network conditions?**
A: Combine an explicit `WebDriverWait` with a specific `ExpectedCondition` on a meaningful DOM state, rather than relying on implicit waits; if network throttling is deliberately applied via CDP, increase the explicit wait's timeout accordingly for that specific test rather than raising a global implicit wait.

**Q3. What is `ChromeOptions` used for?**
A: A typed, Chrome-specific object for configuring browser launch behavior — headless mode, window size, extensions, user-data directories, and capabilities like `acceptInsecureCerts` — passed into the `ChromeDriver` constructor.

### 12.2 Intermediate-Level

**Q4. How would you mock a backend API call in a Selenium test without touching the application's code?**
A: Use Selenium 4's CDP `Fetch` domain: enable interception with a `RequestPattern` matching the API's URL/resource type, then in the `Fetch.requestPaused` listener call `Fetch.fulfillRequest` with a fabricated status/body, or `Fetch.continueRequest` for anything not meant to be mocked.

**Q5. How do you test how your application behaves on a slow 3G connection?**
A: `Network.enable()` followed by `Network.emulateNetworkConditions(false, latencyMs, downloadThroughput, uploadThroughput, ...)` with values matching a standard 3G profile (roughly 750 Kbps down / 250 Kbps up / 300 ms RTT), then drive the app and assert on loading-state UI or timing.

**Q6. What's the difference between `DesiredCapabilities` and `ChromeOptions`, and why did Selenium 4 move away from the former?**
A: `DesiredCapabilities` was an untyped, browser-agnostic map prone to silent misconfiguration across browsers. `ChromeOptions` (and its Firefox/Edge counterparts) are per-browser, strongly typed classes implementing the `Capabilities` interface, giving compile-time safety and IDE support while remaining mergeable for Grid capability matching.

### 12.3 Advanced-Level

**Q7. Explain how CDP's `Fetch` domain differs from the older `Network.setRequestInterception`, and why the former is preferred.**
A: `Network.setRequestInterception` was an earlier, coarser CDP mechanism for pausing requests with limited modification ability. The `Fetch` domain supersedes it with a cleaner pause/resume model (`requestPaused` event + `continueRequest`/`fulfillRequest`/`failRequest`), supports response-stage interception (not just request-stage), and integrates with `Fetch.authRequired` for auth automation — capabilities the older domain lacked or handled less reliably.

**Q8. What are the architectural differences between BiDi and CDP, and when would you choose BiDi over CDP in an enterprise framework?**
A: CDP requires a second WebSocket direct to the Chromium browser process and is versioned per Chrome release, meaning it's fragile across Chrome auto-updates and Chromium-only. BiDi is a W3C standard negotiated as part of the same WebDriver session and (in principle) portable to Firefox and eventually Safari. Choose BiDi when the framework's roadmap requires genuine cross-browser advanced automation (e.g., console log capture or network events across Chrome and Firefox) and current BiDi support in your Selenium version covers the needed features; choose CDP when you need Chromium-only deep capabilities (CPU throttling, full `Fetch`-domain fine-grained mocking) not yet available in BiDi.

**Q9. How would you handle a scenario where CDP interception causes intermittent test hangs in a CI pipeline running hundreds of tests in parallel?**
A: First isolate whether every `Fetch.requestPaused` event has a guaranteed resolution path including a default/catch-all continue — the single most common cause. Second, verify CDP sessions/listeners are scoped per test (not leaking across a shared/pooled driver, which can cause a stale listener from a previous test to intercept a later test's requests). Third, check for CDP-version mismatches between the Selenium jar and the CI's Chrome version, which can cause specific command overloads to silently no-op.

### 12.4 Architect-Level

**Q10. Design a strategy for maintaining CDP-based tests across frequent Chrome version upgrades in a large enterprise Selenium Grid deployment.**
A: Pin Chrome and ChromeDriver versions together in CI images; track the Selenium release notes for the corresponding `devtools.vXXX` package added per release; keep CDP-specific logic behind an abstraction (`NetworkController`/`EmulationController`) so an import-path bump (`v134`→`v135`) is a localized, single-file change rather than scattered across hundreds of test files; add a smoke-test suite specifically exercising CDP features (mocking, geolocation, throttling) that runs first in the pipeline and fails fast on version drift, before the full regression suite executes.

**Q11. A stakeholder asks whether to invest further in CDP-based automation or begin migrating toward BiDi. What factors drive that decision?**
A: Key factors: (1) Is the org's browser support matrix Chromium-only, or does/will it need Firefox/Safari parity? (2) Does the required feature set (CPU throttling, fine-grained `Fetch` body mutation) currently have BiDi coverage in the pinned Selenium version, or is it CDP-exclusive today? (3) What is the org's tolerance for API churn — BiDi's Java API is newer and less stable release-to-release than the mature CDP wrapper. (4) Total cost of maintaining CDP-version-pinned code vs. investing in an abstraction layer that could later swap CDP calls for BiDi calls with minimal test-code churn. Recommendation pattern: keep the CDP abstraction layer from 9.12 as the seam — the moment BiDi covers a needed feature reliably, only that internal implementation swaps, not every test.

### 12.5 India Market — Frequently Asked / Tricky Questions

**Q12. (Common at Cognizant/Capgemini/TCS-style services interviews) "How do you test geo-restricted content — e.g., a feature only visible to users in a specific country — without a VPN?"**
A: Use CDP's `Emulation.setGeolocationOverride` (paired with `Browser.grantPermissions` for the `geolocation` permission on the target origin) to force `navigator.geolocation` to resolve to coordinates inside the target country, then assert on the geo-gated feature's visibility. Note: if the app determines region via **IP address** (server-side GeoIP) rather than browser geolocation API, this CDP approach won't help — that case genuinely needs a VPN/proxy with an exit node in the target country, or a server-side header override if the backend supports one for staging.

**Q13. (Common at Amazon India / Microsoft India product interviews) "How would you test that your page performs acceptably on a throttled 3G network for users in Tier-2/Tier-3 Indian cities?"**
A: Apply `Network.emulateNetworkConditions` with a "Regular 3G" profile (or the app's actual target-market bandwidth data if available) combined with `Emulation.setCPUThrottlingRate` for a mid-range Android device profile, then assert against a defined performance budget (e.g., time-to-interactive under N seconds) rather than just "the page loads" — tie the throttling profile values to real analytics data on the target user base's device/network mix rather than guessing.

**Q14. (Frequently seen as a scenario/whiteboard question, EPAM/ThoughtWorks-style) "Your team's Selenium suite mocks 40 different API endpoints across CDP `Fetch` interception scattered through step definitions. New joiners keep breaking tests by duplicating conflicting patterns. How do you fix the framework?"**
A: This is exactly the anti-pattern the `NetworkController` abstraction (Section 9.12) is designed to prevent — centralize interception logic behind one class with named, discoverable methods (`mockJsonResponse`, `blockResourceType`, `mockAuthChallenge`) instead of ad hoc `devTools.send(Fetch...)` calls per step definition; add a single catch-all listener registered once at construction so individual mock registrations can't create competing/duplicate listeners; add a lightweight registry (`Map<String, MockDefinition>`) so pattern conflicts are detected at registration time with a clear exception rather than a silent race between two listeners.

**Q15. (Tricky conceptual question seen in LinkedIn discussion threads among Indian QA leads) "If BiDi is the W3C standard, why hasn't Selenium deprecated CDP support yet?"**
A: Because BiDi, while standardized, does not yet have full feature parity with CDP for several capabilities enterprises actively depend on today — most notably CPU throttling and the fine-grained request/response body mutation the `Fetch` domain offers. Deprecating CDP before BiDi reaches parity would strand real production test suites with no migration path. Selenium's public direction is to grow BiDi coverage and offer CDP as a Chromium-specific escape hatch for as long as a capability gap exists — not an overnight cutover.

---

## 13. Practice

### 13.1 Hands-On Exercise — Mocked Checkout Failure

**Task:** Using a public demo e-commerce site of your choice (or a local sample app), write a JUnit 5 test that intercepts the checkout API call and forces a `503 Service Unavailable` response, then asserts the UI shows a retry-able error message rather than crashing.

**Acceptance Criteria:**
- Test uses CDP `Fetch` domain (not a proxy or backend flag).
- Only the checkout endpoint is mocked; all other requests (product images, CSS, analytics) pass through unmodified.
- Test asserts on a specific error-state element, not merely "no exception was thrown."
- Includes a `@AfterEach` that disables `Fetch` interception and quits the driver.

**Sample Test Data:** Mock response body: `{"error": "checkout_temporarily_unavailable", "retryAfterSeconds": 30}`, status `503`.

### 13.2 Mini Assignment — Multi-Region Locale Smoke Test

**Task:** Build a parameterized JUnit 5 test (as in Section 9.5) that verifies the correct currency symbol renders for at least 4 geolocation regions (e.g., India ₹, UK £, US $, Japan ¥) on a target application's homepage.

**Acceptance Criteria:**
- Uses `Emulation.setGeolocationOverride` + `Browser.grantPermissions`.
- Data-driven via `@ParameterizedTest`/`@MethodSource`, not four copy-pasted test methods.
- Fails with a clear, region-labeled assertion message if any region's currency symbol is wrong.

**Sample Test Data:** India (19.0760, 72.8777, ₹), UK (51.5072, -0.1276, £), US (40.7128, -74.0060, $), Japan (35.6762, 139.6503, ¥).

### 13.3 Challenge Exercise — Performance Budget Under Constrained Conditions

**Task:** Design and implement a test that combines network throttling (`Regular 3G`) and CPU throttling (`4x slowdown`) simultaneously, measures time from `driver.get()` call to a specific "app fully interactive" DOM marker appearing, and fails the test if that duration exceeds a defined performance budget (e.g., 8 seconds).

**Acceptance Criteria:**
- Both throttling mechanisms are applied via CDP in the same test.
- The "interactive" signal is a real DOM/JS marker (e.g., an element with `data-app-ready="true"`), not a fixed `Thread.sleep()`.
- The performance budget value is a named constant, not a magic number inline in the assertion.
- The test resets both throttles in `@AfterEach` even if the assertion fails, so driver reuse (if pooled) isn't corrupted for subsequent tests.

**Sample Test Data:** Performance budget constant: `Duration.ofSeconds(8)`. Throttle profile: `NetworkProfile.REGULAR_3G` + CPU rate `4`.

---

## 14. Summary

This chapter moved beyond the W3C WebDriver baseline into Selenium 4's advanced Chromium-specific and emerging cross-browser capabilities. You learned the internal architecture distinguishing **CDP** (a rich, versioned, Chromium-proprietary WebSocket protocol tunneled by Selenium's `DevTools` class) from **BiDi** (a W3C-standardized, session-integrated, event-driven protocol aimed at eventual cross-browser parity). You implemented network interception via the `Fetch` domain for mocking, blocking, and basic-auth automation; geolocation override for multi-region testing; network and CPU throttling for realistic performance testing; device emulation for responsive-design validation; and browser permission automation to eliminate unautomatable native prompts. You also completed the migration from legacy `DesiredCapabilities` to typed, per-browser `Options` classes, and learned to weigh CDP against proxy-based and server-side-stub alternatives for network control depending on cross-browser needs, HTTPS handling, and team ownership boundaries.

## 15. Revision Notes

- CDP = Chromium-only, versioned, richer today; BiDi = W3C standard, session-integrated, growing but not yet at parity for CPU throttling/full device emulation.
- `Fetch` domain (pause → continue/fulfill/fail) supersedes the older `Network.setRequestInterception` for interception work.
- Always pair `Emulation.setGeolocationOverride` with `Browser.grantPermissions` for the target origin.
- `Network.emulateNetworkConditions` and `Emulation.setCPUThrottlingRate` throttle inside the browser process — more realistic than external/proxy throttling.
- `ChromeOptions`/`FirefoxOptions`/`EdgeOptions` replace `DesiredCapabilities`; all implement `Capabilities` and support `.merge()`.
- `--headless=new` is the modern headless mode; avoid the legacy `--headless` flag for new frameworks.
- Every `Fetch.requestPaused` code path must resolve (continue/fulfill/fail) — unresolved pauses hang the browser and the test.

## 16. Common Mistakes Checklist

- [ ] Forgot to call the domain's `.enable()` command before registering a listener for it.
- [ ] Left a `Fetch.requestPaused` branch with no `continueRequest`/`fulfillRequest`/`failRequest` call.
- [ ] Used legacy `--headless` instead of `--headless=new`, causing pixel/rendering mismatches.
- [ ] Called `Emulation.setGeolocationOverride` before granting the `geolocation` permission for the origin.
- [ ] Mixed implicit waits with CDP-throttled network conditions, producing unpredictable total timeouts.
- [ ] Left `DesiredCapabilities` in shared framework utility code instead of migrating fully to typed `Options`.
- [ ] Used arbitrary/made-up throttling numbers instead of named, reproducible network profiles.
- [ ] Did not reset CPU/network throttling in teardown when reusing/pooling driver instances.
- [ ] Chose a local MITM proxy for an HSTS/cert-pinned application instead of CDP (which avoids TLS interception entirely).
- [ ] Scattered raw `devTools.send(Fetch...)` calls across step definitions instead of centralizing behind a `NetworkController`-style abstraction.

## 17. Key Takeaways

1. **CDP gives Selenium Chromium-specific superpowers; BiDi is the standardized, cross-browser future — know both, and know today's feature-parity gaps.**
2. **The `Fetch` domain's pause/resume model is the modern, correct way to mock, block, and authenticate network requests — every paused request must be resolved.**
3. **Realistic performance and location testing (throttling, geolocation) is only trustworthy when implemented inside the browser process via CDP, not guessed at externally.**
4. **Typed `*Options` classes are strictly superior to `DesiredCapabilities` — there is no scenario in a Selenium 4 codebase where reverting to the legacy map-based API is justified.**
5. **Architecture discipline (centralized controllers, catch-all listener resolution, named throttling profiles) is what separates an enterprise-grade CDP framework from a fragile pile of ad hoc `devTools.send()` calls.**
