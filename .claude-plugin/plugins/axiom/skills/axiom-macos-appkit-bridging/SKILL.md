---
name: axiom-macos-appkit-bridging
description: Use when integrating AppKit with SwiftUI including NSViewRepresentable, NSHostingView, NSWindow access, NSToolbar customization, NSTouchBar, and accessing AppKit APIs from SwiftUI
user-invocable: true
skill_type: reference
version: 1.0.0
apple_platforms: macOS 10.15+
---

# AppKit and SwiftUI Integration

Reference guide for bridging AppKit and SwiftUI, including wrapping AppKit views, accessing NSWindow, and using AppKit APIs from SwiftUI apps.

## When to Use This Skill

- Wrapping AppKit views for use in SwiftUI (NSViewRepresentable)
- Embedding SwiftUI views in AppKit apps (NSHostingView/Controller)
- Accessing NSWindow from SwiftUI views
- Customizing NSToolbar beyond SwiftUI's toolbar API
- Implementing NSTouchBar
- Using NSMenu programmatically
- Accessing AppKit delegates (NSWindowDelegate, NSApplicationDelegate)

## When to Use AppKit Integration

| Scenario | Approach |
|----------|----------|
| Standard windows/views | Pure SwiftUI (WindowGroup, NavigationStack) |
| Custom window behavior | Access NSWindow via representable |
| Complex text editing | NSTextView via NSViewRepresentable |
| Drag sources with custom images | AppKit drag APIs |
| System integrations (Services, Dock menu) | AppKit delegates |
| Touch Bar | NSTouchBar via representable |

---

## Pattern 1: NSViewRepresentable

### Basic NSViewRepresentable

```swift
import SwiftUI
import AppKit

struct WebView: NSViewRepresentable {
    let url: URL

    func makeNSView(context: Context) -> WKWebView {
        let webView = WKWebView()
        return webView
    }

    func updateNSView(_ webView: WKWebView, context: Context) {
        let request = URLRequest(url: url)
        webView.load(request)
    }
}

// Usage
struct ContentView: View {
    var body: some View {
        WebView(url: URL(string: "https://apple.com")!)
    }
}
```

### NSViewRepresentable with Coordinator (Delegates)

```swift
struct SearchField: NSViewRepresentable {
    @Binding var text: String
    var onSubmit: () -> Void

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    func makeNSView(context: Context) -> NSSearchField {
        let searchField = NSSearchField()
        searchField.delegate = context.coordinator
        return searchField
    }

    func updateNSView(_ searchField: NSSearchField, context: Context) {
        searchField.stringValue = text
    }

    class Coordinator: NSObject, NSSearchFieldDelegate {
        var parent: SearchField

        init(_ parent: SearchField) {
            self.parent = parent
        }

        func controlTextDidChange(_ notification: Notification) {
            guard let searchField = notification.object as? NSSearchField else { return }
            parent.text = searchField.stringValue
        }

        func control(_ control: NSControl, textView: NSTextView, doCommandBy commandSelector: Selector) -> Bool {
            if commandSelector == #selector(NSResponder.insertNewline(_:)) {
                parent.onSubmit()
                return true
            }
            return false
        }
    }
}
```

### NSViewRepresentable with Size Preferences

```swift
struct FixedSizeAppKitView: NSViewRepresentable {
    func makeNSView(context: Context) -> NSView {
        let view = MyCustomNSView()
        view.setContentHuggingPriority(.required, for: .horizontal)
        view.setContentHuggingPriority(.required, for: .vertical)
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {}

    func sizeThatFits(_ proposal: ProposedViewSize, nsView: NSView, context: Context) -> CGSize? {
        // Return intrinsic size or calculate based on proposal
        return CGSize(width: 200, height: 100)
    }
}
```

---

## Pattern 2: NSHostingView (SwiftUI in AppKit)

### Embedding SwiftUI in NSView

```swift
import AppKit
import SwiftUI

class AppKitViewController: NSViewController {
    override func loadView() {
        // Create SwiftUI content
        let swiftUIView = MySwiftUIView()

        // Wrap in hosting view
        let hostingView = NSHostingView(rootView: swiftUIView)
        hostingView.translatesAutoresizingMaskIntoConstraints = false

        // Set as view
        self.view = NSView()
        self.view.addSubview(hostingView)

        NSLayoutConstraint.activate([
            hostingView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingView.topAnchor.constraint(equalTo: view.topAnchor),
            hostingView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }
}

struct MySwiftUIView: View {
    var body: some View {
        VStack {
            Text("SwiftUI in AppKit!")
            Button("Click Me") { print("Clicked") }
        }
        .padding()
    }
}
```

### NSHostingController

```swift
class MainWindowController: NSWindowController {
    convenience init() {
        let hostingController = NSHostingController(rootView: ContentView())
        let window = NSWindow(contentViewController: hostingController)
        window.setContentSize(NSSize(width: 600, height: 400))
        window.title = "My App"
        self.init(window: window)
    }
}
```

---

## Pattern 3: Accessing NSWindow

### Using NSWindow from SwiftUI

```swift
struct WindowAccessor: NSViewRepresentable {
    @Binding var window: NSWindow?

    func makeNSView(context: Context) -> NSView {
        let view = NSView()
        DispatchQueue.main.async {
            self.window = view.window
        }
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {}
}

// Usage
struct ContentView: View {
    @State private var window: NSWindow?

    var body: some View {
        VStack {
            Text("Content")

            Button("Toggle Full Screen") {
                window?.toggleFullScreen(nil)
            }

            Button("Center Window") {
                window?.center()
            }
        }
        .background(WindowAccessor(window: $window))
    }
}
```

### Window Delegate Access

```swift
struct WindowDelegateAccessor: NSViewRepresentable {
    var onWindowWillClose: () -> Void

    func makeCoordinator() -> Coordinator {
        Coordinator(onWindowWillClose: onWindowWillClose)
    }

    func makeNSView(context: Context) -> NSView {
        let view = NSView()
        DispatchQueue.main.async {
            view.window?.delegate = context.coordinator
        }
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {}

    class Coordinator: NSObject, NSWindowDelegate {
        var onWindowWillClose: () -> Void

        init(onWindowWillClose: @escaping () -> Void) {
            self.onWindowWillClose = onWindowWillClose
        }

        func windowWillClose(_ notification: Notification) {
            onWindowWillClose()
        }

        func windowShouldClose(_ sender: NSWindow) -> Bool {
            // Return false to prevent close
            return true
        }
    }
}
```

---

## Pattern 4: Custom NSToolbar

### NSToolbar with SwiftUI Content

```swift
struct ToolbarAccessor: NSViewRepresentable {
    func makeNSView(context: Context) -> NSView {
        let view = NSView()
        DispatchQueue.main.async {
            self.configureToolbar(for: view.window)
        }
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {}

    private func configureToolbar(for window: NSWindow?) {
        guard let window = window else { return }

        let toolbar = NSToolbar(identifier: "MainToolbar")
        toolbar.delegate = ToolbarDelegate.shared
        toolbar.displayMode = .iconOnly
        toolbar.allowsUserCustomization = true
        toolbar.autosavesConfiguration = true

        window.toolbar = toolbar
    }
}

class ToolbarDelegate: NSObject, NSToolbarDelegate {
    static let shared = ToolbarDelegate()

    func toolbar(_ toolbar: NSToolbar, itemForItemIdentifier itemIdentifier: NSToolbarItem.Identifier, willBeInsertedIntoToolbar flag: Bool) -> NSToolbarItem? {
        switch itemIdentifier {
        case .addItem:
            let item = NSToolbarItem(itemIdentifier: itemIdentifier)
            item.label = "Add"
            item.image = NSImage(systemSymbolName: "plus", accessibilityDescription: "Add")
            item.target = self
            item.action = #selector(addItem)
            return item
        default:
            return nil
        }
    }

    func toolbarDefaultItemIdentifiers(_ toolbar: NSToolbar) -> [NSToolbarItem.Identifier] {
        [.flexibleSpace, .addItem]
    }

    func toolbarAllowedItemIdentifiers(_ toolbar: NSToolbar) -> [NSToolbarItem.Identifier] {
        [.addItem, .flexibleSpace, .space]
    }

    @objc func addItem() {
        NotificationCenter.default.post(name: .addItemRequested, object: nil)
    }
}

extension NSToolbarItem.Identifier {
    static let addItem = NSToolbarItem.Identifier("AddItem")
}

extension Notification.Name {
    static let addItemRequested = Notification.Name("AddItemRequested")
}
```

---

## Pattern 5: NSTouchBar

### Touch Bar with SwiftUI

```swift
struct TouchBarAccessor: NSViewRepresentable {
    var onPlay: () -> Void
    var onPause: () -> Void

    func makeNSView(context: Context) -> NSView {
        let view = NSView()
        DispatchQueue.main.async {
            view.window?.touchBar = context.coordinator.makeTouchBar()
        }
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(onPlay: onPlay, onPause: onPause)
    }

    class Coordinator: NSObject, NSTouchBarDelegate {
        var onPlay: () -> Void
        var onPause: () -> Void

        init(onPlay: @escaping () -> Void, onPause: @escaping () -> Void) {
            self.onPlay = onPlay
            self.onPause = onPause
        }

        func makeTouchBar() -> NSTouchBar {
            let touchBar = NSTouchBar()
            touchBar.delegate = self
            touchBar.defaultItemIdentifiers = [.playPauseItem]
            return touchBar
        }

        func touchBar(_ touchBar: NSTouchBar, makeItemForIdentifier identifier: NSTouchBarItem.Identifier) -> NSTouchBarItem? {
            switch identifier {
            case .playPauseItem:
                let item = NSCustomTouchBarItem(identifier: identifier)
                let segmented = NSSegmentedControl(
                    images: [
                        NSImage(systemSymbolName: "play.fill", accessibilityDescription: "Play")!,
                        NSImage(systemSymbolName: "pause.fill", accessibilityDescription: "Pause")!
                    ],
                    trackingMode: .momentary,
                    target: self,
                    action: #selector(segmentTapped(_:))
                )
                item.view = segmented
                return item
            default:
                return nil
            }
        }

        @objc func segmentTapped(_ sender: NSSegmentedControl) {
            if sender.selectedSegment == 0 {
                onPlay()
            } else {
                onPause()
            }
        }
    }
}

extension NSTouchBarItem.Identifier {
    static let playPauseItem = NSTouchBarItem.Identifier("PlayPause")
}
```

### SwiftUI Touch Bar (Simpler)

```swift
struct ContentView: View {
    var body: some View {
        Text("Content")
            .touchBar {
                Button(action: play) {
                    Image(systemName: "play.fill")
                }
                Button(action: pause) {
                    Image(systemName: "pause.fill")
                }
            }
    }
}
```

---

## Pattern 6: NSMenu Integration

### Dock Menu

```swift
// In AppDelegate
class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDockMenu(_ sender: NSApplication) -> NSMenu? {
        let menu = NSMenu()

        menu.addItem(
            withTitle: "New Window",
            action: #selector(newWindow),
            keyEquivalent: ""
        )

        menu.addItem(NSMenuItem.separator())

        menu.addItem(
            withTitle: "Clear Recent",
            action: #selector(clearRecent),
            keyEquivalent: ""
        )

        return menu
    }

    @objc func newWindow() {
        // Open new window
    }

    @objc func clearRecent() {
        // Clear recent documents
    }
}
```

### Dynamic Context Menu

```swift
struct DynamicContextMenuView: NSViewRepresentable {
    var items: [MenuItem]

    func makeNSView(context: Context) -> NSView {
        let view = ClickableView()
        view.coordinator = context.coordinator
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {
        context.coordinator.items = items
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(items: items)
    }

    class Coordinator: NSObject {
        var items: [MenuItem]

        init(items: [MenuItem]) {
            self.items = items
        }

        func buildMenu() -> NSMenu {
            let menu = NSMenu()
            for item in items {
                let menuItem = NSMenuItem(
                    title: item.title,
                    action: #selector(menuItemClicked(_:)),
                    keyEquivalent: ""
                )
                menuItem.target = self
                menuItem.representedObject = item
                menu.addItem(menuItem)
            }
            return menu
        }

        @objc func menuItemClicked(_ sender: NSMenuItem) {
            guard let item = sender.representedObject as? MenuItem else { return }
            item.action()
        }
    }

    class ClickableView: NSView {
        weak var coordinator: Coordinator?

        override func rightMouseDown(with event: NSEvent) {
            guard let menu = coordinator?.buildMenu() else { return }
            NSMenu.popUpContextMenu(menu, with: event, for: self)
        }
    }
}
```

---

## Common Pitfalls

### Pitfall 1: Accessing Window Too Early

```swift
// ❌ WRONG - Window is nil during makeNSView
func makeNSView(context: Context) -> NSView {
    let view = NSView()
    view.window?.title = "Title"  // window is nil!
    return view
}

// ✅ CORRECT - Access window asynchronously
func makeNSView(context: Context) -> NSView {
    let view = NSView()
    DispatchQueue.main.async {
        view.window?.title = "Title"  // window exists now
    }
    return view
}
```

### Pitfall 2: Coordinator Lifetime

```swift
// ❌ WRONG - Coordinator recreated on every update
struct MyView: NSViewRepresentable {
    func makeCoordinator() -> Coordinator {
        Coordinator()  // New instance each time
    }
}

// ✅ CORRECT - Coordinator is cached by SwiftUI
// Just implement makeCoordinator once; SwiftUI reuses it
```

### Pitfall 3: Missing updateNSView

```swift
// ❌ WRONG - SwiftUI state changes not reflected
struct TextField: NSViewRepresentable {
    @Binding var text: String

    func makeNSView(context: Context) -> NSTextField {
        NSTextField()
    }

    func updateNSView(_ nsView: NSTextField, context: Context) {
        // Nothing here - text changes won't update the field!
    }
}

// ✅ CORRECT - Update AppKit view from SwiftUI state
func updateNSView(_ nsView: NSTextField, context: Context) {
    if nsView.stringValue != text {
        nsView.stringValue = text
    }
}
```

### Pitfall 4: Retain Cycles with Coordinator

```swift
// ❌ WRONG - Strong reference to parent
class Coordinator {
    var parent: MyView  // Strong reference can cause cycles
}

// ✅ CORRECT - Store values, not view reference
class Coordinator {
    var onAction: () -> Void  // Closure captures what's needed
}
```

---

## Resources

**Docs**: /swiftui/nsviewrepresentable, /appkit/nshostingview, /appkit/nswindow, /appkit/nstoolbar

**WWDC**: 2019-231 (Integrating SwiftUI), 2022-10072 (Use SwiftUI with AppKit)

**Skills**: axiom-macos-windows, axiom-macos-keyboard-menus

---

**Last Updated**: Based on macOS 10.15+ documentation
**Platforms**: macOS 10.15+
