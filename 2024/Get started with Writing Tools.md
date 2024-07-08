# [**Get started with Writing Tools**](https://developer.apple.com/videos/play/wwdc2024-10168)

---

### **Introducing Writing Tools**

* Users can proofread, rewrite, or transform text in text views
* In native text views, Writing Tools appear on top of the keyboard when you select a piece of text
    * Also appears in the callout bar next to Cut/Copy/Paste
    * Available in the context and Edit menus in macOS
        * Shows up as an affordance to open the Writing Tools panel when hovering over text
* Proofread text
* Rewrite text
    * Can make it more friendly, professional, and concise
* Transform
    * Summary
    * Key Points
    * List
    * Table
* Also works with non-editable text
    * The results will be shown in the panel, an da user can copy or share the results
* When proofreading text, the suggestions are presented inline
    * Can tap and review individual changes

### **Native text views**

* If using UITextView, NSTextView, or WKWebView, Writing Tools work with no extra effort
* Use TextKit 2 to support the full experience
    * TextKit 1 provides a limited experience that just shows the results in a panel
    * [**What's new in TextKit and text views**](https://developer.apple.com/videos/play/wwdc2022/10090/) session from WWDC 2022
    * [**Meet TextKit 2**](https://developer.apple.com/videos/play/wwdc2021/10061) session from WWDC 2021
* How it works
    * May it expand user's selection to full sentences to get better results frm the tools
        * May use more text surrounding the selection to make sure the model is aware of the context of the selected text
    * Fully supports rich text by using attributed strings
        * Preserves styles, links, and attachments as long as the relevant text is still in the rewritten text
    * If the text view supports lists and tables, NSTextList and NSTextTable are sent ot the storage
        * Can control if the text view will handle tables by using `WritingToolsAllowedInputOptions`
* New text view delegate methods for Writing Tools
    * In `textViewWritingToolsWillBegin`,  can prepare your app states for Writing Tools
        * For example, you may want to pause syncing or anything that can directly manipulate the text storage
    * In `textViewWritingToolsDidEnd`, you can restore your app states, like resume syncing
    * New `isWritingToolsActive` property on UITextView
        * To check if Writing Tools is also interacting with the text view

```swift
func textViewWritingToolsWillBegin(_ textView: UITextView) {
    // Take necessary steps to prepare. For example, disable iCloud sync.
}

func textViewWritingToolsDidEnd(_ textView: UITextView) {
    // Take necessary steps to recover. For example, reenable iCloud sync.
}

if !textView.isWritingToolsActive {
    // Do work that needs to be avoided when Writing Tools is interacting with text view
    // For example, in the textViewDidChange callback, app may want to avoid certain things
       when Writing Tools is active
}
```

* Text storage
    * When text is long, the rewritten text may be delivered in separate chunks
        * Animations applied to the text being processed
    * Can tap the `Original` button to switch between original and rewritten text
    * For Proofreading, suggested changes are applied to the view automatically
        * User can review/reject individual suggestions
    * Writing Tools alters text storage
        * Don't persist text storage when Writing Tools is active
        * Changes are pushed to the Undo stack

### **Controlling behavior**

* Can limit tools or opt out by setting the `writingToolsBehavior`

```swift
// Provide a panel experience
textView.writingToolsBehavior = .limited

textView.writingToolsBehavior = .none
```

* Can set accepted text formats with `UIWritingToolsAllowedInputOptions` and `NSWritingToolsAllowedInputOptions`
    * Plain text
    * Rich text
    * Table
    * Assumes Plain/Rich, but not Table, if you don't set this property

```swift
textView.writingToolsAllowedInputOptions = [.plainText]

textView.writingToolsAllowedInputOptions = [.plainText, .richText, .table]
```

* Can specify the behaviors of a WKWebView as well with `WKWebViewConfiguration`

```swift
// For `WKWebView`, the `default` behavior is equivalent to `.limited`

extension WKWebViewConfiguration {
    @available(iOS 18.0, *)
    open var writingToolsBehavior: UIWritingToolsBehavior { get set }
}

extension WKWebViewConfiguration {
    @available(macOS 15.0, *)
    open var writingToolsBehavior: NSWritingToolsBehavior { get set }
}

extension WKWebView {
    /// @discussion If the Writing Tools behavior on the configuration is `.limited`, this will always be `false`.
    @available(iOS 18.0, macOS 15.0, *)
    open var isWritingToolsActive: Bool { get }
}
```

### **Protecting ranges**

* New delegate for protecting ranges
    * For WKWebView, tags like `<blockquote>` and `<pre>` are ignored by Writing Tools

```swift
// Returned `NSRange`s are relative to the substring of the textView’s textStorage from `enclosingRange`
func textView(_ textView: UITextView, writingToolsIgnoredRangesIn
        enclosingRange: NSRange) -> [NSRange] {
    let text = textView.textStorage.attributedSubstring(from: enclosingRange)
    return rangesInappropriateForWritingTools(in: text)
}
```

### **Custom text views**

* If using a custom text view other than `UITextView` or `NSTextView`, you get a basic experience with no extra work
    * Rewritten text appears in a panel
    * Users can copy, share, or apply the text
* On iOS and iPadOS
    * Use `UITextInteraction` to get the callout bar or context menu
    * Or use `UITextSelectionDisplayInteraction` with `UIEditMenuInteraction` if you can't use `UITextInteraction`
    * Writing Tools relies on `UITextsInput` protocol under the hood
    * [**Modernizing your UI for iOS 13**](https://developer.apple.com/videos/play/wwdc2019/224/) session from WWDC 2019
    * [**What's new with text and text interactions**](../2023/What's%20new%20with%20text%20and%20text%20interactions.md) from WWDC 2023 
    * For text views that don't use text interactions, a new optional `isEditable` property added to the `UITextInput` protocol

```swift
protocol UITextInput {
    @available(iOS 18.0, macOS 15.0, *)
    optional var isEditable: Bool { get }
}
```

* On the Mac, Writing Tools will be shown automatically in the context and Edit menus for custom text views
    * Adopt `NSServicesMenuRequestor` to allow the system to read contents from the view and to write contents back to the view
    * Override `validRequestor(forSendType:returnType)` in NSResponder
        * As long as the context menu is added to the view, the Writing Tools menu item will be there
    * Also support the Services menu and Shortcuts
    * [**What's new in AppKit**](https://developer.apple.com/videos/play/wwdc2021/10054/) session from WWDC 2021

```swift
class CustomTextView: NSView, NSServicesMenuRequestor {
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        
        self.menu = NSMenu()
        self.menu?.addItem(NSMenuItem(title: "Custom Text View", action: nil,
            keyEquivalent: ""))
        self.menu?.addItem(NSMenuItem(title: "Copy", action: #selector(copy(_:)), 
            keyEquivalent: ""))
    }
  
    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)
        
        // Custom text drawing code...
    }
}
```

```swift
class CustomTextView: NSView, NSServicesMenuRequestor {
    override func validRequestor(forSendType sendType: NSPasteboard.PasteboardType?, 
                                 returnType: NSPasteboard.PasteboardType?) -> Any? {
        if sendType == .string || sendType == .rtf {
            return self
        }
        return super.validRequestor(forSendType: sendType, returnType: returnType)
    }
    
    nonisolated func writeSelection(to pboard: NSPasteboard,
                                    types: [NSPasteboard.PasteboardType]) -> Bool {
        // Write plain text and/or rich text to pasteboard
        return true
    }

    // Implement readSelection(from pboard: NSPasteboard)
       as well for editable view
}
```
