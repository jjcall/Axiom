---
name: axiom-macos-extensions
description: Use when building macOS extensions including Finder Sync, Safari extensions, Share extensions, Quick Look previews, Spotlight importers, and Services menu integration
user-invocable: true
skill_type: reference
version: 1.0.0
apple_platforms: macOS 10.15+
---

# macOS Extensions

Reference guide for building macOS app extensions including Finder Sync, Safari, Share, Quick Look, and more.

## When to Use This Skill

- Building Finder Sync extensions (folder badges, contextual menus)
- Creating Safari web extensions
- Implementing Share extensions on Mac
- Building Quick Look preview and thumbnail extensions
- Creating Spotlight importers for custom file types
- Adding Services menu actions

## Extension Types Overview

| Extension Type | Purpose | Template in Xcode |
|---------------|---------|-------------------|
| Finder Sync | Badges, context menus in Finder | File Provider |
| Safari Web Extension | Browser functionality | Safari Extension |
| Share Extension | Share sheet integration | Share Extension |
| Quick Look Preview | File previews in Finder/Quick Look | Quick Look Preview |
| Quick Look Thumbnail | Thumbnails for custom files | Quick Look Thumbnail |
| Spotlight Importer | Index custom file types | Spotlight Importer |
| Action Extension | Process selected content | Action Extension |

---

## Pattern 1: Finder Sync Extension

### Overview

Finder Sync extensions can:
- Display sync status badges on files/folders
- Add items to Finder contextual menus
- Add toolbar buttons in Finder windows

### Creating the Extension

1. File → New → Target → Finder Sync Extension
2. Configure the extension principal class

### Finder Sync Implementation

```swift
import Cocoa
import FinderSync

class FinderSync: FIFinderSync {
    override init() {
        super.init()

        // Watch these folder(s)
        let finderSyncController = FIFinderSyncController.default()
        finderSyncController.directoryURLs = [
            URL(fileURLWithPath: "/Users/Shared/MyApp/")
        ]

        // Badge images
        FIFinderSyncController.default().setBadgeImage(
            NSImage(named: "synced")!,
            label: "Synced",
            forBadgeIdentifier: "synced"
        )
        FIFinderSyncController.default().setBadgeImage(
            NSImage(named: "syncing")!,
            label: "Syncing",
            forBadgeIdentifier: "syncing"
        )
    }

    // Called when Finder needs badge for an item
    override func requestBadgeIdentifier(for url: URL) {
        // Determine sync status
        let status = getSyncStatus(for: url)

        FIFinderSyncController.default().setBadgeIdentifier(
            status,
            for: url
        )
    }

    // Contextual menu items
    override func menu(for menuKind: FIMenuKind) -> NSMenu? {
        let menu = NSMenu(title: "")

        switch menuKind {
        case .contextualMenuForItems:
            menu.addItem(
                withTitle: "Sync Now",
                action: #selector(syncNow(_:)),
                keyEquivalent: ""
            )
        case .contextualMenuForContainer:
            menu.addItem(
                withTitle: "Sync Folder",
                action: #selector(syncFolder(_:)),
                keyEquivalent: ""
            )
        case .toolbarItemMenu:
            menu.addItem(
                withTitle: "Open in App",
                action: #selector(openInApp(_:)),
                keyEquivalent: ""
            )
        default:
            break
        }

        return menu
    }

    @objc func syncNow(_ sender: AnyObject?) {
        guard let items = FIFinderSyncController.default().selectedItemURLs() else {
            return
        }
        // Sync selected items
    }

    @objc func syncFolder(_ sender: AnyObject?) {
        guard let target = FIFinderSyncController.default().targetedURL() else {
            return
        }
        // Sync folder
    }

    @objc func openInApp(_ sender: AnyObject?) {
        // Open items in main app
    }
}
```

### Communicating with Main App

```swift
// In extension - use app groups
let sharedDefaults = UserDefaults(suiteName: "group.com.mycompany.myapp")

// Or XPC for more complex communication
// See Apple's XPC documentation
```

---

## Pattern 2: Safari Web Extension

### Creating Safari Extension

1. File → New → Target → Safari Extension
2. Choose "Safari Web Extension" for cross-browser compatibility

### Extension Structure

```
SafariExtension/
├── Info.plist
├── Resources/
│   ├── manifest.json      # Web extension manifest
│   ├── background.js      # Background script
│   ├── content.js         # Content script (injected into pages)
│   ├── popup.html         # Toolbar popup UI
│   ├── popup.js
│   └── images/
│       ├── icon-48.png
│       └── icon-96.png
└── SafariExtensionHandler.swift  # Native messaging handler
```

### manifest.json

```json
{
    "manifest_version": 3,
    "name": "My Extension",
    "version": "1.0",
    "description": "Description of extension",

    "icons": {
        "48": "images/icon-48.png",
        "96": "images/icon-96.png"
    },

    "action": {
        "default_popup": "popup.html",
        "default_icon": {
            "48": "images/icon-48.png"
        }
    },

    "permissions": [
        "activeTab",
        "storage"
    ],

    "background": {
        "service_worker": "background.js"
    },

    "content_scripts": [{
        "matches": ["*://*/*"],
        "js": ["content.js"]
    }]
}
```

### Native Messaging (Swift ↔ JavaScript)

```swift
// SafariExtensionHandler.swift
import SafariServices

class SafariExtensionHandler: SFSafariExtensionHandler {
    override func messageReceived(
        withName messageName: String,
        from page: SFSafariPage,
        userInfo: [String: Any]?
    ) {
        switch messageName {
        case "getData":
            // Fetch data from app
            let response = fetchAppData()
            page.dispatchMessageToScript(
                withName: "dataResponse",
                userInfo: ["data": response]
            )
        default:
            break
        }
    }

    override func toolbarItemClicked(in window: SFSafariWindow) {
        // Handle toolbar click
        window.getActiveTab { tab in
            tab?.getActivePage { page in
                page?.dispatchMessageToScript(
                    withName: "toolbarClicked",
                    userInfo: nil
                )
            }
        }
    }
}
```

```javascript
// background.js - Send message to native handler
browser.runtime.sendNativeMessage(
    "application.id",
    { action: "getData" },
    (response) => {
        console.log("Received:", response);
    }
);
```

---

## Pattern 3: Share Extension

### Creating Share Extension

1. File → New → Target → Share Extension
2. Configure supported data types in Info.plist

### Share Extension View

```swift
import Cocoa
import Social

class ShareViewController: SLComposeServiceViewController {
    override func loadView() {
        super.loadView()

        // Configure extension
        self.title = "Share to My App"
    }

    override func didSelectPost() {
        // Get shared content
        guard let extensionItem = extensionContext?.inputItems.first as? NSExtensionItem,
              let attachments = extensionItem.attachments else {
            extensionContext?.completeRequest(returningItems: nil)
            return
        }

        for attachment in attachments {
            if attachment.hasItemConformingToTypeIdentifier("public.url") {
                attachment.loadItem(forTypeIdentifier: "public.url") { [weak self] url, error in
                    if let url = url as? URL {
                        self?.shareURL(url)
                    }
                    self?.extensionContext?.completeRequest(returningItems: nil)
                }
            }
        }
    }

    override func isContentValid() -> Bool {
        // Validate content before enabling Post button
        return !contentText.isEmpty
    }

    private func shareURL(_ url: URL) {
        // Share to main app via app group or URL scheme
        if let sharedDefaults = UserDefaults(suiteName: "group.com.mycompany.myapp") {
            var urls = sharedDefaults.array(forKey: "sharedURLs") as? [String] ?? []
            urls.append(url.absoluteString)
            sharedDefaults.set(urls, forKey: "sharedURLs")
        }
    }
}
```

### Info.plist Configuration

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <key>NSExtensionActivationRule</key>
        <string>SUBQUERY (
            extensionItems,
            $extensionItem,
            SUBQUERY (
                $extensionItem.attachments,
                $attachment,
                ANY $attachment.registeredTypeIdentifiers UTI-CONFORMS-TO "public.url"
            ).@count == $extensionItem.attachments.@count
        ).@count == 1</string>
    </dict>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.share-services</string>
    <key>NSExtensionPrincipalClass</key>
    <string>$(PRODUCT_MODULE_NAME).ShareViewController</string>
</dict>
```

---

## Pattern 4: Quick Look Extension

### Preview Extension

```swift
import Cocoa
import Quartz

class PreviewViewController: NSViewController, QLPreviewingController {
    override var nibName: NSNib.Name? {
        return NSNib.Name("PreviewViewController")
    }

    func preparePreviewOfFile(
        at url: URL,
        completionHandler handler: @escaping (Error?) -> Void
    ) {
        // Load and display file preview
        do {
            let data = try Data(contentsOf: url)
            let content = parseMyFileFormat(data)

            // Update UI on main thread
            DispatchQueue.main.async {
                self.displayContent(content)
                handler(nil)
            }
        } catch {
            handler(error)
        }
    }

    private func parseMyFileFormat(_ data: Data) -> MyContent {
        // Parse your custom file format
    }

    private func displayContent(_ content: MyContent) {
        // Update view with parsed content
    }
}
```

### Thumbnail Extension

```swift
import QuickLookThumbnailing

class ThumbnailProvider: QLThumbnailProvider {
    override func provideThumbnail(
        for request: QLFileThumbnailRequest,
        _ handler: @escaping (QLThumbnailReply?, Error?) -> Void
    ) {
        let fileURL = request.fileURL
        let maximumSize = request.maximumSize
        let scale = request.scale

        // Option 1: Draw directly
        let reply = QLThumbnailReply(
            contextSize: maximumSize
        ) { context -> Bool in
            // Draw thumbnail in context
            let rect = CGRect(origin: .zero, size: maximumSize)
            context.setFillColor(NSColor.blue.cgColor)
            context.fill(rect)

            // Draw text or image representing file
            return true
        }

        handler(reply, nil)

        // Option 2: Return an image
        // let image = generateThumbnailImage(for: fileURL, size: maximumSize)
        // let reply = QLThumbnailReply(cgImage: image.cgImage!)
        // handler(reply, nil)
    }
}
```

---

## Pattern 5: Spotlight Importer

### Creating Importer

1. File → New → Target → Spotlight Importer
2. Declare supported UTTypes

### Importer Implementation

```swift
import CoreServices

class GetMetadataForFile {
    func getMetadata(
        forFileAt url: URL,
        contentTypeUTI: String
    ) -> [String: Any]? {
        do {
            let data = try Data(contentsOf: url)
            let content = parseFile(data)

            return [
                kMDItemTitle as String: content.title,
                kMDItemDescription as String: content.description,
                kMDItemTextContent as String: content.fullText,
                kMDItemKeywords as String: content.tags,
                kMDItemCreator as String: "My App",
                kMDItemAuthors as String: content.authors
            ]
        } catch {
            return nil
        }
    }
}
```

### Info.plist Configuration

```xml
<key>CFBundleDocumentTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>MDImporter</string>
        <key>LSItemContentTypes</key>
        <array>
            <string>com.mycompany.mydocument</string>
        </array>
    </dict>
</array>
```

---

## Pattern 6: Services Menu

### Adding Services

```swift
// In AppDelegate
class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        NSApp.servicesProvider = ServicesProvider()
    }
}

class ServicesProvider: NSObject {
    // Declare in Info.plist as NSServices
    @objc func processText(_ pboard: NSPasteboard, userData: String, error: AutoreleasingUnsafeMutablePointer<NSString?>) {
        guard let text = pboard.string(forType: .string) else {
            error.pointee = "No text found" as NSString
            return
        }

        let processed = text.uppercased()

        pboard.clearContents()
        pboard.setString(processed, forType: .string)
    }
}
```

### Info.plist for Services

```xml
<key>NSServices</key>
<array>
    <dict>
        <key>NSMenuItem</key>
        <dict>
            <key>default</key>
            <string>Process with My App</string>
        </dict>
        <key>NSMessage</key>
        <string>processText</string>
        <key>NSPortName</key>
        <string>MyApp</string>
        <key>NSSendTypes</key>
        <array>
            <string>public.plain-text</string>
        </array>
        <key>NSReturnTypes</key>
        <array>
            <string>public.plain-text</string>
        </array>
    </dict>
</array>
```

---

## Common Pitfalls

### Pitfall 1: Extension Not Appearing

```
Check:
1. Extension is enabled in System Settings → Extensions
2. Sandbox entitlements are correct
3. App Groups match between app and extension
4. Extension identifier matches in Info.plist
```

### Pitfall 2: Finder Sync Not Showing Badges

```swift
// ❌ WRONG - Watching wrong path
finderSyncController.directoryURLs = [
    URL(fileURLWithPath: "~/MyFolder")  // Tilde not expanded
]

// ✅ CORRECT - Expand path
finderSyncController.directoryURLs = [
    URL(fileURLWithPath: NSHomeDirectory()).appendingPathComponent("MyFolder")
]
```

### Pitfall 3: Share Extension Timeout

```swift
// ❌ WRONG - Long operation without informing system
override func didSelectPost() {
    performLongOperation()  // May timeout
}

// ✅ CORRECT - Complete request promptly
override func didSelectPost() {
    // Copy data to app group storage
    saveToAppGroup(contentText)
    // Complete immediately
    extensionContext?.completeRequest(returningItems: nil)
    // Main app processes data later
}
```

---

## Debugging Extensions

### Enable Extension

```bash
# Finder Sync
pluginkit -m -i com.mycompany.myapp.findersynce

# Safari Extension
# Enable in Safari → Settings → Extensions
```

### View Extension Logs

```bash
# Console.app filter:
# Process: com.mycompany.myapp.extension
# or
log stream --predicate 'subsystem == "com.mycompany.myapp"'
```

### Force Reload Extension

```bash
# Kill Finder Sync host
killall Finder

# Reload pluginkit
pluginkit -e use -i com.mycompany.myapp.findersync
```

---

## Resources

**Docs**: /findersync, /safariservices/safari-web-extensions, /social/slcomposeserviceviewcontroller, /quicklook

**WWDC**: 2019-720 (Introducing Safari Web Extensions), 2021-10109 (What's new in Safari Web Extensions)

**Skills**: axiom-macos-file-handling, axiom-macos-distribution

---

**Last Updated**: Based on macOS 10.15+ documentation
**Platforms**: macOS 10.15+
