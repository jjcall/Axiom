---
name: axiom-macos-keyboard-menus
description: Use when implementing keyboard shortcuts, menu bar commands, CommandGroup, CommandMenu, context menus, and standard macOS menu patterns in SwiftUI apps
user-invocable: true
skill_type: reference
version: 1.0.0
apple_platforms: macOS 13+, iOS 15+ (keyboard shortcuts only)
---

# Keyboard Shortcuts and Menus

Comprehensive reference for implementing keyboard shortcuts, menu commands, and context menus in SwiftUI macOS apps.

## When to Use This Skill

- Adding keyboard shortcuts to buttons and actions
- Customizing the app menu bar (File, Edit, View, etc.)
- Creating custom menus with CommandMenu
- Modifying standard menus with CommandGroup
- Implementing context menus (right-click)
- Following macOS keyboard shortcut conventions

## Quick Reference: Standard macOS Shortcuts

| Action | Shortcut | SwiftUI |
|--------|----------|---------|
| New | ⌘N | `.keyboardShortcut("n")` |
| Open | ⌘O | `.keyboardShortcut("o")` |
| Save | ⌘S | `.keyboardShortcut("s")` |
| Close | ⌘W | `.keyboardShortcut("w")` |
| Quit | ⌘Q | Built-in |
| Cut | ⌘X | Built-in |
| Copy | ⌘C | Built-in |
| Paste | ⌘V | Built-in |
| Undo | ⌘Z | Built-in |
| Redo | ⇧⌘Z | Built-in |
| Select All | ⌘A | Built-in |
| Find | ⌘F | `.keyboardShortcut("f")` |
| Preferences | ⌘, | Settings scene (automatic) |

---

## Pattern 1: Basic Keyboard Shortcuts

### Adding Shortcuts to Buttons

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Button("Save") {
                saveDocument()
            }
            .keyboardShortcut("s")  // ⌘S

            Button("New Item") {
                createItem()
            }
            .keyboardShortcut("n")  // ⌘N

            Button("Delete") {
                deleteItem()
            }
            .keyboardShortcut(.delete)  // ⌘⌫ (backspace)
        }
    }
}
```

### Modifier Keys

```swift
Button("Export") {
    exportDocument()
}
.keyboardShortcut("e", modifiers: [.command, .shift])  // ⇧⌘E

Button("Show Inspector") {
    toggleInspector()
}
.keyboardShortcut("i", modifiers: [.command, .option])  // ⌥⌘I

Button("Quick Look") {
    quickLook()
}
.keyboardShortcut(" ", modifiers: [])  // Space (no modifiers)
```

### Special Keys

```swift
.keyboardShortcut(.return)           // Return/Enter
.keyboardShortcut(.escape)           // Escape
.keyboardShortcut(.delete)           // Delete (backspace)
.keyboardShortcut(.deleteForward)    // Forward delete
.keyboardShortcut(.tab)              // Tab
.keyboardShortcut(.upArrow)          // Up arrow
.keyboardShortcut(.downArrow)        // Down arrow
.keyboardShortcut(.leftArrow)        // Left arrow
.keyboardShortcut(.rightArrow)       // Right arrow
.keyboardShortcut(.home)             // Home
.keyboardShortcut(.end)              // End
.keyboardShortcut(.pageUp)           // Page Up
.keyboardShortcut(.pageDown)         // Page Down
```

### Default Button Styling

```swift
Button("OK") {
    confirm()
}
.keyboardShortcut(.defaultAction)  // Return key, blue highlight
```

---

## Pattern 2: Menu Bar Commands

### CommandGroup (Modify Standard Menus)

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            // Add items to File menu
            CommandGroup(after: .newItem) {
                Button("New from Template...") {
                    newFromTemplate()
                }
                .keyboardShortcut("n", modifiers: [.command, .shift])
            }

            // Add items to Edit menu
            CommandGroup(after: .pasteboard) {
                Divider()
                Button("Duplicate") {
                    duplicate()
                }
                .keyboardShortcut("d")
            }

            // Replace help menu
            CommandGroup(replacing: .help) {
                Button("My App Help") {
                    showHelp()
                }
            }
        }
    }
}
```

### CommandGroup Placement Options

```swift
// File menu placements
.newItem           // After "New" and "New Window"
.saveItem          // After "Save" items
.importExport      // After import/export
.printItem         // After print

// Edit menu placements
.undoRedo          // After undo/redo
.pasteboard        // After cut/copy/paste
.textEditing       // After text editing commands
.textFormatting    // After formatting commands

// View menu placements
.sidebar           // Sidebar toggle area
.toolbar           // Toolbar customization area

// Window menu placements
.windowSize        // Window sizing commands
.windowArrangement // Window arrangement
.windowList        // Window list area
.singleWindowList  // Single window list

// Other
.systemServices    // Services menu
.help              // Help menu
.appInfo           // App information
.appSettings       // App settings
.appVisibility     // App visibility (hide/show)
.appTermination    // Quit command
```

### CommandMenu (Custom Menu)

```swift
@main
struct MyApp: App {
    @State private var showGrid = true
    @State private var zoomLevel = 1.0

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            // Custom menu in menu bar
            CommandMenu("Canvas") {
                Button("Zoom In") {
                    zoomLevel *= 1.25
                }
                .keyboardShortcut("+", modifiers: .command)

                Button("Zoom Out") {
                    zoomLevel /= 1.25
                }
                .keyboardShortcut("-", modifiers: .command)

                Button("Actual Size") {
                    zoomLevel = 1.0
                }
                .keyboardShortcut("0")

                Divider()

                Toggle("Show Grid", isOn: $showGrid)
                    .keyboardShortcut("g", modifiers: [.command, .option])

                Divider()

                Menu("Grid Size") {
                    Button("Small") { /* ... */ }
                    Button("Medium") { /* ... */ }
                    Button("Large") { /* ... */ }
                }
            }
        }
    }
}
```

---

## Pattern 3: Context Menus

### Basic Context Menu

```swift
struct ItemView: View {
    let item: Item

    var body: some View {
        Text(item.title)
            .contextMenu {
                Button("Edit") {
                    editItem(item)
                }

                Button("Duplicate") {
                    duplicateItem(item)
                }

                Divider()

                Button("Delete", role: .destructive) {
                    deleteItem(item)
                }
            }
    }
}
```

### Context Menu with Preview (macOS 14+)

```swift
Text(item.title)
    .contextMenu {
        Button("Edit") { editItem(item) }
        Button("Delete", role: .destructive) { deleteItem(item) }
    } preview: {
        ItemPreview(item: item)
            .frame(width: 300, height: 200)
    }
```

### Context Menu for Selection

```swift
struct ItemListView: View {
    @State private var selection: Set<Item.ID> = []

    var body: some View {
        List(items, selection: $selection) { item in
            ItemRow(item: item)
        }
        .contextMenu(forSelectionType: Item.ID.self) { selectedIDs in
            if selectedIDs.count == 1 {
                Button("Edit") { editItem(selectedIDs.first!) }
            }

            Button("Delete \(selectedIDs.count) item(s)", role: .destructive) {
                deleteItems(selectedIDs)
            }
        } primaryAction: { selectedIDs in
            // Double-click action
            if let id = selectedIDs.first {
                openItem(id)
            }
        }
    }
}
```

---

## Pattern 4: Focus-Based Commands

### Commands That Depend on Focus

```swift
struct DocumentView: View {
    @FocusedValue(\.document) private var document: Document?

    var body: some View {
        // ...
    }
}

// Define focused value key
struct FocusedDocumentKey: FocusedValueKey {
    typealias Value = Document
}

extension FocusedValues {
    var document: Document? {
        get { self[FocusedDocumentKey.self] }
        set { self[FocusedDocumentKey.self] = newValue }
    }
}

// In editor view
struct EditorView: View {
    @ObservedObject var document: Document

    var body: some View {
        TextEditor(text: $document.text)
            .focusedValue(\.document, document)
    }
}

// In commands
.commands {
    CommandGroup(after: .pasteboard) {
        Button("Format Selection") {
            // Access focused document
        }
        .disabled(document == nil)
    }
}
```

### FocusedBinding for Editable State

```swift
struct FocusedTextKey: FocusedValueKey {
    typealias Value = Binding<String>
}

extension FocusedValues {
    var selectedText: Binding<String>? {
        get { self[FocusedTextKey.self] }
        set { self[FocusedTextKey.self] = newValue }
    }
}

// Usage
@FocusedBinding(\.selectedText) private var selectedText

.commands {
    CommandMenu("Format") {
        Button("Make Uppercase") {
            selectedText?.wrappedValue = selectedText?.wrappedValue.uppercased() ?? ""
        }
        .disabled(selectedText == nil)
    }
}
```

---

## Pattern 5: Toolbar and Touch Bar

### Toolbar with Keyboard Shortcuts

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            ContentList()
                .toolbar {
                    ToolbarItem(placement: .primaryAction) {
                        Button("Add", systemImage: "plus") {
                            addItem()
                        }
                        .keyboardShortcut("n")
                    }

                    ToolbarItem(placement: .destructiveAction) {
                        Button("Delete", systemImage: "trash") {
                            deleteItem()
                        }
                        .keyboardShortcut(.delete)
                    }
                }
        }
    }
}
```

### Touch Bar (macOS with Touch Bar)

```swift
struct ContentView: View {
    var body: some View {
        MainContent()
            .touchBar {
                Button("Play") {
                    play()
                }

                Button("Pause") {
                    pause()
                }

                Slider(value: $volume, in: 0...1)
            }
    }
}
```

---

## Pattern 6: Disabling and Hiding Commands

### Conditional Disabling

```swift
.commands {
    CommandGroup(after: .newItem) {
        Button("Save") {
            save()
        }
        .keyboardShortcut("s")
        .disabled(!hasUnsavedChanges)

        Button("Revert to Saved") {
            revert()
        }
        .disabled(!hasUnsavedChanges)
    }
}
```

### Removing Standard Commands

```swift
.commands {
    // Remove the entire sidebar toggle
    SidebarCommands()  // Remove this line to hide sidebar toggle

    // Remove toolbar customization
    ToolbarCommands()  // Remove to hide toolbar commands

    // Remove text editing commands (use carefully)
    // TextEditingCommands()
}
```

### Replacing Commands

```swift
.commands {
    CommandGroup(replacing: .newItem) {
        // Custom new item behavior
        Button("New Document") {
            newDocument()
        }
        .keyboardShortcut("n")

        Button("New from Template") {
            newFromTemplate()
        }
        .keyboardShortcut("n", modifiers: [.command, .shift])
    }
}
```

---

## Common Pitfalls

### Pitfall 1: Shortcut Conflicts

```swift
// ❌ WRONG - Conflicts with system ⌘C (copy)
Button("Create") {
    create()
}
.keyboardShortcut("c")

// ✅ CORRECT - Use different shortcut
Button("Create") {
    create()
}
.keyboardShortcut("n", modifiers: [.command, .shift])
```

### Pitfall 2: Shortcuts on Hidden Views

```swift
// ❌ WRONG - Shortcut won't work when view is hidden
if showButton {
    Button("Action") {
        action()
    }
    .keyboardShortcut("a")
}

// ✅ CORRECT - Keep button, just disable
Button("Action") {
    action()
}
.keyboardShortcut("a")
.disabled(!showButton)
```

### Pitfall 3: Missing Commands Block

```swift
// ❌ WRONG - Commands defined in view
struct ContentView: View {
    var body: some View {
        Text("Hello")
            .commands {  // This doesn't exist!
                // ...
            }
    }
}

// ✅ CORRECT - Commands on Scene
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            // Commands go here
        }
    }
}
```

### Pitfall 4: Forgetting Keyboard Shortcut Display

Users expect to see shortcuts in menus. SwiftUI shows them automatically when using `.keyboardShortcut()` on menu items.

```swift
// ✅ Shortcut appears in menu automatically
CommandMenu("Edit") {
    Button("Find") {
        find()
    }
    .keyboardShortcut("f")  // Shows "⌘F" in menu
}
```

---

## Standard macOS Menu Structure

### Recommended Menu Order

1. **App Menu** (automatic) - About, Preferences, Services, Hide, Quit
2. **File** - New, Open, Save, Export, Print, Close
3. **Edit** - Undo, Cut, Copy, Paste, Select All, Find
4. **View** - Show/Hide UI elements, Zoom
5. **[App-specific menus]** - Your custom functionality
6. **Window** - Minimize, Zoom, Bring All to Front, window list
7. **Help** - Search, documentation

### Example: Complete Menu Setup

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            // File menu additions
            CommandGroup(after: .newItem) {
                Button("New from Template...") {
                    newFromTemplate()
                }
                .keyboardShortcut("n", modifiers: [.command, .shift])
            }

            // Custom menu
            CommandMenu("Canvas") {
                Button("Zoom to Fit") {
                    zoomToFit()
                }
                .keyboardShortcut("0")

                Divider()

                Menu("Guides") {
                    Toggle("Show Grid", isOn: $showGrid)
                    Toggle("Show Rulers", isOn: $showRulers)
                }
            }

            // View menu additions
            CommandGroup(after: .sidebar) {
                Toggle("Show Inspector", isOn: $showInspector)
                    .keyboardShortcut("i", modifiers: [.command, .option])
            }
        }

        Settings {
            SettingsView()
        }
    }
}
```

---

## Resources

**WWDC**: 2020-10037 (Build document-based apps in SwiftUI), 2023-10162 (Beyond the basics of structured concurrency)

**Docs**: /swiftui/keyboardshortcut, /swiftui/commandgroup, /swiftui/commandmenu, /swiftui/focusedvalue

**Skills**: axiom-macos-windows, axiom-macos-document-apps, axiom-accessibility-diag

---

**Last Updated**: Based on macOS 13+ documentation
**Platforms**: macOS 13+ (full support), iOS 15+ (keyboard shortcuts only)
