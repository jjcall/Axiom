# macos-window-auditor

Audits SwiftUI macOS apps for common window architecture mistakes including scene selection, MenuBarExtra issues, and keyboard shortcut conflicts.

## How to Use This Agent

**Natural language (automatic triggering):**
- "Why does ⌘N keep creating duplicate windows?"
- "My MenuBarExtra state resets when I click away"
- "openWindow isn't opening my window"
- "My Settings window won't open with ⌘,"
- "How do I prevent users from resizing my window?"

**Explicit command:**
```bash
/axiom:audit macos-windows
```

## What It Checks

### Critical (Broken Functionality)
- **openWindow ID mismatch** — Window ID doesn't match scene definition
- **MenuBarExtra state in view** — State resets when popover closes

### High Priority (User Experience)
- **WindowGroup for single-instance** — Creates duplicates with ⌘N
- **Window without ID** — Cannot open programmatically
- **MenuBarExtra without quit** — No way to exit the app
- **Keyboard shortcut conflict** — Overrides system ⌘C, ⌘V, etc.
- **Missing openWindow environment** — Compile error or runtime crash

### Medium Priority (Conventions)
- **Missing default size** — Window may be tiny or huge
- **Settings without TabView** — Doesn't follow macOS conventions
- **WindowGroup for value without openWindow** — Dead code

## Example Output

```markdown
# macOS Window Architecture Audit Results

## Summary
- **CRITICAL Issues**: 1 (Broken functionality)
- **HIGH Issues**: 2 (User experience problems)

## Scene Inventory
| Scene | Type | ID | Purpose |
|-------|------|-----|---------|
| Main | WindowGroup | - | Primary content |
| Inspector | Window | "inspector" | Side panel |

### CRITICAL: openWindow ID Mismatch
- **Location**: `ContentView.swift:45`
- **Issue**: `openWindow(id: "Inspector")` but Window defined with `id: "inspector"`
- **Impact**: Window never opens (silent failure)
- **Fix**: Use matching ID: `openWindow(id: "inspector")`

### HIGH: Keyboard Shortcut Conflict
- **Location**: `MyApp.swift:56`
- **Issue**: `.keyboardShortcut("c")` conflicts with system Copy
- **Fix**: Use modifier: `.keyboardShortcut("c", modifiers: [.command, .shift])`
```

## Model & Tools

- **Model**: haiku (fast scene analysis)
- **Tools**: Glob, Grep, Read
- **Color**: blue

## Related Skills

- **axiom-macos-windows** — WindowGroup, Window, MenuBarExtra, Settings scenes
- **axiom-macos-keyboard-menus** — Keyboard shortcuts, CommandMenu, CommandGroup
