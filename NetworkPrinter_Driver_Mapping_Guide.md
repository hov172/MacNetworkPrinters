# NetworkPrinter Driver Mapping: Complete Setup & Testing Guide

## Table of Contents
1. [Overview](#overview)
2. [Quick Start](#quick-start)
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

### 🌐 **Mid-tier (Remote JSON/CSV Endpoint)**
- **Priority**: Medium
- **Best for**: Mid-size companies with web hosting
- **Benefits**: Dynamic updates, version control, scalable

### 📊 **Small Business (Google Sheets Integration)**
- **Priority**: Lower
- **Best for**: Small organizations using Google Workspace
- **Benefits**: Easy collaboration, no technical setup, visual editing

### 📦 **Fallback (Bundled Defaults)**
- **Priority**: Lowest
- **Best for**: Always available backup
- **Benefits**: Offline operation, no configuration needed

The system uses a **priority hierarchy**: `MDM → Remote JSON → Google Sheets → Bundled Defaults`

> **Scope**: NetworkPrinter targets **SMB/CIFS and USB printers only** — it is not an IPP/AirPrint client. All examples below apply to SMB print-server queues.

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
- Everything below still works as documented — it now co-exists with automatic detection.

---

## Quick Start

### 🚀 **5-Minute Setup (Google Sheets)**

1. **Create Google Sheets spreadsheet**:
```
Column A: Printer Name    | Column B: Driver Path
HP-LaserJet-Pro-4015n    | /Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz
Canon-iR-ADV-C5535i      | /Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz
```

2. **Get API key** from [Google Cloud Console](https://console.cloud.google.com/)

3. **Configure NetworkPrinter**:
```bash
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_SHEET_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_API_KEY"

**Note:** Name of your sheet at the bottom
 <key>pfm_default</key>
            <string>printers!A:B</string>
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

**For MDM System:**
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
            
            <!-- Primary Driver Mappings -->
            <key>PrinterDriverMappings</key>
            <dict>
                <key>CORP-HP-4015-FL1</key>
                <string>/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz</string>
                <key>CORP-CANON-5535-FL2</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz</string>
                <key>CORP-XEROX-7835-FL3</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz</string>
                <key>CORP-BROTHER-L2350-FL4</key>
                <string>/Library/Printers/PPDs/Contents/Resources/Brother HL-L2350DW.ppd.gz</string>
            </dict>
            
            <!-- Optional: Fallback to remote source -->
            <key>DriverMappingsURL</key>
            <string>https://your-company.com/printer-drivers.json</string>
            
            <!-- Print Server Configuration -->
            <key>PrintServerHost</key>
            <string>printserver.company.com</string>
            <key>PrintServerPort</key>
            <integer>445</integer>
            <key>PrintServerDomain</key>
            <string>COMPANY</string>
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
            
            <!-- Logging -->
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

# Check specific preferences
defaults read com.networkprinter.preferences PrinterDriverMappings
```

#### **2. Test in NetworkPrinter App**
1. Open NetworkPrinter app
2. Check Console.app for logs: `NetworkPrinter` subsystem
3. Look for: `"Using MDM managed printer driver mappings"`
4. Verify driver suggestions match your mappings

#### **3. Validation Script**
```bash
#!/bin/bash
# validate_mdm_config.sh
# Test MDM driver mappings

echo "🔍 Testing MDM Configuration..."
echo "================================"

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
    
else
    echo "❌ No driver mappings found"
    echo "💡 Check profile payload structure"
fi

echo ""
echo "🔍 Checking other preferences..."

# Check other key preferences
PRINT_SERVER=$(defaults read com.networkprinter.preferences PrintServerHost 2>/dev/null)
if [ -n "$PRINT_SERVER" ]; then
    echo "✅ Print server configured: $PRINT_SERVER"
else
    echo "⚠️  Print server not configured"
fi

echo ""
echo "✅ MDM validation complete"
```

**Usage:**
```bash
chmod +x validate_mdm_config.sh
./validate_mdm_config.sh
```

---

## Option 2: Remote JSON/CSV Endpoint

### 📋 **Prerequisites**
- Web server with HTTPS support
- JSON/CSV file hosting capability
- Basic web development knowledge

### 🔧 **Setup Steps**

#### **1. Create JSON Endpoint**

**Simple JSON Format (Recommended):**
```json
{
  "HP-LaserJet-Pro-4015n": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
  "Canon-iR-ADV-C5535i": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
  "Xerox-WorkCentre-7835": "/Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz",
  "Brother-HL-L2350DW": "/Library/Printers/PPDs/Contents/Resources/Brother HL-L2350DW.ppd.gz",
  "Epson-WorkForce-Pro-WF-4730": "/Library/Printers/PPDs/Contents/Resources/Epson WorkForce Pro WF-4730.ppd.gz"
}
```

**Extended JSON Format (With Metadata):**
```json
{
  "version": "2024.01.15",
  "last_updated": "2024-01-15T10:30:00Z",
  "mappings": {
    "HP-LaserJet-Pro-4015n": {
      "driver_path": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
      "display_name": "HP LaserJet Pro 4015n",
      "notes": "Tested with macOS 14+",
      "features": ["duplex", "color"],
      "last_updated": "2024-01-15"
    },
    "Canon-iR-ADV-C5535i": {
      "driver_path": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
      "display_name": "Canon iR-ADV C5535i",
      "notes": "Enterprise multifunction printer",
      "features": ["duplex", "color", "stapling", "scanning"],
      "last_updated": "2024-01-15"
    }
  }
}
```

**CSV Format Option:**
```csv
Printer Name,Driver Path,Display Name,Notes,Features
HP-LaserJet-Pro-4015n,/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz,HP LaserJet Pro 4015n,Tested with macOS 14+,duplex;color
Canon-iR-ADV-C5535i,/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz,Canon iR-ADV C5535i,Enterprise MFP,duplex;color;stapling;scanning
Xerox-WorkCentre-7835,/Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz,Xerox WorkCentre 7835,High-volume printing,duplex;color
```

#### **2. Deploy Configuration**

**Option A: MDM (Recommended)**
```xml
<key>DriverMappingsURL</key>
<string>https://your-company.com/printer-drivers.json</string>
```

**Option B: Manual Configuration**
```bash
# Set the URL in user defaults
defaults write com.networkprinter.preferences DriverMappingsURL "https://your-company.com/printer-drivers.json"

# Verify configuration
defaults read com.networkprinter.preferences DriverMappingsURL
```

#### **3. Server Setup Examples**

**Apache Configuration (.htaccess):**
```apache
# Enable CORS for NetworkPrinter app
Header set Access-Control-Allow-Origin "*"
Header set Access-Control-Allow-Methods "GET, OPTIONS"
Header set Access-Control-Allow-Headers "Content-Type, Authorization"
Header set Access-Control-Max-Age "86400"

# Set proper content types
<FilesMatch "\.json$">
    Header set Content-Type "application/json; charset=utf-8"
</FilesMatch>

<FilesMatch "\.csv$">
    Header set Content-Type "text/csv; charset=utf-8"
</FilesMatch>

# Enable compression
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE application/json
    AddOutputFilterByType DEFLATE text/csv
</IfModule>

# Cache control
<FilesMatch "\.(json|csv)$">
    ExpiresActive On
    ExpiresDefault "access plus 1 hour"
    Header set Cache-Control "public, max-age=3600"
</FilesMatch>
```

**Nginx Configuration:**
```nginx
server {
    listen 443 ssl;
    server_name your-company.com;
    
    location /printer-drivers {
        # CORS headers
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
        add_header Access-Control-Max-Age 86400 always;
        
        # Content type
        location ~ \.json$ {
            add_header Content-Type "application/json; charset=utf-8";
        }
        
        location ~ \.csv$ {
            add_header Content-Type "text/csv; charset=utf-8";
        }
        
        # Compression
        gzip on;
        gzip_types application/json text/csv;
        
        # Cache control
        expires 1h;
        add_header Cache-Control "public, max-age=3600";
    }
}
```

**Simple Python Flask Server:**
```python
from flask import Flask, jsonify, send_file
from flask_cors import CORS
import json

app = Flask(__name__)
CORS(app)

@app.route('/printer-drivers.json')
def get_printer_drivers():
    drivers = {
        "HP-LaserJet-Pro-4015n": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
        "Canon-iR-ADV-C5535i": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz",
        "Xerox-WorkCentre-7835": "/Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz"
    }
    return jsonify(drivers)

@app.route('/printer-drivers.csv')
def get_printer_drivers_csv():
    return send_file('printer-drivers.csv', mimetype='text/csv')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 🧪 **Testing Remote JSON/CSV**

#### **1. Test URL Accessibility**
```bash
# Test JSON endpoint
curl -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0" \
     https://your-company.com/printer-drivers.json

# Test with verbose output
curl -v -H "Accept: application/json" \
     https://your-company.com/printer-drivers.json

# Test CSV endpoint
curl -H "Accept: text/csv" \
     https://your-company.com/printer-drivers.csv
```

#### **2. Validate JSON Format**
```bash
# Test JSON validity with Python
python3 -c "
import json
import urllib.request

url = 'https://your-company.com/printer-drivers.json'
try:
    with urllib.request.urlopen(url) as response:
        data = json.load(response)
    print('✅ Valid JSON format')
    print(f'📊 Found {len(data)} mappings')
    for key in list(data.keys())[:3]:
        print(f'  {key}: {data[key]}')
except Exception as e:
    print(f'❌ JSON validation failed: {e}')
"

# Or use jq if available
curl -s https://your-company.com/printer-drivers.json | jq '.'
```

#### **3. NetworkPrinter App Testing**
1. Configure the URL in app settings
2. Launch NetworkPrinter app
3. Check Console.app for: `"Successfully parsed JSON driver mappings"`
4. Verify mappings load correctly
5. Test with/without internet connectivity

---

## Option 3: Google Sheets Integration

### 📋 **Prerequisites**
- Google Account with Google Sheets access
- Google Cloud Console project
- Google Sheets API enabled
- Basic understanding of Google APIs

### 🔧 **Setup Steps**

#### **1. Create Google Sheets Spreadsheet**

**Step-by-step:**
1. Go to [Google Sheets](https://sheets.google.com/)
2. Create a new spreadsheet
3. Name it "NetworkPrinter Driver Mappings"
4. Set up the structure:

**Spreadsheet Structure:**
```
     A                         B
1    Printer Name              Driver Path
2    HP-LaserJet-Pro-4015n     /Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz
3    Canon-iR-ADV-C5535i       /Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz
4    Xerox-WorkCentre-7835     /Library/Printers/PPDs/Contents/Resources/Xerox WorkCentre 7835.ppd.gz
5    Brother-HL-L2350DW        /Library/Printers/PPDs/Contents/Resources/Brother HL-L2350DW.ppd.gz
6    Epson-WorkForce-Pro-WF-4730 /Library/Printers/PPDs/Contents/Resources/Epson WorkForce Pro WF-4730.ppd.gz
```

#### **2. Set Up Google Sheets API**

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

#### **3. Get Spreadsheet ID**

From your Google Sheets URL:
```
https://docs.google.com/spreadsheets/d/YOUR_GOOGLE_SHEETS_ID/edit#gid=0
                                   ^--- This is your Spreadsheet ID ---^
```

#### **4. Configure NetworkPrinter App**

**Option A: MDM Configuration**
```xml
<key>GoogleSheetsID</key>
<string>YOUR_GOOGLE_SHEETS_ID</string>
<key>GoogleSheetsAPIKey</key>
<string>YOUR_GOOGLE_SHEETS_API_KEY</string>
<key>GoogleSheetsRange</key>
<string>Sheet1!A:B</string>
```

**Option B: Manual Configuration**
```bash
# Configure via defaults
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"

# Verify configuration
defaults read com.networkprinter.preferences | grep GoogleSheets
```

### 🧪 **Testing Google Sheets Integration**

#### **1. Test API Access**
```bash
# Test Google Sheets API
SHEET_ID="YOUR_GOOGLE_SHEETS_ID"
API_KEY="YOUR_GOOGLE_SHEETS_API_KEY"
RANGE="Sheet1!A:B"

# Basic API test
curl "https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${RANGE}?key=${API_KEY}"

# Test with error handling
curl -v "https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${RANGE}?key=${API_KEY}" \
     -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0"
```

---

## Testing & Validation

### 🔄 **Priority Testing**

Test the priority system by configuring multiple sources:

#### **1. Multi-Source Configuration Test**
```bash
#!/bin/bash
# test_priority_system.sh
# Test driver mapping priority system

echo "🔍 Testing NetworkPrinter Priority System"
echo "========================================"

# Step 1: Clear all existing configurations
echo "🧹 Clearing existing configurations..."
defaults delete com.networkprinter.preferences PrinterDriverMappings 2>/dev/null
defaults delete com.networkprinter.preferences DriverMappingsURL 2>/dev/null
defaults delete com.networkprinter.preferences GoogleSheetsID 2>/dev/null
defaults delete com.networkprinter.preferences GoogleSheetsAPIKey 2>/dev/null

# Step 2: Configure Google Sheets (lowest priority)
echo ""
echo "📊 Setting up Google Sheets (Priority: Lowest)..."
defaults write com.networkprinter.preferences GoogleSheetsID "test-sheet-id"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "test-api-key"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
echo "✅ Google Sheets configured"

# Step 3: Configure Remote JSON (medium priority)
echo ""
echo "🌐 Setting up Remote JSON (Priority: Medium)..."
defaults write com.networkprinter.preferences DriverMappingsURL "https://your-company.com/test-drivers.json"
echo "✅ Remote JSON configured"

# Step 4: Install MDM profile (highest priority)
echo ""
echo "🏢 Installing MDM profile (Priority: Highest)..."
echo "⚠️  Manual step: Install MDM profile with PrinterDriverMappings"
echo "   sudo profiles -I -F /path/to/test-profile.mobileconfig"

# Step 5: Test priority order
echo ""
echo "🔍 Testing priority order..."
echo "1. Launch NetworkPrinter app"
echo "2. Check Console.app for log messages"
echo "3. Verify which source is used"

echo ""
echo "Expected behavior:"
echo "- If MDM profile installed: 'Using MDM managed printer driver mappings'"
echo "- Else if JSON URL works: 'Successfully parsed JSON driver mappings'"
echo "- Else if Google Sheets works: 'Successfully fetched Google Sheets driver mappings'"
echo "- Else: 'Loaded bundled default printer driver mappings'"
```

### 🔍 **Logging and Debugging**

#### **1. Enable Detailed Logging**
```bash
# Enable detailed logging
defaults write com.networkprinter.preferences EnableDetailedLogging -bool true

# View live logs
log stream --predicate 'subsystem == "com.slc.NetworkPrinter"' --level debug

# View specific logs
log show --predicate 'subsystem == "com.slc.NetworkPrinter" and category == "PreferencesManager"' --last 1h
```

#### **2. Key Log Messages to Monitor**
```bash
# Driver mapping source priority
"Loading printer driver mappings with priority hierarchy..."

# MDM configuration
"Using MDM managed printer driver mappings: X mappings"

# Remote JSON
"Successfully parsed JSON driver mappings: X entries"

# Google Sheets
"Successfully fetched Google Sheets driver mappings: X entries"

# Bundled defaults
"Loaded bundled default printer driver mappings: X mappings"

# Driver matching
"Found exact driver mapping for 'PrinterName'"
"Found fuzzy driver mapping for 'PrinterName' -> 'MappedName'"
"No driver mapping found for printer: PrinterName"

# Errors
"Failed to fetch driver mappings from URL: error"
"Failed to fetch driver mappings from Google Sheets: error"
"Invalid JSON format in driver mappings"
```

---

## Troubleshooting

### 🚨 **Common Issues**

#### **1. MDM Profile Not Loading**
**Symptoms:** App uses defaults instead of MDM settings

**Diagnosis:**
```bash
# Check profile installation
sudo profiles -C -v | grep com.networkprinter.preferences

# Check profile contents
defaults read com.networkprinter.preferences
```

**Solutions:**
- Verify profile installation: `sudo profiles -C -v`
- Check profile payload identifier matches `com.networkprinter.preferences`
- Restart app after profile installation
- Check Console.app for MDM-related errors
- Verify profile is signed (if required)

#### **2. JSON Endpoint Unreachable**
**Symptoms:** `"Failed to fetch driver mappings from URL"`

**Diagnosis:**
```bash
# Test URL accessibility
curl -v https://your-url.com/mappings.json

# Test with specific headers
curl -H "Accept: application/json" \
     -H "User-Agent: NetworkPrinter/1.0" \
     https://your-url.com/mappings.json

# Check DNS resolution
nslookup your-url.com
```

**Solutions:**
- Test URL accessibility: `curl -v https://your-url.com/mappings.json`
- Check CORS headers
- Verify JSON format validity
- Test with different network conditions
- Check firewall/proxy settings

#### **3. Google Sheets API Errors**
**Symptoms:** `"Failed to fetch driver mappings from Google Sheets"`

**Diagnosis:**
```bash
# Test API key
SHEET_ID="your-sheet-id"
API_KEY="your-api-key"
curl "https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/Sheet1!A:B?key=${API_KEY}"

# Check API quotas
# Visit Google Cloud Console → APIs & Services → Quotas
```

**Solutions:**
- Verify API key permissions
- Check spreadsheet sharing settings (public or shared with service account)
- Test API endpoint manually
- Check quota limits in Google Cloud Console
- Verify API key restrictions

#### **4. Driver Suggestions Not Working**
**Symptoms:** Generic driver always suggested

**Diagnosis:**
```bash
# Check PPD files exist
ls -la "/Library/Printers/PPDs/Contents/Resources/" | grep -i "hp\|canon\|xerox"

# Check printer name matching
defaults read com.networkprinter.preferences PrinterDriverMappings | grep "PrinterName"

# Check recent logs
log show --predicate 'subsystem == "com.slc.NetworkPrinter" and message contains "driver mapping"' --last 1h
```

**Solutions:**
- Verify PPD file paths exist: `ls -la "/Library/Printers/PPDs/Contents/Resources/"`
- Check printer name matching (case-sensitive)
- Review fuzzy matching logic
- Test with exact printer names from mappings

---

## Best Practices

### 🔒 **Security Considerations**

#### **1. API Key Security**
- Use API key restrictions in Google Cloud Console
- Limit API keys to specific APIs only
- Use IP restrictions when possible
- Rotate API keys regularly
- Store API keys securely (MDM preferred)

#### **2. Network Security**
- Use HTTPS for all remote endpoints
- Implement proper SSL certificate validation
- Use VPN or private networks for internal endpoints
- Implement rate limiting on endpoints
- Monitor API access logs

#### **3. Configuration Security**
- Use MDM for sensitive configurations
- Avoid hardcoding credentials in app
- Use managed preferences for enterprise deployments
- Implement proper error handling to avoid information leakage

### ⚡ **Performance Optimization**

#### **1. Caching Strategy**
- Cache remote mappings locally
- Use appropriate cache expiration times
- Implement cache invalidation mechanisms
- Use checksums to detect changes
- Fallback to cached data when remote unavailable

#### **2. Network Optimization**
- Implement reasonable timeout values (30-60 seconds)
- Use compression for large mapping files
- Implement exponential backoff for retries
- Use connection pooling for multiple requests
- Monitor network performance

### 🔧 **Maintenance**

#### **1. Regular Monitoring**
```bash
#!/bin/bash
# maintenance_check.sh
# Regular maintenance check for NetworkPrinter

echo "🔍 NetworkPrinter Maintenance Check"
echo "=================================="

# Check endpoint health
endpoints=(
    "https://your-company.com/printer-drivers.json"
    "https://sheets.googleapis.com/v4/spreadsheets/YOUR_SHEET_ID/values/Sheet1!A:B?key=YOUR_API_KEY"
)

for endpoint in "${endpoints[@]}"; do
    echo "Testing: $endpoint"
    
    if curl -s -f "$endpoint" > /dev/null; then
        echo "✅ Accessible"
    else
        echo "❌ Not accessible"
        # Send alert here
    fi
done

# Check log file sizes
LOG_SIZE=$(du -h ~/Library/Application\ Support/com.slc.NetworkPrinter/Logs/NetworkPrinter_debug.log 2>/dev/null | cut -f1)
if [ -n "$LOG_SIZE" ]; then
    echo "Log size: $LOG_SIZE"
fi

# Check for errors in recent logs
ERROR_COUNT=$(log show --predicate 'subsystem == "com.slc.NetworkPrinter" and messageType == "Error"' --last 24h | wc -l)
echo "Errors in last 24h: $ERROR_COUNT"
```

#### **2. Version Control**
- Keep mapping configurations in version control
- Document changes with commit messages
- Use branches for testing new configurations
- Tag releases for easy rollback
- Maintain changelog

### 📊 **Deployment Strategy**

#### **1. Phased Rollout**
1. **Pilot Phase**: Test with small group (10-20 users)
2. **Beta Phase**: Expand to larger group (100-200 users)
3. **Production Phase**: Full deployment
4. **Monitoring Phase**: Continuous monitoring and optimization

#### **2. Testing Strategy**
- Test all three mapping methods
- Test fallback behavior
- Test with different network conditions
- Test with various printer models
- Test error handling scenarios

---

## API Reference

### 🔧 **Configuration Keys**

#### **Core Settings**
```bash
# Print Server Configuration
defaults write com.networkprinter.preferences PrintServerHost "printserver.company.com"
defaults write com.networkprinter.preferences PrintServerPort 445
defaults write com.networkprinter.preferences PrintServerDomain "COMPANY"
defaults write com.networkprinter.preferences RequireAuthentication -bool true
defaults write com.networkprinter.preferences DefaultDomain "COMPANY"

# User Permissions
defaults write com.networkprinter.preferences AllowUserServerChange -bool false
defaults write com.networkprinter.preferences AllowUserInstall -bool true
defaults write com.networkprinter.preferences AllowUserUninstall -bool true

# Auto-Installation
defaults write com.networkprinter.preferences AutoInstallPrinters -bool false
defaults write com.networkprinter.preferences AutoInstallDefaultPrinter "DefaultPrinterName"

# Logging
defaults write com.networkprinter.preferences EnableDetailedLogging -bool true
defaults write com.networkprinter.preferences AllowFallbackWithGenericSuggestion -bool false
```

#### **Driver Mapping Settings**
```bash
# MDM Driver Mappings (dictionary format)
defaults write com.networkprinter.preferences PrinterDriverMappings '{
    "HP-LaserJet-Pro-4015n" = "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz";
    "Canon-iR-ADV-C5535i" = "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz";
}'

# Remote JSON/CSV URL
defaults write com.networkprinter.preferences DriverMappingsURL "https://your-company.com/printer-drivers.json"

# Google Sheets Configuration
defaults write com.networkprinter.preferences GoogleSheetsID "YOUR_GOOGLE_SHEETS_ID"
defaults write com.networkprinter.preferences GoogleSheetsAPIKey "YOUR_GOOGLE_SHEETS_API_KEY"
defaults write com.networkprinter.preferences GoogleSheetsRange "Sheet1!A:B"
```

### 📝 **Data Formats**

#### **JSON Format**
```json
{
  "PrinterShareName": "/Library/Printers/PPDs/Contents/Resources/DriverName.ppd.gz",
  "HP-LaserJet-Pro-4015n": "/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz",
  "Canon-iR-ADV-C5535i": "/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz"
}
```

#### **CSV Format**
```csv
Printer Name,Driver Path
HP-LaserJet-Pro-4015n,/Library/Printers/PPDs/Contents/Resources/HP LaserJet Pro 4015n.ppd.gz
Canon-iR-ADV-C5535i,/Library/Printers/PPDs/Contents/Resources/Canon iR-ADV C5535i PS.ppd.gz
```

#### **Google Sheets Format**
```
Column A: Printer Name
Column B: Driver Path
```

### 🔍 **Logging Categories**

#### **PreferencesManager**
- Driver mapping source detection
- Configuration loading
- MDM profile changes
- Remote source fetching

#### **PrinterManager**
- Driver suggestion logic
- Printer installation
- PPD file access
- SNMP detection

#### **NetworkPrinter**
- Application startup
- UI interactions
- Error handling
- Performance metrics

### 📊 **Status Codes**

#### **HTTP Status Codes**
- `200`: Success
- `403`: Forbidden (API key issues)
- `404`: Not found (invalid URL/Sheet ID)
- `429`: Too many requests (rate limiting)
- `500`: Server error

#### **Application Status**
- `0`: Success
- `1`: Configuration error
- `2`: Network error
- `3`: Authentication error
- `4`: Driver not found

---

## Support

### 📞 **Getting Help**

#### **Documentation**
- Review this guide thoroughly
- Check the bundled app documentation
- Review Console.app logs
- Check system requirements

#### **Troubleshooting Steps**
1. Run the validation script
2. Check recent logs
3. Test individual components
4. Verify network connectivity
5. Check MDM profile status

#### **Common Commands**
```bash
# View live logs
log stream --predicate 'subsystem == "com.slc.NetworkPrinter"' --level debug

# Check configuration
defaults read com.networkprinter.preferences

# Validate configuration
./validate_networkprinter_config.sh

# Reset configuration
defaults delete com.networkprinter.preferences
```

### 🛠️ **Advanced Configuration**

#### **Custom Validation Scripts**
Create custom validation scripts for your specific environment:
```bash
#!/bin/bash
# custom_validation.sh
# Custom validation for your organization

# Add organization-specific checks
# - Test specific printer models
# - Validate against your printer inventory
# - Check organizational policies
# - Verify compliance requirements
```

#### **Monitoring Integration**
Integrate with your existing monitoring systems:
```bash
#!/bin/bash
# monitoring_integration.sh
# Integration with organizational monitoring

# Send metrics to monitoring system
# - Endpoint health status
# - Driver mapping success rates
# - Application performance metrics
# - Error rates and types
```

---

**This comprehensive guide provides everything needed to implement, test, and maintain all three driver mapping options for the NetworkPrinter app across any enterprise environment.**
