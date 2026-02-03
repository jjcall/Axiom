# sandbox-distribution-auditor

Audits macOS apps for sandbox entitlements, notarization readiness, code signing issues, and distribution blockers.

## How to Use This Agent

**Natural language (automatic triggering):**
- "Why is my Mac app being blocked by Gatekeeper?"
- "Check if my app is ready for notarization"
- "My app can't access files even though I have the entitlement"
- "What entitlements do I need for my Mac app?"
- "Help me prepare my app for the Mac App Store"

**Explicit command:**
```bash
/axiom:audit sandbox
# or
/axiom:audit distribution
```

## What It Checks

### Critical (Distribution Blockers)
- **Missing network entitlement** — App uses URLSession but sandbox lacks network entitlement
- **Missing file access entitlements** — App accesses files without required entitlements
- **Camera/microphone without entitlement** — AV capture without device entitlements
- **Hardened Runtime missing** — Required for notarization
- **Unsigned frameworks/helpers** — Embedded code not properly signed

### High Priority (App Store/Distribution)
- **Disallowed entitlements for App Store** — JIT, unsigned memory not allowed on MAS
- **Apple Events without entitlement** — Automation fails silently
- **Bookmark usage without entitlement** — Persistent file access fails

### Medium Priority (Configuration)
- **Missing sandbox entitlement** — Required for Mac App Store
- **Excessive/unused entitlements** — May flag during App Store review

## Example Output

```markdown
# Sandbox & Distribution Audit Results

## Summary
- **CRITICAL Issues**: 2 (Will cause failures)
- **HIGH Issues**: 1 (Distribution blockers)

## Distribution Readiness
- **Mac App Store**: NOT READY
- **Developer ID (Direct)**: NOT READY
- **Notarization**: WILL FAIL

### CRITICAL: Missing Network Entitlement
- **Code Usage**: `NetworkManager.swift:45` uses `URLSession.shared`
- **Entitlement**: `com.apple.security.network.client` NOT FOUND
- **Fix**: Add to entitlements file

### CRITICAL: Unsigned Framework
- **Location**: `App.app/Contents/Frameworks/MyFramework.framework`
- **Impact**: Notarization will fail with "unsigned code" error
```

## Model & Tools

- **Model**: haiku (fast entitlement scanning)
- **Tools**: Glob, Grep, Read, Bash
- **Color**: orange

## Related Skills

- **axiom-macos-distribution** — Notarization, code signing, Sparkle updates
- **axiom-macos-file-handling** — Security-scoped URLs, bookmarks, sandbox entitlements
