---
name: axiom-macos-distribution
description: Use when distributing macOS apps including notarization, Developer ID signing, Mac App Store vs direct distribution, Hardened Runtime, Gatekeeper, and Sparkle auto-updates
user-invocable: true
skill_type: discipline
version: 1.0.0
apple_platforms: macOS 10.15+
---

# macOS App Distribution

Comprehensive guide to distributing macOS apps, covering notarization, code signing, Mac App Store, and direct distribution.

## When to Use This Skill

- Preparing app for distribution outside Mac App Store
- Understanding notarization requirements
- Configuring Developer ID signing
- Choosing between Mac App Store and direct distribution
- Setting up Hardened Runtime entitlements
- Implementing auto-updates with Sparkle
- Troubleshooting Gatekeeper issues

## Distribution Methods Overview

| Method | Review | Sandbox | Updates | Revenue Split |
|--------|--------|---------|---------|---------------|
| Mac App Store | Yes | Required | Automatic | 70/30 (85/15 after year 1) |
| Developer ID | No | Optional | You handle | 100% yours |
| Ad Hoc | No | N/A | Manual | N/A |

---

## Pattern 1: Notarization (Required for Developer ID)

### What is Notarization?

Apple's automated security check for apps distributed outside the Mac App Store. Required since macOS 10.15 for apps to open without Gatekeeper warnings.

### Requirements for Notarization

1. **Developer ID certificate** (not Mac App Store certificate)
2. **Hardened Runtime enabled**
3. **Code signed** with timestamp
4. **No unsigned code** (all frameworks, plugins signed)
5. **Supported minimum OS** (macOS 10.9+)

### Notarization Workflow

```bash
# 1. Archive and export from Xcode
# Product → Archive → Distribute App → Developer ID → Upload

# OR command line workflow:

# 2. Sign the app
codesign --deep --force --verify --verbose \
    --sign "Developer ID Application: Your Name (TEAMID)" \
    --options runtime \
    --timestamp \
    MyApp.app

# 3. Create ZIP for notarization
ditto -c -k --keepParent MyApp.app MyApp.zip

# 4. Submit for notarization
xcrun notarytool submit MyApp.zip \
    --apple-id "your@email.com" \
    --team-id "TEAMID" \
    --password "@keychain:AC_PASSWORD" \
    --wait

# 5. Staple the ticket (allows offline verification)
xcrun stapler staple MyApp.app

# 6. Verify
spctl --assess --verbose MyApp.app
```

### Storing Credentials Securely

```bash
# Store App Store Connect password in keychain
xcrun notarytool store-credentials "AC_PASSWORD" \
    --apple-id "your@email.com" \
    --team-id "TEAMID" \
    --password "app-specific-password"

# Use stored credentials
xcrun notarytool submit MyApp.zip \
    --keychain-profile "AC_PASSWORD" \
    --wait
```

### Checking Notarization Status

```bash
# Check status
xcrun notarytool info <submission-id> \
    --keychain-profile "AC_PASSWORD"

# Get detailed log (for failures)
xcrun notarytool log <submission-id> \
    --keychain-profile "AC_PASSWORD"
```

---

## Pattern 2: Hardened Runtime

### Required Entitlements

Add to your `.entitlements` file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
    <!-- Required for most apps -->
    <key>com.apple.security.app-sandbox</key>
    <false/>  <!-- or true if using sandbox -->

    <!-- Allow JIT (for JavaScript engines, etc.) -->
    <key>com.apple.security.cs.allow-jit</key>
    <true/>

    <!-- Allow unsigned executable memory -->
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>

    <!-- Allow DYLD environment variables -->
    <key>com.apple.security.cs.allow-dyld-environment-variables</key>
    <true/>

    <!-- Disable library validation (load unsigned plugins) -->
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>

    <!-- Audio input -->
    <key>com.apple.security.device.audio-input</key>
    <true/>

    <!-- Camera -->
    <key>com.apple.security.device.camera</key>
    <true/>

    <!-- Location -->
    <key>com.apple.security.personal-information.location</key>
    <true/>

    <!-- Apple Events (automation) -->
    <key>com.apple.security.automation.apple-events</key>
    <true/>
</dict>
</plist>
```

### Hardened Runtime Decision Tree

```
What does your app need?

├─ Load plugins/bundles at runtime?
│  └─ cs.disable-library-validation
│
├─ Execute JIT-compiled code?
│  └─ cs.allow-jit
│
├─ Use dynamic linker features?
│  └─ cs.allow-dyld-environment-variables
│
├─ Access camera/microphone?
│  └─ device.camera / device.audio-input
│
├─ Send Apple Events (automation)?
│  └─ automation.apple-events
│
└─ None of the above?
   └─ No additional entitlements needed
```

### Enable in Xcode

1. Select target → Signing & Capabilities
2. Add "Hardened Runtime" capability
3. Check boxes for needed entitlements

---

## Pattern 3: Code Signing

### Developer ID Certificate Setup

1. **Apple Developer Program** membership required ($99/year)
2. **Create certificate** in Certificates, Identifiers & Profiles
3. **Download and install** in Keychain Access

### Signing Commands

```bash
# Sign app bundle
codesign --deep --force --verify --verbose \
    --sign "Developer ID Application: Your Name (TEAMID)" \
    --options runtime \
    --entitlements MyApp.entitlements \
    --timestamp \
    MyApp.app

# Sign a disk image
codesign --sign "Developer ID Application: Your Name (TEAMID)" \
    --timestamp \
    MyApp.dmg

# Sign a pkg installer
productsign --sign "Developer ID Installer: Your Name (TEAMID)" \
    --timestamp \
    MyApp-unsigned.pkg \
    MyApp.pkg

# Verify signature
codesign --verify --verbose MyApp.app
spctl --assess --verbose MyApp.app
```

### Signing Frameworks and Plugins

```bash
# Sign frameworks first (inside out)
codesign --force --verify --verbose \
    --sign "Developer ID Application: Your Name (TEAMID)" \
    --options runtime \
    --timestamp \
    MyApp.app/Contents/Frameworks/MyFramework.framework

# Then sign the app
codesign --force --verify --verbose \
    --sign "Developer ID Application: Your Name (TEAMID)" \
    --options runtime \
    --timestamp \
    MyApp.app
```

---

## Pattern 4: Mac App Store Distribution

### Key Differences from Developer ID

| Aspect | Mac App Store | Developer ID |
|--------|--------------|--------------|
| Certificate | Mac App Distribution | Developer ID Application |
| Sandbox | Required | Optional |
| Review | Yes (1-7 days) | No (automated notarization) |
| Entitlements | Restricted | More flexible |
| Updates | Automatic via App Store | You implement |
| Purchase/IAP | App Store handles | You implement |

### Required Entitlements for App Store

```xml
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>

    <!-- Add only what you need -->
    <key>com.apple.security.network.client</key>
    <true/>

    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
</dict>
```

### Submission Process

1. **Archive** in Xcode (Product → Archive)
2. **Validate** (checks for common issues)
3. **Distribute App** → App Store Connect
4. **Upload** to App Store Connect
5. **Submit for Review** in App Store Connect

---

## Pattern 5: Direct Distribution (DMG/PKG)

### Creating a DMG

```bash
# Create DMG with app and Applications symlink
hdiutil create -volname "MyApp" \
    -srcfolder MyApp.app \
    -ov -format UDZO \
    MyApp.dmg

# Or use create-dmg for fancy DMGs
# brew install create-dmg
create-dmg \
    --volname "MyApp" \
    --window-pos 200 120 \
    --window-size 600 400 \
    --icon-size 100 \
    --icon "MyApp.app" 150 190 \
    --app-drop-link 450 185 \
    --hide-extension "MyApp.app" \
    "MyApp.dmg" \
    "MyApp.app"
```

### Creating a PKG Installer

```bash
# Build component package
pkgbuild --root ./build/MyApp.app \
    --identifier com.mycompany.myapp \
    --version 1.0.0 \
    --install-location /Applications/MyApp.app \
    MyApp-component.pkg

# Build product archive
productbuild --package MyApp-component.pkg \
    --sign "Developer ID Installer: Your Name (TEAMID)" \
    MyApp.pkg

# OR with distribution.xml for customization
productbuild --distribution distribution.xml \
    --resources Resources \
    --sign "Developer ID Installer: Your Name (TEAMID)" \
    MyApp.pkg
```

### distribution.xml Example

```xml
<?xml version="1.0" encoding="utf-8"?>
<installer-gui-script minSpecVersion="2">
    <title>MyApp</title>
    <welcome file="welcome.html"/>
    <license file="license.html"/>
    <conclusion file="conclusion.html"/>

    <options customize="never" require-scripts="false"/>

    <choices-outline>
        <line choice="default"/>
    </choices-outline>

    <choice id="default">
        <pkg-ref id="com.mycompany.myapp"/>
    </choice>

    <pkg-ref id="com.mycompany.myapp" version="1.0.0">
        MyApp-component.pkg
    </pkg-ref>
</installer-gui-script>
```

---

## Pattern 6: Auto-Updates with Sparkle

### Sparkle Setup

1. **Add Sparkle** via SPM: `https://github.com/sparkle-project/Sparkle`
2. **Configure Info.plist**

```xml
<key>SUFeedURL</key>
<string>https://yoursite.com/appcast.xml</string>

<key>SUPublicEDKey</key>
<string>your-public-key</string>

<key>SUEnableAutomaticChecks</key>
<true/>
```

### SwiftUI Integration

```swift
import Sparkle

class UpdaterViewModel: ObservableObject {
    private let updaterController: SPUStandardUpdaterController

    init() {
        updaterController = SPUStandardUpdaterController(
            startingUpdater: true,
            updaterDelegate: nil,
            userDriverDelegate: nil
        )
    }

    func checkForUpdates() {
        updaterController.checkForUpdates(nil)
    }
}

struct SettingsView: View {
    @StateObject private var updater = UpdaterViewModel()

    var body: some View {
        Button("Check for Updates...") {
            updater.checkForUpdates()
        }
    }
}
```

### Creating appcast.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<rss version="2.0" xmlns:sparkle="http://www.andymatuschak.org/xml-namespaces/sparkle">
    <channel>
        <title>MyApp Updates</title>
        <item>
            <title>Version 1.1.0</title>
            <pubDate>Mon, 01 Jan 2024 12:00:00 +0000</pubDate>
            <sparkle:version>1.1.0</sparkle:version>
            <sparkle:shortVersionString>1.1.0</sparkle:shortVersionString>
            <sparkle:minimumSystemVersion>10.15</sparkle:minimumSystemVersion>
            <description><![CDATA[
                <h2>What's New</h2>
                <ul>
                    <li>New feature X</li>
                    <li>Bug fixes</li>
                </ul>
            ]]></description>
            <enclosure
                url="https://yoursite.com/MyApp-1.1.0.dmg"
                sparkle:edSignature="signature-here"
                length="12345678"
                type="application/octet-stream"/>
        </item>
    </channel>
</rss>
```

### Generate Update Signature

```bash
# Generate keys (one time)
./bin/generate_keys

# Sign update
./bin/sign_update MyApp-1.1.0.dmg
# Outputs: sparkle:edSignature="..."
```

---

## Common Pitfalls

### Pitfall 1: Notarization Fails - Unsigned Code

```bash
# Find unsigned code
codesign --verify --deep --strict MyApp.app 2>&1 | grep "not signed"

# Common culprits:
# - Third-party frameworks
# - Helper tools
# - Plugins
```

### Pitfall 2: Gatekeeper Rejects After Notarization

```bash
# Check if ticket is stapled
stapler validate MyApp.app

# Re-staple if needed
xcrun stapler staple MyApp.app

# Check extended attributes (quarantine)
xattr -l MyApp.app
# Remove quarantine if testing locally
xattr -d com.apple.quarantine MyApp.app
```

### Pitfall 3: Wrong Certificate Type

```
Error: "MyApp.app" does not have a valid Developer ID signature

Check certificate type:
- Developer ID Application → for apps
- Developer ID Installer → for pkg installers
- Mac App Distribution → for App Store only
```

### Pitfall 4: Missing Timestamp

```bash
# ❌ WRONG - no timestamp (signature expires with cert)
codesign --sign "Developer ID Application: ..." MyApp.app

# ✅ CORRECT - include timestamp
codesign --sign "Developer ID Application: ..." \
    --timestamp \
    MyApp.app
```

---

## Distribution Checklist

### Before Distribution

- [ ] App runs correctly on clean macOS install
- [ ] Hardened Runtime enabled
- [ ] All code signed (app, frameworks, plugins)
- [ ] Notarization passes
- [ ] Ticket stapled
- [ ] `spctl --assess` passes
- [ ] Test on macOS version you claim to support

### For Mac App Store

- [ ] Sandbox enabled
- [ ] All entitlements justified
- [ ] Privacy descriptions in Info.plist
- [ ] Screenshots prepared (all sizes)
- [ ] App Store metadata complete

### For Direct Distribution

- [ ] DMG/PKG signed
- [ ] Auto-update mechanism (Sparkle)
- [ ] License/EULA included
- [ ] Website with download link
- [ ] Release notes published

---

## Resources

**Docs**: /security/notarizing-macos-software-before-distribution, /xcode/notarizing-macos-software-before-distribution, /security/hardened-runtime

**Tools**: Sparkle (https://sparkle-project.org), create-dmg (https://github.com/create-dmg/create-dmg)

**Skills**: axiom-macos-file-handling, axiom-ios-build

---

**Last Updated**: Based on macOS 10.15+ documentation
**Platforms**: macOS 10.15+
