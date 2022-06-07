# **What's new in SwiftUI**

**Swift Charts**

* Charts build very similar to things like `List`
![](images/swiftui/party_chart1.png)
* Like `List`, you can also put a `ForEach` inside your chart to show more views, like a `RuleMark`
![](images/swiftui/party_chart2.png)
* Handle localization, dark mode, and dynamic type automatically
* Hello Swift Charts #session
* Swift Charts: Rais the bar #session


**Navigation and windows**

* Stacks
	* New `NavigationStack` container view
		* push/pop style navigation
		* works with `NavigationLink` and `.navigationTitle`
	* can now write better data-driven navigation with `.navigationDestination`
		![](images/swiftui/navigation_destination.png)
	*  You can also represent the current navigation stack as explicit state, and jump back to a specific view in the stack
		![](images//swiftuinavigation_explicit.png)
* `NavigationSplitView`
	* Declares two/three column layouts
	* Will automatically collapse into a stack on smaller size classes
		![](images/n/swiftuiavigation_split_view.png)
	* `NavigationSplitView` was designed to work with `NavigationStack`
		![](images//swiftuinavigation_duo.png)
* Updates allow for:
	* Heterogenous navigation paths
	* Advanced deep-linking
	* State restoration
	* Migrating from `NavigationView`
* [The SwiftUI cookbook for navigation](The SwiftUI cookbook for navigation.md) session
* Scenes
	* New `Window` keyword
		* Declares a single unique window for the app
		* Can assign it a keyboard shortcut
		* `@Environment(.openWindow)` to programmatically open new windows
		* modifiers for default size, position. resizability, and more
			* changing these values are remembered across app launches
	* `presentationDetents` modifier for sheets
		* can define multiple states
		![](images/swiftui/half_sheet.png)
	* [What's new in Xcode](What's new in Xcode.md) session
	* Use Xcode to develop a multiplatform app #session
	* Menubar extras (for macOS) can be built entirely in SwiftUI
	* Bring multiple windows to your SwiftUI app #session


**Advanced controls**

* Forms
	* New `.grouped` formStyle for macOS
	* `LabeledContent` View automatically adjust styling and allows selection of text
		* Can wrap any kind of view
		![](images/swiftui/labeled_content.png)
	* `Toggle` (and other SwiftUI form views) now automatically takes multiple pieces of text and use them as titles and subtitles
	![](images/swiftui/toggle_titles.png)
* Controls
	* `TextField` can now be configured to expand using the new `axis` parameter
		* `.lineLimit` now takes a range, like `.lineLimit(5...10)`
	* `MultiDatePicker` allows for non-contiguous date selection
	* Mixed-state controls
		* Allow for groups that each have individual bindings, but a parrent that takes a collection of targets
		![](images/swiftui/mixed_state_controls.png)
		* Pickers can work the same way
	* Steppers now allow providing a format for its values with the `format` parameter
	* watchOS now has `.accessibilityQuickActions` that define what functions accessibility actions perform
* Tables
	* Tables are now supported on iPadOS
		* use same API as macOS
		![](images/swiftui/ipad_table.png)
		* will also render on compact sizes, like on iPhone (showing just the primary column)
	* iPad toolbars can now support user customization and reordering
		* implement explicit identifiers for each toolbar item
		* not all toolbar items allow customization
	![](images/swiftui/customize_toolbar.png)
	* Search fields can now support tokenized inputs and suggestions to help build more structured search queries
		![](images/swiftui/search_tokenized.png)
	* Search scopes are now supported for filtering results
		* appear as scope bar beneath toolbar on macos
		* as a segmented control beneath the search bar on iOS
		![](images/swiftui/search_scope.png)
* More #sessions:
	* SwiftUI on iPad: Organize your interface
	* SwiftUI on iPad: Add toolbars, and documents, and more
	* What's new in iPad app design
	* SwiftUI on the Mac: Build the fundamentals


**Sharing**

* Photos
	* `PhotosPicker` view
	![](images/swiftui/photos_picker.png)
* Sharing
	* `ShareLink` view
		* use in a context menu or toolbar, and it auto creates a standard share button
		* adapt to the context/platform they apply to
		![](images/swiftui/sharelink.png)
* Transferable
	* Utilized by PhotosPicker and ShareLink
	* Defines how types are transferred across applications
	* Powers drag and drop
		* `.dropDestination` API
		![](images/swiftui/drop_destination.png)
		* many standard types already conform to Transferable:
			* String
			* Data
			* URL
			* Attributed String
			* Image
			* More...
	* Can be declared for your own custom types
	![](images/swiftui/transferable.png)
	* Meet Transferable #session


**Graphics and layout**

* Shape Styles
	* `Color` has a new `.gradient` property that defines a subtle gradient from the color
	* `ShapeStyle` got a new `.shadow` modifier
	![](images/swiftui/shape_shadow.png)
	* SwiftUI preview variants allow you to view multiple views/orientations/modes without writing any configuration code
	* Previews now run in live mode by default
	* Text can now be animated between weights, styles, and even layouts using `withAnimation`
* Layout
	* `Grid` is a new container view
		* Arranges views in 2D grid
		* Measures subviews up front to enable cells that expand multiple columns, and allow automatic alignment across rows and columns
		* Can be built using `Grid`, `GridRow`, and `.gridCellColumns` you can build a grid
		![](images/swiftui/grid.png)
	* `AnyLayout` protocol allows creation of custom, reusable layouts
		* Compose custom layouts with SwiftUI #session
		![](images/swiftui/anylayout.png)
		



