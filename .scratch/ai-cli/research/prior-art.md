# Prior art: programmatic access to Portfolio Performance client files

Research for [02-prior-art](../issues/02-prior-art.md). Sources were fetched on 2026-10-09. Relative ages such as "2 weeks ago" are as GitHub showed them on that day. GitHub leaves the year off dates in the current year, so "Jul 29" means 2026-07-29.

## TL;DR

- **Many readers exist, but almost none compute PP's own figures.** I found about 25 third-party tools that read the client file. Only two report TTWROR/IRR:
  - [quovibe](https://github.com/quovibe-web/quovibe) re-implements the maths in TypeScript.
  - [vgritsel12/portfolio-performance-automation](https://github.com/vgritsel12/portfolio-performance-automation) drives a pinned build of PP's Java engine over the XML. This is the closest prior art to a "headless core".

  Everything else gives balances, holdings, transactions, taxonomies or country-specific tax figures.
- **On the XML side, [pfalcon/ppxml2db](https://github.com/pfalcon/ppxml2db) is the de-facto base.**
  - It round-trips XML ⇄ SQLite, which gives read and write access, and it requires the "XML with id attributes" variant.
  - [pp-terminal](https://github.com/ma4nn/pp-terminal) builds on it. pp-terminal is the most mature tool: a CLI and MCP server on PyPI, last commit about 2 weeks ago.
  - quovibe's import also builds on it.
- **More and more tools read the binary `.portfolio` format directly.** They vendor `client.proto` and parse the ZIP, the `PPPBV1` signature and AES encryption themselves. Examples are [Nokke32/pp-mcp](https://github.com/Nokke32/pp-mcp), [tradey](https://github.com/jedimasterjonny/tradey), [exactis](https://github.com/jedimasterjonny/exactis), [portfolio-forecast](https://github.com/christianjann/portfolio-forecast) and [ha-pp-reader](https://github.com/Freakandi/ha-pp-reader). [magnum/ppapi](https://github.com/magnum/ppapi) (Ruby) even **writes** transactions into encrypted `.portfolio` files.
- **Writers are rare and work around missing validation.** The ones I found are ppxml2db, [pp-portfolio-classifier](https://github.com/Alfons1Qto12/pp-portfolio-classifier) (writes taxonomies), ppapi (writes transactions) and [pp-security-master](https://github.com/ByronWilliamsCPA/pp-security-master). Each re-learns PP invariants by trial and error. For example, ppapi must set `otherUuid` "without those PP throws `UnsupportedOperationException` on open".
- **There are already at least four read-only MCP servers.** These are `pp-terminal mcp`, Nokke32/pp-mcp, [dds-org/pp-mcp](https://github.com/dds-org/pp-mcp) and [pyfolio-performance-mcp](https://github.com/leneffets/pyfolio-performance-mcp). None of them exposes PP-computed performance metrics. dds-org lists TWROR/IRR only as a roadmap item.
- **Each tool targets one format variant.**
  - Variants are XML, XML-with-ids, binary or encrypted.
  - Units and scaling factors are "discovered by hand".
  - XStream relative XPath references are the main pain point for XML readers.
  - The only tool that records a tested format version is load-portfolio-performance-xml, at version 56. The core is now at version 70.
- **Maintainer, 2016–2025:** Buchen was "not averse" to a CLI in 2016, but did not build one. He kept XML so that users "can fully extract the data later on". He prefers protobuf because it "can be used in other languages as well".
- **Maintainer, 2026:**
  - In March he drafted a headless CLI (draft PR [#5550](https://github.com/portfolio-performance/portfolio/pull/5550)) and then called it "of limited use".
  - He does not want a CLI that manipulates the file "bypassing the input validation".
  - In July he switched to an **in-app loopback REST API** ([#5870](https://github.com/portfolio-performance/portfolio/pull/5870)), which includes IRR/TTWROR. He also published agent [skills](https://github.com/portfolio-performance/skills).
  - He plans to "merge the REST API as a first increment and release as an experimental feature". He called a read-only CLI over the same API "a reasonable starting point".
- **Where that leaves a gap:**
  - A *headless, read-only* entry point (library and/or CLI) that runs PP's own calculation engine on a *closed* file. This is the open niche.
  - It should keep the vocabulary and response shape of the #5870 API so it does not fork the contract.
  - It should leave writes alone.

## How the client file can look (from this repo's source)

These variants determine what a third-party reader must handle:

- **XML (XStream), with relative XPath references by default.** The "XML with id attributes" variant uses `XStream.ID_REFERENCES`. See [ClientFactory.java](../../../name.abuchen.portfolio/src/name/abuchen/portfolio/model/ClientFactory.java#L134) and [ClientFileType.java](../../../name.abuchen.portfolio/src/name/abuchen/portfolio/model/ClientFileType.java#L10).
- **Binary.** A ZIP (`PK\x03\x04`, see [ClientFactory.java](../../../name.abuchen.portfolio/src/name/abuchen/portfolio/model/ClientFactory.java#L214-L217)) that contains a protobuf payload prefixed with the signature `PPPBV1` ([ProtobufWriter.java](../../../name.abuchen.portfolio/src/name/abuchen/portfolio/model/ProtobufWriter.java#L99)).
- **Encrypted.** The signature `PORTFOLIO` is followed by a method byte (AES128 or AES256) ([ClientFactory.java](../../../name.abuchen.portfolio/src/name/abuchen/portfolio/model/ClientFactory.java#L271-L293)).
- **Model version.** The current version is `Client.CURRENT_VERSION = 70` ([Client.java](../../../name.abuchen.portfolio/src/name/abuchen/portfolio/model/Client.java#L34)).

## Tools

Abbreviations: **R** = read, **W** = write, **XML** = plain XStream XML, **XML-id** = "XML with id attributes", **bin** = protobuf `.portfolio` (ZIP), **enc** = encrypted `.portfolio`.

### Libraries, converters and CLIs

| Tool | Lang / dist | R/W & coverage | PP figures (IRR/TTWROR)? | Format handling | Last activity / status |
|---|---|---|---|---|---|
| [pfalcon/ppxml2db](https://github.com/pfalcon/ppxml2db) | Python scripts, MIT | **R+W**: XML → SQLite (one table per PP object) and SQLite → XML, aiming at a near-perfect round trip. Investment plans, exchange rates and CPI are not supported. | No (it is a DB, "anything you want") | Requires XML-id (PP ≥ 0.70.3). Known round-trip diff on empty `<events/>`. | Last commit 5 months ago. The author calls it "experimental project and work-in-progress". 26★. |
| [ma4nn/pp-terminal](https://github.com/ma4nn/pp-terminal) ([PyPI](https://pypi.org/project/pp-terminal/)) | Python CLI + MCP, GPL-3.0 | **R**: view accounts, cash-flows, securities, taxonomies; validate. Exports an anonymized copy of the XML. Plugin entry points for extra commands. | No TTWROR/IRR. Computes German taxes (Vorabpauschale, FIFO sell simulations, interest checks). | XML with or without ids, PP ≥ 0.70.3. Reads via a vendored, patched ppxml2db submodule. "Beta… there might be Portfolio Performance files that are not compatible". | v0.12.1, released 2 months ago. Last commit about 2 weeks ago. 25★, 11 forks. Most active. |
| [bendun-io/pyfolio-performance](https://github.com/bendun-io/pyfolio-performance) | Python, PyPI `pyfolio-performance`, MIT | R: accounts, depots, securities, transactions | No | XML | Last commit 2 years ago |
| [leneffets/pyfolio-performance-mcp](https://github.com/leneffets/pyfolio-performance-mcp) | Python lib + MCP (fork of the above) | R: summary, P/L, holdings, transactions, price history, "performance by year" (totals by transaction type) | No (P/L only) | XML. Regression fixtures copied from the upstream PP project. | Last commit 4 months ago. 6★. |
| [from68/pp-cli](https://github.com/from68/pp-cli) | Go CLI, MIT, release binaries | R: info, securities, accounts, portfolios, transactions, validate. Output as table, JSON, CSV or TSV. | No | XML | v0.1.1, 7 months ago. 0★. |
| [d-a-n/portfolio-performance-xml-parser](https://github.com/d-a-n/portfolio-performance-xml-parser) | JS, npm `@d-a-n/portfolio-performance-xml-parser` | R: securities and transactions → JSON | No | XML | Last commit 6 years ago. Unmaintained. |
| [load-portfolio-performance-xml](https://www.npmjs.com/package/load-portfolio-performance-xml) | TS, npm, AGPL-3.0 (repo on GitLab) | R: generic node tree, no PP types. Resolves XStream references itself. | No | XML. "Developed and tested with version `56` of the application's file format". | v1.0.3, 4 years ago |
| [dvett01/Portfolio-Performance-Export](https://github.com/dvett01/Portfolio-Performance-Export) | Python (one file) | R | No | XML | 4 years ago |
| [bschramke/ppxml](https://github.com/bschramke/ppxml) | Kotlin | R (partial models) | No | XML | 4 years ago. Incomplete. |
| dop89/portfolio-performance-xml-importer | Kotlin | R ("importer") | n/a | XML | Repo metadata only (GitHub repo search API): last updated 2020-06-10. Not inspected. |
| [inazr/PortfolioPerformanceTaxes](https://github.com/inazr/PortfolioPerformanceTaxes) (SteuerPP) | Python GUI + `--cli-mode`, PyInstaller | R | No. Computes German taxes (Vorabpauschale, Freibetrag, FIFO sell planning). | XML or `.portfolio` ZIP | 8 months ago |
| [vgritsel12/portfolio-performance-automation](https://github.com/vgritsel12/portfolio-performance-automation) | Python + Java adapters (`upstream-overlay/name.abuchen.portfolio/...`) | R: XML → "pinned Java engine" → `report.json` → HTML | **Yes**: TTWROR, annualized TTWROR, IRR and Profit "from the pinned upstream engine" (commit `2f39175…`). Downloads Java 21, Maven and upstream sources on the first run. | XML | 3 weeks ago (an internship write-up) |

### MCP servers and agent-facing tools

| Tool | Lang | R/W & coverage | PP figures? | Format handling | Last activity / status |
|---|---|---|---|---|---|
| [Nokke32/pp-mcp](https://github.com/Nokke32/pp-mcp) | Python, MIT, Docker | **R only**: about 20 tools over one or more files (holdings, allocation by taxonomy, realized/unrealized gains, transactions, price history). `refresh_prices` keeps new quotes in memory only ([forum](https://forum.portfolio-performance.info/t/39153.json)). | Gains only. No TTWROR/IRR. | bin + enc: its [vendored parser](https://raw.githubusercontent.com/Nokke32/pp-mcp/main/src/pp_parser/pp_parser.py) handles `PK`/`PORTFOLIO`/`PPPBV1`, AES with PBKDF2 and a hard-coded salt, and `client_pb2`. It explicitly rejects XML. | Last commit 2 months ago. 7★. |
| [dds-org/pp-mcp](https://github.com/dds-org/pp-mcp) | Python, Apache-2.0 | **R only**: summary, cash balance as of a date, year summary, holdings, transactions, dividends, allocation, price history | No. Lists "TWROR / IRR computation" under roadmap ideas. No FX conversion. | Unencrypted XML. Resolves XStream references with `lxml.xpath()`. Scaling factors "not documented anywhere — discovered by hand". | v0.4.1, 5 months ago |
| `pp-terminal mcp` | see above | R, anonymized | No | XML | see above |
| [leneffets/pyfolio-performance-mcp](https://github.com/leneffets/pyfolio-performance-mcp) | see above | R | No | XML | see above |
| [AndreevDm MCP PR #5823](https://github.com/portfolio-performance/portfolio/pull/5823) | Java, inside PP | R+W of transactions on files open in the app | n/a | in-app | Closed by Buchen on Jul 13 in favour of #5870 |

### Apps and integrations that read (or write) the file

| Tool | Lang | R/W & coverage | PP figures? | Format handling | Last activity / status |
|---|---|---|---|---|---|
| [quovibe-web/quovibe](https://github.com/quovibe-web/quovibe) | TS web app, AGPL-3.0 | One-way **import** into its own SQLite. After that it is a separate tracker. | **Yes, re-implemented**: TTWROR, IRR, Sharpe, drawdown, FIFO/moving-average cost | XML (it calls this the "ppxml2db-compatible format") or a DB produced by ppxml2db | v1.7.1, 2 months ago. 18★. |
| [magnum/ppapi](https://github.com/magnum/ppapi) | Ruby (Sinatra) | R: JSON/CSV balances per account. **W**: `bin/import fineco` appends transactions and `bin/sync` syncs both ways with Google Sheets. "Close Portfolio Performance before importing." | Balances and market values only | enc `.portfolio` (protobuf). Uses protobuf enum names for types and sets `otherUuid`/`otherUpdatedAt` so that PP can open the file. | 3 days ago. 0★. |
| [Freakandi/ha-pp-reader](https://github.com/Freakandi/ha-pp-reader) | Python, Home Assistant integration | R: `.portfolio` → SQLite. Adds live Yahoo prices and FX. Computes gains and day-change ([legacy README](https://github.com/Freakandi/ha-pp-reader/tree/dev/_legacy_v1)). | Own gains, not TTWROR/IRR | `.portfolio` with vendored `client.proto` | v0.15.6, 10 months ago. Now being rewritten ("Project Phoenix"). |
| [christianjann/portfolio-forecast](https://github.com/christianjann/portfolio-forecast) | Rust (prost) + GPUI, desktop and Android | R: NAV history chart and forecast | Own NAV analysis | `.portfolio` "ZIP or binary format" | 5 months ago |
| [jedimasterjonny/tradey](https://github.com/jedimasterjonny/tradey) | Python CLI | R: holdings and a taxonomy → optimizer for where to put new cash | No | bin. Vendored `client.proto` with "hand-simplified" stubs. | 4 days ago (Renovate). Features 3 months ago. |
| [jedimasterjonny/exactis](https://github.com/jedimasterjonny/exactis) | TS / Next.js | R: Asset Allocation taxonomy only, parsed in the browser | No | bin. A hand-written protobuf wire reader that uses the field numbers from `client.proto`. | Active (hours ago) |
| [benjamingolder/portfolio-dashboard](https://github.com/benjamingolder/portfolio-dashboard) | Python + JS | R: "Multi-client Portfolio Performance dashboard with SharePoint sync" | Unknown (no README) | Has a `proto/` dir with `client.proto` | 7 months ago |
| [Alfons1Qto12/pp-portfolio-classifier](https://github.com/Alfons1Qto12/pp-portfolio-classifier) | Python | **W**: creates or updates taxonomies from Morningstar data. "The script will overwrite some of your data… risk of data loss." | No | **Only** unencrypted plain XML. "Doesn't work with… 'XML with "id" attributes'". | 2 months ago. 60★ (fork of fizban99). |
| [rylorin/pp-portfolio-classifier](https://www.npmjs.com/package/pp-portfolio-classifier) | TS, npm | W: same as above, writes a new `.classified.xml` | No | "Only supports the unencrypted XML (without IDs)" | v0.1.4, 4 months ago |
| [ByronWilliamsCPA/pp-security-master](https://github.com/ByronWilliamsCPA/pp-security-master) | Python + Postgres | R+W: "PP XML round-trip" backup and restore, plus broker imports | No | XML | 3 days ago. "Foundation phase". |

### Leads I did not open

GitHub code search matched PP XML element names or `PPPBV1` in these repos, but I did not fetch them, so I make no claims about them. They are listed only as search hits:

- julian-klode/jak-fintools (`bin/import_portfolio_performance.py` + `bin/client.proto`). Its [README](https://github.com/julian-klode/jak-fintools) does not mention it.
- Marvin1912/stock-portfolio-tracker
- jemrobinson/uk-tax-report
- traderonfire/pp_charts
- ronanren/portfolio-performance-sync
- leonarduk/allotmint
- flaviostutz/agentme
- fuwa2019/dca-tracker
- pawel491/portfolio-performance-tests (looks like a fork of PP)

### Not relevant (no client-file access)

- [lastunicorn/portfolioperformance-toolkit](https://github.com/lastunicorn/portfolioperformance-toolkit) (NuGet). Parses only PP's transaction **CSV** export.
- Most `topic:portfolio-performance` repos are CSV/PDF converters *into* PP: pytr, PP-P2P-Parser and others.
- **Maven Central:** a search for `g:name.abuchen*` returns only `name.abuchen:fix-info-plist-maven-plugin` ([query](https://search.maven.org/solrsearch/select?q=g:name.abuchen*&rows=20&wt=json)). A free-text search for `portfolio-performance` returns nothing ([query](https://search.maven.org/solrsearch/select?q=portfolio-performance&rows=20&wt=json)). PP's model is not published as a library, so every Java-based reuse (for example vgritsel12) builds it from source.
- **PyPI:** the search and some project pages returned a CAPTCHA. Only the [pp-terminal page](https://pypi.org/project/pp-terminal/) loaded. pyfolio-performance's PyPI name is taken from its README.

## Maintainer stance (Andreas Buchen, `buchen` / `AndreasB`)

Chronological. Only statements by Buchen are labelled as his. Other people's views are attributed to them by name.

1. **2016-12-30, forum "CLI Interface für Portfolio Performance"** ([t/362](https://forum.portfolio-performance.info/t/362.json)).
   - PP once had a command-line option that opened an XML file and exported all bookings to CSV. He removed it during the migration to Eclipse e4.
   - He was "einer CLI Interface nicht abgeneigt" ("not averse to a CLI interface") and asked for contributions on GitHub.
   - He pointed to a user who automates macOS Automator to open the file, update quotes and close it.
2. **2021-02-14, issue #2085, "[new] REST API"** ([#2085](https://github.com/portfolio-performance/portfolio/issues/2085)).
   - He drafted an experimental embedded web server: `/api/v0/transactions`, `X-PP-TOKEN`, and it only works on files open in PP.
   - On 2021-11-16 he replied to the community's read-only REST API PR: "I like the idea, I want to pursue it, but I do not find the time" ([#2478](https://github.com/portfolio-performance/portfolio/pull/2478)).
3. **2021-08, protobuf format** ([#2363](https://github.com/portfolio-performance/portfolio/issues/2363)).
   - He chose protobuf partly because it "provides language support for many other languages which might create new use cases in the future".
   - He refused a DB backend because the refactoring is too large for his "limited time".
   - On XML: "I do not plan to get rid of the XML… users are only willing to input so much data if they can fully extract the data later on… the XML is the full picture."
   - On XPath references: "With protobuf, it is all UUIDs now."
4. **2023-06-25, XML interoperability** ([#3417](https://github.com/portfolio-performance/portfolio/issues/3417)).
   - "The format in a way just happened 11 years ago."
   - He was willing to switch to id references if both formats stay readable.
   - "Generally, I want to move to more use of the protobuf format. It is much faster… and can be used in other languages as well."
   - He also warned that "an incompatible change would also break the other tools".
   - Outcome: the "XML with id attributes" save option (pfalcon's PR #4117; noted as shipped on 2025-01-04 in the same thread).
5. **2023-06-25, DB sync** ([#2216](https://github.com/portfolio-performance/portfolio/issues/2216)). The Portfolio Report sync API is "not productively used… behind the 'experimental features' flag".
6. **2026-03-04, issue #5547 "AI/CLI access to PP data"** ([#5547](https://github.com/portfolio-performance/portfolio/issues/5547)).
   - He was thinking about "an MCP server", but "the CLI approach could also work".
   - He currently deletes `PortfolioPerformancec.exe` from Windows builds because "there is no meaningful CLI".
   - **"I wouldn't want the CLI to be a tool to manipulate the file itself (and potentially bypassing the input validation that is currently in the dialogs)."**
7. **2026-03-04 to 06, draft PR #5550 "Draft CLI test"** ([#5550](https://github.com/portfolio-performance/portfolio/pull/5550)).
   - A headless Eclipse application, run as `-nosplash -application name.abuchen.portfolio.cli.app --file … --export transactions --format json`, that reuses the existing JSON/CSV exporters.
   - His verdict: "with the 'garbage' on the output stream, the CLI is of limited use", and "As the CLI is not the primary use case for PP… I wouldn't want to spend time there now."
   - It is still a draft.
8. **2026-07-13 onwards, PR #5870 "Local REST API for scripts and agents"** ([#5870](https://github.com/portfolio-performance/portfolio/pull/5870)).
   - New plugin `name.abuchen.portfolio.rest`, served only on loopback `127.0.0.1:5712`.
   - Off by default and protected by a bearer token / pairing. Users enable access per file.
   - All calls run on the UI thread. Writes only mark the file dirty: "The API cannot save; only the user does."
   - Endpoints cover instruments, accounts, holdings, **performance (IRR, TTWROR)** and matched trades. There is an OpenAPI 3.1 spec, and later commits add an in-app MCP endpoint.
   - On 2026-07-13 he closed the MCP PR #5823 in favour of #5870, hoping "that with skills the agents can work with this or we add a MCP layer on top".
   - **On the CLI (2026-07-27):** "The CLI has the problem, that I cannot have the file open at the same time - the risk for dual edits is too high and there is no conflict resolution." He added: "one could expose the same API - the one that is exposed via REST - also via CLI." He keeps the surface small "to avoid exposing to many idiosyncrasies that have been added to PP over the time".
   - **2026-07-28:** "the CLI could be read-only - that avoids the whole problem domain of parallel edits. That could be a reasonable starting point." He still needs to check whether the CLI uses the same Eclipse workspace lock.
   - On 2026-07-29 he closed #2085 and #2478 in favour of #5870.
   - The PR was still **open** when I fetched it. No `name.abuchen.portfolio.rest` plugin exists in this repo checkout.
9. **About 2026-10-04, on community PR #6072** (it extends #5870 with transaction writes, save/open and imports) ([#6072](https://github.com/portfolio-performance/portfolio/pull/6072)).
   - "I plan to merge the REST API as a first increment and release as an experimental feature."
   - On transactions: "I am hesitant to expose it as is, but rather think about a ledger implementation" (#5874).
10. **Agent skills.** Buchen maintains [portfolio-performance/skills](https://github.com/portfolio-performance/skills) (`pp-connect`, `pp-inspect`, `pp-edit`, `pp-analyze`). They drive the #5870 API with `curl`. "Portfolio Performance running, with the REST API enabled" is a prerequisite.

### Other voices (not the maintainer)

- **Sn1kk3r5** (forum moderator / repo triager):
  - He closed headless-mode issue [#1411](https://github.com/portfolio-performance/portfolio/issues/1411) as "not planned" on Sep 1 without comment.
  - On the forum (2026-08-27): "Im ersten Schritt wird es mit hoher Wahrscheinlichkeit eine Read-Only-Version werden. Schreiben ist aufgrund von Validierung nochmal um ein Vielfaches schwieriger" ("The first step will most likely be a read-only version. Writing is many times harder because of validation.") ([t/17493](https://forum.portfolio-performance.info/t/17493.json)).
  - On 2026-04-08 he pointed users to ppxml2db (same thread).
- **georgemac-labs** (#5547 author) on #5870: "ANY solution that provides access to calculated metrics! That's the part that's missing from the CLI prototype and the various side-projects… we don't want to start reimplementing the calculation engine." He prefers the headless CLI (#5550) over REST because it does not need a running GUI.
- **Forum users** say the same thing. In 2021, one user wrote "Das rohe XML zu bearbeiten/zu parsen, kommt für mich nicht in Frage, da eine doppelte Business Logik nicht Sinn der Sache ist" ("Editing or parsing the raw XML is out of the question for me, because duplicating the business logic misses the point"). In 2024, another wrote that you "müsste … die gesamte Buchungsengine von PP in Python reimplementieren" ("would have to re-implement PP's entire booking engine in Python") ([t/17493](https://forum.portfolio-performance.info/t/17493.json)).
- **#4924** ([link](https://github.com/portfolio-performance/portfolio/issues/4924)): a request for an XML schema and namespaces for extensions. Contributor Morpheus1w3 said it "will not take in place". The issue is closed.

## Implications for this project

- **The format-parsing niche is already crowded.** XML has ppxml2db and pp-terminal; binary has several vendored-`client.proto` readers. The **unfilled** need is PP-engine-computed figures (TTWROR, IRR, statement of assets) without a running GUI. Today only an internship repo does this, by building PP from source.
- **Two design choices would make a headless CLI or library acceptable upstream:**
  - **Read-only first.** It meets Buchen's validation and dual-edit concerns and Sn1kk3r5's "read-only first".
  - **Reuse the #5870 resource model and naming.** Buchen wants "the same API… also via CLI".
- **Known obstacles to a headless Eclipse app,** from #5550:
  - log noise on stderr (SPI Fly, `jdk.incubator.vector` messages);
  - long launcher arguments;
  - the Eclipse workspace lock.
