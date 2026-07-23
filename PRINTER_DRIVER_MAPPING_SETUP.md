# NetworkPrinter Driver Mapping Setup & Testing Guide

## Overview

The NetworkPrinter app supports three flexible methods for mapping printer names to their optimal macOS drivers:

1. **MDM Configuration Profile** (Enterprise - Highest Priority)
2. **Remote JSON/CSV Endpoint** (Mid-tier IT - Medium Priority)
3. **Google Sheets Integration** (Small Business - Lower Priority)

The system uses a priority hierarchy: **MDM → Remote JSON → Google Sheets → Bundled Defaults**

> **Scope**: NetworkPrinter targets **SMB/CIFS and USB printers only** — it is not an IPP/AirPrint client. All mapping methods apply to SMB print-server queues (USB printers are matched by model, not by mapping name).

---

## 🧭 Mapping Is Now Fallback-Optional

**You no longer have to map every printer.** The mapping table (MDM `PrinterDriverMappings`, a remote JSON/CSV endpoint, or a Google Sheet) remains the **highest-priority** driver source — an explicit entry always wins — but it is now a *fallback-optional convenience* for overrides and edge cases, not a hard requirement.

When a discovered SMB printer is **not** in the mapping, the app auto-resolves a driver the way Apple's **Add Printer** effectively does, instead of falling back to a generic PostScript PPD:

### 🎯 Driver resolution order
1. **IT driver mapping (highest priority)** — an exact/fuzzy entry from MDM, the remote endpoint, or the Google Sheet always wins. Use it for overrides.
2. **SMB print-server queues** — the app parses the model out of the share **comment/location** text (e.g. `Financial Aid HP LaserJet P3015n`) and **fuzzy-matches** it against the CUPS driver database (`lpinfo -m`); the tokenized score weights model numbers and tolerates suffixes, so `P3015n` matches the `HP LaserJet P3015` PPD.
3. **SNMP** is only queried against a real **printer IP**, never against an SMB print server.
4. **Generic driver + manual picker (last resort)** — used **only** when nothing above clears the confidence threshold. Unmapped printers are no longer given a generic driver by default; `AllowFallbackWithGenericSuggestion` still gates whether generic is offered.

### 💡 What this means for IT
- **Mapping is for overrides and edge cases**, not full inventory coverage. Add an entry when auto-detection picks the wrong PPD, when you want to pin a specific driver, or for printers whose share text is missing or ambiguous.

---

## Option 1: MDM Configuration Profile Setup

### Prerequisites
- MDM solution (Jamf Pro, Microsoft Intune, etc.)
- Administrator access to MDM console
- Understanding of macOS configuration profiles

### Setup Steps

#### 1. Create Configuration Profile

**For Jamf Pro:**
1. Navigate to **Computers** → **Configuration Profiles**
2. Click **New**
3. Select **Application & Custom Settings**
4. Configure:
   - **Source**: Upload file
   - **Preference Domain**: `com.networkprinter.preferences`
   - Upload your `.mobileconfig` file

**For Microsoft Intune:**
1. Navigate to **Devices** → **Configuration profiles**
2. Create **New policy** → **macOS** → **Custom**
3. Upload the XML configuration profile
4. Assign to appropriate device groups

#### 2. Configuration Profile Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadDisplayName</key>
    <string>NetworkPrinter Configuration</string>
    <key>PayloadIdentifier</key>
    <string>com.yourcompany.networkprinter.config</string>
    <key>PayloadOrganization</key>
    <string>Your Organization</string>
    <key>PayloadRemovalDisallowed</key>
    <false/>
    <key>PayloadScope</key>
    <string>System</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>12345678-1234-1234-1234-123456789ABC</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadDisplayName</key>
            <string>NetworkPrinter App Preferences</string>
            <key>PayloadIdentifier</key>
            <string>com.networkprinter.preferences</string>
            <key>PayloadType</key>
            <string>com.apple.ManagedClient.preferences</string>
            <key>PayloadUUID</key>
            <string>87654321-4321-4321-4321-CBA987654321</string>
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
                                <!-- SMB Server Configuration -->
                                <key>PrintServerHost</key>
                                <string>printserver.company.com</string>
                                <key>PrintServerPort</key>
                                <integer>445</integer>
                                <key>PrintServerDomain</key>
                                <string>COMPANY</string>

                                <!-- Authentication Settings -->
                                <key>RequireAuthentication</key>
                                <true/>
                                <key>DefaultDomain</key>
                                <string>COMPANY</string>

                                <!-- User Permissions (enforced) -->
                                <key>AllowUserServerChange</key>
                                <false/>
                                <key>AllowUserInstall</key>
                                <true/>
                                <key>AllowUserUninstall</key>
                                <true/>

                                <!-- Installation Behavior -->
                                <key>AutoInstallPrinters</key>
                                <false/>
                                <!-- Non-empty: only these named printers auto-install (case-insensitive).
                                     Empty: only printers published by the configured print server(s) — never USB. -->
                                <key>AutoInstallPrinterNames</key>
                                <array/>
                                <key>AutoInstallDefaultPrinter</key>
                                <string></string>

                                <!-- Logging and Fallback -->
                                <key>EnableDetailedLogging</key>
                                <false/>
                                <key>AllowFallbackWithGenericSuggestion</key>
                                <false/>

                                <!-- Driver Mappings (overrides; fallback-optional) -->
                                <key>PrinterDriverMappings</key>
                                <dict>
                                    <key>CORP-HP-4015-FL1</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz</string>
                                    <key>CORP-CANON-5535-FL2</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz</string>
                                </dict>

                                <!-- Remote Driver Management -->
                                <key>DriverMappingsURL</key>
                                <string>https://your-company.com/printer-drivers.json</string>

                                <!-- Google Sheets Integration (placeholders only) -->
                                <key>GoogleSheetsID</key>
                                <string>YOUR_GOOGLE_SHEETS_ID</string>
                                <key>GoogleSheetsAPIKey</key>
                                <string>YOUR_GOOGLE_SHEETS_API_KEY</string>
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

#### 3. Deploy Profile
- Assign to target computer groups
- Verify deployment in MDM console
- Check installation on test machines

### Testing MDM Configuration

#### 1. Verify Profile Installation
```bash
# Check if profile is installed
sudo profiles -C -v | grep com.networkprinter.preferences

# Check specific preferences
defaults read com.networkprinter.preferences PrinterDriverMappings
defaults read com.networkprinter.preferences AutoInstallPrinterNames
```

#### 2. Verify Discovery & Mapping
1. Open NetworkPrinter app
2. Check Console.app for logs: `com.slc.NetworkPrinter` subsystem
3. Look for: `"Using MDM managed printer driver mappings"`
4. Confirm SMB discovery works and driver suggestions appear

---

## Option 2: Remote JSON/CSV Endpoint

### Prerequisites
- Web server with HTTPS support
- JSON/CSV file hosting capability

### Setup Steps

#### 1. Simple Flat JSON (Recommended)
```json
{
  "CORP-HP-4015-FL1": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
  "CORP-CANON-5535-FL2": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
  "Finance-HPLJ451c": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet 400 Series.ppd.gz"
}
```

#### 2. Testing Remote JSON
```bash
# Test JSON endpoint
curl -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0" \
     https://your-company.com/printer-drivers.json

# Validate JSON format
python3 -c "
import json, urllib.request
url = 'https://your-company.com/printer-drivers.json'
try:
    with urllib.request.urlopen(url) as response:
        data = json.load(response)
    print('✅ Valid JSON format')
    print(f'📊 Found {len(data)} mappings')
    for key in list(data.keys())[:5]:
        print(f'  {key}: {data[key]}')
except Exception as e:
    print(f'❌ JSON validation failed: {e}')
"
```

---

## Option 3: Google Sheets Integration

### Prerequisites
- Google Account with Google Sheets access
- Google Cloud Console project
- Google Sheets API enabled

### Setup

#### 1. Create Spreadsheet

**Sheet Structure:**
```
     A                         B
1    Printer Name              Driver Path
2    CORP-HP-4015-FL1          /Library/Printers/PPDs/.../HP LaserJet Pro 4015n.ppd.gz
3    CORP-CANON-5535-FL2       /Library/Printers/PPDs/.../Canon iR-ADV C5535i PS.ppd.gz
```

#### 2. Configuration for Google Sheets
```bash
# Placeholders only — supply your real values via MDM, not source control
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
```

#### 3. Testing Google Sheets Integration
```bash
SHEET_ID="YOUR_GOOGLE_SHEETS_ID"
API_KEY="YOUR_GOOGLE_SHEETS_API_KEY"
RANGE="Sheet1!A:B"

curl "https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${RANGE}?key=${API_KEY}"
```

> **Security**: never commit a real Google Sheets ID or API key. Use placeholders in source control and inject real values through MDM. Keys committed in earlier versions remain in git history and must be rotated/revoked.

---

## SMB/CIFS Testing

```bash
# Test SMB connectivity (use Kerberos SSO where available; avoid embedding a password)
smbutil view //username@server

# Watch SMB printer discovery
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PrinterDiscoveryService"' --level debug

# Enable detailed file logging while troubleshooting
defaults write com.networkprinter.preferences EnableDetailedLogging -bool true
```

Detailed file logging is gated by `EnableDetailedLogging` (default off); error and fault lines are always written, and all log output is credential-redacted.

---

## Key Benefits of the Driver System

### 🧠 Automatic
- Unmapped SMB printers auto-resolve via share-comment model parsing + fuzzy `lpinfo -m` matching
- Mapping is reserved for overrides and edge cases

### 📊 Unified Management
- Single source of truth (MDM, JSON, or Google Sheets) for driver overrides
- Consistent MDM configuration approach

### 🔍 Debuggable
- `EnableDetailedLogging`-gated, credential-redacted logs
- Clear identification of matched drivers and confidence

---

This guide covers SMB/CIFS driver mapping and testing. NetworkPrinter installs SMB print-server queues and local USB printers only — it is not an IPP/AirPrint client.
