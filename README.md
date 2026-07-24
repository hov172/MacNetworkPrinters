# 👨‍💼 Maintain and Build

**Jesus M. Ayala - Ayala Solutions**

> For bugs or enhancements, please open an issue.

---

# 🖨️ NetworkPrinter macOS Application

NetworkPrinter is a comprehensive macOS application designed to help users discover and install network printers from both SMB/Windows print servers and IPP (Internet Printing Protocol) servers. The application features a modern SwiftUI interface with comprehensive driver selection capabilities and is built to robustly read and apply settings managed via configuration profiles.

![NetworkPrinter Main UI](docs/images/main_ui.png)

![NetworkPrinter App Settings UI](docs/images/app_settings_ui.png)

![Print Servers Settings](docs/images/print_servers_settings.png)

![Driver Mappings (Google Sheets) Settings](docs/images/driver_mappings_settings.png)

![Auto-Installation and Printer Install Helper Settings](docs/images/auto_install_helper_settings.png)

---

## ✨ Features

* **🎨 Modern SwiftUI Interface**: Clean, intuitive design with smooth animations and responsive layouts
* **🔍 Multi-Protocol Discovery**: Automatically discover printers across SMB print servers, IPP servers, local mDNS/AirPrint, IPP Everywhere, and USB — all sources run **concurrently** and are de-duplicated centrally
* **🧠 Hybrid Driver Detection**: When a printer isn't in the IT driver mapping, the app resolves a real driver the way Apple's *Add Printer* does — advertised make-and-model / driverless "everywhere" for IPP, share-comment parsing plus a fuzzy match against the CUPS driver database for SMB queues — instead of blindly falling back to generic PostScript
* **🎫 Kerberos / SSO**: On a machine with a valid Kerberos ticket the app skips the password prompt, browses SMB without a password, and installs queues with `auth-info-required=negotiate` — no password stored. The auth mode is re-evaluated before each operation with realm validation, so an expired ticket cleanly falls back to username/password (distinct "ticket expired" / "realm mismatch" errors)
* **🧑‍💻 Standard-User Installs**: An optional bundled privileged helper (SMAppService LaunchDaemon with a code-signing-pinned XPC interface) lets **non-admin** users install and remove printers; admins need no setup
* **🔐 Enforced MDM Policy**: Managed permission keys are actually enforced — install/remove actions are hidden *and* guarded in the manager, server-field edits are gated, authentication prompts are gated, and IPP SSL/port settings drive the real target
* **🤖 Working Auto-Install**: Silently install a named allowlist of printers (or all print-server-published queues) at launch, optionally setting a system default — idempotent per session
* **📡 Real-time Status Updates**: Live printer status monitoring and installation progress tracking; installation status is checked with a single bulk `lpstat -p`
* **🔎 Search and Filter**: Quickly find printers by name or location across all protocols
* **📦 Batch Operations**: Select and install multiple printers at once from any protocol
* **🔐 MDM Configuration**: Supports managed deployment via macOS configuration profiles
* **🪵 Redacted, Gated Logging**: Credentials are redacted before anything is written; the detailed file log is gated by `EnableDetailedLogging` (errors are always logged)
* **🌐 Protocol Flexibility**: Support for SMB/CIFS, IPP/IPPS, and mixed environments

---

## 🏗️ Architecture

The app is built using a modern, modular service-based architecture:

* **PrinterManager**: Orchestrates concurrent multi-source discovery, installation, and status updates; enforces MDM permission policy (blocked actions throw a "not permitted" error)
* **PreferencesManager**: Reads and manages both local and MDM-supplied preferences for all protocols
* **ProcessRunner**: The single async runner for every subprocess call (`lpadmin`, `lpinfo`, `smbutil`, `ippfind`, `ipptool`, `dig`, `security`, `snmpget`, `gunzip`, `lpstat`, …) — enforces per-command timeouts, drains stdout/stderr concurrently to avoid pipe deadlocks, and terminates the child process on cancellation
* **PrinterDiscoveryService** (SMB) and **IPPDiscoveryService** (IPP/AirPrint): Dedicated per-protocol discovery; independent sources and individual mDNS service types run in parallel and are de-duplicated centrally
* **DriverMatching** (`ModelExtractor` + `DriverMatcher`): Extracts a printer model from advertised attributes or SMB share text and fuzzy-matches it against the CUPS driver database (`lpinfo -m`)
* **KerberosService**: Detects a valid ticket (`klist`) and drives the password-free SSO/negotiate install path
* **HelperClient + PrinterInstallHelper daemon**: XPC client and the bundled SMAppService LaunchDaemon (strict, code-signing-pinned interface) that lets standard users install/remove printers; the app routes `lpadmin` through it when enabled, otherwise runs `lpadmin` directly
* **PrinterInstallationService**: Unified installation for both protocols (direct or via the helper)
* **PrinterStatusService**: Bulk installation-status checks (`lpstat -p`)
* **DriverManagementService**: Loads driver mappings (bundled, MDM, remote JSON, Google Sheets)
* **KeychainService**: Stores and reuses validated domain credentials so users aren't re-prompted every launch
* **FileLogging**: Credential-redacted, `EnableDetailedLogging`-gated OSLog + file logging
* **UI Components**: SwiftUI-based views like `PrinterListView`, `DriverSelectionView`, `PrinterHeaderView`
* **Styling System**: Unified design language with consistent colors, fonts, and components

---

## 🧠 Driver Management

When a printer **is** in the IT driver mapping, that mapping always wins. When it is **not**, the app resolves a driver the way Apple's *Add Printer* effectively does — instead of silently falling back to generic PostScript — using this order:

1. **IT driver mapping** — an exact/normalized match from the Google Sheet or MDM `PrinterDriverMappings` takes priority over everything else.
2. **Advertised make-and-model (IPP/AirPrint)** — for IPP/AirPrint printers the app reads the printer's own attributes (`ipptool get-printer-attributes`) and prefers driverless **`everywhere`** when the printer supports it.
3. **SMB share-comment parse + fuzzy match** — for SMB print-server queues the model is parsed out of the share comment/location text (e.g. *"Financial Aid HP LaserJet P3015n"*) and fuzzy-matched against the CUPS driver database (`lpinfo -m`) with a tokenized score that weights model numbers and tolerates suffixes (so `P3015n` matches the `HP LaserJet P3015` PPD).
4. **SNMP** — queried **only** against a real printer IP, never against the SMB print server.
5. **Generic + manual picker** — the generic driver and the manual driver picker are used only when nothing clears the confidence threshold.

This hybrid resolution runs entirely through **`DriverMatching`** (`ModelExtractor` + `DriverMatcher`) and the `DriverManagementService`, so IT can override any result with an explicit mapping.

### Detection by Connection Type

Driver auto-detection works on **every** connection type — all installs route through the same hybrid resolver:

| Connection | Detection path | If nothing matches |
|---|---|---|
| **IPP server / AirPrint / IPP Everywhere** | IT mapping → driverless `everywhere` when the printer advertises AirPrint/IPP Everywhere → the printer's own advertised make-and-model via `ipptool` (the mapping can also match that string) → fuzzy match against the CUPS driver database | Falls back to **driverless** — always a working endpoint |
| **SMB print server** | IT mapping → model parsed from the share comment/location (e.g. *"Financial Aid HP LaserJet P3015n"*) → SNMP (only against a real printer IP, never the print server) → PPD cache + `lpinfo -m` fuzzy match | Manual driver picker |
| **USB** | IT mapping → the device-reported make-and-model → PPD cache + `lpinfo -m` fuzzy match | Manual driver picker |

Three guarantees, regardless of connection type:

1. **The IT mapping (Google Sheets / MDM) always wins first**, so fleet-managed printers behave identically however they are discovered.
2. **Low-confidence results never install silently** — a generic or below-threshold match surfaces the manual driver picker instead of installing the wrong driver.
3. **Standard-user installs resolve identically** — the privileged helper re-validates the chosen driver against its own `lpinfo -m` inventory and accepts driverless for IPP.


---

## 🚀 Configuration Setup

### 📘 Quick Start Guide

1. Copy the file: `NetworkPrinter/Resources/NetworkPrinterSettings.mobileconfig`
2. Edit placeholders:

   * `<string>Your Organization</string>` → `<string>Acme Corporation</string>`
   * Replace `YOUR_PRINTSERVER_HOSTNAME_OR_IP` and other placeholders
   * Configure both SMB and IPP settings as needed
3. Deploy via MDM (Jamf, Intune, etc.) or manually:
   ```bash
   sudo profiles -I -F NetworkPrinterSettings.mobileconfig
   ```

### 🔧 Complete Configuration Examples

#### 🏢 Example 1: Mixed Protocol Business Setup

**Scenario**: Business with both SMB Windows print server and modern IPP server

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadDisplayName</key>
    <string>Network Printer Settings</string>
    <key>PayloadIdentifier</key>
    <string>com.acmecorp.networkprinter.configuration</string>
    <key>PayloadOrganization</key>
    <string>Acme Corporation</string>
    <key>PayloadRemovalDisallowed</key>
    <false/>
    <key>PayloadScope</key>
    <string>System</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>12345678-1234-1234-1234-123456789012</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadDescription</key>
            <string>Configures Network Printer application settings for mixed SMB/IPP environment</string>
            <key>PayloadDisplayName</key>
            <string>Network Printer App Preferences</string>
            <key>PayloadIdentifier</key>
            <string>com.networkprinter.preferences</string>
            <key>PayloadOrganization</key>
            <string>Acme Corporation</string>
            <key>PayloadType</key>
            <string>com.apple.ManagedClient.preferences</string>
            <key>PayloadUUID</key>
            <string>12345678-1234-1234-1234-123456789013</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadEnabled</key>
            <true/>
            <key>PayloadContent</key>
            <dict>
                <key>com.networkprinter.preferences</key>
                <dict>
                    <key>Forced</key>
                    <array>
                        <dict>
                            <key>mcx_preference_settings</key>
                            <dict>
                                <!-- Multi-Protocol Discovery -->
                                <key>DiscoveryMode</key>
                                <string>Both</string>
                                
                                <!-- SMB Server Configuration -->
                                <key>PrintServerHost</key>
                                <string>printserver.acmecorp.com</string>
                                <key>PrintServerPort</key>
                                <integer>445</integer>
                                <key>PrintServerDomain</key>
                                <string>ACME</string>
                                
                                <!-- IPP Server Configuration -->
                                <key>IPPServerHost</key>
                                <string>ippserver.acmecorp.com</string>
                                <key>IPPServerPort</key>
                                <integer>631</integer>
                                <key>IPPUseSSL</key>
                                <false/>
                                <key>IPPRequireAuthentication</key>
                                <true/>
                                
                                <!-- User authentication -->
                                <key>RequireAuthentication</key>
                                <true/>
                                <key>DefaultDomain</key>
                                <string>ACME</string>
                                
                                <!-- User permissions (flexible) -->
                                <key>AllowUserServerChange</key>
                                <true/>
                                <key>AllowUserInstall</key>
                                <true/>
                                <key>AllowUserUninstall</key>
                                <true/>
                                
                                <!-- Installation preferences -->
                                <key>AutoInstallPrinters</key>
                                <false/>
                                <!-- Non-empty: only these named printers auto-install.
                                     Empty: only print-server-published queues auto-install. -->
                                <key>AutoInstallPrinterNames</key>
                                <array>
                                    <string>Office-Printer-Main</string>
                                </array>
                                <key>AutoInstallDefaultPrinter</key>
                                <string></string>
                                
                                <!-- Troubleshooting -->
                                <key>EnableDetailedLogging</key>
                                <false/>
                                <key>AllowFallbackWithGenericSuggestion</key>
                                <true/>
                                
                                <!-- Driver mappings work for BOTH SMB and IPP -->
                                <key>PrinterDriverMappings</key>
                                <dict>
                                    <!-- SMB Printers -->
                                    <key>CORP-HP-LaserJet-FL1</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz</string>
                                    <key>CORP-Canon-Copier-FL2</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz</string>
                                    
                                    <!-- IPP Printers -->
                                    <key>Office-Printer-Main</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz</string>
                                    <key>Reception-Color-Printer</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz</string>
                                </dict>
                                
                                <!-- Remote management disabled for simplicity -->
                                <key>DriverMappingsURL</key>
                                <string></string>
                                <key>GoogleSheetsID</key>
                                <string></string>
                                <key>GoogleSheetsAPIKey</key>
                                <string></string>
                                <key>GoogleSheetsRange</key>
                                <string>printers!A:B</string>
                            </dict>
                        </dict>
                    </array>
                </dict>
            </dict>
        </dict>
    </array>
</dict>
</plist>
```

**Key Features**:
- Discovers printers from both SMB and IPP servers
- Users can modify server settings if needed
- Authentication configured for both protocols
- Driver mappings work for both SMB and IPP printers
- Fallback to generic drivers allowed

#### 🌐 Example 2: IPP-Only Modern Environment

**Scenario**: Modern organization using only IPP/CUPS servers

```xml
<key>mcx_preference_settings</key>
<dict>
    <!-- IPP-Only Discovery -->
    <key>DiscoveryMode</key>
    <string>IPP</string>
    
    <!-- IPP Server Configuration -->
    <key>IPPServerHost</key>
    <string>cups.moderncompany.com</string>
    <key>IPPServerPort</key>
    <integer>631</integer>
    <key>IPPUseSSL</key>
    <true/>
    <key>IPPRequireAuthentication</key>
    <true/>
    
    <!-- Authentication -->
    <key>RequireAuthentication</key>
    <true/>
    <key>DefaultDomain</key>
    <string>MODERN</string>
    
    <!-- IPP Printer Mappings -->
    <key>PrinterDriverMappings</key>
    <dict>
        <key>Office-Color-Printer</key>
        <string>/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz</string>
        <key>Engineering-Plotter</key>
        <string>/Library/Printers/PPDs/Contents/Resources/HP DesignJet.ppd.gz</string>
        <key>Reception-MFP</key>
        <string>/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz</string>
    </dict>
</dict>
```

#### 🏭 Example 3: SMB-Only Legacy Environment

**Scenario**: Traditional Windows-based print server environment

```xml
<key>mcx_preference_settings</key>
<dict>
    <!-- SMB-Only Discovery -->
    <key>DiscoveryMode</key>
    <string>SMB</string>
    
    <!-- SMB Server Configuration -->
    <key>PrintServerHost</key>
    <string>legacy-printserver.company.local</string>
    <key>PrintServerPort</key>
    <integer>445</integer>
    <key>PrintServerDomain</key>
    <string>COMPANY</string>
    
    <!-- Authentication -->
    <key>RequireAuthentication</key>
    <true/>
    <key>DefaultDomain</key>
    <string>COMPANY</string>
    
    <!-- Locked-down permissions -->
    <key>AllowUserServerChange</key>
    <false/>
    <key>AllowUserInstall</key>
    <true/>
    <key>AllowUserUninstall</key>
    <false/>
</dict>
```

### 🔧 Advanced Configuration

| Key | Type | Description | Default | Protocols |
|-----|------|-------------|---------|-----------|
| `DiscoveryMode` | String | Discovery mode: "SMB", "IPP", or "Both" | `"Both"` | Both |
| `PrintServerHost` | String | Hostname or IP address of SMB print server | `""` | SMB |
| `PrintServerPort` | Integer | SMB port for print server connection | `445` | SMB |
| `PrintServerDomain` | String | Domain name for SMB authentication | `""` | SMB |
| `IPPServerHost` | String | Hostname or IP address of IPP print server | `""` | IPP |
| `IPPServerPort` | Integer | IPP server port (631 for HTTP, 443 for HTTPS) | `631` | IPP |
| `IPPUseSSL` | Boolean | Use HTTPS/TLS for secure IPP connections | `false` | IPP |
| `IPPRequireAuthentication` | Boolean | Require authentication for IPP access | `true` | IPP |
| `RequireAuthentication` | Boolean | Whether users must authenticate to access SMB printers | `false` | SMB |
| `DefaultDomain` | String | Pre-fills domain field in authentication dialog | `""` | Both |
| `AllowUserServerChange` | Boolean | Can users modify server settings | `true` | Both |
| `AllowUserInstall` | Boolean | Can users install printers | `true` | Both |
| `AllowUserUninstall` | Boolean | Can users remove installed printers | `true` | Both |
| `AutoInstallPrinters` | Boolean | Silently install printers at launch (implemented, idempotent per session) | `false` | Both |
| `AutoInstallPrinterNames` | Array | Names to auto-install (case-insensitive). Empty → only print-server-published queues | `[]` | Both |
| `AutoInstallDefaultPrinter` | String | Set as system default if it matches an installed printer | `""` | Both |
| `EnableDetailedLogging` | Boolean | Gates the detailed file log (errors always logged); output is credential-redacted | `false` | Both |
| `AllowFallbackWithGenericSuggestion` | Boolean | Allow generic drivers when specific unavailable | `false` | Both |

### 🔧 Feature Implementation
```swift
// Multi-protocol printer discovery
class PrinterManager: ObservableObject {
    @Published var printers: [NetworkPrinter] = []
    private let smbDiscoveryService = PrinterDiscoveryService()
    private let ippDiscoveryService = IPPDiscoveryService()
    
    func discoverPrinters() {
        let mode = preferencesManager.discoveryMode
        
        switch mode {
        case .smb:
            // SMB-only discovery
            Task {
                await discoverSMBPrinters()
            }
        case .ipp:
            // IPP-only discovery
            Task {
                await discoverIPPPrinters()
            }
        case .both:
            // Parallel discovery from both protocols
            Task {
                async let smbPrinters = discoverSMBPrinters()
                async let ippPrinters = discoverIPPPrinters()
                
                let allPrinters = await smbPrinters + ippPrinters
                await MainActor.run {
                    self.printers = allPrinters
                }
            }
        }
    }
    
    private func discoverSMBPrinters() async -> [NetworkPrinter] {
        // SMB discovery implementation
    }
    
    private func discoverIPPPrinters() async -> [NetworkPrinter] {
        // IPP discovery implementation using IPPDiscoveryService
    }
}

// Unified driver mapping system
class PreferencesManager: ObservableObject {
    func getSuggestedDriverPath(for printerName: String) -> String? {
        // Works for both SMB and IPP printer names
        // Priority: MDM → Remote JSON → Google Sheets → Bundled
        
        // Try exact match first
        if let exactMatch = printerDriverMappings[printerName] {
            return exactMatch
        }
        
        // Try fuzzy matching for both protocols
        return findFuzzyMatch(for: printerName)
    }
}
```

### 🔧 Protocol-Specific Configuration

#### SMB/CIFS Configuration
```xml
<!-- Traditional Windows print server -->
<key>DiscoveryMode</key>
<string>SMB</string>
<key>PrintServerHost</key>
<string>windows-printserver.domain.com</string>
<key>PrintServerDomain</key>
<string>DOMAIN</string>
<key>RequireAuthentication</key>
<true/>
```

#### IPP/IPPS Configuration
```xml
<!-- Modern CUPS or IPP server -->
<key>DiscoveryMode</key>
<string>IPP</string>
<key>IPPServerHost</key>
<string>cups.company.com</string>
<key>IPPServerPort</key>
<integer>631</integer>
<key>IPPUseSSL</key>
<false/>
<key>IPPRequireAuthentication</key>
<true/>
```

#### Mixed Environment Configuration
```xml
<!-- Support both protocols -->
<key>DiscoveryMode</key>
<string>Both</string>
<key>PrintServerHost</key>
<string>smb-server.company.com</string>
<key>IPPServerHost</key>
<string>ipp-server.company.com</string>
<key>RequireAuthentication</key>
<true/>
<key>IPPRequireAuthentication</key>
<true/>
```

---

### 📝 Configuration Profile Setup Steps

#### Step 1: Generate UUIDs
Generate unique UUIDs for your configuration:
```bash
# Generate UUID for main payload
uuidgen
# Generate UUID for preferences payload  
uuidgen
```

#### Step 2: Choose Your Protocol Strategy
**Decision Matrix:**

| Environment | SMB Server | IPP Server | Discovery Mode | Best For |
|-------------|------------|------------|----------------|----------|
| **Traditional** | ✅ | ❌ | `SMB` | Windows-based environments |
| **Modern** | ❌ | ✅ | `IPP` | CUPS/Linux-based environments |
| **Mixed** | ✅ | ✅ | `Both` | Organizations in transition |
| **Cloud** | ❌ | ✅ | `IPP` | Cloud-native printing solutions |

#### Step 3: Configure Server Settings
Set your print server details based on chosen protocols:

**For SMB environments:**
```xml
<key>DiscoveryMode</key>
<string>SMB</string>
<key>PrintServerHost</key>
<string>your-smb-server.domain.com</string>
<key>PrintServerPort</key>
<integer>445</integer>
<key>PrintServerDomain</key>
<string>YOUR-DOMAIN</string>
```

**For IPP environments:**
```xml
<key>DiscoveryMode</key>
<string>IPP</string>
<key>IPPServerHost</key>
<string>your-ipp-server.domain.com</string>
<key>IPPServerPort</key>
<integer>631</integer>
<key>IPPUseSSL</key>
<false/>
<key>IPPRequireAuthentication</key>
<true/>
```

**For mixed environments:**
```xml
<key>DiscoveryMode</key>
<string>Both</string>
<!-- Configure both SMB and IPP settings -->
```

#### Step 4: Configure Driver Mappings
The same driver mapping system works for both protocols:

```xml
<key>PrinterDriverMappings</key>
<dict>
    <!-- SMB printer names -->
    <key>CORP-HP-4015-FL1</key>
    <string>/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz</string>
    
    <!-- IPP printer names -->
    <key>Office-Printer-Main</key>
    <string>/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz</string>
    
    <!-- Generic mappings work for both -->
    <key>Reception-Printer</key>
    <string>/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz</string>
</dict>
```

#### Step 5: Deploy Profile
Deploy using your MDM or manually:
```bash
# Manual deployment
sudo profiles -I -F YourNetworkPrinterSettings.mobileconfig

# Verify deployment
profiles -P | grep networkprinter

# Check settings (both protocols)
defaults read com.networkprinter.preferences DiscoveryMode
defaults read com.networkprinter.preferences PrintServerHost
defaults read com.networkprinter.preferences IPPServerHost
```

---

### 🎯 Configuration Decision Matrix

| Requirement | Small Business | Enterprise | Kiosk/Lab | Remote Managed | Mixed Environment |
|-------------|---------------|------------|-----------|----------------|-------------------|
| **Protocols supported** | SMB or IPP | Both | Both | Both | Both |
| **User can change server** | ✅ | ❌ | ❌ | ❌ | ✅ |
| **User can install printers** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **User can uninstall printers** | ✅ | ❌ | ❌ | ✅ | ✅ |
| **Auto-install all printers** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Authentication required** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Detailed logging** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Profile removal allowed** | ✅ | ❌ | ❌ | ✅ | ✅ |
| **Driver management** | Static | Static + Sheets | Static | Remote JSON | Remote + Static |
| **Fallback drivers** | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Discovery mode** | Single | Both | Both | Both | Both |

### 🔧 Common Use Cases

#### 🏢 Use Case 1: Mixed Protocol Environment

* 500+ users, Windows domain + CUPS servers, both SMB and IPP support

#### 🌐 Use Case 2: Cloud-Native IPP

* Modern organization, CUPS-based cloud printing, IPP-only

#### 🏭 Use Case 3: Legacy SMB Migration

* Transitioning from Windows print servers to IPP, temporary both protocols

#### 🤝 Use Case 4: Hybrid Management

* Static mappings for core printers, remote JSON for dynamic updates, support both protocols

---

## 🔑 Configuration Key Reference

### Multi-Protocol Discovery

| Key                 | Type    | Description                 | Protocols |
| ------------------- | ------- | --------------------------- | --------- |
| `DiscoveryMode`     | String  | "SMB", "IPP", or "Both"     | Both      |
| `PrintServerHost`   | String  | Hostname/IP of SMB server   | SMB       |
| `PrintServerPort`   | Integer | SMB port (default: 445)     | SMB       |
| `PrintServerDomain` | String  | Auth domain for SMB         | SMB       |
| `IPPServerHost`     | String  | Hostname/IP of IPP server   | IPP       |
| `IPPServerPort`     | Integer | IPP port (default: 631) — **enforced**: drives the real `ipp(s)://host:port` target | IPP       |
| `IPPUseSSL`         | Boolean | Use IPPS (SSL/TLS) — **enforced**: selects `ipps://` and the SSL port | IPP       |

### Authentication

| Key                     | Type    | Description            | Protocols |
| ----------------------- | ------- | ---------------------- | --------- |
| `RequireAuthentication` | Boolean | **Enforced** — gates the login prompt for SMB creds | SMB       |
| `IPPRequireAuthentication` | Boolean | Prompt for IPP creds | IPP       |
| `DefaultDomain`         | String  | Default domain prefill | Both      |

### 🔐 Domain Configuration & Authentication

The application supports comprehensive domain configuration to streamline user authentication across both protocols:

#### Domain Settings Overview

| Setting | Purpose | User Experience | Protocols |
|---------|---------|-----------------|-----------|
| `PrintServerDomain` | Server's domain for SMB connections | Used internally for SMB communication | SMB |
| `DefaultDomain` | Pre-fills authentication dialog | Users see this value in domain field | Both |
| `IPPRequireAuthentication` | Controls IPP authentication | Shows/hides IPP login dialog | IPP |

#### Multi-Protocol Authentication Flow

1. **Discovery mode determines** which protocols are used
2. **For SMB printers**: Uses `RequireAuthentication` and `DefaultDomain`
3. **For IPP printers**: Uses `IPPRequireAuthentication` setting
4. **Mixed environments**: May require authentication for both protocols
5. **Domain validation** occurs before attempting printer discovery

#### Configuration Examples

**Example 1: Mixed Environment Authentication**
```xml
<key>DiscoveryMode</key>
<string>Both</string>
<key>PrintServerDomain</key>
<string>CORP</string>
<key>RequireAuthentication</key>
<true/>
<key>IPPRequireAuthentication</key>
<true/>
<key>DefaultDomain</key>
<string>CORP</string>
```

**Example 2: IPP-Only with Authentication**
```xml
<key>DiscoveryMode</key>
<string>IPP</string>
<key>IPPRequireAuthentication</key>
<true/>
<key>DefaultDomain</key>
<string>COMPANY</string>
```

**Example 3: SMB-Only Traditional Setup**
```xml
<key>DiscoveryMode</key>
<string>SMB</string>
<key>PrintServerDomain</key>
<string>DOMAIN.LOCAL</string>
<key>RequireAuthentication</key>
<true/>
<key>DefaultDomain</key>
<string>DOMAIN</string>
```

#### User Experience Benefits

- **Protocol awareness**: Users understand which servers require authentication
- **Reduced friction**: Pre-configured domains reduce typing
- **Consistent experience**: Same authentication flow regardless of protocol
- **Error prevention**: Pre-configured domains prevent authentication failures
- **MDM control**: IT can lock authentication settings to prevent user changes

#### Common Multi-Protocol Configurations

| Scenario | DiscoveryMode | SMB Auth | IPP Auth | Notes |
|----------|---------------|----------|----------|-------|
| Windows + CUPS | `Both` | `true` | `true` | Both protocols need auth |
| Public IPP | `IPP` | N/A | `false` | Open IPP server |
| Secure Mixed | `Both` | `true` | `true` | All printers require auth |
| Legacy Only | `SMB` | `true` | N/A | Traditional Windows domain |

### Installation

| Key                                  | Type    | Description                      | Protocols |
| ------------------------------------ | ------- | -------------------------------- | --------- |
| `AutoInstallPrinters`                | Boolean | **Implemented** — silently install printers at launch (idempotent per session). Behavior depends on `AutoInstallPrinterNames` (see below) | Both      |
| `AutoInstallPrinterNames`            | Array   | Names of printers to auto-install (case-insensitive). **Non-empty** → only these named printers are installed. **Empty** → only printers published by the configured print server(s) are installed — never ad-hoc mDNS/AirPrint discoveries, never USB | Both      |
| `AutoInstallDefaultPrinter`          | String  | **Implemented** — if it matches an installed printer, it is set as the system default | Both      |
| `AllowFallbackWithGenericSuggestion` | Boolean | Allow generic driver fallback    | Both      |

### User Permissions

All permission keys below are **enforced**: the corresponding actions are hidden in the UI *and* guarded inside `PrinterManager` (a blocked action throws a "not permitted" error).

| Key                     | Type    | Description                    | Protocols |
| ----------------------- | ------- | ------------------------------ | --------- |
| `AllowUserServerChange` | Boolean | **Enforced** — lets a non-admin edit the server fields when policy allows | Both      |
| `AllowUserInstall`      | Boolean | **Enforced** — hides and guards user-initiated installs | Both      |
| `AllowUserUninstall`    | Boolean | **Enforced** — hides and guards printer removal | Both      |

### Driver Management (Protocol Agnostic)

| Key                     | Type       | Description                  | Protocols |
| ----------------------- | ---------- | ---------------------------- | --------- |
| `PrinterDriverMappings` | Dictionary | Printer name → path to PPD   | Both      |
| `DriverMappingsURL`     | String     | Remote JSON with driver info | Both      |
| `GoogleSheetsID`        | String     | Google Sheets spreadsheet ID | Both      |
| `GoogleSheetsAPIKey`    | String     | API Key for Sheet access     | Both      |
| `GoogleSheetsRange`     | String     | Range (e.g., `printers!A:B`) | Both      |

### Advanced

| Key                     | Type    | Description               | Protocols |
| ----------------------- | ------- | ------------------------- | --------- |
| `EnableDetailedLogging` | Boolean | **Gates the detailed file log.** Error/fault lines are always written; info/debug lines are written only when this is `true` (default `false`). Enable it to capture a full troubleshooting log. All log output is credential-redacted | Both      |

---

## 🧪 Deployment Instructions

### Jamf Pro

1. Create or edit a Configuration Profile
2. Use "Application & Custom Settings"
3. Upload your `.mobileconfig`
4. Configure discovery mode and both SMB/IPP settings as needed
5. Assign to desired scope and deploy

### Microsoft Intune

1. Devices → Configuration profiles → Create profile
2. Platform: **macOS**, Profile type: **Custom**
3. Upload `.mobileconfig`, assign, and deploy

### Manual

```bash
sudo profiles -I -F NetworkPrinterSettings.mobileconfig
```

To check settings:

```bash
# Check discovery mode and servers
defaults read com.networkprinter.preferences DiscoveryMode
defaults read com.networkprinter.preferences PrintServerHost
defaults read com.networkprinter.preferences IPPServerHost
```

### 🧑‍💻 Standard-User Installs (Privileged Helper)

By default, installing or removing printers requires admin rights. To let **standard (non-admin)** users install and remove printers, the app ships an optional bundled **SMAppService LaunchDaemon** (`PrinterInstallHelper`, daemon identifier `com.ayala.solutions.NetworkPrinter.installhelper`) with a strict, code-signing-pinned XPC interface; when it is enabled the app routes `lpadmin` through it, otherwise it uses `lpadmin` directly (admins need no setup).

Deployment facts to be aware of:

* The helper **only works in a properly signed & notarized build.** The helper tool embeds an `Info.plist` so its signed identity matches the app's code-signing requirement (without it, every XPC connection is rejected).
* It must be **approved once** — either by the user in **System Settings → General → Login Items**, or **pre-approved via MDM** (a managed-login-items / privileged-helper payload).
* **Admins** install directly and are unaffected whether or not the helper is enabled. A **standard user** whose helper isn't yet approved gets an actionable *"open Login Items"* message rather than a silent failure. XPC calls are bounded by a timeout so an unreachable helper can't hang the UI.
* Server-side, the daemon builds the `lpadmin` arguments itself from validated pieces (printer name, device-URI scheme, and a driver confirmed against its own `lpinfo -m` inventory — driverless `everywhere` included) and stages PPD files into a root-owned temporary file. It never accepts raw arguments or credentials.

To pre-approve the helper fleet-wide (no Login Items click on each Mac), deploy a **Managed Login Items** payload alongside the settings profile:

```xml
<dict>
    <key>PayloadType</key>
    <string>com.apple.servicemanagement</string>
    <key>PayloadIdentifier</key>
    <string>com.yourorg.networkprinter.helper-approval</string>
    <key>PayloadUUID</key>
    <string>GENERATED-UUID-HERE</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
    <key>Rules</key>
    <array>
        <dict>
            <key>RuleType</key>
            <string>TeamIdentifier</string>
            <key>RuleValue</key>
            <string>YOUR_TEAM_ID</string>
            <key>Comment</key>
            <string>Pre-approve NetworkPrinter privileged install helper</string>
        </dict>
    </array>
</dict>
```

**Local Network permission cannot be pre-granted.** There is no MDM payload for macOS's Local Network privacy prompt — on first discovery every Mac asks the user to Allow once (System Settings › Privacy & Security › Local Network). Plan help-desk messaging accordingly.


---

## 🔄 Updating the App

Fleet updates are distributed through MDM (Jamf Pro / Intune) as versioned, signed PKGs — build a new PKG for each release, bump the version, and push it through your MDM's normal software-update workflow. The app deliberately ships **without** an in-app updater (e.g. Sparkle): on managed fleets the MDM owns the update channel, which keeps versions consistent across the fleet and avoids updates that bypass IT review.

---

## 🧱 Building & Testing

* **Optimized Release builds**: Release configuration builds with `-O` and whole-module optimization.
* **Signing**: `build_and_notarize.sh` takes its signing identity from the `DEVELOPER_ID` environment variable, notarizes via a `notarytool` keychain profile (`NOTARY_PROFILE`), and signs inside-out (helper and daemon first, app last) for a properly signed & notarized bundle.
* **CI**: A GitHub Actions workflow builds the project and runs the unit tests on every push and pull request.
* **Unit tests (~100)**: Cover the parsers, credential redaction, `ProcessRunner`, CIDR expansion, the auto-install allowlist policy, Kerberos parsing, helper input validation, driver matching, discovery-mode server usage, and `ippfind` exit-status interpretation.

---

## 🧰 Logging

### 🔒 Redaction & Gating

* **Credential-redacted**: before anything is written to OSLog or the file log, credentials are stripped — URI `user:pass@` forms, `smbutil //DOMAIN;user:pass` specs, `-w`/`--password` values, `PASSWD`-style env names, and Google API keys.
* **`EnableDetailedLogging` gates the detailed file log**: error/fault lines are **always** written, but info/debug lines are written only when `EnableDetailedLogging` is `true` (default `false`). To capture a full troubleshooting file log, enable `EnableDetailedLogging` first.

### 📄 File Logs

* Location: `~/Library/Application Support/com.slc.NetworkPrinter/Logs/NetworkPrinter_debug.log`
* Logs contain timestamps, log levels, protocol information, and operation details

### 🖥️ Console Logs

* Use Console.app → filter by `com.slc.NetworkPrinter` or `cfprefsd`
* Protocol-specific filtering available:
  * SMB: Search for "PrinterDiscoveryService" or "SMB"
  * IPP: Search for "IPPDiscoveryService" or "IPP"

### Enhanced Logging Commands

```bash
# View live logs for both protocols
log stream --predicate 'subsystem == "com.slc.NetworkPrinter"' --level debug

# Protocol-specific logging
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PrinterDiscoveryService"' --level debug
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "IPPDiscoveryService"' --level debug

# View recent multi-protocol activity
log show --predicate 'subsystem == "com.slc.NetworkPrinter"' --last 1h
```

---

## 🛠️ Troubleshooting

### MDM Settings Not Applying

* Verify profile in System Settings → Profiles
* Restart Mac to reset `cfprefsd`
* Check with:

```bash
profiles -C -v | grep networkprinter
```

### Multi-Protocol Discovery Issues

* **Only SMB printers found**: Check `IPPServerHost` configuration and IPP server connectivity
* **Only IPP printers found**: Check `PrintServerHost` configuration and SMB server connectivity  
* **No printers found**: Verify `DiscoveryMode` is set correctly and both servers are accessible
* **Local Network permission**: on first discovery, macOS asks *"NetworkPrinter would like to find and connect to devices on your local network"* — AirPrint / IPP Everywhere / mDNS discovery finds nothing until this is allowed. Check **System Settings → Privacy & Security → Local Network** if the app isn't listed or was denied.
* **Printer on another VLAN**: Bonjour/mDNS does not cross VLANs. Either enable mDNS reflection on the gateway (e.g. UniFi's *Multicast DNS*), add the printer's subnet under **Settings → Cross-VLAN Discovery**, or configure the print server address directly.

```bash
# Test SMB connectivity
smbutil view //username:password@smb-server

# Test IPP connectivity  
ippfind -T 5

# Is anything on this segment advertising print services at all?
dns-sd -B _ipp._tcp local.

# Check discovery mode
defaults read com.networkprinter.preferences DiscoveryMode
```

Note: `ippfind` exits `1` when it simply finds no printers — the app reports that
as zero results, not an error.

Note: printer names appear with hyphens instead of spaces (e.g.
`HP-ColorLaserJet-MFP-M282-M285`) because CUPS queue names cannot contain
spaces or most punctuation — the app sanitizes names automatically.

### Driver Mapping Failures

* Check PPD paths work for both SMB and IPP printers
* Enable `EnableDetailedLogging` in preferences
* Verify PPDs in `/Library/Printers/PPDs/Contents/Resources/`
* Test with both SMB and IPP printer names

### Protocol-Specific Authentication Issues

* **SMB Authentication**: Verify `PrintServerDomain` and `DefaultDomain` match your AD
* **IPP Authentication**: Check `IPPRequireAuthentication` setting
* **Mixed Environment**: Ensure both protocol credentials are configured

#### Testing Protocol Configuration:

```bash
# Check SMB configuration
defaults read com.networkprinter.preferences PrintServerHost
defaults read com.networkprinter.preferences PrintServerDomain

# Check IPP configuration
defaults read com.networkprinter.preferences IPPServerHost
defaults read com.networkprinter.preferences IPPRequireAuthentication

# Test SMB connection with domain
smbutil view -I 192.168.1.100 -W DOMAIN //username@server

# Test IPP connection
ippfind -T 5 -x echo "Found: {service_uri}" \;
```

---

## 🔮 Future Roadmap

* Enhanced protocol support (LPR, Google Cloud Print successor)
* Advanced mixed-environment management tools
* Real-time cross-protocol printer job queue visibility
* Cloud-based printer management integration

> Hybrid driver auto-detection, Kerberos/SSO, standard-user installs via the privileged helper, enforced MDM policy, working auto-install, and redacted/gated logging have all **shipped** — see the Features and Architecture sections above.

---

## 🌟 Additional Features

### 🌈 Protocol-Aware UI

The app's UI adapts to show protocol-specific information, with clear indicators for SMB vs IPP printers and their respective connection methods.

### 🚀 Advanced Multi-Protocol Deployment Options

Support for complex mixed environments with automated deployment via MDM and custom scripts that handle both SMB and IPP configurations.

### 📈 Performance Enhancements

Discovery is fully concurrent: independent sources (SMB, IPP server, local mDNS/AirPrint, IPP Everywhere, USB) and the individual mDNS service types run in parallel and are de-duplicated centrally, and installation status is checked with a single bulk `lpstat -p` rather than one process per printer. A refresh completes before the UI reports success, and when every source fails the app surfaces a real error instead of an empty list.

### 📊 Security Enhancements

Credentials are never placed in process arguments — the `security` keychain calls receive the command on **stdin**, and `ipptool` gets its password via the `CUPS_PASSWORD` environment variable — and are never embedded in persisted CUPS device URIs. Validated domain credentials are stored in the Keychain and reused on next launch (subject to `RequireAuthentication` and Kerberos), so users are not re-prompted every launch. All log output is credential-redacted, and IPPS provides encrypted communication.

---

## 🏗️ Enhanced Architecture

The app's enhanced architecture supports multiple sources and protocols seamlessly:

* **PrinterManager**: Orchestrates concurrent discovery and enforces MDM permission policy
* **PreferencesManager**: Manages unified configuration for both protocols
* **ProcessRunner**: Single async runner for all subprocess calls (timeouts, concurrent stdout/stderr draining, child termination on cancel)
* **PrinterDiscoveryService**: Handles SMB/CIFS printer discovery and validation
* **IPPDiscoveryService**: Dedicated IPP/IPPS/AirPrint printer discovery and management
* **DriverMatching** (`ModelExtractor` + `DriverMatcher`): Hybrid driver resolution against `lpinfo -m`
* **KerberosService**: Ticket detection and the password-free SSO/negotiate install path
* **HelperClient + PrinterInstallHelper daemon**: Signed SMAppService LaunchDaemon that enables standard-user installs
* **PrinterInstallationService**: Unified installation service (direct or via the helper)
* **PrinterStatusService / KeychainService / FileLogging**: Bulk status, credential storage, redacted/gated logging
* **UI Components**: Protocol-aware views with visual indicators for printer types
* **Unified Driver System**: IT mappings plus hybrid detection across all protocols

---

## 📊 Complete Configuration Keys Reference Chart

### 🎯 Enhanced Configuration Keys Overview

This comprehensive chart explains every available configuration key for both SMB and IPP protocols:

| Category | Key | Type | Default | Description | User Impact | Protocols | MDM Manageable |
|----------|-----|------|---------|-------------|-------------|-----------|----------------|
| **🔍 Discovery** | `DiscoveryMode` | String | `"Both"` | Discovery mode: "SMB", "IPP", or "Both" | Controls which protocols are used | Both | ✅ |
| **🖥️ SMB Server** | `PrintServerHost` | String | `""` | Hostname or IP address of SMB print server | Required for SMB printer discovery | SMB | ✅ |
| **🖥️ SMB Server** | `PrintServerPort` | Integer | `445` | SMB port for print server connection | Usually 445 for SMB/CIFS | SMB | ✅ |
| **🖥️ SMB Server** | `PrintServerDomain` | String | `""` | Domain name for SMB authentication | Used internally for server communication | SMB | ✅ |
| **🌐 IPP Server** | `IPPServerHost` | String | `""` | Hostname or IP address of IPP print server | Required for IPP printer discovery | IPP | ✅ |
| **🌐 IPP Server** | `IPPServerPort` | Integer | `631` | IPP server port (631 for HTTP, 443 for HTTPS) — **enforced** in the `ipp(s)://host:port` target | Standard IPP ports | IPP | ✅ |
| **🌐 IPP Server** | `IPPUseSSL` | Boolean | `false` | Use HTTPS/TLS for secure IPP connections — **enforced** (selects `ipps://`) | Enables encrypted communication | IPP | ✅ |
| **🌐 IPP Server** | `IPPRequireAuthentication` | Boolean | `true` | Require authentication for IPP access | Shows/hides IPP login dialog | IPP | ✅ |
| **🔐 Auth** | `RequireAuthentication` | Boolean | `false` | **Enforced** — gates the SMB login prompt | Shows/hides SMB login dialog | SMB | ✅ |
| **🔐 Auth** | `DefaultDomain` | String | `""` | Pre-fills domain field in authentication dialog | Reduces user typing, improves UX | Both | ✅ |
| **👤 Permissions** | `AllowUserServerChange` | Boolean | `true` | **Enforced** — lets a non-admin edit server fields when allowed | Shows/hides server configuration UI | Both | ✅ |
| **👤 Permissions** | `AllowUserInstall` | Boolean | `true` | **Enforced** — hides *and* guards installs (blocked action throws) | Shows/hides install buttons | Both | ✅ |
| **👤 Permissions** | `AllowUserUninstall` | Boolean | `true` | **Enforced** — hides *and* guards removal (blocked action throws) | Shows/hides uninstall buttons | Both | ✅ |
| **🤖 Auto-Install** | `AutoInstallPrinters` | Boolean | `false` | **Implemented** — silently install at launch, idempotent per session | Installs without user interaction | Both | ✅ |
| **🤖 Auto-Install** | `AutoInstallPrinterNames` | Array | `[]` | Names to auto-install (case-insensitive). Non-empty → only these; empty → only print-server-published queues (never ad-hoc mDNS/AirPrint, never USB) | Scopes what auto-install adds | Both | ✅ |
| **🤖 Auto-Install** | `AutoInstallDefaultPrinter` | String | `""` | **Implemented** — set as system default if it matches an installed printer | Sets macOS default printer | Both | ✅ |
| **🚗 Drivers** | `PrinterDriverMappings` | Dictionary | `{}` | Printer name → path to PPD (works for both protocols) | Provides specific drivers for all printers | Both | ✅ |
| **🚗 Drivers** | `DriverMappingsURL` | String | `""` | Remote JSON with driver info (protocol agnostic) | Enables dynamic driver updates | Both | ✅ |
| **📊 Sheets** | `GoogleSheetsID` | String | `""` | Google Sheets spreadsheet ID (supports both protocols) | Enables Google Sheets driver management | Both | ✅ |
| **📊 Sheets** | `GoogleSheetsAPIKey` | String | `""` | API Key for Sheet access | Required for Sheets authentication | Both | ✅ |
| **📊 Sheets** | `GoogleSheetsRange` | String | `"printers!A:B"` | Range (e.g., `printers!A:B`) | Defines data location in spreadsheet | Both | ✅ |
| **🔧 Advanced** | `EnableDetailedLogging` | Boolean | `false` | Gates the detailed file log — errors always logged, info/debug only when `true`; output is credential-redacted | Enable to capture a full troubleshooting log | Both | ✅ |
| **🔧 Advanced** | `AllowFallbackWithGenericSuggestion` | Boolean | `false` | Allow generic drivers when specific unavailable | Enables installation with basic drivers | Both | ✅ |

---

### 🔄 Enhanced Configuration Key Relationships

```mermaid
graph TD
    A[DiscoveryMode] --> B{Protocol Selection}
    B -->|SMB| C[PrintServerHost]
    B -->|IPP| D[IPPServerHost]
    B -->|Both| E[Both Servers]
    
    C --> F[SMB Authentication]
    D --> G[IPP Authentication]
    E --> H[Multi-Protocol Auth]
    
    F --> I[SMB Printer Discovery]
    G --> J[IPP Printer Discovery]
    H --> K[Combined Discovery]
    
    I --> L[Unified Driver Mapping]
    J --> L
    K --> L
    
    L --> M[Printer Installation]
    M --> N[Both SMB and IPP Support]
```

---

### 💡 Enhanced Best Practices for Multi-Protocol Configuration

#### ✅ Do's
- **Always set DiscoveryMode** - Determines which protocols are used
- **Configure appropriate servers** - Set both SMB and IPP servers for mixed environments
- **Use unified driver mappings** - Same mappings work for both protocols
- **Test both protocols separately** - Ensure each works independently
- **Enable logging during testing** - Set EnableDetailedLogging: true
- **Use protocol-specific naming** - Help distinguish SMB vs IPP printers

#### ❌ Don'ts
- **Don't forget discovery mode** - Missing DiscoveryMode defaults to "Both"
- **Don't mix authentication settings** - Be clear about which protocols need auth
- **Don't ignore protocol-specific ports** - 445 for SMB, 631 for IPP
- **Don't assume same credentials** - SMB and IPP may use different auth
- **Don't enable logging in production** - Performance impact
- **Don't forget SSL settings** - IPP can use encrypted IPPS

---

### 🔍 Enhanced Configuration Troubleshooting Guide

| Issue | Likely Cause | Check These Keys | Solution | Protocols |
|-------|-------------|------------------|----------|-----------|
| **No printers found** | Server connection | `DiscoveryMode`, `PrintServerHost`, `IPPServerHost` | Verify both server connections | Both |
| **Only SMB printers** | IPP not configured | `IPPServerHost`, `DiscoveryMode` | Configure IPP server settings | IPP |
| **Only IPP printers** | SMB not configured | `PrintServerHost`, `DiscoveryMode` | Configure SMB server settings | SMB |
| **SMB auth fails** | Domain mismatch | `DefaultDomain`, `PrintServerDomain` | Ensure domains match your AD | SMB |
| **IPP auth fails** | Authentication settings | `IPPRequireAuthentication`, `DefaultDomain` | Check IPP auth configuration | IPP |
| **Users can't install** | Permissions | `AllowUserInstall` | Set to `true` or use admin unlock | Both |
| **Wrong drivers selected** | Missing mappings | `PrinterDriverMappings`, `DriverMappingsURL` | Add protocol-specific mappings | Both |
| **Auto-install not working** | Configuration | `AutoInstallPrinters`, `AutoInstallDefaultPrinter` | Enable and specify default | Both |
| **Settings not applying** | MDM sync | All managed keys | Restart and check profile installation | Both |

**This comprehensive README now fully reflects the enhanced multi-protocol capabilities of the NetworkPrinter application, supporting both SMB/CIFS and IPP/IPPS environments with unified driver management.**
