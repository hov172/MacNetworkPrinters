# NetworkPrinter Driver Mapping: Complete Setup & Testing Guide

## Table of Contents
1. [Overview](#overview)
2. [Mapping Is Now Fallback-Optional](#-mapping-is-now-fallback-optional-read-this-first)
3. [Quick Start](#quick-start)
3. [Option 1: MDM Configuration Profile](#option-1-mdm-configuration-profile)
4. [Option 2: Remote JSON/CSV Endpoint](#option-2-remote-jsoncsv-endpoint)
5. [Option 3: Google Sheets Integration](#option-3-google-sheets-integration)
6. [Testing & Validation](#testing--validation)
7. [Troubleshooting](#troubleshooting)
8. [Best Practices](#best-practices)
9. [API Reference](#api-reference)
10. [Support](#support)

---

## Overview

The NetworkPrinter app supports three flexible methods for mapping printer names to their optimal macOS drivers, designed to work with any enterprise infrastructure:

### 🏢 **Enterprise (MDM Configuration Profile)**
- **Priority**: Highest
- **Best for**: Large enterprises with MDM infrastructure
- **Benefits**: Centralized management, offline operation, secure
- **Supports**: Both SMB/CIFS and IPP/IPPS printers

### 🌐 **Mid-tier (Remote JSON/CSV Endpoint)**
- **Priority**: Medium
- **Best for**: Mid-size companies with web hosting
- **Benefits**: Dynamic updates, version control, scalable
- **Supports**: Both SMB/CIFS and IPP/IPPS printers

### 📊 **Small Business (Google Sheets Integration)**
- **Priority**: Lower
- **Best for**: Small organizations using Google Workspace
- **Benefits**: Easy collaboration, no technical setup, visual editing
- **Supports**: Both SMB/CIFS and IPP/IPPS printers

### 📦 **Fallback (Bundled Defaults)**
- **Priority**: Lowest
- **Best for**: Always available backup
- **Benefits**: Offline operation, no configuration needed

**🆕 NEW**: The system now supports both **SMB/CIFS** and **IPP/IPPS** protocols seamlessly! The same driver mapping system works for all printer types.

The system uses a **priority hierarchy**: `MDM → Remote JSON → Google Sheets → Bundled Defaults`

---

## 🧭 Mapping Is Now Fallback-Optional (Read This First)

**You no longer have to map every printer.** The mapping table (MDM `PrinterDriverMappings`, a remote JSON/CSV endpoint, or a Google Sheet) is still the **highest-priority** driver source — an explicit entry always wins — but it has become a *fallback-optional convenience* for overrides and edge cases rather than a hard requirement.

When a discovered printer is **not** found in the mapping, the app auto-resolves a driver the way Apple's own **Add Printer** effectively does, instead of dropping straight to a generic PostScript PPD:

### 🎯 **Driver resolution order**
1. **IT driver mapping (highest priority)** — an exact/fuzzy entry from MDM, the remote endpoint, or the Google Sheet always wins. Use this for overrides and any printer whose auto-detected driver is wrong.
2. **IPP / AirPrint printers** — the app reads the printer's advertised **make-and-model** (`ipptool` get-printer-attributes) and prefers **driverless "everywhere"** when the device supports it.
3. **SMB print-server queues** — the app parses the model out of the share **comment/location** text (for example, `Financial Aid HP LaserJet P3015n`) and **fuzzy-matches** it against the local CUPS driver database (`lpinfo -m`). The tokenized score weights model numbers and tolerates suffixes, so `P3015n` still matches the `HP LaserJet P3015` PPD.
4. **SNMP** is only queried against a real **printer IP**, never against an SMB print server.
5. **Generic driver + manual picker (last resort)** — used **only** when nothing above clears the confidence threshold. This is no longer the default outcome for unmapped printers.

### 💡 **What this means for IT**
- **Mapping is for overrides and edge cases**, not full inventory coverage. Add an entry when auto-detection picks the wrong PPD, when you want to pin a specific driver, or for printers whose make-and-model / share text is missing or ambiguous.
- Unmapped printers are **not** silently given a generic driver by default — they go through auto-resolution first and only fall back to generic + the manual picker when confidence is too low.
- Everything below (MDM / Remote JSON / Google Sheets) still works exactly as documented; it now simply co-exists with automatic detection.

---

## Quick Start

### 🚀 **5-Minute Setup (Google Sheets)**

1. **Create Google Sheets spreadsheet**:
```
Column A: Printer Name          | Column B: Driver Path
HP-LaserJet-Pro-4015n          | /Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz
Canon-iR-ADV-C5535i            | /Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz
Office-Printer-Main            | /Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz
IPP-CUPS-COLOR-01              | /Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd
```

2. **Get API key** from [Google Cloud Console](https://console.cloud.google.com/)

3. **Configure NetworkPrinter**:
```bash
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "printers!A:B"
defaults write com.networkprinter.preferences DiscoveryMode "Both"

# Note: Name of your sheet at the bottom - update range accordingly
# Example: "Sheet1!A:B" or "printers!A:B"
```

4. **Test**: Launch NetworkPrinter and check Console.app for `"Successfully fetched Google Sheets driver mappings"`

---

## Option 1: MDM Configuration Profile

### 📋 **Prerequisites**
- MDM solution (Jamf Pro, Microsoft Intune, etc.)
- Administrator access to MDM console
- Understanding of macOS configuration profiles

### 🔧 **Setup Steps**

#### **1. Create Configuration Profile**

**Enhanced Configuration Profile with Full IPP Support:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadDisplayName</key>
            <string>NetworkPrinter Driver Mappings</string>
            <key>PayloadIdentifier</key>
            <string>com.networkprinter.preferences</string>
            <key>PayloadType</key>
            <string>com.networkprinter.preferences</string>
            <key>PayloadUUID</key>
            <string>12345678-1234-1234-1234-123456789ABC</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            
            <!-- Discovery Mode: SMB, IPP, or Both -->
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
            
            <!-- Authentication -->
            <key>RequireAuthentication</key>
            <true/>
            <key>DefaultDomain</key>
            <string>COMPANY</string>
            
            <!-- Driver Mappings (Works for BOTH SMB and IPP printers!) -->
            <key>PrinterDriverMappings</key>
            <dict>
                <!-- SMB Printers -->
                <key>CORP-HP-4015-FL1</key>
                <string>/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz</string>
                <key>CORP-CANON-5535-FL2</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz</string>
                <key>CORP-XEROX-7835-FL3</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz</string>
                <key>CORP-BROTHER-L2350-FL4</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Brother HL-L2350DW.ppd.gz</string>
                
                <!-- IPP Printers -->
                <key>Office-Printer-Main</key>
                <string>/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz</string>
                <key>IPP-CUPS-COLOR-01</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd</string>
                <key>IPP-CUPS-BW-01</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Generic PostScript Printer.ppd</string>
                <key>Reception-Printer</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz</string>
            </dict>
            
            <!-- Optional: Fallback to remote source -->
            <key>DriverMappingsURL</key>
            <string>https://your-company.com/printer-drivers.json</string>
            
            <!-- Optional: Google Sheets Integration -->
            <key>GoogleSheetsID</key>
            <string>YOUR_GOOGLE_SHEETS_ID</string>
            <key>GoogleSheetsAPIKey</key>
            <string>YOUR_GOOGLE_SHEETS_API_KEY</string>
            <key>GoogleSheetsRange</key>
            <string>printers!A:B</string>
            
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
            
            <!-- Advanced Options -->
            <key>EnableDetailedLogging</key>
            <false/>
            <key>AllowFallbackWithGenericSuggestion</key>
            <false/>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>NetworkPrinter Configuration</string>
    <key>PayloadIdentifier</key>
    <string>com.yourcompany.networkprinter.config</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>87654321-4321-4321-4321-CBA987654321</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**For Microsoft Intune:**
1. Navigate to **Devices** → **Configuration profiles**
2. Create **New policy** → **macOS** → **Custom**
3. Upload the XML configuration profile above
4. Assign to appropriate device groups

#### **2. Deploy Profile**
- Assign to target computer groups
- Verify deployment in MDM console
- Check installation on test machines

### 🧪 **Testing MDM Configuration**

#### **1. Verify Profile Installation**
```bash
# Check if profile is installed
sudo profiles -C -v | grep com.networkprinter.preferences

# Check specific preferences including new IPP settings
defaults read com.networkprinter.preferences PrinterDriverMappings
defaults read com.networkprinter.preferences DiscoveryMode
defaults read com.networkprinter.preferences IPPServerHost
defaults read com.networkprinter.preferences IPPServerPort
```

#### **2. Test in NetworkPrinter App**
1. Open NetworkPrinter app
2. Check Console.app for logs: `NetworkPrinter` subsystem
3. Look for: `"Using MDM managed printer driver mappings"`
4. Verify both SMB and IPP discovery modes work
5. Test driver suggestions for both SMB and IPP printers

#### **3. Enhanced Validation Script**
```bash
#!/bin/bash
# validate_mdm_config_enhanced.sh
# Test MDM driver mappings for both SMB and IPP

echo "🔍 Testing Enhanced MDM Configuration..."
echo "======================================="

# Check if profile exists
if profiles -C -v | grep -q "com.networkprinter.preferences"; then
    echo "✅ Configuration profile installed"
    
    # Get profile details
    PROFILE_INFO=$(profiles -C -v | grep -A 20 "com.networkprinter.preferences")
    echo "📄 Profile details:"
    echo "$PROFILE_INFO"
    
else
    echo "❌ Configuration profile NOT found"
    echo "💡 Run: sudo profiles -I -F /path/to/profile.mobileconfig"
    exit 1
fi

echo ""
echo "🔍 Checking discovery mode and server configuration..."

# Check discovery mode
DISCOVERY_MODE=$(defaults read com.networkprinter.preferences DiscoveryMode 2>/dev/null)
if [ -n "$DISCOVERY_MODE" ]; then
    echo "✅ Discovery mode: $DISCOVERY_MODE"
else
    echo "⚠️  Discovery mode not configured (will default to Both)"
fi

# Check SMB server configuration
SMB_SERVER=$(defaults read com.networkprinter.preferences PrintServerHost 2>/dev/null)
if [ -n "$SMB_SERVER" ]; then
    echo "✅ SMB server configured: $SMB_SERVER"
else
    echo "⚠️  SMB server not configured"
fi

# Check IPP server configuration
IPP_SERVER=$(defaults read com.networkprinter.preferences IPPServerHost 2>/dev/null)
if [ -n "$IPP_SERVER" ]; then
    echo "✅ IPP server configured: $IPP_SERVER"
    
    IPP_PORT=$(defaults read com.networkprinter.preferences IPPServerPort 2>/dev/null)
    IPP_SSL=$(defaults read com.networkprinter.preferences IPPUseSSL 2>/dev/null)
    echo "📡 IPP Port: ${IPP_PORT:-631}, SSL: ${IPP_SSL:-false}"
else
    echo "⚠️  IPP server not configured"
fi

echo ""
echo "🔍 Checking driver mappings..."

# Check driver mappings
MAPPINGS=$(defaults read com.networkprinter.preferences PrinterDriverMappings 2>/dev/null)
if [ $? -eq 0 ]; then
    echo "✅ Driver mappings found in preferences"
    echo "📊 Mappings:"
    echo "$MAPPINGS"
    
    # Count mappings
    COUNT=$(echo "$MAPPINGS" | grep -c "=")
    echo "📈 Total mappings: $COUNT"
    
    # Check for both SMB and IPP printer examples
    if echo "$MAPPINGS" | grep -q "CORP-"; then
        echo "✅ SMB printer mappings found"
    fi
    if echo "$MAPPINGS" | grep -q "IPP-\|Office-"; then
        echo "✅ IPP printer mappings found"
    fi
    
else
    echo "❌ No driver mappings found"
    echo "💡 Check profile payload structure"
fi

echo ""
echo "✅ Enhanced MDM validation complete"
```

**Usage:**
```bash
chmod +x validate_mdm_config_enhanced.sh
./validate_mdm_config_enhanced.sh
```

---

## Option 2: Remote JSON/CSV Endpoint

### 📋 **Prerequisites**
- Web server with HTTPS support
- JSON/CSV file hosting capability
- Basic web development knowledge

### 🔧 **Setup Steps**

#### **1. Create Enhanced JSON Endpoint**

**Multi-Protocol JSON Format:**
```json
{
  "version": "2025.01.11",
  "last_updated": "2025-01-11T10:30:00Z",
  "protocol_support": ["SMB", "IPP"],
  "discovery_modes": ["SMB", "IPP", "Both"],
  "mappings": {
    "SMB_Printers": {
      "CORP-HP-4015-FL1": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
      "CORP-CANON-5535-FL2": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
      "Finance-HPLJ451c": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet 400 Series.ppd.gz",
      "CORP-XEROX-7835-FL3": "/Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz"
    },
    "IPP_Printers": {
      "Office-Printer-Main": "/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz",
      "IPP-CUPS-COLOR-01": "/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd",
      "Reception-Printer": "/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz",
      "Conference-Room-Printer": "/Library/Printers/PPDs/Contents/Resources/Brother Color Laser.ppd.gz"
    },
    "Universal_Mappings": {
      "HP-LaserJet-Pro-4015n": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
      "Canon-iR-ADV-C5535i": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz"
    }
  }
}
```

**Simple Flat JSON Format (Recommended):**
```json
{
  "CORP-HP-4015-FL1": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
  "CORP-CANON-5535-FL2": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
  "Office-Printer-Main": "/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz",
  "IPP-CUPS-COLOR-01": "/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd",
  "Reception-Printer": "/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz",
  "Conference-Room-Printer": "/Library/Printers/PPDs/Contents/Resources/Brother Color Laser.ppd.gz",
  "Finance-HPLJ451c": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet 400 Series.ppd.gz"
}
```

**CSV Format Option (Protocol Agnostic):**
```csv
Printer Name,Driver Path,Display Name,Notes,Protocol
CORP-HP-4015-FL1,/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz,HP LaserJet Pro 4015n,SMB printer Floor 1,SMB
Office-Printer-Main,/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz,Office Color Printer,IPP main office printer,IPP
IPP-CUPS-COLOR-01,/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd,Generic Color Laser,Generic IPP color printer,IPP
Reception-Printer,/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz,Reception Printer,Front desk printer,Both
```

#### **2. Deploy Configuration**

**Option A: MDM (Recommended)**
```xml
<key>DriverMappingsURL</key>
<string>https://your-company.com/printer-drivers.json</string>
<key>DiscoveryMode</key>
<string>Both</string>
```

**Option B: Manual Configuration**
```bash
# Set the URL and discovery mode in user defaults
defaults write com.networkprinter.preferences DriverMappingsURL "https://your-company.com/printer-drivers.json"
defaults write com.networkprinter.preferences DiscoveryMode "Both"

# Verify configuration
defaults read com.networkprinter.preferences DriverMappingsURL
defaults read com.networkprinter.preferences DiscoveryMode
```

### 🧪 **Testing Remote JSON/CSV**

#### **1. Test URL Accessibility**
```bash
# Test JSON endpoint
curl -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0" \
     https://your-company.com/printer-drivers.json

# Test with verbose output for debugging
curl -v -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0" \
     https://your-company.com/printer-drivers.json

# Test CSV endpoint
curl -H "Accept: text/csv" \
     https://your-company.com/printer-drivers.csv
```

#### **2. Enhanced JSON Validation**
```bash
# Test JSON validity with enhanced validation
python3 -c "
import json
import urllib.request

url = 'https://your-company.com/printer-drivers.json'
try:
    with urllib.request.urlopen(url) as response:
        data = json.load(response)
    print('✅ Valid JSON format')
    
    # Count total mappings
    total_mappings = 0
    if isinstance(data, dict):
        if 'mappings' in data:
            # Structured format
            for category, mappings in data['mappings'].items():
                if isinstance(mappings, dict):
                    total_mappings += len(mappings)
                    print(f'  {category}: {len(mappings)} printers')
        else:
            # Flat format
            total_mappings = len(data)
            print(f'  Flat format: {total_mappings} printers')
    
    print(f'📊 Total mappings found: {total_mappings}')
    
    # Show sample mappings
    if isinstance(data, dict) and 'mappings' not in data:
        # Flat format - show first few
        for i, (key, value) in enumerate(list(data.items())[:3]):
            print(f'  {key}: {value}')
    elif 'mappings' in data:
        # Structured format - show samples from each category
        for category, mappings in data['mappings'].items():
            if isinstance(mappings, dict) and mappings:
                sample_key, sample_value = next(iter(mappings.items()))
                print(f'  {category} sample: {sample_key} -> {sample_value}')
                
except Exception as e:
    print(f'❌ JSON validation failed: {e}')
"
```

#### **3. NetworkPrinter App Testing**
1. Configure the URL in app settings
2. Set discovery mode to "Both" to test with all protocols
3. Launch NetworkPrinter app
4. Check Console.app for: `"Successfully parsed JSON driver mappings"`
5. Verify mappings load correctly for both SMB and IPP printers
6. Test with/without internet connectivity

---

## Option 3: Google Sheets Integration

### 📋 **Prerequisites**
- Google Account with Google Sheets access
- Google Cloud Console project
- Google Sheets API enabled
- Basic understanding of Google APIs

### 🔧 **Enhanced Setup Steps for Multi-Protocol Support**

#### **1. Create Multi-Protocol Google Sheets Spreadsheet**

**Enhanced Spreadsheet Structure:**
```
     A                         B                                    C            D
1    Printer Name              Driver Path                          Protocol     Notes
2    CORP-HP-4015-FL1          /Library/Printers/PPDs/.../HP...     SMB         Floor 1 SMB printer
3    Office-Printer-Main       /Library/Printers/PPDs/.../HP...     IPP         Main office IPP printer  
4    CORP-CANON-5535-FL2       /Library/Printers/PPDs/.../Canon...  SMB         Floor 2 Canon MFP
5    IPP-CUPS-COLOR-01         /Library/Printers/PPDs/.../Generic... IPP         Generic IPP color printer
6    Reception-Printer         /Library/Printers/PPDs/.../Canon...  Both        Works with both protocols
7    Finance-HPLJ451c          /Library/Printers/PPDs/.../HP...     SMB         Finance department printer
8    Conference-Room-Printer   /Library/Printers/PPDs/.../Brother... IPP         Conference room printer
```

**Note**: Columns C and D are optional - the app uses printer name for mapping regardless of protocol. The app reads columns A:B by default.

#### **2. Set Up Google Sheets API (Same Process)**

**Step 1: Create Google Cloud Project**
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing
3. Name it "NetworkPrinter Integration"

**Step 2: Enable Google Sheets API**
1. Navigate to **APIs & Services** → **Library**
2. Search for "Google Sheets API"
3. Click **Enable**

**Step 3: Create API Key**
1. Go to **APIs & Services** → **Credentials**
2. Click **Create Credentials** → **API Key**
3. Copy the API key
4. Click **Restrict Key** (recommended)
5. Under **API restrictions**, select **Google Sheets API**

#### **3. Enhanced NetworkPrinter Configuration**

**Option A: MDM Configuration (Recommended)**
```xml
<key>GoogleSheetsID</key>
<string>YOUR_GOOGLE_SHEETS_ID</string>
<key>GoogleSheetsAPIKey</key>
<string>YOUR_GOOGLE_SHEETS_API_KEY</string>
<key>GoogleSheetsRange</key>
<string>Sheet1!A:B</string>
<key>DiscoveryMode</key>
<string>Both</string>
```

**Option B: Manual Configuration**
```bash
# Configure for multi-protocol environment
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
defaults write com.networkprinter.preferences DiscoveryMode "Both"

# Verify configuration
defaults read com.networkprinter.preferences | grep GoogleSheets
defaults read com.networkprinter.preferences DiscoveryMode
```

### 🧪 **Enhanced Google Sheets Testing**

#### **1. Test API Access with Multi-Protocol Data**
```bash
# Test Google Sheets API with enhanced validation
SHEET_ID="YOUR_GOOGLE_SHEETS_ID"
API_KEY="YOUR_GOOGLE_SHEETS_API_KEY"
RANGE="Sheet1!A:B"

# Basic API test
echo "Testing Google Sheets API access..."
curl "https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${RANGE}?key=${API_KEY}"

# Enhanced test with error handling and protocol identification
python3 -c "
import json
import urllib.request

sheet_id = '${SHEET_ID}'
api_key = '${API_KEY}'
range_spec = '${RANGE}'

url = f'https://sheets.googleapis.com/v4/spreadsheets/{sheet_id}/values/{range_spec}?key={api_key}'

try:
    with urllib.request.urlopen(url) as response:
        data = json.load(response)
    
    if 'values' in data:
        values = data['values']
        print(f'✅ Successfully retrieved {len(values)} rows from Google Sheets')
        
        # Count printer types
        smb_count = 0
        ipp_count = 0
        
        for i, row in enumerate(values[1:], 2):  # Skip header row
            if len(row) >= 2:
                printer_name = row[0]
                driver_path = row[1]
                
                # Try to identify protocol based on naming
                if 'CORP-' in printer_name or 'SMB' in printer_name.upper():
                    smb_count += 1
                elif 'IPP' in printer_name.upper() or 'Office-' in printer_name:
                    ipp_count += 1
                
                if i <= 4:  # Show first few entries
                    print(f'  Row {i}: {printer_name} -> {driver_path[:50]}...')
        
        print(f'📊 Estimated printer types:')
        print(f'  SMB printers: {smb_count}')
        print(f'  IPP printers: {ipp_count}')
        print(f'  Other/Unknown: {len(values)-1-smb_count-ipp_count}')
        
    else:
        print('❌ No data found in response')
        
except Exception as e:
    print(f'❌ Google Sheets API test failed: {e}')
"
```

---

## Testing & Validation

### 🔄 **Multi-Protocol Priority Testing**

Test the priority system with both SMB and IPP printers:

#### **1. Enhanced Multi-Source Configuration Test**
```bash
#!/bin/bash
# test_multiprotocol_priority_system.sh
# Test driver mapping priority system for both SMB and IPP

echo "🔍 Testing NetworkPrinter Multi-Protocol Priority System"
echo "======================================================="

# Step 1: Clear all existing configurations
echo "🧹 Clearing existing configurations..."
defaults delete com.networkprinter.preferences PrinterDriverMappings 2>/dev/null
defaults delete com.networkprinter.preferences DriverMappingsURL 2>/dev/null
defaults delete com.networkprinter.preferences GoogleSheetsID 2>/dev/null
defaults delete com.networkprinter.preferences GoogleSheetsAPIKey 2>/dev/null
defaults delete com.networkprinter.preferences DiscoveryMode 2>/dev/null

# Step 2: Configure for multi-protocol support
echo ""
echo "🌐 Setting up multi-protocol discovery..."
defaults write com.networkprinter.preferences DiscoveryMode "Both"
echo "✅ Discovery mode set to Both (SMB + IPP)"

# Step 3: Configure Google Sheets (lowest priority)
echo ""
echo "📊 Setting up Google Sheets (Priority: Lowest)..."
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
echo "✅ Google Sheets configured"

# Step 4: Configure Remote JSON (medium priority)
echo ""
echo "🌐 Setting up Remote JSON (Priority: Medium)..."
defaults write com.networkprinter.preferences DriverMappingsURL "https://your-company.com/multiprotocol-drivers.json"
echo "✅ Remote JSON configured"

# Step 5: Install MDM profile (highest priority)
echo ""
echo "🏢 Installing MDM profile (Priority: Highest)..."
echo "⚠️  Manual step: Install MDM profile with PrinterDriverMappings"
echo "   sudo profiles -I -F /path/to/multiprotocol-profile.mobileconfig"

echo ""
echo "🔍 Testing priority order with multi-protocol support..."
echo "1. Launch NetworkPrinter app"
echo "2. Check Console.app for log messages"
echo "3. Verify which source is used for both SMB and IPP printers"

echo ""
echo "Expected behavior:"
echo "- Discovery mode should show: 'Both SMB & IPP'"
echo "- If MDM profile installed: 'Using MDM managed printer driver mappings'"
echo "- Driver mappings should work for both SMB and IPP printer names"
echo "- Else if JSON URL works: 'Successfully parsed JSON driver mappings'"
echo "- Else if Google Sheets works: 'Successfully fetched Google Sheets driver mappings'"
echo "- Else: 'Loaded bundled default printer driver mappings'"
```

### 🔍 **Enhanced Logging and Debugging**

#### **1. Enable Multi-Protocol Detailed Logging**
```bash
# Enable detailed logging for both protocols
defaults write com.networkprinter.preferences EnableDetailedLogging -bool true
defaults write com.networkprinter.preferences DiscoveryMode "Both"

# View live logs with protocol filtering
log stream --predicate 'subsystem == "com.slc.NetworkPrinter"' --level debug

# View specific service logs
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and (category == "PrinterDiscoveryService" or category == "IPPDiscoveryService")' --level debug

# View driver mapping logs
log show --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PreferencesManager"' --last 1h
```

#### **2. Enhanced Key Log Messages to Monitor**
```bash
# Discovery mode configuration
"Discovery mode: Both SMB & IPP"
"Discovery mode: SMB Only"
"Discovery mode: IPP Only"

# Protocol-specific discovery
"SMB printer discovery started"
"SMB printer discovery completed: X printers found"
"IPP printer discovery started"  
"IPP printer discovery completed: X printers found"

# Driver mapping source priority
"Loading printer driver mappings with priority hierarchy..."

# Source-specific loading
"Using MDM managed printer driver mappings: X mappings"
"Successfully parsed JSON driver mappings: X entries"
"Successfully fetched Google Sheets driver mappings: X entries"
"Loaded bundled default printer driver mappings: X mappings"

# Protocol-specific driver matching
"Found exact driver mapping for 'SMB-PrinterName'"
"Found exact driver mapping for 'IPP-PrinterName'"
"Found fuzzy driver mapping for 'PrinterName' -> 'MappedName'"

# Installation logs
"Preparing to install SMB network printer: PrinterName"
"Preparing to install IPP printer: PrinterName"

# Protocol-specific errors
"SMB printer discovery failed: error"
"IPP printer discovery failed: error"
"Failed to validate IPP printer access: error"
"Failed to validate SMB printer access: error"
```

---

## Troubleshooting

### 🚨 **Enhanced Common Issues**

#### **1. Mixed Protocol Discovery Issues**
**Symptoms:** Only SMB or only IPP printers discovered, not both

**Diagnosis:**
```bash
# Check discovery mode
defaults read com.networkprinter.preferences DiscoveryMode

# Check both server configurations
defaults read com.networkprinter.preferences PrintServerHost
defaults read com.networkprinter.preferences IPPServerHost

# Test both protocols separately
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PrinterDiscoveryService"' &
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "IPPDiscoveryService"' &
```

**Solutions:**
- Verify `DiscoveryMode` is set to "Both"
- Ensure both `PrintServerHost` and `IPPServerHost` are configured
- Check network connectivity to both servers
- Verify authentication settings for both protocols

#### **2. IPP-Specific Issues**
**Symptoms:** `"IPP discovery failed"` or `"IPP connection failed"`

**Diagnosis:**
```bash
# Test IPP server connectivity
ippfind -T 5

# Test specific IPP server
ippfind -T 5 -x echo "Found: {service_uri}" \;

# Check IPP server configuration
defaults read com.networkprinter.preferences IPPServerHost
defaults read com.networkprinter.preferences IPPServerPort
defaults read com.networkprinter.preferences IPPUseSSL
```

**Solutions:**
- Verify IPP server is running and accessible
- Check IPP port configuration (631 for HTTP, 443 for HTTPS)
- Test SSL settings if using secure IPP
- Verify firewall settings allow IPP traffic

#### **3. Protocol-Specific Driver Mapping Issues**
**Symptoms:** Drivers found for SMB but not IPP printers (or vice versa)

**Diagnosis:**
```bash
# Check if mappings include both protocols
defaults read com.networkprinter.preferences PrinterDriverMappings

# Look for protocol-specific naming patterns
log show --predicate 'subsystem == "com.slc.NetworkPrinter" and message contains "driver mapping"' --last 1h | grep -E "(SMB|IPP|CORP-|Office-)"
```

**Solutions:**
- Ensure driver mappings include names for both SMB and IPP printers
- Use consistent naming conventions for easier mapping
- Test with generic driver paths that work for both protocols
- Check that PPD files exist at specified paths

---

## Best Practices

### 🔒 **Enhanced Security for Multi-Protocol Environments**

#### **1. Protocol-Specific Security**
- **SMB Security**: Use domain authentication, secure SMB versions
- **IPP Security**: Enable SSL/TLS (IPPS), use proper certificates
- **Mixed Security**: Apply least-privilege principle for both protocols

#### **2. Network Security**
- Segment SMB and IPP traffic appropriately
- Use VPN for remote access to both protocols
- Monitor both SMB and IPP access logs
- Implement proper firewall rules for ports 445 (SMB) and 631/443 (IPP)

### ⚡ **Multi-Protocol Performance Optimization**

#### **1. Discovery Optimization**
- Use "SMB" or "IPP" mode when only one protocol is needed
- Implement reasonable timeouts for both protocols
- Cache discovered printers across protocols
- Use parallel discovery when discovery mode is "Both"

#### **2. Driver Management Optimization**
- Use unified naming conventions across protocols
- Implement protocol-agnostic driver mappings when possible
- Cache driver mappings locally for both protocols
- Monitor mapping effectiveness across both protocols

### 🔧 **Multi-Protocol Maintenance**

#### **1. Enhanced Monitoring**
```bash
#!/bin/bash
# multiprotocol_maintenance_check.sh
# Regular maintenance check for both SMB and IPP

echo "🔍 NetworkPrinter Multi-Protocol Maintenance Check"
echo "================================================"

# Check both protocol endpoints
smb_server=$(defaults read com.networkprinter.preferences PrintServerHost 2>/dev/null)
ipp_server=$(defaults read com.networkprinter.preferences IPPServerHost 2>/dev/null)

if [ -n "$smb_server" ]; then
    echo "Testing SMB server: $smb_server"
    if nc -z "$smb_server" 445 2>/dev/null; then
        echo "✅ SMB server accessible"
    else
        echo "❌ SMB server not accessible"
    fi
fi

if [ -n "$ipp_server" ]; then
    echo "Testing IPP server: $ipp_server"
    ipp_port=$(defaults read com.networkprinter.preferences IPPServerPort 2>/dev/null)
    ipp_port=${ipp_port:-631}
    
    if nc -z "$ipp_server" "$ipp_port" 2>/dev/null; then
        echo "✅ IPP server accessible on port $ipp_port"
    else
        echo "❌ IPP server not accessible on port $ipp_port"
    fi
fi

# Check recent errors for both protocols
SMB_ERRORS=$(log show --predicate 'subsystem == "com.slc.NetworkPrinter" and messageType == "Error" and message contains "SMB"' --last 24h | wc -l)
IPP_ERRORS=$(log show --predicate 'subsystem == "com.slc.NetworkPrinter" and messageType == "Error" and message contains "IPP"' --last 24h | wc -l)

echo "Errors in last 24h:"
echo "  SMB errors: $SMB_ERRORS"
echo "  IPP errors: $IPP_ERRORS"
```

---

## API Reference

### 🔧 **Enhanced Configuration Keys**

#### **Discovery Configuration**
```bash
# Multi-protocol discovery mode
defaults write com.networkprinter.preferences DiscoveryMode "Both"
# Options: "SMB", "IPP", "Both"

# SMB server configuration
defaults write com.networkprinter.preferences PrintServerHost "smb-server.company.com"
defaults write com.networkprinter.preferences PrintServerPort 445
defaults write com.networkprinter.preferences PrintServerDomain "COMPANY"

# IPP server configuration  
defaults write com.networkprinter.preferences IPPServerHost "ipp-server.company.com"
defaults write com.networkprinter.preferences IPPServerPort 631
defaults write com.networkprinter.preferences IPPUseSSL -bool false
defaults write com.networkprinter.preferences IPPRequireAuthentication -bool true

# Authentication
defaults write com.networkprinter.preferences RequireAuthentication -bool true
defaults write com.networkprinter.preferences DefaultDomain "COMPANY"
```

#### **Enhanced Driver Mapping Settings**
```bash
# Protocol-agnostic driver mappings
defaults write com.networkprinter.preferences PrinterDriverMappings '{
    "CORP-HP-4015-FL1" = "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz";
    "Office-Printer-Main" = "/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz";
    "IPP-CUPS-COLOR-01" = "/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd";
    "Reception-Printer" = "/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz";
}'

# Remote sources (work for both protocols)
defaults write com.networkprinter.preferences DriverMappingsURL "https://your-company.com/multiprotocol-drivers.json"

# Google Sheets (supports both protocols)
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
```

### 📝 **Enhanced Data Formats**

#### **Multi-Protocol JSON Format**
```json
{
  "CORP-HP-4015-FL1": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
  "Office-Printer-Main": "/Library/Printers/PPDs/Contents/Resources/HP Color LaserJet.ppd.gz",
  "IPP-CUPS-COLOR-01": "/Library/Printers/PPDs/Contents/Resources/Generic ColorLaser.ppd",
  "Reception-Printer": "/Library/Printers/PPDs/Contents/Resources/Canon Color ImageCLASS.ppd.gz"
}
```

#### **Enhanced Google Sheets Format**
```
Column A: Printer Name (works for both SMB and IPP)
Column B: Driver Path (same PPD files work for both)
Column C: Protocol (optional: SMB, IPP, Both)
Column D: Notes (optional)
```

### 🔍 **Enhanced Logging Categories**

#### **Multi-Protocol Categories**
- **PrinterDiscoveryService**: SMB printer discovery and validation
- **IPPDiscoveryService**: IPP printer discovery and validation
- **PrinterInstallationService**: Installation for both protocols
- **PreferencesManager**: Configuration and driver mapping for both protocols

---

## Support

### 📞 **Enhanced Getting Help**

#### **Multi-Protocol Troubleshooting Steps**
1. Check discovery mode configuration
2. Verify both SMB and IPP server connectivity
3. Test individual protocol discovery
4. Validate driver mappings for both protocols
5. Review protocol-specific logs
6. Test authentication for each protocol

#### **Enhanced Common Commands**
```bash
# View protocol-specific logs
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PrinterDiscoveryService"' --level debug
log stream --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "IPPDiscoveryService"' --level debug

# Check multi-protocol configuration
defaults read com.networkprinter.preferences | grep -E "(DiscoveryMode|PrintServerHost|IPPServerHost)"

# Reset to default multi-protocol configuration
defaults delete com.networkprinter.preferences
defaults write com.networkprinter.preferences DiscoveryMode "Both"
```

---

**This comprehensive enhanced guide now provides complete support for both SMB/CIFS and IPP/IPPS printer protocols while maintaining the same unified driver mapping system across all printer types and deployment methods.**
