---
name: axiom-macos-document-apps
description: Use when building document-based Mac apps including DocumentGroup, FileDocument, ReferenceFileDocument, autosave, undo/redo integration, and file type declarations
user-invocable: true
skill_type: discipline
version: 1.0.0
apple_platforms: macOS 13+, iOS 16+, iPadOS 16+
---

# Document-Based Apps

Comprehensive guide to building document-based applications with SwiftUI's DocumentGroup, FileDocument, and ReferenceFileDocument protocols.

## When to Use This Skill

- Building apps that create/edit files (text editors, image editors, spreadsheets)
- Implementing FileDocument for value-type documents
- Implementing ReferenceFileDocument for reference-type documents
- Setting up autosave and dirty state tracking
- Integrating undo/redo with document changes
- Declaring supported file types (UTType)
- Handling recent documents and file browser

## Example Prompts

#### 1. "How do I create a document-based Mac app?"
→ Use DocumentGroup with FileDocument or ReferenceFileDocument

#### 2. "What's the difference between FileDocument and ReferenceFileDocument?"
→ FileDocument uses value semantics (struct), ReferenceFileDocument uses reference semantics (class)

#### 3. "How do I implement undo/redo in my document app?"
→ Use UndoManager with ReferenceFileDocument, or structure changes for value-type diffing

#### 4. "My document isn't showing as modified (no dot in close button)"
→ Check that you're properly binding to $document or registering undo actions

#### 5. "How do I declare custom file types?"
→ Define UTType in Info.plist and code, implement readableContentTypes/writableContentTypes

---

## Choosing the Right Document Protocol

### Decision Tree

```
What kind of data does your document hold?

├─ Simple value types (Codable structs)?
│  └─ FileDocument (simpler, automatic dirty tracking)
│
├─ Complex object graphs or large files?
│  └─ ReferenceFileDocument (manual dirty tracking, better performance)
│
└─ Need fine-grained undo/redo?
   ├─ Few operations → FileDocument (SwiftUI handles diffing)
   └─ Many operations → ReferenceFileDocument (explicit UndoManager)
```

### Comparison Table

| Feature | FileDocument | ReferenceFileDocument |
|---------|--------------|----------------------|
| Data Type | Struct (value) | Class (reference) |
| Dirty Tracking | Automatic | Manual (via UndoManager) |
| Undo/Redo | Automatic diffing | Manual registration |
| Performance | Copies entire document | References, incremental |
| Complexity | Simpler | More control |
| Best For | Text, JSON, small files | Large files, complex graphs |

---

## Pattern 1: FileDocument (Value-Type Documents)

### Basic FileDocument

```swift
import SwiftUI
import UniformTypeIdentifiers

struct TextDocument: FileDocument {
    var text: String

    // Supported file types
    static var readableContentTypes: [UTType] { [.plainText] }

    // Initialize empty document
    init() {
        text = ""
    }

    // Initialize from file
    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents,
              let string = String(data: data, encoding: .utf8)
        else {
            throw CocoaError(.fileReadCorruptFile)
        }
        text = string
    }

    // Save to file
    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        let data = text.data(using: .utf8)!
        return FileWrapper(regularFileWithContents: data)
    }
}
```

### Using FileDocument with DocumentGroup

```swift
@main
struct TextEditorApp: App {
    var body: some Scene {
        DocumentGroup(newDocument: TextDocument()) { file in
            TextEditorView(document: file.$document)
        }
    }
}

struct TextEditorView: View {
    @Binding var document: TextDocument

    var body: some View {
        TextEditor(text: $document.text)
            .padding()
    }
}
```

### FileDocument with Codable

```swift
struct TodoDocument: FileDocument {
    var items: [TodoItem]

    static var readableContentTypes: [UTType] { [.json] }

    init() {
        items = []
    }

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents else {
            throw CocoaError(.fileReadCorruptFile)
        }
        items = try JSONDecoder().decode([TodoItem].self, from: data)
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        let data = try JSONEncoder().encode(items)
        return FileWrapper(regularFileWithContents: data)
    }
}

struct TodoItem: Codable, Identifiable {
    let id: UUID
    var title: String
    var isCompleted: Bool
}
```

---

## Pattern 2: ReferenceFileDocument (Reference-Type Documents)

### Basic ReferenceFileDocument

```swift
import SwiftUI
import UniformTypeIdentifiers

final class DrawingDocument: ReferenceFileDocument, ObservableObject {
    @Published var shapes: [Shape]

    // Snapshot for saving
    typealias Snapshot = [Shape]

    static var readableContentTypes: [UTType] { [.json] }

    init() {
        shapes = []
    }

    required init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents else {
            throw CocoaError(.fileReadCorruptFile)
        }
        shapes = try JSONDecoder().decode([Shape].self, from: data)
    }

    // Create snapshot for saving (called on background thread)
    func snapshot(contentType: UTType) throws -> Snapshot {
        shapes
    }

    // Write snapshot to file
    func fileWrapper(snapshot: Snapshot, configuration: WriteConfiguration) throws -> FileWrapper {
        let data = try JSONEncoder().encode(snapshot)
        return FileWrapper(regularFileWithContents: data)
    }
}
```

### Using ReferenceFileDocument with DocumentGroup

```swift
@main
struct DrawingApp: App {
    var body: some Scene {
        DocumentGroup(newDocument: { DrawingDocument() }) { file in
            DrawingEditorView(document: file.document)
        }
    }
}

struct DrawingEditorView: View {
    @ObservedObject var document: DrawingDocument
    @Environment(\.undoManager) private var undoManager

    var body: some View {
        Canvas { context, size in
            for shape in document.shapes {
                // Draw shapes
            }
        }
        .onTapGesture { location in
            addShape(at: location)
        }
    }

    func addShape(at location: CGPoint) {
        let newShape = Shape(center: location)
        let oldShapes = document.shapes

        document.shapes.append(newShape)

        undoManager?.registerUndo(withTarget: document) { doc in
            doc.shapes = oldShapes
        }
    }
}
```

---

## Pattern 3: Undo/Redo Integration

### Automatic Undo with FileDocument

FileDocument gets basic undo/redo automatically through SwiftUI's binding system. Each change to `$document` creates an undo point.

```swift
struct TextEditorView: View {
    @Binding var document: TextDocument

    var body: some View {
        // Changes to text automatically support undo/redo
        TextEditor(text: $document.text)
    }
}
```

### Manual Undo with ReferenceFileDocument

```swift
struct ShapeEditorView: View {
    @ObservedObject var document: DrawingDocument
    @Environment(\.undoManager) private var undoManager

    func deleteShape(_ shape: Shape) {
        guard let index = document.shapes.firstIndex(of: shape) else { return }

        let deletedShape = document.shapes.remove(at: index)

        undoManager?.registerUndo(withTarget: document) { doc in
            doc.shapes.insert(deletedShape, at: index)
        }
        undoManager?.setActionName("Delete Shape")
    }

    func moveShape(_ shape: Shape, to newPosition: CGPoint) {
        guard let index = document.shapes.firstIndex(of: shape) else { return }

        let oldPosition = document.shapes[index].center
        document.shapes[index].center = newPosition

        undoManager?.registerUndo(withTarget: document) { doc in
            doc.shapes[index].center = oldPosition
        }
        undoManager?.setActionName("Move Shape")
    }
}
```

### Grouping Undo Operations

```swift
func performMultipleChanges() {
    undoManager?.beginUndoGrouping()

    // Multiple changes grouped as single undo
    addShape(at: point1)
    addShape(at: point2)
    addShape(at: point3)

    undoManager?.endUndoGrouping()
    undoManager?.setActionName("Add Shapes")
}
```

---

## Pattern 4: Custom File Types (UTType)

### Declaring Custom UTType

**1. Add to Info.plist (Exported Type Identifiers):**

```xml
<key>UTExportedTypeDeclarations</key>
<array>
    <dict>
        <key>UTTypeConformsTo</key>
        <array>
            <string>public.data</string>
            <string>public.content</string>
        </array>
        <key>UTTypeDescription</key>
        <string>My Document</string>
        <key>UTTypeIdentifier</key>
        <string>com.mycompany.mydocument</string>
        <key>UTTypeTagSpecification</key>
        <dict>
            <key>public.filename-extension</key>
            <array>
                <string>mydoc</string>
            </array>
        </dict>
    </dict>
</array>
```

**2. Define UTType in Code:**

```swift
import UniformTypeIdentifiers

extension UTType {
    static var myDocument: UTType {
        UTType(exportedAs: "com.mycompany.mydocument")
    }
}

struct MyDocument: FileDocument {
    static var readableContentTypes: [UTType] { [.myDocument] }
    static var writableContentTypes: [UTType] { [.myDocument] }

    // ... rest of implementation
}
```

### Multiple File Types

```swift
struct ImageDocument: FileDocument {
    var imageData: Data
    var format: UTType

    static var readableContentTypes: [UTType] {
        [.png, .jpeg, .gif, .tiff]
    }

    static var writableContentTypes: [UTType] {
        [.png, .jpeg]
    }

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents else {
            throw CocoaError(.fileReadCorruptFile)
        }
        imageData = data
        format = configuration.contentType
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        // Convert to requested format if needed
        let outputData: Data
        if configuration.contentType == .jpeg {
            outputData = convertToJPEG(imageData)
        } else {
            outputData = convertToPNG(imageData)
        }
        return FileWrapper(regularFileWithContents: outputData)
    }
}
```

---

## Pattern 5: Autosave Behavior

### Understanding Autosave

macOS document apps autosave automatically. Key behaviors:

- **Autosave triggers**: After user stops typing (~3 seconds), app backgrounds, window closes
- **Dirty indicator**: Dot in close button when unsaved changes exist
- **Version browser**: Users can browse/restore previous versions

### Controlling Autosave

```swift
// ReferenceFileDocument - mark document as modified
final class MyDocument: ReferenceFileDocument, ObservableObject {
    @Published var content: String {
        didSet {
            // Document automatically marked dirty when @Published changes
        }
    }
}

// For manual control, use UndoManager
func markDocumentDirty() {
    undoManager?.registerUndo(withTarget: self) { _ in }
}
```

### Disabling Autosave (Rare)

Generally not recommended, but for specific cases:

```swift
// In document view
.onDisappear {
    // Prompt for save before closing
}
```

---

## Pattern 6: File Browser Integration

### Recent Documents

macOS automatically tracks recent documents. Access via:

```swift
// Open recent document
NSDocumentController.shared.openDocument(
    withContentsOf: url,
    display: true
) { document, wasOpen, error in
    // Handle result
}

// Clear recent documents
NSDocumentController.shared.clearRecentDocuments(nil)
```

### Open Panel Customization

```swift
struct ContentView: View {
    @State private var showingOpenPanel = false

    var body: some View {
        Button("Open...") {
            let panel = NSOpenPanel()
            panel.allowedContentTypes = [.myDocument, .json]
            panel.allowsMultipleSelection = false
            panel.canChooseDirectories = false

            if panel.runModal() == .OK, let url = panel.url {
                // Open document at url
            }
        }
    }
}
```

### Save Panel Customization

```swift
func exportDocument() {
    let panel = NSSavePanel()
    panel.allowedContentTypes = [.png, .jpeg]
    panel.nameFieldStringValue = "Untitled"

    if panel.runModal() == .OK, let url = panel.url {
        // Save to url
    }
}
```

---

## Common Pitfalls

### Pitfall 1: FileDocument Not Showing as Modified

```swift
// ❌ WRONG - Modifying copy, not binding
struct EditorView: View {
    var document: TextDocument  // Missing @Binding

    var body: some View {
        TextField("", text: .constant(document.text))  // Changes lost
    }
}

// ✅ CORRECT - Use binding
struct EditorView: View {
    @Binding var document: TextDocument

    var body: some View {
        TextField("", text: $document.text)  // Changes tracked
    }
}
```

### Pitfall 2: ReferenceFileDocument Not Triggering Save

```swift
// ❌ WRONG - No undo registration
func addItem(_ item: Item) {
    document.items.append(item)
    // Document not marked dirty!
}

// ✅ CORRECT - Register undo action
func addItem(_ item: Item) {
    let oldItems = document.items
    document.items.append(item)

    undoManager?.registerUndo(withTarget: document) { doc in
        doc.items = oldItems
    }
}
```

### Pitfall 3: Missing UTType Declaration

```swift
// ❌ WRONG - UTType not declared in Info.plist
extension UTType {
    static var myDoc: UTType {
        UTType(exportedAs: "com.myapp.mydoc")  // Won't work without plist entry
    }
}

// ✅ CORRECT - Also add to Info.plist UTExportedTypeDeclarations
```

### Pitfall 4: Heavy Work in fileWrapper

```swift
// ❌ WRONG - Blocking main thread
func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
    let processedData = heavyProcessing(data)  // Blocks UI during save
    return FileWrapper(regularFileWithContents: processedData)
}

// ✅ CORRECT - Use ReferenceFileDocument snapshot pattern
func snapshot(contentType: UTType) throws -> Snapshot {
    // Called on background thread
    return heavyProcessing(data)
}
```

---

## Pressure Scenario: Rich Text Editor

### Requirements
- Support .rtf and .txt formats
- Undo/redo for text changes
- Autosave
- Export to PDF

### Solution

```swift
import SwiftUI
import UniformTypeIdentifiers

final class RichTextDocument: ReferenceFileDocument, ObservableObject {
    @Published var attributedText: NSAttributedString

    typealias Snapshot = NSAttributedString

    static var readableContentTypes: [UTType] { [.rtf, .plainText] }
    static var writableContentTypes: [UTType] { [.rtf, .plainText, .pdf] }

    init() {
        attributedText = NSAttributedString(string: "")
    }

    required init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents else {
            throw CocoaError(.fileReadCorruptFile)
        }

        if configuration.contentType == .rtf {
            attributedText = try NSAttributedString(
                data: data,
                options: [.documentType: NSAttributedString.DocumentType.rtf],
                documentAttributes: nil
            )
        } else {
            let string = String(data: data, encoding: .utf8) ?? ""
            attributedText = NSAttributedString(string: string)
        }
    }

    func snapshot(contentType: UTType) throws -> Snapshot {
        attributedText
    }

    func fileWrapper(snapshot: Snapshot, configuration: WriteConfiguration) throws -> FileWrapper {
        let data: Data

        switch configuration.contentType {
        case .rtf:
            data = try snapshot.data(
                from: NSRange(location: 0, length: snapshot.length),
                documentAttributes: [.documentType: NSAttributedString.DocumentType.rtf]
            )
        case .pdf:
            data = try createPDF(from: snapshot)
        default:
            data = snapshot.string.data(using: .utf8) ?? Data()
        }

        return FileWrapper(regularFileWithContents: data)
    }

    private func createPDF(from text: NSAttributedString) throws -> Data {
        // PDF generation logic
        let printInfo = NSPrintInfo.shared
        let printOp = NSPrintOperation(view: NSTextView())
        // Configure and return PDF data
        return Data()
    }
}

@main
struct RichTextApp: App {
    var body: some Scene {
        DocumentGroup(newDocument: { RichTextDocument() }) { file in
            RichTextEditorView(document: file.document)
        }
        .commands {
            CommandGroup(after: .importExport) {
                Button("Export as PDF...") {
                    // Trigger PDF export
                }
                .keyboardShortcut("e", modifiers: [.command, .shift])
            }
        }
    }
}
```

---

## Resources

**WWDC**: 2020-10039 (Build document-based apps in SwiftUI), 2022-10061 (Bring multiple windows to your SwiftUI app)

**Docs**: /swiftui/documentgroup, /swiftui/filedocument, /swiftui/referencefiledocument, /uniformtypeidentifiers/uttype

**Skills**: axiom-macos-windows, axiom-macos-file-handling, axiom-macos-keyboard-menus

---

**Last Updated**: Based on WWDC 2020-2022, macOS 13+ documentation
**Platforms**: macOS 13+, iOS 16+, iPadOS 16+
