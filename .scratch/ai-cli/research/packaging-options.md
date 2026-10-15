# pp-cli packaging options (research)

Research for ticket `.scratch/ai-cli/issues/09-packaging-options.md`. Scope: a headless Java 21 CLI (`pp-cli`) that runs the core bundle jar (`name.abuchen.portfolio`) plus target-platform jars on a plain classpath, one JVM per command, called by AI agents from a shell on macOS first. This document lists trade-offs only. It does not pick an option.

Conventions:
- Repo paths are relative to `/Users/idodo/dev/portfolio` (branch `feat/cli-support`, version `0.88.1-SNAPSHOT`).
- **[measured]** means it was run on this machine (Apple M1 Pro, macOS) during this research.
- **[uncertain]** means not verified here.
- External doc URLs come from memory of the official doc sites and were not fetched in this session. The Tycho source link was checked via GitHub code search.

## Comparison table

| | (a) Fat/shaded jar + script | (b) jlink runtime + jars + script | (c) jpackage app image / dmg/pkg | (d) Launcher reusing installed PP |
|---|---|---|---|---|
| Build here | New module. The shade plugin needs the dependencies as Maven artifacts, but Tycho resolves them from p2 (see Cross-cutting §1). Shade config must handle services, signatures, multi-release jars and the pdfbox clash. | Same jar collection as (a), but no shading. Plus a `jlink` step (JDK tool, or exec/antrun from Maven). | Same jar collection as (b). Plus a `jpackage` step that runs jlink internally. | Only the CLI's own jar (or no jar) plus a shell script. No dependency collection at all. |
| User must install | A Java 21+ runtime (Homebrew `openjdk@21`, Temurin, …) | Nothing (runtime bundled) | Nothing (runtime bundled) | Portfolio Performance.app (distro build, which bundles Zulu JRE 21) |
| Artifact size (rough) | Fat jar ~30–50 MB. [measured] Core-relevant jars total ≈49 MB with a generous 137-jar classpath; a trimmed set is likely smaller. [uncertain] | ~64 MB runtime [measured] + ~30–50 MB jars ≈ 95–115 MB | Like (b), wrapped in `.app` (~100–115 MB). A dmg is compressed, size [uncertain]. | Script of a few KB (+ CLI jar <1 MB). Reuses the 230 MB install [measured]. |
| Per-command startup | ~0.47–0.52 s; `TieredStopAtLevel=1` ≈0.42 s; plus AppCDS ≈0.37 s [measured on bundled Zulu] | Same order. An image without a CDS archive measured 0.53 s; first run 2.2 s (cold disk cache) [measured]. Use `--generate-cds-archive`. | Like (b), plus native launcher overhead (small, [uncertain]) | Same as (a) [measured: these were the numbers] |
| macOS signing / quarantine | No Mach-O in the artifact. The JVM comes from a signed vendor JDK. Quarantine on a downloaded jar/script: low impact [uncertain]. | The image contains Mach-O (`java`, `libjvm.dylib`). Fine if built locally. If downloaded, it needs codesign + notarize. | `--mac-sign` and notarization supported. Unsigned images built locally are fine. Downloaded unsigned images are blocked. | Reuses Apple-notarized, Developer-ID-signed app. The launcher must live **outside** the `.app` so it doesn't break the seal. |
| Maintenance on upstream dep bumps | Shade rules may need updates (new signed or MR jars, new service files). Collection is automatic only if it goes through Tycho resolution. | Plus re-check the jdeps module list. Plus rebuild for JDK security updates. | Like (b), plus jpackage options/versioning. | No dependency maintenance. Coupled to whatever PP version is installed. Version skew is likely (see §d). |
| How agents find it | Script on PATH (Homebrew tap formula, or symlink in a PATH dir) | Same | Symlink to `.app/Contents/MacOS/<launcher>` on PATH ([uncertain] symlink resolution) | Script on PATH |

## (a) Fat/shaded jar + launcher script

**Build here.** Add a new module to `portfolio-app/pom.xml` (module list at [portfolio-app/pom.xml](../../../../portfolio/portfolio-app/pom.xml) lines 46–58). There are two ways to get the dependency jars:
- **Tycho-resolved (`eclipse-plugin` packaging, manifest-first).** Tycho resolves the target platform. Then `tycho-dependency-tools-plugin:list-dependencies` writes `target/dependencies-list.txt`, one absolute jar path per line ([ListDependenciesMojo.java](https://github.com/eclipse-tycho/tycho/blob/main/tycho-extras/tycho-dependency-tools-plugin/src/main/java/org/eclipse/tycho/extras/pde/ListDependenciesMojo.java), `outputFile` default `${project.build.directory}/dependencies-list.txt`). Those paths can feed a shade/assembly step, or a small antrun/exec step that unpacks them into one jar.
  - [uncertain] Whether `maven-shade-plugin` sees p2-resolved bundles directly as Maven dependencies under Tycho 5.0.4 ([portfolio-app/pom.xml](../../../../portfolio/portfolio-app/pom.xml) line 16). Expect to drive shading from the list file instead.
- **Plain Maven (`jar` packaging, pom-first).** Depend on the reactor's `name.abuchen.portfolio` jar, and re-declare every third-party dependency with Maven coordinates.
  - The target definition already uses Maven-type locations for most libraries: pdfbox, xstream, commons, jsoup, protobuf, guava, gson, json-path, … ([portfolio-target-definition/portfolio-target-definition.target](../../../../portfolio/portfolio-target-definition/portfolio-target-definition.target) lines 69–240).
  - Eclipse platform bundles come from p2 IU locations: `core.runtime`, `equinox.common`, `osgi`, `swt`, `equinox.security` from the Eclipse 4.40 repo (line 62), and BouncyCastle `bcpg`/`bcpkix`, `commons.math3` and `slf4j` from Orbit (lines 5–28). These would need their Maven Central equivalents (`org.eclipse.platform:*`, `org.bouncycastle:*`) at matching versions [uncertain: exact version alignment].
  - Duplicating the GAV list is the main maintenance cost.

**Shading pitfalls specific to this dependency set:**
- **`META-INF/services` must be merged.** The core jar carries five service files: `Check`, `ExchangeRateProvider`, `DividendFeed`, `QuoteFeed`, `SecuritySearchProvider` ([name.abuchen.portfolio/META-INF/services/](../../../../portfolio/name.abuchen.portfolio/META-INF/services)). Core loads them via `ServiceLoader` (`Factory.java` lines 95–105, `ExchangeRateProviderFactory.java` line 165, `Checker.java` line 17). Shade needs `ServicesResourceTransformer` (https://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html). Issue 01 also notes you must use the built bundle jar, not `target/classes`, because services are packaged there (`.scratch/ai-cli/issues/01-headless-core-feasibility.md`).
- **Signed jars break shading.** `bcprov` ships `META-INF/BCRSA204.SF/.RSA` [measured]. Eclipse jars are typically signed too. You must filter `META-INF/*.SF|*.RSA|*.DSA|*.EC`, otherwise "Invalid signature file digest" at startup. Core doesn't register BC as a JCE provider (no `BouncyCastleProvider`/`Security.addProvider` in `name.abuchen.portfolio/src` [measured grep]), so losing the signature should be harmless [uncertain].
- **Multi-release jars.** `bcprov` has `META-INF/versions/` [measured]. A shaded jar must keep `Multi-Release: true` in its manifest, or version-specific classes are silently ignored.
- **pdfbox 1.8 and 3.x share package names (`org.apache.pdfbox.*`).** OSGi isolates them through the wrapper bundles `name.abuchen.portfolio.pdfbox1` (`Require-Bundle: org.apache.pdfbox;bundle-version="[1.8.17,2.0.0)"`) and `name.abuchen.portfolio.pdfbox3` (`[3.0.3,4.0.0)`). Core imports both adapters ([PDFInputFile.java](../../../../portfolio/name.abuchen.portfolio/src/name/abuchen/portfolio/datatransfer/pdf/PDFInputFile.java) lines 17–19, 155, 203).
  - On a flat classpath only one version wins.
  - Issue 01 says PDFBox isn't needed for load/calc/save. So either exclude pdfbox entirely (PDF import unavailable), or, in (a) only, **relocate** pdfbox 1.8 together with the pdfbox1 adapter (shade `relocation`).
  - This applies to all four options; only (a) can fix it with relocation. (b)/(c)/(d) would need a separate classloader.
- **SWT natives.** `org.eclipse.swt.cocoa.macosx.aarch64_*.jar` contains `libswt-*-cocoa-*.jnilib` [measured]. It's per-architecture (a fragment of `org.eclipse.swt`), so a fat jar is arch-specific unless you bundle both. Issue 01 says SWT only needs to be loadable, not running. Classes like `RGB` (`ColorConversion.java`) are pure Java. `Image` paths (`ImageManager`, `ImageUtil*`) would load natives, which SWT extracts to `~/.swt` at runtime [uncertain for the CLI's code paths].
- **Notarization (only if distributed).** Apple notarization inspects nested code, including Mach-O inside archives. Jars containing unsigned `.jnilib` files are a known source of notarization failures [uncertain whether `.jar` contents are scanned]. Not relevant for local personal use. https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution

**User installs.** A Java 21+ runtime. Options: `brew install openjdk@21` (keg-only, needs a PATH or `JAVA_HOME` hint), the Temurin pkg, or reuse PP's bundled JRE (hybrid with (d)). The launcher script picks `$JAVA_HOME`, `/usr/libexec/java_home -v 21`, or a fixed path.

**Size.** Bundle jars measured from the installed PP 0.87.0:
- core 3.4 MB
- bcprov 8.5 MB
- SWT cocoa aarch64 2.7 MB
- pdfbox 3 2.0 MB
- all non-UI, non-nl jars (137 files) ≈49 MB [measured]

A trimmed fat jar is likely 25–45 MB [uncertain].

**Startup.** See Cross-cutting §3. AppCDS works with a single fat jar: CDS requires classpath entries to be jars, not non-empty directories [measured error: `Cannot have non-empty directory in paths`].

**Signing/quarantine.** The artifact contains no Mach-O, except SWT jnilibs inside the jar, which SWT extracts at runtime. Gatekeeper evaluates executables launched with a quarantine attribute. A shell script run through `/bin/sh` plus `java -jar` is generally not blocked [uncertain]. `xattr -d com.apple.quarantine` removes the concern for downloads.

**Maintenance.**
- If jars are collected through Tycho (`dependencies-list.txt`), upstream bumps flow in automatically on rebuild. Shade filters may still need updates when a new signed or MR jar appears.
- If GAVs are duplicated pom-first, every upstream target-definition change must be mirrored by hand. Renovate already manages this repo (`renovate.json`) but only for upstream files.

## (b) jlink custom runtime image

**Build here.** The CLI's dependencies aren't proper modules. Most are plain jars or OSGi bundles; some have `Automatic-Module-Name` (core: `name.abuchen.portfolio/META-INF/MANIFEST.MF` line 83). jlink can't link automatic modules. So in practice (b) means: **jlink only the JDK runtime, and ship the app jars next to it on a classpath.** The jar collection is the same as (a), minus shading, which removes the shading pitfalls but **not** the pdfbox clash.

jlink is a JDK tool. Call it via `exec-maven-plugin`/antrun in the CLI module, or use `maven-jlink-plugin`, which expects modular input [uncertain fit]. Docs: https://docs.oracle.com/en/java/javase/21/docs/specs/man/jlink.html

**Module set.** `jdeps --ignore-missing-deps --print-module-deps` on the core jar plus xstream, protobuf, httpclient5 and bcprov printed [measured]: `java.base, java.desktop, java.management, java.naming, java.net.http, java.security.jgss, java.sql, jdk.httpserver, jdk.net, jdk.unsupported`.
- Recommended additions: `jdk.localedata` (non-English number/currency formats), `jdk.charsets`, `jdk.crypto.ec`, `jdk.zipfs` [uncertain which are strictly required].
- `java.desktop` is pulled in by ImageIO/`java.beans` usage and is large.
- `--add-modules=ALL-SYSTEM` is what the desktop product passes ([portfolio-product/name.abuchen.portfolio.product](../../../../portfolio/portfolio-product/name.abuchen.portfolio.product) line 11). That hints that some code reflects into modules outside the static jdeps view [uncertain].

**User installs.** Nothing.

**Size.** The jlink image with the modules above plus `--strip-debug --no-header-files --no-man-pages --compress=zip-6` came to **64 MB** [measured], against 335 MB for the full Temurin 21 JDK and 151 MB for PP's bundled Zulu JRE [measured]. Add ~30–50 MB of jars.

**Startup.**
- The jlinked image has no `lib/server/classes.jsa` (only `libjsig.dylib`, `libjvm.dylib`) [measured]. The probe took 0.53 s warm vs 0.47–0.48 s on Zulu with its default CDS archive; the first run was 2.24 s [measured, likely cold disk cache].
- jlink's `--generate-cds-archive` plugin is available in JDK 21 [measured: `jlink --list-plugins`] and should close the gap.
- `--compress=zip-*` trades disk for decompression at class load [uncertain magnitude].

**Signing/quarantine.**
- The image contains Mach-O binaries (`bin/java`, `lib/**/*.dylib`).
- Built locally: no quarantine attribute, runs as-is.
- Downloaded (e.g. a release tarball): needs `codesign --options runtime` with entitlements suitable for a JVM (JIT needs `com.apple.security.cs.allow-jit`; compare PP's `portfolio-product/signing/entitlements.plist`), plus notarization.
- On Apple Silicon, every arm64 Mach-O must carry at least an ad-hoc signature. jlink-copied binaries keep the JDK vendor's signature unless modified [uncertain whether `--strip-debug` touches native signatures; it targets class files].

**Maintenance.**
- Re-run jdeps when dependencies change. A missing module shows up only at runtime as `NoClassDefFoundError`.
- Rebuild to pick up JDK security updates (the runtime is frozen).
- Upstream jar bumps: same as (a) without shading rules.

## (c) jpackage app image (and/or installer)

**Build here.**
- `jpackage --type app-image --input lib/ --main-jar pp-cli.jar --main-class … --java-options …`, optionally with `--runtime-image` from (b). Otherwise jpackage runs jlink itself.
- `--type dmg|pkg` produces installers.
- jpackage is a JDK tool, invoked like jlink from the CLI module. Docs: https://docs.oracle.com/en/java/javase/21/docs/specs/man/jpackage.html and https://docs.oracle.com/en/java/javase/21/jpackage/
- The repo already builds dmgs, but with `create-dmg`, not jpackage ([portfolio-product/pom.xml](../../../../portfolio/portfolio-product/pom.xml) lines 416–430, 465–479). There is no jpackage precedent here.

**User installs.**
- app-image: copy the `.app`, or extract a tarball, and symlink the launcher.
- pkg: installs into `/Applications`. Can also drop a symlink via postinstall [uncertain how easily jpackage pkg supports that].

**Size.** Roughly (b) plus the launcher and `Info.plist`: ~100–115 MB unpacked [uncertain].

**Startup.** Same JVM as (b). The jpackage native launcher reads `Contents/app/<name>.cfg` and starts the JVM in-process, so overhead is small [uncertain, not measured]. CDS: `--java-options -XX:SharedArchiveFile=$APPDIR/...` or `-XX:+AutoCreateSharedArchive` pointing to a writable location (the `.app` itself shouldn't be written to once signed).

**Signing/quarantine.**
- jpackage supports `--mac-sign`, `--mac-signing-key-user-name`, `--mac-entitlements` and `--mac-package-identifier`. Notarization is a separate `xcrun notarytool` step, as PP does ([portfolio-product/pom.xml](../../../../portfolio/portfolio-product/pom.xml) lines 498–575).
- Unsigned app images built locally run fine.
- A downloaded unsigned `.app` gets quarantined and Gatekeeper-blocked.
- Personal use with no Developer ID: build locally or ad-hoc sign (`codesign -s -`).

**Caveats.**
- [uncertain] On macOS, `--app-version` must be 1–3 integers with the first one > 0, so `0.88.1` may be rejected. Use a CLI-specific version.
- [uncertain] Whether the jpackage launcher resolves its app directory correctly when invoked through a symlink on PATH. Test it, or use a wrapper script that execs the absolute path.
- A GUI-style `.app` bundle is unusual for a pure CLI. Agents only need the inner executable.

**Maintenance.** Like (b), plus the jpackage flag surface. The `.app` bundle adds no value for a shell-only tool except as a signing container.

## (d) Launcher alongside an existing Portfolio Performance install

**What's on disk** (installed `/Applications/PortfolioPerformance.app`, listed read-only) [measured]:
- **Layout.** `Contents/{Eclipse,Info.plist,MacOS,Resources,_CodeSignature}`.
  - `Contents/Eclipse/{PortfolioPerformance.ini, artifacts.xml, configuration, features, p2, plugins}`.
  - Total 230 MB, of which `plugins/` is 226 MB and 1092 entries, many `*.nl_*` Babel fragments.
- **Bundled JRE.** `plugins/name.abuchen.zulu.jre.macosx.aarch64_21.0.5/jre/Contents/Home`:
  - Zulu 21.0.5 (2024-10-15), 151 MB.
  - Includes `lib/server/classes.jsa` (default CDS).
  - `release` lists 52 modules.
  - The JRE is signed by Azul (TeamIdentifier `TDTHCUPYFR`); the app by PP (`P3N6NZSUQK`); both with hardened-runtime flags.
- **How the JRE gets in.** The distro product includes feature `name.abuchen.zulu.jre.feature` ([portfolio-product/name.abuchen.portfolio.distro.product](../../../../portfolio/portfolio-product/name.abuchen.portfolio.distro.product) line 65). The plain product doesn't (lines 41–64 of [name.abuchen.portfolio.product](../../../../portfolio/portfolio-product/name.abuchen.portfolio.product)). [uncertain] The feature's p2 source isn't in `portfolio-target-definition.target` (no `zulu` match in the target or `portfolio-app/pom.xml`). It's presumably injected by the release pipeline.
  - So a local `-Plocal-dev` product build has **no JRE**: `portfolio-product/target/products/.../PortfolioPerformance.app/Contents/Eclipse/plugins` has no `zulu` entry [measured].
- **How the launcher uses it.** `PortfolioPerformance.ini` points `-vm` at `../Eclipse/plugins/name.abuchen.zulu.jre.macosx.aarch64_21.0.5/jre/Contents/Home/lib/libjli.dylib` [measured].
- **Calling it from a shell works.** The bundled `java` binary runs fine when invoked directly; the probe below used it.
- **Plugin jars present.** All core dependencies, e.g. `name.abuchen.portfolio_0.87.0.jar`, `org.eclipse.swt.cocoa.macosx.aarch64_3.134.0…jar`, `org.apache.servicemix.bundles.xstream_1.4.21.1.jar`, `com.google.protobuf_4.35.1.jar`, `bcprov_1.84.0.jar`, `org.eclipse.core.runtime_3.34.200…jar`, `org.eclipse.osgi_3.24.200…jar`, `org.eclipse.equinox.security_1.4.700…jar`.
  - Both `org.apache.pdfbox_1.8.17.jar` and `org.apache.pdfbox_3.0.7.jar` are present (the clash from (a)).
- **Ground truth for the active jar set.** `configuration/org.eclipse.equinox.simpleconfigurator/bundles.info` lists `symbolicName,version,plugins/<file>` per active bundle (e.g. `portfolio-product/target/products/.../bundles.info` line 11 for guava). This is the robust way to build the classpath after p2 updates, which can leave old versions in `plugins/` [uncertain how often p2 GC removes them].

**Build here.** Almost nothing:
- A shell script that reads `bundles.info`, keeps the needed symbolic names, builds `-cp`, and execs the bundled `java`.
- Plus the CLI's own small jar. That jar can be a normal Tycho `eclipse-plugin` module in the reactor, so it compiles against the target platform.

**User installs.** The PP distro app (already present). Nothing else.

**Size.** KBs for the script, under 1 MB for the CLI jar.

**Startup.** [measured] Probe: load a test XML client file + `ExchangeRateProviderFactory` + `CurrencyConverterImpl` + `ClientSnapshot.create`, with 137 PP jars on the classpath and the bundled Zulu 21.0.5:

| JVM flags | Wall time |
|---|---|
| default | 0.47–0.52 s |
| `-XX:TieredStopAtLevel=1` | 0.42 s |
| dynamic AppCDS (`-XX:SharedArchiveFile`, 8.5 MB archive) | 0.46 s |
| AppCDS + `TieredStopAtLevel=1` | 0.37–0.38 s |
| AppCDS + `TieredStopAtLevel=1` + `-XX:+UseSerialGC` | 0.37–0.38 s (no extra gain) |
| `java -version` (baseline) | 0.03 s |

The CDS archive must live outside the `.app`, e.g. `~/Library/Caches/pp-cli/`.

**Signing/quarantine.**
- Reuses the Developer-ID-signed, notarized app and the Azul-signed JRE. The installed app has a `com.apple.quarantine` attribute [measured], but direct `java` invocation from the shell worked.
- **Don't place files inside `PortfolioPerformance.app`.** Adding or modifying anything under `Contents/` invalidates the bundle's code-signature seal, and Gatekeeper may then report the app as damaged [uncertain severity for an already-approved app]. PP's own signing deletes `p2/.../cache` before `codesign --deep` ([portfolio-product/signing/codesign-macos-product.sh](../../../../portfolio/portfolio-product/signing/codesign-macos-product.sh) lines 21–31). Put the launcher in `~/.local/bin` or a Homebrew prefix instead.
- p2 self-updates rewrite `plugins/`. A launcher outside the bundle survives that.

**Maintenance and coupling.**
- **Version skew is observed today.** The installed app is **0.87.0** while the fork is **0.88.1-SNAPSHOT** ([name.abuchen.portfolio/META-INF/MANIFEST.MF](../../../../portfolio/name.abuchen.portfolio/META-INF/MANIFEST.MF) line 5) [measured]. Two sub-variants:
  - **(d1) Use the installed core jar.** The CLI computes exactly what the desktop app computes, and file format/version stay aligned with the app the user opens. But the CLI jar compiled against fork core must stay binary-compatible with whatever core PP auto-updates to, and the fork's own core changes are unavailable.
  - **(d2) Ship the fork's core jar and reuse only the JRE and third-party jars.** The fork's core can then write a newer file version than the installed app reads [uncertain; see issue 08, format version support]. Third-party jar versions can also drift: e.g. installed guava 33.6.0 vs target 33.7.2 [measured].
- No dependency-collection maintenance at all.
- Breaks if PP moves the JRE or plugins, renames bundles, or the user runs the non-distro build without a JRE.
- The bundled Zulu 21.0.5 only gets updates when PP ships them.
- macOS-first only. Windows/Linux layouts differ (`portfolio/plugins`, no `.app`) [uncertain whether the JRE is bundled there].

## Cross-cutting

### 1. Collecting target-platform/p2 jars outside OSGi

- The reactor is manifest-first Tycho, with `pomDependencies=consider` ([portfolio-app/pom.xml](../../../../portfolio/portfolio-app/pom.xml) line 150). The target platform mixes p2 IU and Maven locations (§a), across six OS/arch environments (lines 166–197). Only `macosx/aarch64` (and maybe `x86_64`) matter here.
- Three sources of a jar set:
  1. **Tycho resolution in a CLI module**, via `tycho-dependency-tools-plugin:list-dependencies` (`dependencies-list.txt`). Tracks upstream automatically. Includes all `Require-Bundle` transitive bundles: `org.eclipse.core.runtime`, `httpclient5`, `json-path`, `gson`, `osgi.service.component`, `pdfbox1/3` wrappers (core `MANIFEST.MF` lines 75–82), plus the `Import-Package` providers (lines 33–72).
  2. **Pom-first GAV duplication.** Manual and drifts.
  3. **The materialized product** `portfolio-product/target/products/name.abuchen.portfolio.product/macosx/cocoa/aarch64/PortfolioPerformance.app/Contents/Eclipse/plugins` (already present locally with 0.88.1 jars [measured]) or an installed app (d). Needs a full product build or install.
- AGENTS.md test commands show core tests run inside Tycho/OSGi, so no existing plain-classpath launch config exists to copy.

### 2. Non-jar runtime needs of core off-OSGi

- **`META-INF/services`** (5 files): merge when shading (§a). Third-party jars bring their own ImageIO SPI service files (twelvemonkeys). OSGi wires these via `OSGI-INF/name.abuchen.portfolio.util.ImageioSpiRegistration.xml` (`Service-Component`, `Require-Capability … osgi.serviceloader`, MANIFEST lines 85–86). On a flat classpath, plain `ServiceLoader`/ImageIO scanning applies and the DS component is inert.
- **Bundle resources.** `build.properties` packages `META-INF/`, `.`, `about.html` and `OSGI-INF/` ([name.abuchen.portfolio/build.properties](../../../../portfolio/name.abuchen.portfolio/build.properties) lines 1–5). Sources include `protos/` (line 1). The jar holds 255 `services`/`OSGI-INF`/`.properties` entries [measured]: NLS `messages*.properties` and bundled data, e.g. the built-in ECB rates mentioned in issue 01. These work as classpath resources.
  - Bundle localization (`%Bundle-Name`) is irrelevant off-OSGi.
- **Native libraries.** SWT cocoa jnilibs inside the arch-specific fragment jar [measured], loaded only if `Image`/`GC`-backed paths run. Core imports SWT in `ImageManager`, `LimitPriceSettings`, `ColorConversion`, `ImageUtil*`. `equinox.security.macosx` (Keychain, native) is only reached through `TokenStorage` (OAuth login), which issue 01 lists as OSGi-only.
- **OSGi-dependent code paths** that degrade off-OSGi:
  - `FrameworkUtil.getBundle(...)` returns null off-OSGi. Used in `PortfolioLog` (lines 26, 55, 71), feed user-agent strings (`PortfolioPerformanceFeed.java` line 331, `PortfolioReportQuoteFeed.java` line 151, `DivvyDiary*`, `MyDividends24Uploader`), and the ECB cache location (`ECBExchangeRateProvider.java` lines 204–205).
  - `Platform.getOS()` in `ClientFactory.java` line 734.
  - Issue 01 rates load/calc/save as safe. This is a code concern, not a packaging one, but whichever option is chosen must ship the same `equinox.common`/`core.runtime`/`osgi` jars.
- **Stdout pollution.** The probe printed `Status INFO: name.abuchen.portfolio.model.ClientFactory code=0 Loaded … with [XML]` to stdout [measured]. A JSON-emitting CLI must redirect `System.out` or `PortfolioLog`, as issue 01 also says.
- **JVM flags the desktop app uses.** `-Djdk.xml.elementAttributeLimit=0` and `--add-modules=ALL-SYSTEM` ([name.abuchen.portfolio.product](../../../../portfolio/portfolio-product/name.abuchen.portfolio.product) line 11). `elementAttributeLimit` plausibly matters for large XML client files [uncertain]. Every launcher should carry both.
  - `-XstartOnFirstThread` (Mac) is SWT-UI-only and not needed.

### 3. Per-command JVM startup and mitigations

- The measured floor is ~0.37–0.5 s per command (Cross-cutting tables in §d). The JVM itself is ~0.03 s, so most of the time is class loading and initialization of XStream, core and the ECB data.
- `-XX:TieredStopAtLevel=1` saved ~15%. Dynamic AppCDS alone saved ~2% in this probe, and ~20% combined with C1 [measured].
- Larger real client files shift cost to XML/protobuf parsing, which CDS doesn't help.
- **CDS.**
  - Docs: https://docs.oracle.com/en/java/javase/21/vm/class-data-sharing.html
  - JDK 19+ `-XX:+AutoCreateSharedArchive -XX:SharedArchiveFile=<path>` regenerates the archive automatically when the JDK or classpath changes, which suits (d) after PP updates [uncertain: not tested here].
  - The archive is classpath-sensitive and needs jar-only classpath entries [measured]. It must live in a writable location outside signed bundles.
- **Other levers** [uncertain, not measured]:
  - A trimmed classpath: the probe used 137 jars, many unnecessary.
  - `-Xshare` defaults; `-XX:-UsePerfData`.
  - Avoid `-Xms512m` (the desktop's setting).
  - A long-running daemon is excluded by ADR 0002 ([docs/adr/0002-headless-java-cli-module-in-fork.md](../../../../portfolio/docs/adr/0002-headless-java-cli-module-in-fork.md)).

### 4. How an agent's shell finds `pp-cli`

- **PATH.**
  - macOS's default PATH doesn't include `~/.local/bin`.
  - Login shells read `~/.zprofile`; interactive ones read `~/.zshrc`; every zsh reads `~/.zshenv`.
  - Agent tools may spawn non-interactive or non-login shells:
    - VS Code's integrated terminal (used by Copilot) inherits VS Code's resolved environment. On macOS, VS Code resolves the login-shell environment at startup [uncertain detail].
    - Claude Code runs commands in a shell that snapshots the user's environment [uncertain detail].
  - The safest place for the PATH export is `~/.zshenv` (or `~/.zprofile`, plus restarting the agent).
- **Homebrew tap.**
  - A personal tap repo `homebrew-<name>` holds a formula that downloads a release artifact and `depends_on "openjdk@21"` for (a). Homebrew provides helpers to write a jar wrapper script that sets `JAVA_HOME` [uncertain exact helper names in current Homebrew; see https://docs.brew.sh/Formula-Cookbook and https://docs.brew.sh/Taps].
  - Formulae land in `/opt/homebrew/bin`, already on PATH via `brew shellenv` in `.zprofile`.
  - [uncertain] Formula installs aren't quarantined (casks are, unless `--no-quarantine`).
  - Building through Tycho inside a formula is impractical (heavy, needs network/p2). Ship prebuilt artifacts instead.
- **Symlink in `~/.local/bin` or `/opt/homebrew/bin`.** Simplest for personal use: `ln -s <repo>/…/pp-cli ~/.local/bin/pp-cli`. The script must resolve its own real path (`realpath "$0"`) to find `lib/` next to it.
- **Absolute path in agent instructions** (`AGENTS.md`/`CLAUDE.md`/an MCP wrapper). Sidesteps PATH entirely. Ticket 11 (MCP surface) may make this moot.

### 5. Signing summary for personal, macOS-first use

- Anything built locally on the same Mac carries no quarantine attribute and runs without Developer ID signing. Apple Silicon only requires the ad-hoc signatures that compilers/linkers or JDK vendors already apply [uncertain for jlink/jpackage outputs, test once].
- Signing and notarization costs (Apple Developer ID, `notarytool`, as in [portfolio-product/pom.xml](../../../../portfolio/portfolio-product/pom.xml) lines 498–575) only arise when the artifact is downloaded on another machine, or via a browser on this one.
