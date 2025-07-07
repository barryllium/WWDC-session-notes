# [**Meet PaperKit**](https://developer.apple.com/videos/play/wwdc2025/285)

---

* PaperKit is the framework that powers Apple’s markup experience system-wide
    * Used in apps like Notes, Screenshots, QuickLook, and the Journal app
* Provides a canvas that you can both draw on and add a variety of markup elements to, including shapes, images, textboxes, and more
* Now available in macOS Tahoe

### **Getting to know PaperKit**

* Three main components
    * The markup controller (`PaperMarkupViewController`) that interactively creates and displays PaperKit markup and drawing
    * The data model container (`PaperMarkup`)
        * Handles saving and loading both the PaperKit markup and PencilKit drawing data, and handles rendering the markup
    * New insertion menu that allows annotation with markup elements into the canvas
        * On iOS, iPadOS and visionOS 26, this is called a `MarkupEditViewController`
        * On macOS, an alternative option is a new toolbar called `MarkupToolbarViewController`, complete with drawing tools and buttons for annotating

### **Getting started with PaperKit**

* iOS
    * Begin by subclassing UIViewController
    * Within the viewDidLoad method:
        * Initialize a markup data model container, configuring it to match the view’s bounds to ensure proper layout and rendering context
        * Next, create a markup controller configured with the complete set of the latest features provided by PaperKit
        * Add it to the view hierarchy
        * To create a tool picker, initialize a PencilKit tool picker and register the markup controller as an observer
        * To display the tool picker in the app, assign it to the activeToolPicker property on the PencilKit responder state
            * This is a new API available on UIResponder that controls the tool picker visibility for the current responder
    * Configure the accessory item as a button, triggering a function to present the insertion menu in `plusButtonPressed(_ button:)
    * [Introducing PencilKit](https://developer.apple.com/videos/play/wwdc2019/221/) Session from WWDC 2019
    * [Squeeze the most out of Apple Pencil](https://developer.apple.com/videos/play/wwdc2024/10214/) Session from WWDC 2024

```swift
// Adopt PaperKit in iOS

override func viewDidLoad() {
    super.viewDidLoad()
    
    let markupModel = PaperMarkup(bounds: view.bounds)
    let paperViewController = PaperMarkupViewController(markup: markupModel, supportedFeatureSet: .latest)
    view.addSubview(paperViewController.view)
    addChild(paperViewController)
    paperViewController.didMove(toParent: self)
    becomeFirstResponder()    

    let toolPicker = PKToolPicker()
    toolPicker.addObserver(paperViewController)
    
    pencilKitResponderState.activeToolPicker = toolPicker
    pencilKitResponderState.toolPickerVisibility = .visible
    
    toolPicker.accessoryItem = UIBarButtonItem(barButtonSystemItem: .add, target: self, action: #selector(plusButtonPressed(_:)))
}

@objc func plusButtonPressed(_ button: UIBarButtonItem) {
    let markupEditViewController = MarkupEditViewController(supportedFeatureSet: .latest)    
    markupEditViewController.delegate = paperViewController
    markupEditViewController.modalPresentationStyle = .popover
    markupEditViewController.popoverPresentationController?.barButtonItem = button
    present(markupEditViewController, animated: true)
}
```

* macOS
    * Setting up the markup model and markup controller are essentially the same as iOS
    * The only difference on macOS is that a toolbar can be created to present the insertion UI
        * Set the toolbar’s delegate to the markup controller and add it to the view hierarchy using the standard NSViewController embedding process

```swift
// Adopt PaperKit in macOS

override func viewDidLoad() {
    super.viewDidLoad()
    
    let markupModel = PaperMarkup(bounds: view.bounds)
    let paperViewController = PaperMarkupViewController(markup: markupModel, supportedFeatureSet: .latest)
    view.addSubview(paperViewController.view)
    addChild(paperViewController)

    // Create toolbar for macOS
    let toolbarViewController = MarkupToolbarViewController(supportedFeatureSet: .latest)
    toolbarViewController.delegate = paperViewController
    view.addSubview(toolbarViewController.view)
    
    // Set layout
    setupLayoutConstraints()
}
```

* PaperKit can be integrated into a SwiftUI environment with `UIViewControllerRepresentable` or `NSViewControllerRepresentable`
    * [What's new in SwiftUI](https://developer.apple.com/videos/play/wwdc2020/10041/) Session from WWDC 2020

* The markup controller includes a delegate that allows for custom handling of callbacks
    * Can implement the markup change delegate method to automatically save any modifications to the markup model
    * The markup controller conforms to Observable, providing an alternative to using a delegate for managing state and updates

```swift
// Auto-save markup changes
    
func paperMarkupViewControllerDidChangeMarkup(_ paperMarkupViewController: PaperMarkupViewController) {
    let markupModel = paperMarkupViewController.markup
    Task {
        // Create a data blob and save it
        let data = try await markupModel.dataRepresentation()
        try data.write(toFile: paperKitDataURL)
    }
}
```

* When loading data from disk, it is essential to always verify the content version for forwards compatibility
* Two common approaches for handling version mismatches:
    * Presenting an alert to inform of the need to upgrade
    * Showing a pre-rendered thumbnail of the markup
* To show a pre-rendered thumbnail:
    * Create a CGContext to render the thumbnail into
    * Generate a thumbnail image of the model using the markup model’s draw function
    * Save it alongside the markup

```swift
// Thumbnail for forward compatibility

func updateThumbnail(_ markupModel: PaperMarkup) async throws {
    // Set up CGContext to render thumbnail in
    let thumbnailSize = CGSize(width: 200, height: 200)
    let context = makeCGContext(size: thumbnailSize)
    context.setFillColor(gray: 1, alpha: 1)
    context.fill(renderer.format.bounds)            

    // Render the PaperKit markup
    await markupModel.draw(in: context, frame: CGRect(origin: .zero, size: thumbnailSize))
    
    thumbnail = context.makeImage()
}
```

### **Customizing the feature set**

* The set of all available markup functionality in PaperKit is called a FeatureSet
    * Defines the capabilities and tools exposed to both the markup and the insertion controllers
    * `FeatureSet.latest` gives you the full set of markup features supported by PaperKit
    * `remove()`/`insert()` functions allow editing of the available features
    * Add/remove HDR support by setting the `colorMaximumLinearExposure` properties to a number greater than 1
        * Set it on the `toolpicker` as well for HDR inks
        * Use 1 for SDR
        * The value you set for `colorMaximumLinearExposure` will tone-map down to the supported HDR headroom for your device’s screen
        * Or just use the screen’s supported value, which you can get from UIScreen or NSScreen
            * `UIScreen.potentialEHDRHeadroom`
            * `NSScreen.maximumPotentialExtendedDynamicRangeColorComponentValue`
    * By using the latest feature set and enabling HDR, you get all the new features
        * e.g. the new Reed tool, for calligraphy
    * Once the feature set is configured to your requirements, assign the customized feature set to both the markup controller and the insertion controller
        * Ensures consistency across the app’s markup and insertion functionality

```swift
// Customized markup FeatureSet
    
var featureSet: FeatureSet = .latest

featureSet.remove(.text)
featureSet.insert(.stickers)

// HDR support
featureSet.colorMaximumLinearExposure = 4
toolPicker.colorMaximumLinearExposure = 4

let paperViewController = PaperMarkupViewController(supportedFeatureSet: featureSet)
let markupEditViewController = MarkupEditViewController(supportedFeatureSet: featureSet)
```

* Can also customize the markup controller's contentView
    * Can be set to any UIView

```swift
// Custom background on markup controller

let template = UIImage(named: "MyTemplate.jpg")
let templateView = UIImageView(image: template)
paperViewController.contentView = templateView
```
