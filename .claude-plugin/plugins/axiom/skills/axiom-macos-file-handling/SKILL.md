---
name: axiom-macos-file-handling
description: Use when accessing files outside the sandbox including security-scoped URLs, bookmarks for persistent access, entitlements, open/save panels, drag-and-drop, and file coordination
user-invocable: true
skill_type: discipline
version: 1.0.0
apple_platforms: macOS 10.15+
---

# macOS File Handling and Sandboxing

Comprehensive guide to file access in sandboxed macOS apps, covering security-scoped URLs, bookmarks, entitlements, and file coordination.

## When to Use This Skill

- Accessing files outside your app's sandbox container
- Persisting user-granted file access across launches
- Implementing open/save panels
- Handling drag-and-drop of files
- Understanding and configuring sandbox entitlements
- Coordinating file access with other apps

## Key Concept: The Sandbox

macOS sandboxed apps have limited file access by default:

| Location | Access | Notes |
|----------|--------|-------|
| App container | Read/Write | ~/Library/Containers/com.you.app/ |
| User-selected files | Temporary | Via open/save panel, drag-drop |
| Bookmarked files | Persistent | Must store and restore bookmark |
| Downloads folder | Optional | Requires entitlement |
| Pictures/Music/Movies | Optional | Requires entitlements |
| Arbitrary locations | No | Not available in sandbox |

---

## Pattern 1: Open and Save Panels

### NSOpenPanel

```swift
import AppKit

func openFile() {
    let panel = NSOpenPanel()
    panel.allowsMultipleSelection = false
    panel.canChooseDirectories = false
    panel.canChooseFiles = true
    panel.allowedContentTypes = [.text, .json]  // UTType

    panel.begin { response in
        if response == .OK, let url = panel.url {
            // User granted access - use immediately
            do {
                let contents = try String(contentsOf: url)
                processContents(contents)
            } catch {
                print("Error reading file: \(error)")
            }
        }
    }
}

// Or modal (blocks main thread - use sparingly)
func openFileModal() -> URL? {
    let panel = NSOpenPanel()
    panel.allowedContentTypes = [.text]

    if panel.runModal() == .OK {
        return panel.url
    }
    return nil
}
```

### NSSavePanel

```swift
func saveFile(content: String) {
    let panel = NSSavePanel()
    panel.allowedContentTypes = [.text]
    panel.nameFieldStringValue = "Untitled.txt"
    panel.canCreateDirectories = true

    panel.begin { response in
        if response == .OK, let url = panel.url {
            do {
                try content.write(to: url, atomically: true, encoding: .utf8)
            } catch {
                print("Error saving: \(error)")
            }
        }
    }
}
```

### SwiftUI Integration

```swift
struct ContentView: View {
    @State private var showingOpenPanel = false
    @State private var fileContent = ""

    var body: some View {
        VStack {
            Text(fileContent)

            Button("Open File") {
                let panel = NSOpenPanel()
                panel.allowedContentTypes = [.text]
                if panel.runModal() == .OK, let url = panel.url {
                    fileContent = (try? String(contentsOf: url)) ?? ""
                }
            }
        }
    }
}
```

---

## Pattern 2: Security-Scoped URLs

When a user selects a file, you get temporary access. To persist access across app launches, use security-scoped bookmarks.

### Creating a Bookmark

```swift
func createBookmark(for url: URL) throws -> Data {
    // Start accessing security-scoped resource
    guard url.startAccessingSecurityScopedResource() else {
        throw BookmarkError.accessDenied
    }
    defer { url.stopAccessingSecurityScopedResource() }

    // Create bookmark data
    let bookmarkData = try url.bookmarkData(
        options: .withSecurityScope,
        includingResourceValuesForKeys: nil,
        relativeTo: nil
    )

    return bookmarkData
}

// Store bookmark data in UserDefaults or your own storage
func saveBookmark(_ data: Data, forKey key: String) {
    UserDefaults.standard.set(data, forKey: "bookmark_\(key)")
}
```

### Resolving a Bookmark

```swift
func resolveBookmark(data: Data) throws -> URL {
    var isStale = false

    let url = try URL(
        resolvingBookmarkData: data,
        options: .withSecurityScope,
        relativeTo: nil,
        bookmarkDataIsStale: &isStale
    )

    if isStale {
        // Bookmark needs refresh - recreate it
        let newData = try createBookmark(for: url)
        // Save new bookmark data
    }

    return url
}
```

### Using Security-Scoped URLs

```swift
func readBookmarkedFile(bookmarkData: Data) throws -> String {
    let url = try resolveBookmark(data: bookmarkData)

    // MUST call startAccessingSecurityScopedResource before access
    guard url.startAccessingSecurityScopedResource() else {
        throw BookmarkError.accessDenied
    }
    defer { url.stopAccessingSecurityScopedResource() }  // MUST call stop

    return try String(contentsOf: url)
}
```

### Complete Bookmark Manager

```swift
class BookmarkManager {
    static let shared = BookmarkManager()
    private let userDefaults = UserDefaults.standard

    func saveBookmark(for url: URL, withKey key: String) throws {
        guard url.startAccessingSecurityScopedResource() else {
            throw BookmarkError.accessDenied
        }
        defer { url.stopAccessingSecurityScopedResource() }

        let data = try url.bookmarkData(
            options: .withSecurityScope,
            includingResourceValuesForKeys: nil,
            relativeTo: nil
        )

        userDefaults.set(data, forKey: "bookmark_\(key)")
    }

    func loadBookmarkedURL(forKey key: String) throws -> URL? {
        guard let data = userDefaults.data(forKey: "bookmark_\(key)") else {
            return nil
        }

        var isStale = false
        let url = try URL(
            resolvingBookmarkData: data,
            options: .withSecurityScope,
            relativeTo: nil,
            bookmarkDataIsStale: &isStale
        )

        if isStale {
            try saveBookmark(for: url, withKey: key)
        }

        return url
    }
}

// Usage
class FileAccessor {
    func accessSavedFile(key: String) throws -> String {
        guard let url = try BookmarkManager.shared.loadBookmarkedURL(forKey: key) else {
            throw BookmarkError.notFound
        }

        guard url.startAccessingSecurityScopedResource() else {
            throw BookmarkError.accessDenied
        }
        defer { url.stopAccessingSecurityScopedResource() }

        return try String(contentsOf: url)
    }
}
```

---

## Pattern 3: Sandbox Entitlements

### Essential Entitlements

Add to your `.entitlements` file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
    <!-- App Sandbox (required for Mac App Store) -->
    <key>com.apple.security.app-sandbox</key>
    <true/>

    <!-- User-selected files (read-only) -->
    <key>com.apple.security.files.user-selected.read-only</key>
    <true/>

    <!-- User-selected files (read-write) -->
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>

    <!-- Downloads folder -->
    <key>com.apple.security.files.downloads.read-write</key>
    <true/>

    <!-- Pictures folder -->
    <key>com.apple.security.assets.pictures.read-write</key>
    <true/>

    <!-- Music folder -->
    <key>com.apple.security.assets.music.read-write</key>
    <true/>

    <!-- Movies folder -->
    <key>com.apple.security.assets.movies.read-write</key>
    <true/>

    <!-- Bookmarks (for persistent file access) -->
    <key>com.apple.security.files.bookmarks.app-scope</key>
    <true/>

    <!-- Document-scoped bookmarks (for document-based apps) -->
    <key>com.apple.security.files.bookmarks.document-scope</key>
    <true/>
</dict>
</plist>
```

### Entitlement Decision Tree

```
What file access do you need?

├─ Files user explicitly selects?
│  └─ com.apple.security.files.user-selected.read-write
│
├─ Remember user's file selections?
│  └─ com.apple.security.files.bookmarks.app-scope
│
├─ Access standard folders (without picker)?
│  ├─ Downloads → files.downloads.read-write
│  ├─ Pictures → assets.pictures.read-write
│  ├─ Music → assets.music.read-write
│  └─ Movies → assets.movies.read-write
│
├─ Document stores external files?
│  └─ com.apple.security.files.bookmarks.document-scope
│
└─ Network access?
   ├─ Outgoing connections → network.client
   └─ Incoming connections → network.server
```

---

## Pattern 4: Drag and Drop

### Receiving Dropped Files

```swift
struct DropTargetView: View {
    @State private var droppedURLs: [URL] = []

    var body: some View {
        Rectangle()
            .fill(.secondary.opacity(0.2))
            .frame(width: 300, height: 200)
            .overlay(
                Text("Drop files here")
            )
            .onDrop(of: [.fileURL], isTargeted: nil) { providers in
                for provider in providers {
                    provider.loadItem(forTypeIdentifier: UTType.fileURL.identifier) { data, error in
                        if let data = data as? Data,
                           let url = URL(dataRepresentation: data, relativeTo: nil) {
                            // Access the file immediately
                            DispatchQueue.main.async {
                                droppedURLs.append(url)
                                processDroppedFile(url)
                            }
                        }
                    }
                }
                return true
            }
    }

    func processDroppedFile(_ url: URL) {
        // File access is granted during drop handling
        // For persistent access, create a bookmark
        do {
            let bookmark = try url.bookmarkData(
                options: .withSecurityScope,
                includingResourceValuesForKeys: nil,
                relativeTo: nil
            )
            // Save bookmark for later access
        } catch {
            print("Could not create bookmark: \(error)")
        }
    }
}
```

### Providing Files for Drag

```swift
struct DraggableFileView: View {
    let fileURL: URL

    var body: some View {
        Text(fileURL.lastPathComponent)
            .draggable(fileURL)  // Simple file drag
    }
}

// For custom drag behavior
struct CustomDraggableView: View {
    let fileURL: URL

    var body: some View {
        Text(fileURL.lastPathComponent)
            .onDrag {
                NSItemProvider(contentsOf: fileURL) ?? NSItemProvider()
            }
    }
}
```

---

## Pattern 5: File Coordination

When multiple processes might access the same file, use NSFileCoordinator.

### Reading with Coordination

```swift
func readFileCoordinated(at url: URL) throws -> String {
    var readError: NSError?
    var contents: String?

    let coordinator = NSFileCoordinator()
    coordinator.coordinate(
        readingItemAt: url,
        options: [],
        error: &readError
    ) { coordinatedURL in
        do {
            contents = try String(contentsOf: coordinatedURL)
        } catch {
            // Handle read error
        }
    }

    if let error = readError {
        throw error
    }

    return contents ?? ""
}
```

### Writing with Coordination

```swift
func writeFileCoordinated(content: String, to url: URL) throws {
    var writeError: NSError?

    let coordinator = NSFileCoordinator()
    coordinator.coordinate(
        writingItemAt: url,
        options: .forReplacing,
        error: &writeError
    ) { coordinatedURL in
        do {
            try content.write(to: coordinatedURL, atomically: true, encoding: .utf8)
        } catch {
            // Handle write error
        }
    }

    if let error = writeError {
        throw error
    }
}
```

### Monitoring File Changes

```swift
class FileMonitor {
    private var fileDescriptor: Int32 = -1
    private var source: DispatchSourceFileSystemObject?

    func startMonitoring(url: URL, onChange: @escaping () -> Void) {
        fileDescriptor = open(url.path, O_EVTONLY)
        guard fileDescriptor != -1 else { return }

        source = DispatchSource.makeFileSystemObjectSource(
            fileDescriptor: fileDescriptor,
            eventMask: [.write, .delete, .rename],
            queue: .main
        )

        source?.setEventHandler {
            onChange()
        }

        source?.setCancelHandler { [weak self] in
            if let fd = self?.fileDescriptor, fd != -1 {
                close(fd)
            }
        }

        source?.resume()
    }

    func stopMonitoring() {
        source?.cancel()
        source = nil
    }
}
```

---

## Common Pitfalls

### Pitfall 1: Forgetting to Start/Stop Security Scope

```swift
// ❌ WRONG - No security scope management
func readFile(url: URL) throws -> String {
    return try String(contentsOf: url)  // May fail for security-scoped URLs
}

// ✅ CORRECT - Proper scope management
func readFile(url: URL) throws -> String {
    guard url.startAccessingSecurityScopedResource() else {
        throw FileError.accessDenied
    }
    defer { url.stopAccessingSecurityScopedResource() }

    return try String(contentsOf: url)
}
```

### Pitfall 2: Bookmark Staleness

```swift
// ❌ WRONG - Ignoring stale bookmarks
func loadURL(from data: Data) throws -> URL {
    var isStale = false
    return try URL(
        resolvingBookmarkData: data,
        options: .withSecurityScope,
        relativeTo: nil,
        bookmarkDataIsStale: &isStale
    )
    // isStale ignored - bookmark may fail next time
}

// ✅ CORRECT - Handle staleness
func loadURL(from data: Data, key: String) throws -> URL {
    var isStale = false
    let url = try URL(
        resolvingBookmarkData: data,
        options: .withSecurityScope,
        relativeTo: nil,
        bookmarkDataIsStale: &isStale
    )

    if isStale {
        // Refresh the bookmark
        let newData = try url.bookmarkData(
            options: .withSecurityScope,
            includingResourceValuesForKeys: nil,
            relativeTo: nil
        )
        // Save newData
    }

    return url
}
```

### Pitfall 3: Missing Entitlements

```swift
// ❌ WRONG - Using bookmarks without entitlement
// App crashes or fails silently

// ✅ CORRECT - Add to entitlements file:
// <key>com.apple.security.files.bookmarks.app-scope</key>
// <true/>
```

### Pitfall 4: Blocking Main Thread

```swift
// ❌ WRONG - Modal panel blocks UI
Button("Open") {
    let panel = NSOpenPanel()
    panel.runModal()  // Blocks main thread
}

// ✅ CORRECT - Use async panel
Button("Open") {
    let panel = NSOpenPanel()
    panel.begin { response in
        // Handle response on main thread
    }
}
```

---

## Pressure Scenario: Project Folder Access

### Requirements
- User selects a project folder
- App needs access to all files within
- Access persists across launches
- Works with App Sandbox

### Solution

```swift
class ProjectManager: ObservableObject {
    @Published var projectURL: URL?
    private let bookmarkKey = "projectFolder"

    init() {
        loadSavedProject()
    }

    func selectProjectFolder() {
        let panel = NSOpenPanel()
        panel.canChooseFiles = false
        panel.canChooseDirectories = true
        panel.allowsMultipleSelection = false

        panel.begin { [weak self] response in
            guard response == .OK, let url = panel.url else { return }
            self?.setProject(url: url)
        }
    }

    private func setProject(url: URL) {
        do {
            let bookmark = try url.bookmarkData(
                options: .withSecurityScope,
                includingResourceValuesForKeys: nil,
                relativeTo: nil
            )
            UserDefaults.standard.set(bookmark, forKey: bookmarkKey)
            projectURL = url
        } catch {
            print("Failed to create bookmark: \(error)")
        }
    }

    private func loadSavedProject() {
        guard let data = UserDefaults.standard.data(forKey: bookmarkKey) else {
            return
        }

        do {
            var isStale = false
            let url = try URL(
                resolvingBookmarkData: data,
                options: .withSecurityScope,
                relativeTo: nil,
                bookmarkDataIsStale: &isStale
            )

            if isStale {
                setProject(url: url)  // Refresh bookmark
            }

            projectURL = url
        } catch {
            print("Failed to restore bookmark: \(error)")
        }
    }

    func withProjectAccess<T>(_ operation: (URL) throws -> T) throws -> T {
        guard let url = projectURL else {
            throw ProjectError.noProject
        }

        guard url.startAccessingSecurityScopedResource() else {
            throw ProjectError.accessDenied
        }
        defer { url.stopAccessingSecurityScopedResource() }

        return try operation(url)
    }
}

// Usage
struct ProjectView: View {
    @StateObject private var projectManager = ProjectManager()

    var body: some View {
        VStack {
            if let url = projectManager.projectURL {
                Text("Project: \(url.lastPathComponent)")

                Button("Read File") {
                    do {
                        let contents = try projectManager.withProjectAccess { url in
                            let fileURL = url.appendingPathComponent("README.md")
                            return try String(contentsOf: fileURL)
                        }
                        print(contents)
                    } catch {
                        print("Error: \(error)")
                    }
                }
            } else {
                Button("Select Project Folder") {
                    projectManager.selectProjectFolder()
                }
            }
        }
    }
}
```

---

## Resources

**Docs**: /security/app-sandbox, /foundation/url/bookmarkdata(options:includingresourcevaluesforkeys:relativeto:), /appkit/nsopenpanel

**Skills**: axiom-macos-document-apps, axiom-macos-distribution

---

**Last Updated**: Based on macOS 10.15+ documentation
**Platforms**: macOS 10.15+
