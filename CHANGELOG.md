# 📝 Changelog

All notable changes to **NetworkPrinter** are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to semantic versioning. NetworkPrinter installs
**SMB/CIFS print-server queues and local USB printers only** — it is not an
IPP/AirPrint client. App and fleet updates are distributed to managed Macs as
versioned PKGs via MDM — there is deliberately no in-app updater.

## [3.11.0] — 2026-07-23

### 🔧 Changed

* **🆔 Unified bundle identifiers** — the privileged helper and both test bundles moved from the legacy `edu.slc.helpdesk.NetworkPrinter` namespace to `com.ayala.solutions.NetworkPrinter`, matching the app. The helper daemon's label, Mach service, and LaunchDaemon plist are now `com.ayala.solutions.NetworkPrinter.installhelper`. **Deployment note:** any MDM payload or approval keyed on the old helper *label* must be updated (Team-ID-keyed pre-approval is unaffected), and Macs with the old helper approved will re-prompt once in Login Items.
* **🔏 Notarization via keychain profile** — `build_and_notarize.sh` now authenticates `notarytool` with a stored keychain profile (`NOTARY_PROFILE`, validated up front) instead of passing the Apple ID, Team ID, and app-specific password through environment variables.

### ✨ Added

* **↔️ Resizable settings sheet** — the Settings sheet now opens at 640×700 and can be freely resized (minimum 560×480).

### 🐛 Fixed

* **💬 Zero printers no longer presented as an error** — a completed discovery that simply finds nothing used to raise an error alert (misleadingly titled *"No local (USB) printers were found"*). It is now a normal empty state with guidance: add a print server (with a one-tap path to Settings) or check the named server. Printers that *are* found now always render, fixing an ordering bug where an empty server field hid discovered USB printers behind a "No Server Configured" screen.
* **🔐 Local Network permission never requested** — macOS silently denies a spawned tool's traffic to local-network hosts (`smbutil`, `lpinfo`) when the app has never been granted Local Network access, so a print server on the local subnet could be unreachable with no prompt and the app never appeared in **System Settings → Privacy & Security → Local Network**. The app now declares `NSLocalNetworkUsageDescription` and briefly browses `_smb._tcp` in-process with `NWBrowser` at the start of discovery — the supported way to raise the consent prompt; once granted, the permission covers the spawned tools.

## [3.10.0] — 2026-07-22

This release is a large overhaul focused on smarter driver detection, real
single-sign-on and standard-user support, enforced MDM policy, faster and more
truthful discovery, and hardened subprocess/credential handling.

### ✨ Added

* **🧠 Hybrid driver detection** — when a printer isn't in the IT driver mapping, the app now resolves a driver much the way Apple's *Add Printer* does instead of falling back to generic PostScript. It parses the model out of SMB share comments/locations and fuzzy-matches it against the CUPS driver database (`lpinfo -m`); the tokenized score weights model numbers and tolerates suffixes. SNMP is queried only against a real printer IP, never against an SMB print server, and generic + the manual driver picker appear only when nothing clears a confidence threshold. **Mapping is now fallback-optional** — an explicit entry still wins, but full inventory coverage is no longer required.
* **🎫 Kerberos / single sign-on** — on a machine with a valid Kerberos ticket (`klist -s`), the app skips the password prompt, browses SMB without a password, and installs SMB queues using `auth-info-required=negotiate` with no password stored. It falls back to the normal username/password flow when no ticket exists.
* **🧑‍💼 Standard-user installs via a privileged helper** — a bundled `SMAppService` LaunchDaemon (`edu.slc.helpdesk.NetworkPrinter.installhelper`) with a strict, code-signing-pinned XPC interface lets standard (non-admin) users install and remove printers. The helper requires a properly signed & notarized build and a one-time approval (System Settings › Login Items, or pre-approved via an MDM Managed Login Items profile keyed on the app's Team ID). When it isn't enabled the app silently falls back to installing directly; admins need no setup.
* **⚙️ Working auto-install** — `AutoInstallPrinters` now actually installs printers. A new `AutoInstallPrinterNames` key controls scope: with names listed, only those printers are auto-installed (case-insensitive); with the list empty, only printers published by the configured print server(s) are auto-installed — never ad-hoc discoveries and **never USB**. `AutoInstallDefaultPrinter` is set as the system default when it matches an installed printer. Auto-install is idempotent within a session.
* **📥 Reused domain credentials** — validated domain credentials are stored in the Keychain and reused on next launch, so users aren't re-prompted every time (subject to `RequireAuthentication` and Kerberos).
* **🧪 Unit-test suite** — a real test suite now covers the parsers, credential redaction, subprocess runner, CIDR expansion, the auto-install allowlist policy, Kerberos parsing, helper input validation, and driver matching.
* **🤖 Continuous integration** — a GitHub Actions workflow builds the app and runs the unit tests on every push and pull request.

### 🔧 Changed

* **🎫 Operation-scoped Kerberos** — the authentication mode is now recomputed immediately before each discovery/install/remove (re-checking `klist` and validating the ticket's realm against the configured domain) instead of being latched once at launch. If a ticket expires mid-session and the user signs in with a password, the queue is no longer configured for the stale Kerberos credentials; distinct "ticket expired" and "realm mismatch" errors are surfaced.
* **🧑‍💼 Helper readiness** — XPC calls to the privileged helper are bounded by a deadline so a hung or interrupted helper can no longer freeze the UI, and helper status refreshes when the app becomes active. A standard user whose helper isn't approved now gets an actionable "open Login Items" message instead of a silent fallback to a direct install that would fail.
* **🖨️ Helper-owned driver transaction** — the daemon validates a requested driver model against its own live `lpinfo -m` inventory, stages PPD files itself into a root-owned temporary file, routes post-install authentication configuration through the helper (surfacing failures), and treats printer removal as tri-state (present / absent / query-failed) so a transient CUPS error is never reported as a successful removal.
* **⚡ Concurrent discovery** — independent sources (SMB and USB) now run in parallel and are de-duplicated centrally. Installation status is checked with a single bulk query instead of one process per printer. Discovery is no longer sequential.
* **✅ Truthful discovery results** — a refresh now finishes before the UI reports success, and when every discovery source fails the app surfaces a real error instead of showing an empty list.
* **🛡️ Enforced MDM policy** — policy keys that were previously stored but ignored are now enforced: `AllowUserInstall` / `AllowUserUninstall` hide the install/remove actions and are also guarded internally (a blocked action reports "not permitted"); `AllowUserServerChange` lets a non-admin edit the server fields when policy allows; and `RequireAuthentication` gates the login prompt.
* **🧵 Hardened subprocess handling** — all external command calls (e.g. `lpadmin`, `lpinfo`, `smbutil`, `lpstat`) now run through one async runner with per-command timeouts, concurrent stdout/stderr draining to avoid pipe deadlocks, and child-process termination on cancellation.
* **🚀 Optimized Release builds** — Release builds are now compiled with optimization and whole-module compilation. The build-and-notarize script takes signing identities from environment variables and signs inside-out.
* **📦 MDM-delivered updates** — app and fleet updates ship as versioned PKGs through MDM; there is intentionally no in-app updater.

### 🐛 Fixed

* **🪪 Privileged-helper signing identity** — the helper command-line-tool target now embeds an `Info.plist` section, so its signed binary carries the correct bundle identifier. Without it the signature reported `Identifier=PrinterInstallHelper`, which failed the app's code-signing requirement and caused **every** XPC connection to be rejected — silently forcing standard users onto a direct install they can't perform. Standard-user installs now work end-to-end on a signed build.
* **🖨️ AutoInstallPrinterNames** is now present in the sample configuration profile.
* **🌐 Cross-VLAN subnet scanning** now does correct CIDR math with a safety cap and warns when a range is too large. Previously a `/16` silently scanned only a `/24`-sized slice.
* **🔑 Keychain trusted-application ACL** path was corrected — the obsolete `PrinterKit.framework` path (gone on modern macOS) is no longer passed; only existing trusted-application paths such as `cupsd` are included.
* **↩️ Helper routing fallback** — when the privileged daemon is unreachable, installs and removals fall back to running directly.
* **🤫 Reduced log noise** — removed noisy per-line debug logging.

### 🔒 Security

* **📄 Root-owned PPD staging** — when the privileged daemon installs a driver, it copies the PPD into a root-owned temporary file it creates with `mkstemps`/`O_EXCL` and symlink guards, rather than trusting an app-writable path in the temporary directory. This closes a window where a local process could have swapped the file between validation and use.
* **🗑️ Truthful removal** — a failed `lpstat` query (e.g. CUPS temporarily unavailable) is no longer treated as "already removed"; removal reports present / absent / query-failed distinctly so a printer is never falsely reported as gone.
* **🕵️ Credential redaction in logs** — credentials are redacted before anything is written to the system log or file log, including URI `user:pass@` forms, `smbutil` `//DOMAIN;user:pass` specs, `-w`/`--password` values, `PASSWD`-style environment variable names, and Google API keys.
* **📓 Gated detailed file logging** — file logging is now controlled by the `EnableDetailedLogging` preference. Error and fault lines are always written; info and debug lines are written only when `EnableDetailedLogging` is enabled (default off).
* **🔐 Passwords kept out of process arguments and URIs** — passwords are never placed in command arguments (the `security` keychain command reads its input from stdin) and are never embedded in persisted CUPS device URIs.
* **🧹 Placeholder secrets only** — the sample configuration profile, plist, and guides no longer contain real Google Sheets API keys or a real sheet ID; they now use placeholders (`YOUR_GOOGLE_SHEETS_API_KEY` / `YOUR_GOOGLE_SHEETS_ID`).
* **⚠️ Rotate the previously committed Google API keys** — real Google Sheets API keys and a sheet ID were committed in earlier versions. They have since been **purged from the git history** via a history rewrite, but because they were exposed they must still be **rotated/revoked** in Google Cloud Console.
* **🗑️ Removed dead code** — the non-functional Windows discovery path and its supporting service, plus a superseded managed-preferences helper, have been removed.
