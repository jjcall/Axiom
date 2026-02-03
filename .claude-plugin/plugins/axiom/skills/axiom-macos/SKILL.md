---
name: axiom-macos
description: Use when building ANY macOS app including SwiftUI Mac apps, AppKit integration, menu bar apps, document-based apps, window management. Covers WindowGroup, MenuBarExtra, Settings scenes, sandboxing, notarization, keyboard shortcuts.
user-invocable: false
skill_type: router
version: 1.0.0
apple_platforms: macOS 13+ (Ventura), macOS 14+ (Sonoma), macOS 15+ (Sequoia), macOS 26+ (Tahoe)
---

# macOS Development Router

**You MUST use this skill for ANY macOS-specific development question including window management, document apps, menu bar apps, keyboard shortcuts, distribution, and AppKit integration.**

## When to Use

Use this router when working with:
- macOS app architecture (WindowGroup, Window, Settings, MenuBarExtra)
- Document-based apps (DocumentGroup, FileDocument, ReferenceFileDocument)
- Menu bar and keyboard shortcuts
- macOS distribution (notarization, Developer ID, Mac App Store)
- AppKit integration with SwiftUI
- Sandboxing and security-scoped file access
- macOS extensions (Finder Sync, Safari, Share)
- macOS-specific UI patterns

## Cross-Platform Note

Many SwiftUI concepts work identically on macOS and iOS. This router focuses on **macOS-unique concepts**. For cross-platform topics, use the corresponding iOS skill:

| Topic | Use Skill |
|-------|-----------|
| SwiftUI views, state, bindings | axiom-ios-ui |
| Navigation (NavigationStack, NavigationSplitView) | axiom-swiftui-nav-ref |
| Data persistence (SwiftData, Core Data) | axiom-ios-data |
| Networking | axiom-ios-networking |
| Swift concurrency | axiom-ios-concurrency |
| Testing | axiom-ios-testing |
| Build issues | axiom-ios-build |

## Routing Logic

### 1. Window Management → **macos-windows**

**Triggers**:
- WindowGroup vs Window vs Settings
- MenuBarExtra for menu bar apps
- Window sizing, positioning, styling
- Window lifecycle (open, close, minimize)
- Multi-window apps
- Toolbar customization

**Invoke**: `/skill axiom-macos-windows`

---

### 2. Document-Based Apps → **macos-document-apps**

**Triggers**:
- DocumentGroup scene type
- FileDocument protocol (value types)
- ReferenceFileDocument protocol (reference types)
- Autosave and dirty state
- Undo/redo integration
- File type declarations (UTType)
- Recent documents

**Invoke**: `/skill axiom-macos-document-apps`

---

### 3. Keyboard & Menus → **macos-keyboard-menus**

**Triggers**:
- `.keyboardShortcut()` modifier
- CommandGroup and CommandMenu
- Menu bar customization
- Context menus
- Standard menu commands (Edit, View, Window)
- Keyboard navigation patterns

**Invoke**: `/skill axiom-macos-keyboard-menus`

---

### 4. File Handling & Sandboxing → **macos-file-handling**

**Triggers**:
- Security-scoped URLs
- Bookmarks for persistent file access
- Sandboxing entitlements
- Open/save panels
- Drag and drop file handling
- File promises

**Invoke**: `/skill axiom-macos-file-handling`

---

### 5. App Distribution → **macos-distribution**

**Triggers**:
- Notarization workflow
- Developer ID signing
- Mac App Store vs direct distribution
- Hardened Runtime requirements
- Gatekeeper considerations
- Sparkle for auto-updates

**Invoke**: `/skill axiom-macos-distribution`

---

### 6. AppKit Integration → **macos-appkit-bridging**

**Triggers**:
- NSHostingController / NSHostingView
- NSViewRepresentable
- NSWindowDelegate access
- NSToolbar customization
- NSTouchBar support
- NSMenuItem with SwiftUI

**Invoke**: `/skill axiom-macos-appkit-bridging`

---

### 7. macOS Extensions → **macos-extensions**

**Triggers**:
- Finder Sync extensions
- Safari extensions
- Share extensions on Mac
- Quick Look previews
- Spotlight importers
- Services menu

**Invoke**: `/skill axiom-macos-extensions`

---

### 8. UI & Design (macOS-specific) → **hig** + **ios-ui**

**Triggers**:
- macOS Human Interface Guidelines
- Pointer interactions (hover, right-click)
- Dense layouts appropriate for Mac
- Window chrome and controls

**Invoke**: `/skill axiom-hig` for design, `/skill axiom-ios-ui` for SwiftUI implementation

---

### 9. Navigation (macOS patterns) → **swiftui-nav-ref**

**Triggers**:
- NavigationSplitView sidebar patterns
- NSToolbar integration
- Source list sidebars
- Inspector panels

**Invoke**: `/skill axiom-swiftui-nav-ref` (includes macOS patterns)

---

### 10. Accessibility (macOS) → **accessibility-diag**

**Triggers**:
- Keyboard navigation
- Focus rings and focus management
- NSAccessibility patterns
- VoiceOver on macOS

**Invoke**: `/skill axiom-accessibility-diag` (includes macOS patterns)

---

## Decision Tree

1. Window/scene management? → macos-windows
2. Document-based app? → macos-document-apps
3. Keyboard shortcuts or menus? → macos-keyboard-menus
4. File access with sandbox? → macos-file-handling
5. Distribution/notarization? → macos-distribution
6. Deep AppKit integration? → macos-appkit-bridging
7. macOS extensions? → macos-extensions
8. Design guidelines? → hig
9. Navigation patterns? → swiftui-nav-ref
10. Accessibility? → accessibility-diag
11. Cross-platform SwiftUI? → ios-ui

## Anti-Rationalization

| Thought | Reality |
|---------|---------|
| "I'll just use iOS patterns for my Mac app" | Mac apps have unique expectations (menus, keyboard, windows). Users notice. |
| "WindowGroup is enough, I don't need Window" | Single-window apps, settings, and utilities benefit from the right scene type. |
| "I'll skip notarization for now" | Unsigned apps trigger Gatekeeper warnings, blocking most users. |
| "I know AppKit, I'll just use NSHostingView" | SwiftUI scenes handle 90% of cases better. AppKit bridging is for specific needs. |
| "Sandbox is optional" | App Store requires it, and users increasingly expect sandboxed apps. |

## Example Invocations

User: "How do I create a menu bar app on macOS?"
→ Invoke: `/skill axiom-macos-windows` (MenuBarExtra section)

User: "How do I implement keyboard shortcuts?"
→ Invoke: `/skill axiom-macos-keyboard-menus`

User: "My Mac app needs to access files the user chooses"
→ Invoke: `/skill axiom-macos-file-handling`

User: "What's the difference between WindowGroup and Window?"
→ Invoke: `/skill axiom-macos-windows`

User: "How do I create a document-based Mac app?"
→ Invoke: `/skill axiom-macos-document-apps`

User: "How do I notarize my Mac app for distribution?"
→ Invoke: `/skill axiom-macos-distribution`

User: "How do I access NSWindow from SwiftUI?"
→ Invoke: `/skill axiom-macos-appkit-bridging`

User: "How do I create a Finder extension?"
→ Invoke: `/skill axiom-macos-extensions`

User: "How do I implement proper keyboard navigation on macOS?"
→ Invoke: `/skill axiom-accessibility-diag`

User: "Should I use NavigationSplitView or a sidebar on Mac?"
→ Invoke: `/skill axiom-swiftui-nav-ref`
