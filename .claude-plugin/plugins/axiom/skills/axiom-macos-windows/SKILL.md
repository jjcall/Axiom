---
name: axiom-macos-windows
description: Use when managing macOS windows, scenes, or app lifecycle including WindowGroup, Window, MenuBarExtra, Settings scenes, window sizing, styling, and multi-window apps
user-invocable: true
skill_type: discipline
version: 1.0.0
apple_platforms: macOS 13+ (Ventura), macOS 14+ (Sonoma), macOS 15+ (Sequoia), macOS 26+ (Tahoe)
---

# macOS Window Management

Comprehensive guide to SwiftUI scenes and window management on macOS, covering WindowGroup, Window, MenuBarExtra, and Settings.

## When to Use This Skill

- Choosing between WindowGroup, Window, and other scene types
- Creating menu bar apps with MenuBarExtra
- Implementing Settings/Preferences windows
- Managing window sizing, positioning, and styling
- Building multi-window applications
- Controlling window lifecycle (open, close, minimize)
- Customizing toolbars and title bars

## Example Prompts

#### 1. "What's the difference between WindowGroup and Window?"
→ WindowGroup allows multiple instances (⌘N creates new window), Window is singleton

#### 2. "How do I create a menu bar app?"
→ Use MenuBarExtra scene type with or without a main window

#### 3. "How do I open a new window programmatically?"
→ Use @Environment(\.openWindow) with window identifiers

#### 4. "How do I make my window a specific size?"
→ Use .defaultSize(), .windowResizability(), frame constraints

#### 5. "How do I hide the title bar?"
→ Use .windowStyle(.hiddenTitleBar) or .titlebarAppearsTransparent

---

## Scene Types Overview

### Decision Tree: Choosing a Scene Type

```
What kind of window do you need?

├─ Main app content with multiple windows?
│  └─ WindowGroup (users can ⌘N to create new windows)
│
├─ Single-instance utility window?
│  └─ Window (only one instance, opens via openWindow)
│
├─ Settings/Preferences?
│  └─ Settings (standard ⌘, shortcut, macOS conventions)
│
├─ Menu bar icon with popover?
│  └─ MenuBarExtra (can be window-based or menu-based)
│
└─ Auxiliary panel/inspector?
   └─ Window with specific styling
```

### Comparison Table

| Scene Type | Multiple Instances | ⌘N Support | Use Case |
|------------|-------------------|------------|----------|
| WindowGroup | Yes | Yes | Main document windows, browsers |
| Window | No (singleton) | No | Utilities, auxiliary windows |
| Settings | No | No (⌘,) | Preferences window |
| MenuBarExtra | N/A | N/A | Menu bar apps, status items |
| DocumentGroup | Yes | Yes (⌘N) | Document-based apps |

---

## Pattern 1: WindowGroup (Multi-Window Apps)

### Basic WindowGroup

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

**Behavior**:
- ⌘N creates a new window instance
- Each window has independent state
- Windows appear in Window menu
- State restoration across launches

### WindowGroup with Value Binding (macOS 13+)

```swift
@main
struct MyApp: App {
    var body: some Scene {
        // Main window
        WindowGroup {
            ContentView()
        }

        // Detail windows bound to specific items
        WindowGroup(for: Item.ID.self) { $itemID in
            if let itemID {
                ItemDetailView(itemID: itemID)
            }
        }
    }
}

// Open a detail window for specific item
struct ContentView: View {
    @Environment(\.openWindow) private var openWindow

    var body: some View {
        Button("Open Item Details") {
            openWindow(value: item.id)
        }
    }
}
```

### WindowGroup Modifiers

```swift
WindowGroup {
    ContentView()
}
.defaultSize(width: 800, height: 600)          // Initial size
.defaultPosition(.center)                       // Initial position (macOS 14+)
.windowResizability(.contentSize)              // Fit to content
.windowResizability(.contentMinSize)           // Minimum = content size
.windowResizability(.automatic)                // User-resizable (default)
.windowStyle(.hiddenTitleBar)                  // Hide title bar
.windowToolbarStyle(.unified)                  // Unified toolbar style
.commands { /* Menu commands */ }
```

---

## Pattern 2: Window (Single-Instance Windows)

### Auxiliary Window

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        // Single-instance inspector window
        Window("Inspector", id: "inspector") {
            InspectorView()
        }
        .defaultSize(width: 300, height: 400)
        .defaultPosition(.topTrailing)
        .keyboardShortcut("i", modifiers: [.command, .option])
    }
}
```

### Opening Windows Programmatically

```swift
struct ContentView: View {
    @Environment(\.openWindow) private var openWindow
    @Environment(\.dismissWindow) private var dismissWindow  // macOS 14+

    var body: some View {
        VStack {
            Button("Show Inspector") {
                openWindow(id: "inspector")
            }

            Button("Hide Inspector") {
                dismissWindow(id: "inspector")
            }
        }
    }
}
```

### Window with Value

```swift
Window("Item Details", id: "item-details", for: Item.ID.self) { $itemID in
    if let itemID {
        ItemDetailView(itemID: itemID)
    } else {
        Text("Select an item")
    }
}

// Open with specific value
openWindow(id: "item-details", value: selectedItem.id)
```

---

## Pattern 3: Settings Scene

### Standard Settings Window

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        Settings {
            SettingsView()
        }
    }
}

struct SettingsView: View {
    var body: some View {
        TabView {
            GeneralSettingsView()
                .tabItem {
                    Label("General", systemImage: "gear")
                }

            AdvancedSettingsView()
                .tabItem {
                    Label("Advanced", systemImage: "gearshape.2")
                }
        }
        .frame(width: 450, height: 250)
    }
}
```

**Behavior**:
- Opens with ⌘, (standard macOS shortcut)
- Single instance (reopens existing window)
- Automatically titled "Settings" (or "Preferences" on older macOS)

### Settings with Form Layout

```swift
struct GeneralSettingsView: View {
    @AppStorage("showWelcome") private var showWelcome = true
    @AppStorage("refreshInterval") private var refreshInterval = 60

    var body: some View {
        Form {
            Toggle("Show welcome screen on launch", isOn: $showWelcome)

            Picker("Refresh interval", selection: $refreshInterval) {
                Text("30 seconds").tag(30)
                Text("1 minute").tag(60)
                Text("5 minutes").tag(300)
            }
        }
        .formStyle(.grouped)
        .padding()
    }
}
```

---

## Pattern 4: MenuBarExtra (Menu Bar Apps)

### Menu Bar App with Popover Window

```swift
@main
struct MenuBarApp: App {
    var body: some Scene {
        MenuBarExtra("My App", systemImage: "star.fill") {
            MenuBarContentView()
        }
        .menuBarExtraStyle(.window)  // Popover-style window
    }
}

struct MenuBarContentView: View {
    var body: some View {
        VStack(spacing: 12) {
            Text("Quick Status")
                .font(.headline)

            Divider()

            Button("Open Main Window") {
                NSApplication.shared.activate(ignoringOtherApps: true)
            }

            Button("Quit") {
                NSApplication.shared.terminate(nil)
            }
            .keyboardShortcut("q")
        }
        .padding()
        .frame(width: 200)
    }
}
```

### Menu Bar App with Traditional Menu

```swift
@main
struct MenuBarApp: App {
    var body: some Scene {
        MenuBarExtra("My App", systemImage: "star.fill") {
            Button("Action One") { /* ... */ }
            Button("Action Two") { /* ... */ }
            Divider()
            Button("Quit") {
                NSApplication.shared.terminate(nil)
            }
            .keyboardShortcut("q")
        }
        .menuBarExtraStyle(.menu)  // Traditional dropdown menu
    }
}
```

### Menu Bar App with Main Window

```swift
@main
struct MyApp: App {
    var body: some Scene {
        // Main window (optional - can be hidden on launch)
        WindowGroup {
            ContentView()
        }

        // Menu bar presence
        MenuBarExtra("My App", systemImage: "star.fill") {
            MenuBarView()
        }
        .menuBarExtraStyle(.window)
    }
}
```

### Menu Bar Only App (No Dock Icon)

Add to Info.plist:
```xml
<key>LSUIElement</key>
<true/>
```

Or in Xcode: Target → Info → "Application is agent (UIElement)" = YES

---

## Pattern 5: Window Sizing and Positioning

### Size Constraints

```swift
WindowGroup {
    ContentView()
}
.defaultSize(width: 800, height: 600)  // Initial size
.defaultSize(CGSize(width: 800, height: 600))  // Alternative

// Content-based sizing
.windowResizability(.contentSize)      // Window = content size exactly
.windowResizability(.contentMinSize)   // Minimum = content, can grow
.windowResizability(.automatic)        // Default, user controls

// Frame constraints in view
struct ContentView: View {
    var body: some View {
        MainContent()
            .frame(minWidth: 400, idealWidth: 800, maxWidth: .infinity,
                   minHeight: 300, idealHeight: 600, maxHeight: .infinity)
    }
}
```

### Window Position (macOS 14+)

```swift
WindowGroup {
    ContentView()
}
.defaultPosition(.center)           // Center of screen
.defaultPosition(.topLeading)       // Top-left
.defaultPosition(.topTrailing)      // Top-right
.defaultPosition(.bottomLeading)    // Bottom-left
.defaultPosition(.bottomTrailing)   // Bottom-right

// Or with UnitPoint
.defaultPosition(UnitPoint(x: 0.25, y: 0.25))
```

### Restoring Window State

```swift
WindowGroup {
    ContentView()
}
.defaultSize(width: 800, height: 600)
// SwiftUI automatically restores size/position for WindowGroup
// Use handlesExternalEvents for URL handling to target correct window
```

---

## Pattern 6: Window Styling

### Title Bar Styles

```swift
WindowGroup {
    ContentView()
}
// Hidden title bar (content extends to top)
.windowStyle(.hiddenTitleBar)

// Automatic (default)
.windowStyle(.automatic)

// Toolbar styles
.windowToolbarStyle(.unified)           // Toolbar in title bar
.windowToolbarStyle(.unifiedCompact)    // Compact unified toolbar
.windowToolbarStyle(.expanded)          // Toolbar below title bar
.windowToolbarStyle(.automatic)         // System decides
```

### Transparent Title Bar (Full Bleed Content)

```swift
struct ContentView: View {
    var body: some View {
        ZStack {
            // Full-bleed background
            Image("hero")
                .resizable()
                .ignoresSafeArea()

            // Content
            VStack {
                // Title bar area is transparent
                Text("Welcome")
                    .font(.largeTitle)
            }
        }
        .toolbar {
            ToolbarItem(placement: .navigation) {
                // Toolbar items still work
            }
        }
    }
}

// In App
WindowGroup {
    ContentView()
}
.windowStyle(.hiddenTitleBar)
```

### Window Background

```swift
struct ContentView: View {
    var body: some View {
        MainContent()
            .background(.ultraThinMaterial)  // Vibrancy
            .background(Color.clear)          // Transparent (needs entitlement)
    }
}
```

---

## Pattern 7: Window Lifecycle

### Observing Window Events

```swift
struct ContentView: View {
    @Environment(\.controlActiveState) private var controlActiveState

    var body: some View {
        Text("Window is \(controlActiveState == .key ? "active" : "inactive")")
            .onChange(of: controlActiveState) { oldValue, newValue in
                if newValue == .key {
                    // Window became active
                }
            }
    }
}
```

### Preventing Window Close

```swift
// Using AppKit integration
class WindowDelegate: NSObject, NSWindowDelegate {
    func windowShouldClose(_ sender: NSWindow) -> Bool {
        // Return false to prevent close
        // Show confirmation dialog
        return false
    }
}
```

### Window Identifier Access

```swift
struct ContentView: View {
    @Environment(\.window) private var window  // NSWindow reference

    var body: some View {
        Button("Get Window Info") {
            if let window {
                print("Frame: \(window.frame)")
                print("Title: \(window.title)")
            }
        }
    }
}
```

---

## Common Pitfalls

### Pitfall 1: Using WindowGroup for Single-Instance Windows

```swift
// ❌ WRONG - Creates duplicate windows with ⌘N
WindowGroup("Inspector") {
    InspectorView()
}

// ✅ CORRECT - Single instance
Window("Inspector", id: "inspector") {
    InspectorView()
}
```

### Pitfall 2: Forgetting Default Size

```swift
// ❌ WRONG - Window may be tiny or huge
WindowGroup {
    ContentView()  // No frame constraints
}

// ✅ CORRECT - Explicit sizing
WindowGroup {
    ContentView()
        .frame(minWidth: 400, minHeight: 300)
}
.defaultSize(width: 800, height: 600)
```

### Pitfall 3: Menu Bar App Without Quit

```swift
// ❌ WRONG - No way to quit
MenuBarExtra("App", systemImage: "star") {
    Text("Hello")
}

// ✅ CORRECT - Include quit option
MenuBarExtra("App", systemImage: "star") {
    Button("Quit") {
        NSApplication.shared.terminate(nil)
    }
    .keyboardShortcut("q")
}
```

### Pitfall 4: Opening Window Without ID

```swift
// ❌ WRONG - How to open this window?
Window("Details") {
    DetailsView()
}

// ✅ CORRECT - Include id for openWindow
Window("Details", id: "details") {
    DetailsView()
}
// Now: openWindow(id: "details")
```

---

## Pressure Scenario: Multi-Window Document App

### Requirements
- Multiple document windows
- Inspector panel (single instance)
- Settings
- Menu bar presence for quick actions

### Solution

```swift
@main
struct DocumentApp: App {
    var body: some Scene {
        // Document windows
        DocumentGroup(newDocument: MyDocument()) { file in
            DocumentView(document: file.$document)
        }
        .commands {
            CommandGroup(after: .windowArrangement) {
                Button("Show Inspector") {
                    // Toggle inspector
                }
                .keyboardShortcut("i", modifiers: [.command, .option])
            }
        }

        // Single inspector window
        Window("Inspector", id: "inspector") {
            InspectorView()
        }
        .defaultSize(width: 280, height: 400)
        .defaultPosition(.topTrailing)

        // Settings
        Settings {
            SettingsView()
        }

        // Menu bar quick access
        MenuBarExtra("My App", systemImage: "doc.text") {
            Button("New Document") {
                NSDocumentController.shared.newDocument(nil)
            }
            .keyboardShortcut("n")
            Divider()
            Button("Quit") {
                NSApplication.shared.terminate(nil)
            }
            .keyboardShortcut("q")
        }
        .menuBarExtraStyle(.menu)
    }
}
```

---

## Resources

**WWDC**: 2022-10061 (Bring multiple windows to your SwiftUI app), 2023-10141 (What's new in SwiftUI), 2024-10148 (Tailor macOS windows with SwiftUI)

**Docs**: /swiftui/windowgroup, /swiftui/window, /swiftui/settings, /swiftui/menubarextra, /swiftui/openwindowaction

**Skills**: axiom-macos-document-apps, axiom-macos-keyboard-menus, axiom-macos-appkit-bridging

---

**Last Updated**: Based on WWDC 2022-2024, macOS 13-15 documentation
**Platforms**: macOS 13+ (Ventura and later)
