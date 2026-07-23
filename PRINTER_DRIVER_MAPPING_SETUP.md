# NetworkPrinter Driver Mapping Setup & Testing Guide

## Overview

The NetworkPrinter app supports three flexible methods for mapping printer names to their optimal macOS drivers:

1. **MDM Configuration Profile** (Enterprise - Highest Priority)
2. **Remote JSON/CSV Endpoint** (Mid-tier IT - Medium Priority)
3. **Google Sheets Integration** (Small Business - Lower Priority)

The system uses a priority hierarchy: **MDM → Remote JSON → Google Sheets → Bundled Defaults**

**NEW**: All driver mapping methods work seamlessly with both **SMB/CIFS** and **IPP/IPPS** printers!

---

## 🧭 Mapping Is Now Fallback-Optional

**You no longer have to map every printer.** The mapping table (MDM `PrinterDriverMappings`, a remote JSON/CSV endpoint, or a Google Sheet) remains the **highest-priority** driver source — an explicit entry always wins — but it is now a *fallback-optional convenience* for overrides and edge cases, not a hard requirement.

When a discovered printer is **not** in the mapping, the app auto-resolves a driver the way Apple's **Add Printer** effectively does, instead of falling back to a generic PostScript PPD:

### 🎯 Driver resolution order
1. **IT driver mapping (highest priority)** — an exact/fuzzy entry from MDM, the remote endpoint, or the Google Sheet always wins. Use it for overrides.
2. **IPP / AirPrint printers** — the app reads the printer's advertised **make-and-model** (`ipptool` get-printer-attributes) and prefers **driverless "everywhere"** when supported.
3. **SMB print-server queues** — the app parses the model out of the share **comment/location** text (e.g. `Financial Aid HP LaserJet P3015n`) and **fuzzy-matches** it against the CUPS driver database (`lpinfo -m`); the tokenized score weights model numbers and tolerates suffixes, so `P3015n` matches the `HP LaserJet P3015` PPD.
4. **SNMP** is only queried against a real **printer IP**, never against an SMB print server.
5. **Generic driver + manual picker (last resort)** — used **only** when nothing above clears the confidence threshold. Unmapped printers are no longer given a generic driver by default.

### 💡 What this means for IT
- **Mapping is for overrides and edge cases**, not full inventory coverage. Add an entry when auto-detection picks the wrong PPD, when you want to pin a specific driver, or for printers whose make-and-model / share text is missing or ambiguous.
- Everything below still works as documented — it now co-exists with automatic detection.

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

#### 2. Enhanced Configuration Profile with IPP Support

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
                                <!-- Discovery Mode Configuration -->
                                <key>DiscoveryMode</key>
                                <string>Both</string>
                                
                                <!-- SMB Server Configuration -->
                                <key>PrintServerHost</key>
                                <string>printserver.company.com</string>
                                <key>PrintServerPort</key>
                                <integer>445</integer>
                                <key>PrintServerDomain</key>
                                <string>COMPANY</string>
                                
                                <!-- IPP Server Configuration -->
                                <key>IPPServerHost</key>
                                <string>ippserver.company.com</string>
                                <key>IPPServerPort</key>
                                <integer>631</integer>
                                <key>IPPUseSSL</key>
                                <false/>
                                <key>IPPRequireAuthentication</key>
                                <true/>
                                
                                <!-- Authentication Settings -->
                                <key>RequireAuthentication</key>
                                <true/>
                                <key>DefaultDomain</key>
                                <string>COMPANY</string>
                                
                                <!-- User Permissions -->
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
                                     Empty: only printers published by the configured print server(s) — never ad-hoc mDNS/AirPrint, never USB. -->
                                <key>AutoInstallPrinterNames</key>
                                <array/>
                                <key>AutoInstallDefaultPrinter</key>
                                <string></string>
                                
                                <!-- Logging and Fallback -->
                                <key>EnableDetailedLogging</key>
                                <false/>
                                <key>AllowFallbackWithGenericSuggestion</key>
                                <false/>
                                
                                <!-- Driver Mappings (Works for both SMB and IPP!) -->
                                <key>PrinterDriverMappings</key>
                                <dict>
                                    <!-- SMB Printers -->
                                    <key>CORP-HP-4015-FL1</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz</string>
                                    <key>CORP-CANON-5535-FL2</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz</string>
                                    
                                    <!-- IPP Printers -->
                                    <key>IPP-CUPS-COLOR-01</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd</string>
                                    <key>Office-Printer-Main</key>
                                    <string>/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz</string>
                                </dict>
                                
                                <!-- Remote Driver Management -->
                                <key>DriverMappingsURL</key>
                                <string>https://your-company.com/printer-drivers.json</string>
                                
                                <!-- Google Sheets Integration -->
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
defaults read com.networkprinter.preferences DiscoveryMode
defaults read com.networkprinter.preferences IPPServerHost
```

#### 2. Test Both Protocols
1. Open NetworkPrinter app
2. Check Console.app for logs: `NetworkPrinter` subsystem
3. Look for: `"Using MDM managed printer driver mappings"`
4. Verify both SMB and IPP discovery work
5. Test driver suggestions for both printer types

---

## Option 2: Remote JSON/CSV Endpoint

### Prerequisites
- Web server with HTTPS support
- JSON/CSV file hosting capability
- Basic web development knowledge

### Setup Steps

#### 1. Create Enhanced JSON Format (Supports Both Protocols)

```json
{
  "version": "2025.01.11",
  "last_updated": "2025-01-11T10:30:00Z",
  "protocol_support": ["SMB", "IPP"],
  "mappings": {
    "SMB_Printers": {
      "CORP-HP-4015-FL1": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
      "CORP-CANON-5535-FL2": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
      "Finance-HPLJ451c": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet 400 Series.ppd.gz"
    },
    "IPP_Printers": {
      "Office-Printer-Main": "/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz",
      "IPP-CUPS-COLOR-01": "/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd",
      "Reception-Printer": "/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz"
    },
    "Universal_Mappings": {
      "HP-LaserJet-Pro-4015n": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
      "Canon-iR-ADV-C5535i": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz"
    }
  }
}
```

#### 2. Simple Flat JSON (Recommended)
```json
{
  "CORP-HP-4015-FL1": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
  "CORP-CANON-5535-FL2": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
  "Office-Printer-Main": "/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz",
  "IPP-CUPS-COLOR-01": "/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd",
  "Reception-Printer": "/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz"
}
```

#### 3. Testing Remote JSON
```bash
# Test JSON endpoint
curl -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0" \
     https://your-company.com/printer-drivers.json

# Validate JSON format
python3 -c "
import json
import urllib.request

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

### Enhanced Google Sheets Setup

#### 1. Create Multi-Protocol Spreadsheet

**Sheet Structure (works for both SMB and IPP):**
```
     A                         B                                    C
1    Printer Name              Driver Path                          Protocol
2    CORP-HP-4015-FL1          /Library/Printers/PPDs/.../HP...     SMB
3    Office-Printer-Main       /Library/Printers/PPDs/.../HP...     IPP
4    CORP-CANON-5535-FL2       /Library/Printers/PPDs/.../Canon...  SMB
5    IPP-CUPS-COLOR-01         /Library/Printers/PPDs/.../Generic... IPP
6    Reception-Printer         /Library/Printers/PPDs/.../Canon...  Both
```

**Note**: Column C (Protocol) is optional - the app will use the printer name for mapping regardless of protocol.

#### 2. Configuration for Google Sheets
```bash
# Configure for mixed environment
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
defaults write com.networkprinter.preferences DiscoveryMode "Both"
```

#### 3. Testing Google Sheets Integration
```bash
# Test API access
SHEET_ID="YOUR_GOOGLE_SHEETS_ID"
API_KEY="YOUR_GOOGLE_SHEETS_API_KEY"
RANGE="Sheet1!A:B"

curl "https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${RANGE}?key=${API_KEY}"
```

---

## Protocol-Specific Testing

### SMB/CIFS Testing
```bash
# Test SMB connectivity
smbutil view //username:password@server

# Check SMB printer discovery
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PrinterDiscoveryService"' --level debug

# Test SMB printer installation
# Look for logs: "Preparing to install SMB network printer"
```

### IPP Testing
```bash
# Test IPP connectivity
ippfind -T 5 -x echo "Found: {service_uri}" \;

# Check IPP printer discovery
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "IPPDiscoveryService"' --level debug

# Test IPP printer installation  
# Look for logs: "Preparing to install IPP printer"
```

### Mixed Environment Testing
```bash
# Enable detailed logging for both protocols
defaults write com.networkprinter.preferences EnableDetailedLogging -bool true
defaults write com.networkprinter.preferences DiscoveryMode "Both"

# Launch app and check Console.app for:
# - "Discovery mode: Both SMB & IPP"
# - "SMB printer discovery completed"
# - "IPP printer discovery completed"
# - Driver mapping logs for both protocols
```

---

## Key Benefits of Updated Driver System

### 🔄 **Protocol Agnostic**
- Same driver mappings work for SMB and IPP printers
- No need to maintain separate mapping systems
- Seamless mixed environment support

### 📊 **Unified Management**
- Single source of truth for all printer drivers
- Same Google Sheets/JSON works for all protocols
- Consistent MDM configuration approach

### 🎯 **Enhanced Flexibility**
- Discovery mode selection (SMB, IPP, Both)
- Protocol-specific server configurations
- Fallback and priority systems maintained

### 🔍 **Improved Debugging**
- Enhanced logging for both protocols
- Clear identification of printer types
- Separate validation for each protocol

---

## Advanced Configuration Examples

### Mixed Environment Configuration
```xml
<!-- Support both SMB and IPP in same environment -->
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

### IPP-Only Environment
```xml
<!-- IPP-only setup for modern environments -->
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

### Legacy SMB-Only Environment
```xml
<!-- Traditional SMB setup -->
<key>DiscoveryMode</key>
<string>SMB</string>
<key>PrintServerHost</key>
<string>printserver.company.com</string>
<key>PrintServerDomain</key>
<string>COMPANY</string>
<key>RequireAuthentication</key>
<true/>
```

---

This enhanced guide now fully supports both SMB/CIFS and IPP/IPPS printer protocols while maintaining the same simple driver mapping system across all printer types.
