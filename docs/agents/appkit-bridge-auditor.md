# appkit-bridge-auditor

Audits AppKit/SwiftUI hybrid apps for integration issues including NSViewRepresentable lifecycle, Coordinator patterns, and responder chain problems.

## How to Use This Agent

**Natural language (automatic triggering):**
- "My NSViewRepresentable doesn't update when SwiftUI state changes"
- "Keyboard shortcuts stop working in my AppKit view"
- "How do I access NSWindow from my SwiftUI view?"
- "My delegate callbacks aren't reaching SwiftUI"
- "Memory leak in my NSViewRepresentable"

**Explicit command:**
```bash
/axiom:audit appkit-bridge
```

## What It Checks

### Critical (Broken Functionality)
- **Window access in makeNSView** — `view.window` is nil during creation
- **Missing updateNSView** — SwiftUI state changes not reflected in AppKit

### High Priority (Memory/Delegates)
- **Strong reference to parent** — Memory leak in Coordinator
- **Delegate not set to coordinator** — Callbacks never fire
- **Coordinator missing protocol** — Delegate methods not called
- **Responder chain break** — Keyboard shortcuts stop working
- **NotificationCenter observer leak** — Observer never removed
- **Main thread violation** — UI updates from background

### Medium Priority (Layout/Sizing)
- **NSHostingView without constraints** — SwiftUI content doesn't resize
- **Missing sizeThatFits** — View may be wrong size
- **Updating in makeNSView** — Dynamic values set in wrong lifecycle method

## Example Output

```markdown
# AppKit Bridge Audit Results

## Summary
- **CRITICAL Issues**: 2 (Broken functionality)
- **HIGH Issues**: 1 (Memory leaks)

## Bridge Components Found
| Component | Type | File | Issues |
|-----------|------|------|--------|
| SearchField | NSViewRepresentable | SearchField.swift | 2 |

### CRITICAL: Window Access in makeNSView
- **Location**: `WindowAccessor.swift:23`
- **Issue**: `view.window?.title = "Title"` — window is nil here
- **Fix**: Access window asynchronously:
  ```swift
  DispatchQueue.main.async {
      view.window?.title = "Title"
  }
  ```

### HIGH: Strong Reference Cycle
- **Location**: `SearchField.swift:56`
- **Issue**: `var parent: SearchFieldRepresentable` — strong reference
- **Fix**: Use closure pattern instead of storing parent reference
```

## Model & Tools

- **Model**: haiku (fast lifecycle analysis)
- **Tools**: Glob, Grep, Read
- **Color**: purple

## Related Skills

- **axiom-macos-appkit-bridging** — NSViewRepresentable, NSHostingView, NSWindow access
