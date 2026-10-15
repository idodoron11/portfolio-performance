# Client file format support in core: headless load/save, versioning, passwords, upstream churn

Research for ticket `.scratch/ai-cli/issues/07-format-support-facts.md`.

Sources: source read from the `/Users/idodo/dev/portfolio` checkout (branch `feat/cli-support`, merge-base with `upstream/master` is `6737e6335`, 2026-10-06; both have `CURRENT_VERSION = 70`). Paths below are relative to that repo root. Git history from `upstream/master` (tip `67aa8629e`, 2026-09-15). Statements marked **(inferred)** are reasoned from code and were not executed.

## Summary

1. **Formats.** Core knows seven `ClientFileType`s: XML, XML with ID references, XML in ZIP, binary (protobuf in ZIP), XML+AES-256, XML+AES-128, binary+AES-256. `ClientFactory.load(File, char[], IProgressMonitor)` reads all of them, and `ClientFactory.save(Client, File)` writes back with the same flags. Encrypted files are saved again with the `SecretKey` cached at load time, so no password is needed on save. The format comes from the file extension plus magic bytes. The flags are kept on the `Client` in a transient, package-private `saveFlags` set, and `save` reuses that set. `save` takes its format from the `Client`, not the target file name, so writing to a temp file and renaming it keeps the format. The only Eclipse call on this path is `Platform.getOS()`, which is harmless outside OSGi per ticket 01. No dialogs, preferences, or UI are involved: passwords are `char[]` parameters.
2. **Version.** The ceiling is `Client.CURRENT_VERSION = 70` (`Client.java#L34`). XML and encrypted files with a higher version are rejected with `IOException("This file was created with a newer version of Portfolio Performance. (file format version {0})")`. **Unencrypted binary (`.portfolio` ZIP+protobuf) has no ceiling check.** A newer file loads silently, keeps its higher version number, and is written back with that number. Fields core doesn't know are likely dropped **(inferred)**. A CLI must guard this itself with `client.getFileVersionAfterRead() > Client.CURRENT_VERSION`. Older files are migrated in memory by `upgradeModel`'s fall-through `switch`, and saving writes `CURRENT_VERSION`, which is one-way: older apps can no longer open the file.
3. **Passwords.** Passwords go in through `load(File, char[] password, IProgressMonitor)` and `saveAs/exportAs(Client, File, char[] password, Set<SaveFlag>)`. A wrong password almost always fails a sanity check on the decrypted header and throws `IOException("Password incorrect")`. A missing password throws `IOException("Password is missing")`. Encryption can be written headlessly (JDK `javax.crypto` only).
4. **Churn.** Over 5 years (2021-10 to 2026-10) the version went up **17 times (53 → 70)**, about 3.4 per year, bunched up (4 in Aug 2024). Another 6 bumps came in Dec 2020 to Sep 2021. `client.proto` changed in 9 commits since 2021-10, some without a version bump. Format code lives entirely in core. If the fork stays additive, a merge from `upstream` picks up format changes for free. The real requirement is to keep the fork's core **not older and not newer** than the user's desktop app.

---

## 1. Formats: detection, load, save, flag preservation, OSGi/UI dependencies

### 1.1 Format matrix

The authoritative list is `ClientFileType` (`name.abuchen.portfolio/src/name/abuchen/portfolio/model/ClientFileType.java#L9-L15`). `SaveFlag` is `XML, BINARY, ID_REFERENCES, COMPRESSED, ENCRYPTED, AES128, AES256` (`.../model/SaveFlag.java#L3-L6`).

| Type | Flags | On-disk layout | Load headless | Save headless, format preserved |
|---|---|---|---|---|
| XML | `XML` | XStream XML, XPath-relative references | yes | yes |
| XML_ID | `XML, ID_REFERENCES` | XStream XML, `ID_REFERENCES` mode (`ClientFactory.java#L130-L138`) | yes; `.xml` only, detected by `<client id=` in the first 100 chars (`#L506-L519`, `#L551-L561`) | yes |
| XML_ZIP | `XML, COMPRESSED` | ZIP (`PK\x03\x04`), entry `data.xml` (`#L217`, `#L254-L258`) | yes | yes |
| BINARY | `BINARY, COMPRESSED` | ZIP, entry `data.portfolio` containing `PPPBV1` + protobuf `PClient` (`ProtobufWriter.java#L99`, `#L879-L882`) | yes | yes |
| XML_AES256 / XML_AES128 | `XML, ENCRYPTED, AES256/AES128` | `PORTFOLIO` signature, method byte, IV, then AES/CBC over: content type (1 = XML), version, ZIP entry `data` (`ClientFactory.java#L264-L279`, `#L428-L459`) | yes | yes; the desktop UI no longer offers these in Save As (see 1.5) |
| BINARY_AES256 | `BINARY, ENCRYPTED, AES256` | same, content type 2 = protobuf | yes | yes |

Other combinations core can produce but cannot read back through `load(File)`:
- `BINARY` without `COMPRESSED` (raw `PPPBV1…`). `buildPersister` would write it (`#L766-L767`, `#L776-L779`). On load, content sniffing finds no `PORTFOLIO`, `PK`, or `<` prefix, falls back to `XML` (`#L566-L567`), and the XML parse fails **(inferred)**. No `ClientFileType` produces this.
- `BINARY + AES128` is not in `ClientFileType`. Core would handle it: the key length comes from the flags (`#L300-L305`).

### 1.2 Detection on load

`ClientFactory.load(File file, char[] password, IProgressMonitor monitor)` (`#L599-L634`):
1. `getFlags(file)` (`#L494-L525`):
   - `*.zip` → `{XML, COMPRESSED}`, with no content check (`#L498-L501`).
   - `*.portfolio` → `getFlagsByContent` (`#L502-L505`).
   - `*.xml` → `{XML}`, plus `ID_REFERENCES` if the first 100 chars contain `<client id=` (`#L506-L520`).
   - Any other extension → `getFlagsByContent` (`#L524`).
2. `getFlagsByContent` (`#L527-L568`) reads 100 bytes. `PORTFOLIO` → `{ENCRYPTED}`, `PK\x03\x04` → `{COMPRESSED}`, `<` → `{XML}` (+ `ID_REFERENCES`). Empty or unknown content → `{XML}`. Unit tests cover these cases (`name.abuchen.portfolio.tests/src/name/abuchen/portfolio/model/ClientFactoryTest.java#L21-L60` and on).
3. `buildPersister(flags, password)` (`#L762-L780`) chains the readers. If the flags give no body format (the `.portfolio` case), the body is picked from content:
   - ZIP: the entry name ending in `.portfolio` → `ProtobufWriter`, otherwise `PlainWriter` (`#L232-L240`).
   - Encrypted: the decrypted content-type int, 2 → protobuf, else XML (`#L362-L368`).
4. Each reader adds the flags it actually found to `client.getSaveFlags()`:
   - `PlainWriter`: `XML` (+ `ID_REFERENCES`) (`#L196-L200`).
   - `ProtobufWriter`: `BINARY` (`ProtobufWriter.java#L140`).
   - ZIP: `COMPRESSED` (`ClientFactory.java#L243`).
   - Decryptor: `ENCRYPTED` + `AES256`/`AES128` from the method byte (`#L376-L377`).

   So after loading, the `Client` holds a full flag set such as `{BINARY, ENCRYPTED, AES256}`.

Edge case **(inferred)**: a `*.zip` whose XML uses ID references is read with `PlainWriter(false)` (XPath mode), because the `.zip` branch never sniffs for `<client id=`. The desktop never produces this combination: `XML_ZIP` has no `ID_REFERENCES` flag (`ClientFileType.java#L11`).

### 1.3 How flags are kept on save

- `Client.saveFlags` is `transient` and **package-private** (`Client.java#L85`, `#L559-L564`). It starts empty for a new `Client`.
- `ClientFactory.save(Client, File)` copies `client.getSaveFlags()`, uses `{XML}` if it is empty, and requires a cached secret when `ENCRYPTED` is set (`ClientFactory.java#L677-L688`). It then calls `writeFile(..., password=null, flags, updateFlags=true)`.
- `saveAs(Client, File, char[] password, Set<SaveFlag>)` (`#L690-L700`) sets a new format and replaces `client.saveFlags` afterwards (`#L754-L758`). `exportAs` (`#L702-L712`) writes a copy and leaves the flags unchanged.
- Encrypted round trip: on load the derived `SecretKey` is cached with `client.setSecret(secret)` (`#L399`). On `save` with `password == null`, the cached key is reused (`#L421-L426`). The method byte comes from the key length (`#L432`), so AES-128 stays AES-128 and AES-256 stays AES-256. A new random IV is written on every save (`#L435-L440`).
- `save` does not use the target file name at all. Writing to `foo.portfolio.tmp` and then calling `Files.move(..., ATOMIC_MOVE)` keeps the format. **Note:** `writeFile` opens the target with a plain `FileOutputStream` and truncates in place (`#L720-L722`). Core has no temp-and-rename of its own, so the CLI must do it, as planned.
- Desktop parity: plain *Save* in the app calls exactly `ClientFactory.save(client, clientFile)` (`name.abuchen.portfolio.ui/src/name/abuchen/portfolio/ui/editor/ClientInput.java#L261`). *Save As* collects a password from a `PasswordDialog` and calls `saveAs` (`ClientInput.java#L299-L314`).
- API visibility caveat: a CLI outside package `name.abuchen.portfolio.model` cannot read `getSaveFlags()`, `getSecret()`, or `getVersion()`. The public tools are `ClientFactory.isEncrypted(File)` (`ClientFactory.java#L482-L492`) and `Client.getFileVersionAfterRead()` (`Client.java#L135-L138`). `getFlags(File)` is package-private (`ClientFactory.java#L494`). If the CLI must *report* the format, it has to re-sniff it, or place a helper class in the same package on the flat classpath. A split package works without OSGi/JPMS **(inferred; not tested)**.

### 1.4 OSGi, preferences, and UI dependencies on the load/save path

- `ClientFactory` imports only `IProgressMonitor` and `Platform` from Eclipse (`ClientFactory.java#L66-L67`).
  - `load` needs an `IProgressMonitor`; `NullProgressMonitor` from equinox.common works (as the tests do: `name.abuchen.portfolio.tests/src/fileversions/ReadingHistoricClientFilesTest.java#L44-L53`).
  - `writeFile` calls `Platform.getOS()` to skip `FileLock` on macOS (`ClientFactory.java#L731-L735`). Ticket 01's bytecode analysis found that outside OSGi this returns `"unknown"` instead of throwing. The only effect is that `tryLock()` is also tried on macOS, and its `IOException` is caught (`#L737-L744`). Source: branch `research/headless-core-feasibility`, file `.scratch/ai-cli/research/headless-core-feasibility.md` (table row for `ClientFactory`). Not re-verified here.
- `PortfolioLog.info/warning` (`ClientFactory.java#L620`, `#L717`, `#L742`) goes through `Platform.getLog(FrameworkUtil.getBundle(...))`. Outside OSGi it falls back to stdout/stderr (`name.abuchen.portfolio/src/name/abuchen/portfolio/PortfolioLog.java#L22-L35`, and ticket 01). This is a stdout-pollution issue, not a blocker.
- `Messages` uses `org.eclipse.osgi.util.NLS` (`name.abuchen.portfolio/src/name/abuchen/portfolio/Messages.java#L3`, `#L379`). It needs the `org.eclipse.osgi` jar on the classpath but no running framework. Exception messages are localized to the JVM default locale (for example `messages_de.properties#L625`). The CLI should compare against `Messages.MsgIncorrectPassword` etc., or force a locale, and not match English text.
- XStream gets the classloader explicitly (`ClientFactory.java#L1879`) and only allows `name.abuchen.portfolio.model.**` (`#L1877`). This works on a flat classpath.
- Crypto uses only JDK `PBKDF2WithHmacSHA1` + `AES/CBC/PKCS5Padding` (`#L287-L294`). There are no Eclipse secure-storage calls and no preference reads. The only migration step that needs preferences (client filters, v56→57) is explicitly left to the UI layer (`#L930-L933`; `Client.shouldDoFilterMigration()` at `Client.java#L146-L149`).
- UI-only pieces that a CLI replaces: the password prompt (`name.abuchen.portfolio.ui/src/name/abuchen/portfolio/ui/editor/ClientInputFactory.java#L49`, `PortfolioPart.java#L130`), backup-before-save driven by preferences (`ClientInput.java#L258-L259`), and the format-picker dialog.

### 1.5 Formats the desktop still offers

The *Save As* menu offers `binary+aes256`, `binary`, `xml`, `xml+id`, and `xml+zip` (`name.abuchen.portfolio.bootstrap/Application.e4xmi#L35`, `#L38`, `#L41`, `#L44`, `#L47`). The handler still accepts `xml+aes256`/`xml+aes128` (`name.abuchen.portfolio.ui/src/name/abuchen/portfolio/ui/handlers/SaveAsFileHandler.java#L55-L60`), but no menu item passes them. The historic-files test says *"encrypted files where the content is stored as XML are not supported anymore"* and has no XML+pwd fixture for v69 (`name.abuchen.portfolio.tests/src/fileversions/ReadingHistoricClientFilesTest.java#L58-L66`, added in `78c004945`). In core the XML+AES read/write path is unchanged, so "not supported" means "no longer offered or tested" **(inferred)**. Existing XML+AES files still load and save in their own format through `save`.

---

## 2. File version: detection, ceiling, newer files, migration

- **Ceiling constant:** `public static final int CURRENT_VERSION = 70;` in `name.abuchen.portfolio/src/name/abuchen/portfolio/model/Client.java#L34`. `Client.version` defaults to it (`#L45`). A transient `fileVersionAfterRead` records the on-disk version (`#L50`, getter `#L135-L138`).
- **Where the version is read:**
  - XML: the `<version>` element is deserialized into `Client.version` by XStream. It is checked **after** the full unmarshal (`ClientFactory.java#L144-L148`).
  - Protobuf: `PClient.version` (`ProtobufWriter.java#L115`).
  - Encrypted: a 4-byte int in the decrypted header, before the body (`ClientFactory.java#L343-L360`).
- **Behaviour on a newer-than-supported file**, by format:

| Format | Check | Result |
|---|---|---|
| XML (plain / ZIP) | `client.getVersion() > CURRENT_VERSION` (`ClientFactory.java#L146-L148`) | `java.io.IOException`, message `MsgUnsupportedVersionClientFiled` = "This file was created with a newer version of Portfolio Performance. (file format version {0})" (`messages.properties#L665`). **(inferred)**: XStream has no `ignoreUnknownElements` (`ClientFactory.java#L1873-L1955`), so a newer XML with unknown elements more likely fails first with `IOException("Invalid XML Format: …")` (`#L154-L157`, `messages.properties#L667`). Either way it is refused. |
| Encrypted (XML or binary) | header version `> CURRENT_VERSION` (`ClientFactory.java#L359-L360`) | same `IOException`/message, thrown before the body is parsed. Versions above `CURRENT_VERSION + 20` are reported as "Password incorrect" instead (`#L357-L358`). |
| **Binary, unencrypted** | **none.** `ProtobufWriter.load` has no ceiling check (`ProtobufWriter.java#L102-L145`; no `CURRENT_VERSION` reference in the file) | loads. `upgradeModel` hits `default: break` (`ClientFactory.java#L971-L972`), so `version` stays at e.g. 71 and is written back as 71 (`ProtobufWriter.java#L863`). Data in protobuf fields this core doesn't know is not mapped into the model, so it is **lost on save** **(inferred)**. Only `Any` extensions are carried through (`Client.java#L71-L75`; `loadExtensions` at `ProtobufWriter.java#L138`). |

  **CLI consequence:** after every load, refuse to write if `client.getFileVersionAfterRead() > Client.CURRENT_VERSION`. `fileVersionAfterRead` is set from the on-disk version before migration (`ClientFactory.java#L786`) and its getter is public.
- **Migration of older files:** `upgradeModel(Client)` (`ClientFactory.java#L782-L975`) runs for every format, called from XML (`#L150`) and protobuf (`ProtobufWriter.java#L142`). It calls `doPostLoadInitialization`, records `fileVersionAfterRead`, then runs a `switch(version)` whose cases fall through from the file's version up to `case 69`. That last case sets `client.setVersion(CURRENT_VERSION)` (`ClientFactory.java#L964-L967`). Many cases are no-ops for additive fields, others rewrite data (for example share precision `#L910-L914`, transaction UUIDs `#L917-L918`). Migration happens in memory only. The next save writes `CURRENT_VERSION` in every format (`ProtobufWriter.java#L863`, `ClientFactory.java#L451`, XStream `version` field), so an app older than the fork's core can no longer open that file.

---

## 3. Passwords

- **Supply (load):** `public static Client load(File file, char[] password, IProgressMonitor monitor) throws IOException` (`ClientFactory.java#L599`). Null password on an encrypted file → `IOException(Messages.MsgPasswordMissing)` = "Password is missing" (`#L603-L604`, `messages.properties#L655`). For unencrypted files the password is ignored: `buildPersister` only gives it to `Decryptor` (`ClientFactory.java#L769-L770`). `ClientFactory.isEncrypted(File)` (`#L482-L492`) tells the caller whether to ask for one.
- **Key derivation:** PBKDF2WithHmacSHA1, fixed salt, 65 536 iterations, 128- or 256-bit key (`#L284-L294`, `#L470-L476`). The key length is checked against `Cipher.getMaxAllowedKeyLength` → `IOException("Key length not supported. Install JCE Unlimited Strength Jurisdiction Policy.")` (`#L325-L326`, `#L587-L597`). This never triggers on a modern JDK 21 **(inferred)**.
- **Wrong password:** decryption with the wrong key yields garbage. The first two decrypted ints must satisfy `contentType ∈ {1,2}` and `1 ≤ version ≤ CURRENT_VERSION + 20`, otherwise `IOException(Messages.MsgIncorrectPassword)` = "Password incorrect" (`ClientFactory.java#L357-L358`, `messages.properties#L625`). There is no special exception type: callers must compare the message. **(inferred)**: random garbage passes this check with probability ≈ 2·90 / 2^64. In that case the failure would surface later as a ZIP/parse `IOException`, or as `IOException("Error while decrypting output: …")` wrapping a `GeneralSecurityException` (`#L403-L406`).
- **Writing encryption headlessly:** yes.
  - New file or re-key: `ClientFactory.saveAs(client, file, password, EnumSet.of(BINARY, ENCRYPTED, AES256))` (`#L690-L700`). This derives a fresh key (`#L421`), caches it (`#L426`), and replaces the flags.
  - In-place rewrite: `ClientFactory.save(client, file)` reuses the cached key, no password needed (`#L683-L684`, `#L421`).
  - `exportAs` with a password also works in core, though the UI forbids encrypted export (`ClientInput.java#L346-L348`).

  All of this is JDK crypto only.

---

## 4. Upstream churn of format and version

### 4.1 `CURRENT_VERSION` bumps on `upstream/master`

Command: `git log upstream/master -G'CURRENT_VERSION = [0-9]+' -- .../model/Client.java`. A parallel-branch 52→53 clash was resolved to 54 in merge `05788e6f9`.

| → ver | Commit | Date | Reason |
|---|---|---|---|
| 48 | `b011e46e0` | 2020-12-25 | fees on dividend transactions |
| 49 | `26a26faff` | 2021-01-03 | share precision 6 → 8 decimals (data migration) |
| 50 | `ad779b13a` | 2021-01-12 | fix limit prices off by ×1000 |
| 51 | `7b765f328` | 2021-03-09 | UUIDs on transactions |
| 52 | `e249012a2` / `b6a3456b0` | 2021-08-06 / 08-22 | drop CPI data; protobuf binary format completed |
| 53 | `6790ab4ef` / `10ce6a98c` | 2021-09-18 / 10-03 | `source` on transactions; typed attribute-type properties |
| 54 | `05788e6f9` (merge) | 2021-10-30 | reconcile the two 53s (`fixSourceAttributeOfTransactions`) |
| 55 | `5d8f40906` | 2022-03-16 | key on taxonomy classifications |
| 56 | `a9298cdf8` | 2022-05-08 | fix gross-value units |
| 57 | `53e54695d` | 2022-07-27 | client filters moved into file |
| 58 | `4151340ec` | 2023-07-22 | remove watchlist-only securities |
| 59 | `ffe603479` | 2023-12-26 | taxes data series label |
| 60 | `a27997045` | 2024-04-02 | null security properties fix |
| 61 | `4f89ad3f9` | 2024-05-05 | investment plan type "Interest" (proto change) |
| 62 | `eee963607` | 2024-04-28 (author date) | Portfolio Report API properties cleanup |
| 63 | `6cc2c1093` | 2024-08-02 | chart label configuration |
| 64 | `176b7507f` | 2024-08-15 | null security events fix |
| 65 | `bb06f63fb` | 2024-08-21 | dashboard identifiers (proto change) |
| 66 | `8c233e04c` | 2024-08-24 | `source` moved to security event (proto `09ae69898`); first released in 0.70.4 |
| 67 | `b7ab2b9d0` | 2025-09-20 | logo attribute name fix; first released in 0.80.0 |
| 68 | `a2d0e0831` | 2025-11-22 | remove taxonomy sync source; first released in 0.81.0 |
| 69 | `78c004945` | 2026-02-13 | ex-date on dividends (proto change); first released in 0.82.0 |
| 70 | `ddaa3b3dd` | 2026-06-10 | ownership weights in client filters; first released in 0.85.0 |

Release tags come from `git tag --contains <commit>`, earliest match.

- **Last ~5 years (2021-10-15 → 2026-10-15): 17 bumps (53 → 70), ≈ 3.4 per year.** Last 3 years (2023-10 →): 12 bumps (58 → 70). The bumps are bursty: 4 in August 2024, then none for 13 months.
- **Schema (`name.abuchen.portfolio/src/name/abuchen/portfolio/model/client.proto`):** 9 commits since 2021-10. Not all of them bump the version:
  - `fdee80c3d` (2023-04-29): `baseCurrency` added to protobuf.
  - `e09b8039c` (2025-08-31): protobuf `Any` extensions.
  - `145cc26d2` (2021-10-15): ECB rate messages, not part of the client file.

  So the version number undercounts wire-format changes. Generated classes live in `name.abuchen.portfolio/protos/.../proto/v1/` and are committed. The magic `PPPBV1` has never changed since protobuf was introduced (`e3a1f07fc`, 2021-08-01).
- **Churn of the code:** since 2021-10, `ClientFactory.java` changed in 49 commits and `ProtobufWriter.java` in 22. The container formats (ZIP, encryption header, `PORTFOLIO`/AES layout) have not changed since AES-128 was added in `306dde937` (2014-05-17). Every change since then is in the model or migrations.

### 4.2 What this means for the fork

- Format support is entirely inside core: `Client`, `ClientFactory`, `ProtobufWriter`, `client.proto`, and the generated classes. ADR 0002 keeps the fork additive, so a routine merge from `upstream` brings new versions and migrations with no conflicts in the CLI module. **The effort to keep the code in step is close to zero.** At the time of writing the fork is 4 commits behind `upstream/master` and both are at version 70.
- The real constraint is **version alignment with the user's desktop app**, in both directions:
  - **Fork core older than the app.** XML and encrypted files are refused (safe). Unencrypted binary files load silently and may lose data on save (section 2). The CLI needs its own `getFileVersionAfterRead() > CURRENT_VERSION` guard.
  - **Fork core newer than the app.** Any CLI save upgrades the file to the fork's `CURRENT_VERSION`, and the desktop app then refuses it with "created with a newer version". The CLI should refuse to save when that would raise the version, unless the user asks for it.
- At about 3–4 bumps a year, with each bump shipping in the next desktop release (for example v69 in 0.82.0, v70 in 0.85.0), the fork should merge `upstream` at least at every desktop release the user installs. Ideally it builds from the release tag that matches the installed app, rather than `master`. `master` can be ahead of the latest release by one version.
