# Research: Can PP core load, calculate and save a client file outside the workbench?

Ticket: `.scratch/ai-cli/issues/01-headless-core-feasibility.md`
Sources: the repository source at branch `dev` (commit `16b52e1b9`), plus bytecode of the Eclipse platform jars that the Tycho build resolved into `~/.m2/repository/p2/osgi/bundle/` (`org.eclipse.core.runtime` 3.34.200, `org.eclipse.osgi` 3.24.200). Paths are relative to the repo root unless noted otherwise.

## TL;DR

- **Yes, on a plain classpath.** You can load a client with `ClientFactory`, compute `ClientSnapshot`, `ClientPerformanceSnapshot`, IRR, TTWROR (`PerformanceIndex`) and realized/unrealized gains, convert currencies, change the model, and save it again. None of this needs a running OSGi framework, a workbench, an SWT `Display` or e4 DI.
- `name.abuchen.portfolio` has **no `Bundle-Activator`**. `PortfolioPlugin` is in the UI bundle, so no activator blocks the core.
- The packages on the load/compute path (`snapshot`, `math`, `money`) import nothing from SWT, e4 or OSGi. The only exceptions are `IProgressMonitor`, e4 DI *annotations* on `ExchangeRateProviderFactory`, and `Platform`/`FrameworkUtil` inside `ECBExchangeRateProvider`. All of them are inert outside OSGi.
- SWT appears only in `ImageManager`, `LimitPriceSettings`, `ImageUtil*` and `ColorConversion`. Those are UI helpers, or are reached only through colour/logo methods. The classpath still needs the SWT jar, plus `org.eclipse.equinox.common`, `org.eclipse.core.runtime` and `org.eclipse.osgi` (for `NLS`, `Platform`, `FrameworkUtil`, `IProgressMonitor`), but none of them has to be *running*.
- Outside OSGi, `PortfolioLog` writes to **`System.out`**: `ClientFactory` logs "Loaded … / Saving …" on every file load and save. A CLI that prints JSON on stdout must redirect or filter this.
- Exchange rates: providers are found with `java.util.ServiceLoader`, so you must use the **built bundle jar**, which contains `META-INF/services`. Without OSGi, the ECB provider has no state location: it neither reads nor writes PP's cached `ecb_exchange_rates.pb`. It falls back to built-in 2015 default rates unless you call `update()`, which goes to the network (ECB). Securities that hold their own exchange-rate prices are used offline.
- Quote feeds and the PP/PortfolioReport feeds (network, `FrameworkUtil.getBundle(..).getVersion()`, OAuth `platform:/meta` secure storage) **do** need OSGi or will NPE/fail. They are not needed for load/compute/save.
- The existing tests are Tycho `eclipse-test-plugin` tests: an OSGi fragment that runs headless under tycho-surefire. Many of them already do exactly "load an XML client and compute snapshots/performance" with a test converter. Code comments and `infinitest.filters` show the same tests are also run on a plain JVM (Infinitest), except a few exchange-rate tests.
- **Recommendation:** use a plain classpath of the Tycho-built bundle jar plus its resolved dependency jars. Inject a `ExchangeRateProviderFactory(client)` → `CurrencyConverterImpl`. Take embedded Equinox (headless `IApplication`, `-data <PP workspace>`) only if you need PP's cached ECB rates or online quote feeds.

## Findings

### 1. Bundle wiring: no activator, OSGi only declarative

- The manifest has no `Bundle-Activator` header, only `Bundle-ActivationPolicy: lazy` ([name.abuchen.portfolio/META-INF/MANIFEST.MF](../../../name.abuchen.portfolio/META-INF/MANIFEST.MF) line 84). The only OSGi component is `ImageioSpiRegistration` (line 85; `OSGI-INF/name.abuchen.portfolio.util.ImageioSpiRegistration.xml`), which binds optional ImageIO SPIs for logos.
- `PortfolioPlugin` lives in `name.abuchen.portfolio.ui/src/name/abuchen/portfolio/ui/PortfolioPlugin.java`, not in the core bundle.
- The manifest imports `org.eclipse.e4.core.di.*`, `org.eclipse.equinox.security.storage`, `org.eclipse.osgi.util` and `org.eclipse.swt{,.graphics,.widgets}` (MANIFEST.MF lines 59-65). It requires `org.eclipse.core.runtime`, httpclient5, json-path, gson, `org.osgi.service.component`, and `name.abuchen.portfolio.pdfbox1/3` (lines 75-82). These are needed for *resolution* inside OSGi. On a plain classpath only the classes that actually get loaded matter (see §2).
- `build.properties` packs `META-INF/`, `.` (classes from `src/,protos/`) and `OSGI-INF/` into the jar ([name.abuchen.portfolio/build.properties](../../../name.abuchen.portfolio/build.properties)). So the built jar contains `META-INF/services/*` and the `messages*.properties`/taxonomy template resources that the code loads from the classpath.

### 2. What the load/compute/save path actually references

Grep of `^import org.eclipse.(swt|e4|equinox|osgi|core|jface)` and `^import org.osgi` across `name.abuchen.portfolio/src` (16 + 9 files). Relevant hits:

| Class | Eclipse/OSGi reference | Effect on a plain JVM |
|---|---|---|
| `model/ClientFactory.java` L65-66 | `IProgressMonitor`, `Platform` | `load(File, char[], IProgressMonitor)` (L599) only needs a monitor (`NullProgressMonitor` from equinox.common). `writeFile` calls `Platform.getOS()` (L734). Bytecode of `InternalPlatform.getContextProperty` falls back to `System.getProperty("osgi.os")` when there is no `BundleContext`, so it returns `"unknown"`: no crash. The only effect is that the file lock is attempted on macOS too (L734-735; the `IOException` is caught at L737-743). |
| `model/ClientFactory.java` L86/L990 | `online.impl.YahooFinanceQuoteFeed.ID` | A compile-time `String` constant, inlined. The feed class is not loaded. |
| `PortfolioLog.java` L5-9 | `Platform.getLog`, `FrameworkUtil.getBundle` | Outside OSGi `FrameworkUtil.getBundle` returns `null`, and `ILog.of(null)` → `InternalPlatform.getLog`: when `!isRunning()` it returns `new Log(bundle, null)`. `Log.log` with a null logger then **prints the status to `System.out`** (bytecode of `org.eclipse.core.internal.runtime.Log.log`). `PortfolioLog.log` additionally catches NPE/IAE and prints to `System.err` (L22-34; comment: "when running unit tests via Infinitest, the platform log is not available"). |
| `Messages.java` L3, L376-379 | `org.eclipse.osgi.util.NLS` | `NLS.initializeMessages` loads `name/abuchen/portfolio/messages*.properties` from the class loader. It needs the `org.eclipse.osgi` jar on the classpath, but no running framework. |
| `money/ExchangeRateProviderFactory.java` L20-23, L173, L187-189 | `jakarta.inject.Inject`, e4 `@Optional`, `@EventTopic` | These are only annotations: the public constructor `ExchangeRateProviderFactory(Client)` can be called directly. The event hook `onSecurityEdited` only clears the cache. Without DI you call `clearCache()` yourself after editing exchange-rate securities. |
| `money/ExchangeRateProvider.java` L6 | `IProgressMonitor` | Only in `load/update/save` signatures. |
| `money/impl/ECBExchangeRateProvider.java` L18-20, L62-71, L200-212 | `Platform.getStateLocation`, `FrameworkUtil` | The storage dir is resolved lazily. When the bundle is `null`, it returns `null` → `Optional.empty()`, so `load()` is a no-op and `save()` logs a warning (L153). The constructor always seeds default rates from 2015-12-18 (`fillInDefaultData`, L71, L236ff). `update()` fetches `https://www.ecb.europa.eu/stats/eurofxref/` (`ECBUpdater.java` L34). |
| `model/Classification.java` L492-517 → `util/ColorConversion.java` | `org.eclipse.swt.graphics.RGB/RGBA` | Pure-Java value classes in the SWT jar, used only by the colour-assignment helpers. They need the SWT jar on the classpath, but no native library or `Display`. |
| `model/ImageManager.java`, `model/LimitPriceSettings.java`, `util/ImageUtil*.java` | `Image`, `ImageData`, `Color`, `GC` | UI-only helpers (logos, limit-price colours). They are not referenced from `Client`, persistence or snapshots. `ImageUtil` already falls back from ImageIO to SWT (`util/ImageUtil.java` L17-42). |
| `datatransfer/csv/CSVConfigManager.java` L16-23 | e4 `@Preference`, `IEclipsePreferences` | Only for CSV import configuration. Not on the path. |
| `oauth/impl/TokenStorage.java` L40, L207-208 | `FileLocator.resolve("platform:/meta/...")`, `SecurePreferencesFactory` | `platform:` URLs need the OSGi URL handlers. Outside OSGi this fails with an `IOException`, which is caught and logged (L58). The token stays null, so the PP quote feed is unauthenticated. It is reached via the `OAuthClient.INSTANCE` static (`oauth/OAuthClient.java` L38, L42) when `PortfolioPerformanceFeed` is loaded. |
| `online/impl/{PortfolioPerformanceFeed,PortfolioReportQuoteFeed,DivvyDiary*,MyDividends24Uploader}.java` | `FrameworkUtil.getBundle(X.class).getVersion()` | **NPE outside OSGi** when these feeds build their User-Agent (e.g. `PortfolioPerformanceFeed.java` L331, `PortfolioReportQuoteFeed.java` L151). They are only hit when you update quotes or dividends online. |

The calculation packages are clean. Grep of `^import (org.eclipse|org.osgi|jakarta)` in `snapshot/**`, `math/**` and `money/**` finds only the three `money` files listed above. `model/**` finds only `ClientFactory`, `ImageManager` and `LimitPriceSettings`. There are no pdfbox, httpclient or `datatransfer.pdf` references from `model/snapshot/math/money`. The one `online` import, in `snapshot/QuoteQualityMetrics.java` L10, is the `QuoteFeed` interface. **The pdfbox bundles are irrelevant** unless you import PDFs.

The public entry points need only a `Client` and a `CurrencyConverter`:
- `ClientSnapshot.create(Client, CurrencyConverter, LocalDate)`: `snapshot/ClientSnapshot.java` L33
- `new ClientPerformanceSnapshot(Client, CurrencyConverter, Interval[, useFifo])`: `snapshot/ClientPerformanceSnapshot.java` L179-189
- `PerformanceIndex.forClient(Client, CurrencyConverter, Interval, List<Exception>)` (TTWROR): `snapshot/PerformanceIndex.java` L73-79
- `ClientIRRYield.create(Client, ClientSnapshot, ClientSnapshot)`: `snapshot/ClientIRRYield.java` L22
- `LazySecurityPerformanceSnapshot.create(Client, CurrencyConverter, Interval)` (per-security IRR/TTWROR, realized/unrealized gains): `snapshot/security/LazySecurityPerformanceSnapshot.java` L14
- `new CurrencyConverterImpl(ExchangeRateProviderFactory, String termCurrency)`: `money/CurrencyConverterImpl.java` L17
- Save: `ClientFactory.save(Client, File)` (L677) and `saveAs(Client, File, char[], Set<SaveFlag>)` (L690). Both use `ProtobufWriter`, `PlainWriter`, XStream and the encryptor (`buildPersister`, L762ff), which are all plain Java.

`Client` uses `java.beans.PropertyChangeSupport` (`model/Client.java` L4, L38), so the JVM must include the `java.desktop` module. That is standard in a full JDK, and it works headless.

### 3. Service discovery (ServiceLoader, not the extension registry)

- Exchange-rate providers come from a `ServiceLoader` in a static initializer (`money/ExchangeRateProviderFactory.java` L165). They are registered in `name.abuchen.portfolio/META-INF/services/name.abuchen.portfolio.money.ExchangeRateProvider`: ECB, Fixed (GBX/ILA/ZAC …), SecurityBased.
- Quote feeds, dividend feeds and search providers also use `ServiceLoader` (`online/Factory.java` L95-105). Checks use `META-INF/services/name.abuchen.portfolio.checks.Check`.
- No `plugin.xml` or extension points exist in the core bundle. The Eclipse extension registry is not used.
- Consequence: the `META-INF/services` files sit at the bundle root, **not** under `src/`. A classpath made of `target/classes` (as in an IDE/Infinitest run) finds **no providers**, so every non-trivial conversion yields an `EmptyExchangeRateTimeSeries` plus a warning (`ExchangeRateProviderFactory.java` L238-245). This is very likely why `name.abuchen.portfolio.tests/infinitest.filters` excludes `ExchangeRateProviderGBXTest`, `…ILATest`, `…AEDTest`, `…ZACTest` and `ECBExchangeRateProviderTest`. This is inferred: the GBX test calls `new ExchangeRateProviderFactory(new Client()).getTimeSeries("EUR","GBX")`, which needs the ECB and Fixed providers (`ExchangeRateProviderGBXTest.java` L23-26). **Use the packaged jar**, or add the bundle root to the classpath.

### 4. Exchange rates and network

- The UI loads and refreshes ECB rates in `StartupAddon.UpdateExchangeRatesJob`: `provider.load(monitor)` from disk, then `provider.update(monitor)` online, then `save` on shutdown (`name.abuchen.portfolio.ui/src/name/abuchen/portfolio/ui/addons/StartupAddon.java` L85-117, L145-169). The core does none of this by itself.
- Inside OSGi the cache lives in the state location: `<instance area>/.metadata/.plugins/name.abuchen.portfolio/ecb_exchange_rates.pb` (`ECBExchangeRateProvider.java` L194-212). The product sets `osgi.instance.area.default`, e.g. `@user.home/Library/Application Support/name.abuchen.portfolio.product/workspace` on macOS (`portfolio-product/name.abuchen.portfolio.product` L76-78).
- On a plain classpath there is no state location, so the options are:
  - (a) accept the 2015 defaults. This is wrong for any real valuation of foreign-currency positions.
  - (b) call `provider.update(new NullProgressMonitor())` for each provider before computing. This needs network access to ECB, every run, because there is no persistence.
  - (c) rely on exchange-rate *securities* in the client file (`SecurityBasedExchangeRateProvider`, offline).
  - (d) a small core change: make the storage `Supplier<Path>` configurable, for example with a system property. The constructor `ECBExchangeRateProvider(Supplier<Path>)` is currently package-private and test-only (L67-72).
- Quote updates are entirely network-based, and several feeds need OSGi (see the §2 table). They are not required to load, compute or save.

### 5. How the tests run

- `name.abuchen.portfolio.tests` has packaging `eclipse-test-plugin` (`name.abuchen.portfolio.tests/pom.xml` L15). It is a **fragment** of the core bundle (`META-INF/MANIFEST.MF`: `Fragment-Host: name.abuchen.portfolio`). Tycho-surefire runs it inside an Equinox runtime. There is no `useUIHarness`/`<application>` configuration (`portfolio-app/pom.xml` L486-494), so it runs headless, without a workbench or `Display`.
- These are plain JUnit 4 tests that load clients and compute figures, for example:
  - `scenarios/CurrencyTestCase.java` L56 loads `currency_sample.xml` and calls `ClientSnapshot.create(client, converter, …)` (L72) with `name.abuchen.portfolio.junit.TestCurrencyConverter`.
  - `scenarios/ClientPerformanceSnapshotTestCase.java` L32 and `snapshot/trades/TradeCollector*Test` do the same.
  - `fileversions/ReadingHistoricClientFilesTest.java` L44-62 loads XML, binary and encrypted files via `ClientFactory.load(File, char[], NullProgressMonitor)`.
  - `fileversions/events/SecurityEventSourceMovedTest.java` L56-57 saves with `saveAs(..., BINARY, COMPRESSED)` and reloads.
- The same tests are also run outside OSGi via Infinitest. `PortfolioLog.java` L30-31 handles that case explicitly, and `infinitest.filters` excludes only eight tests: the five exchange-rate tests, two `*PDFExtractorPDFTest`s and `TaxonomyLoadingTest`. The reason for excluding the last one was not determined.

### 6. Dependencies from the target platform, not Maven Central

The third-party libraries (XStream + xpp3, protobuf-java, guava, gson, commons-csv/codec, jsoup, json-simple, json-path, java-jwt, pdfbox …) are Maven-type locations in `portfolio-target-definition/portfolio-target-definition.target` (L69-375). The Eclipse bundles (core.runtime, equinox.common, osgi, swt, e4.core.di, equinox.security, httpclient5) come from p2 repositories (L5-66). The core bundle itself is an `eclipse-plugin` produced by Tycho, not published to Maven. A plain-classpath launcher therefore has to collect jars from the Tycho build: `name.abuchen.portfolio/target/*.jar` plus the resolved bundles under `~/.m2/repository/p2/osgi/bundle/...`. Another option is a separate Maven module with ordinary Maven coordinates for the third-party libraries and the Eclipse jars.

## Recommendation: minimal viable setup

**Plain JVM classpath. No OSGi launch, no `IApplication`.**

1. Classpath:
   - the built `name.abuchen.portfolio_*.jar`. It must include `META-INF/services`; do not use bare `target/classes`.
   - `org.eclipse.equinox.common`, `org.eclipse.core.runtime`, `org.eclipse.osgi` (for `NLS`, `Platform`, `FrameworkUtil`, `IProgressMonitor`/`NullProgressMonitor`), and the platform SWT jar (for `RGB`, `RGBA`, and class verification of UI helpers if touched).
   - XStream + xpp3, protobuf-java, guava (+ failureaccess), gson, commons-csv, commons-math3, bouncycastle.
   - jakarta.inject/annotation and e4.core.di are optional, because annotations are ignored if absent. Add them anyway to be safe.
   - pdfbox, httpclient5, jsoup and json-path only if you use import or online features.
   - JVM flags as in the product and tests: `-Djdk.xml.elementAttributeLimit=0` (`portfolio-app/pom.xml` L493; product `vmArgs` L11).
2. Code: `ClientFactory.load(file, password, new NullProgressMonitor())` → `var factory = new ExchangeRateProviderFactory(client)` → `var converter = new CurrencyConverterImpl(factory, client.getBaseCurrency())` → snapshot/performance APIs → mutate the model → `ClientFactory.save(client, file)`.
3. Required workarounds:
   - **stdout pollution**: redirect `System.out` around core calls, or reserve stdout for output via a dedicated stream, because `PortfolioLog` prints there outside OSGi.
   - **ECB rates**: either call `update()` on the providers (network), or accept that only in-file exchange-rate securities and 2015 defaults are available. A small follow-up change to make the ECB storage directory configurable would let a CLI reuse PP's `ecb_exchange_rates.pb` offline.
   - Do not call online quote/dividend feeds that use `FrameworkUtil.getBundle(..).getVersion()` (NPE).
4. Escalate to an **embedded headless Equinox launch** (an `IApplication` in a new bundle, started with `-application … -data "<PP workspace>" -consoleLog`, without `org.eclipse.e4.ui.workbench.swt`) only if you need PP's persisted ECB cache via `Platform.getStateLocation`, proper platform logging, OAuth secure storage, or the online feeds unchanged. The Tycho tests show that the core bundle resolves and runs headless in Equinox without a workbench.
