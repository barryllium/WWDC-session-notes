Coming Soon# [**The SwiftUI cookbook for focus**](https://developer.apple.com/videos/play/wwdc2023/10162/)

---

### **What is focus**

* Focus is a tool for deciding how to respond when someone presses a key on a keyboard, swipes on an Apple TV Remote, or turns the Digital Crown on their watch
    * On their own, they don't provide enough information to identify which on-screen control their input is intended for
* Emphasizing the focus view helps by:
    * Indicating where input goes
    * Provides a visual reference point

### **Ingredients**

#### Focusable views

* The view the system uses as a starting point when responding to focus input
    * e.g. Text fields - controls focus for editing, it captures continuous focus input
    * Buttons are not a focusable view by default, they do not gain focus when you tap them
        * Turning on keyboard navigation in the Keyboard Settings pane of macOS System SEttings changes this
    * iOS 17 and macOS Sonoma have new APIs for custom controls to participate in a focus system
        * Can specify the type of focus interactions your control supports (`.edit` and `.activate`)
        * If you don't provide arguments, the system gives the control focus for all interactions

```swift
// Focusable views

struct RecipeGrid: View {
    var body: some View {
        LazyVGrid(columns: [GridItem(), GridItem()]) {
            ForEach(0..<4) { _ in Capsule() }
        }
        .focusable(interactions: .edit)
    }
}

struct RatingPicker: View {
    var body: some View {
        HStack { Capsule() ; Capsule() }
            .focusable(interactions: .activate)
    }
}
```

#### Focus state

* The system keeps track of which view has focus, and the app can use that information in its logic to determine how to handle input and how to style views
* To observe the state of the system, you create bindings that associate values that you provide with focus being on a particular view
* Views can read these bindings to get notified when focus changes, such as a view becoming focused, or when focus is dismissed

```swift
// Focus state

struct GroceryListView: View {
    @FocusState private var isItemFocused
    @State private var itemName = ""

    var body: some View {
        TextField("Item Name", text: $itemName)
            .focused($isItemFocused)

        Button("Done") { isItemFocused = false }
            .disabled(!isItemFocused)
    }
}
```

#### Focused values

* The Focused Values API solves the problem of how to build data dependencies that link remote parts of your user interface
* Use this API to update your app's commands based on what's happening in the active scene

```swift
// Focused values

struct SelectedRecipeKey: FocusedValueKey {
    typealias Value = Binding<Recipe>
}

extension FocusedValues {
    var selectedRecipe: Binding<Recipe>? {
        get { self[SelectedRecipeKey.self] }
        set { self[SelectedRecipeKey.self] = newValue }
    }
}

struct RecipeView: View {
    @Binding var recipe: Recipe

    var body: some View {
        VStack {
            Text(recipe.title)
        }
        .focusedSceneValue(\.selectedRecipe, $recipe)
    }
}

struct RecipeCommands: Commands {
    @FocusedBinding(\.selectedRecipe) private var selectedRecipe: Recipe?

    var body: some Commands {
        CommandMenu("Recipe") {
            Button("Add to Grocery List") {
                if let selectedRecipe {
                    addRecipe(selectedRecipe)
                }
            }
            .disabled(selectedRecipe == nil)
        }
    }

    private func addRecipe(_ recipe: Recipe) { /* ... */ }
}

struct Recipe: Hashable, Identifiable {
    let id = UUID()
    var title = ""
    var isFavorite = false
}
```

#### Focus sections

* Focus sections give you a way to influence how focus moves when someone swipes on an Apple TV Remote or presses the Tab key on a keyboard
* Focus sections become targets for movement gestures, but they don't become focusable - they guide focus towards the nearest focusable content
    * To be effective, the focus sections have to take up more space than their contents

```swift
// Focus sections

struct ContentView: View {
    @State private var favorites = Recipe.examples
    @State private var selection = Recipe.examples.first!

    var body: some View {
        VStack {
            HStack {
                ForEach(favorites) { recipe in
                    Button(recipe.name) { selection = recipe }
                }
            }

            Image(selection.imageName)

            HStack {
                Spacer()
                Button("Add to Grocery List") { addIngredients(selection) }
                Spacer()
            }
            .focusSection()
        }
    }

    private func addIngredients(_ recipe: Recipe) { /* ... */ }
}

struct Recipe: Hashable, Identifiable {
    static let examples: [Recipe] = [
        Recipe(name: "Apple Pie"),
        Recipe(name: "Baklava"),
        Recipe(name: "Crème Brûlée")
    ]

    let id = UUID()
    var name = ""
    var imageName = ""
}
```

### **Recipes**

* First a recipe to automatically put focus on the empty item whenever a list appears
    * Use the ID of the item on the list to track state of the focused item in a `@FocusState` variable
    * Use `.defaultFocus($focusedItem, list.items.last?.id)` to have the system evaluate focus the when the screen appears
* Next, in our `addEmptyItem()` function, we can move focus to every new field as it is added

```swift
struct GroceryListView: View {
    @State private var list = GroceryList.examples
    @FocusState private var focusedItem: GroceryList.Item.ID?

    var body: some View {
        NavigationStack {
            List($list.items) { $item in
                HStack {
                    Toggle("Obtained", isOn: $item.isObtained)
                    TextField("Item Name", text: $item.name)
                        .onSubmit { addEmptyItem() }
                        .focused($focusedItem, equals: item.id)
                }
            }
            .defaultFocus($focusedItem, list.items.last?.id)
            .toggleStyle(.checklist)
        }
        .toolbar {
            Button(action: addEmptyItem) {
                Label("New Item", systemImage: "plus")
            }
        }
    }

    private func addEmptyItem() {
        let newItem = list.addItem()
        focusedItem = newItem.id
    }
}

struct GroceryList: Codable {
    static let examples = GroceryList(items: [
        GroceryList.Item(name: "Apples"),
        GroceryList.Item(name: "Lasagna"),
        GroceryList.Item(name: "")
    ])

    struct Item: Codable, Hashable, Identifiable {
        var id = UUID()
        var name: String
        var isObtained: Bool = false
    }

    var items: [Item] = []

    mutating func addItem() -> Item {
        let item = GroceryList.Item(name: "")
        items.append(item)
        return item
    }
}

struct ChecklistToggleStyle: ToggleStyle {
    func makeBody(configuration: Configuration) -> some View {
        Button {
            configuration.isOn.toggle()
        } label: {
            Image(systemName: configuration.isOn ? "checkmark.circle.fill" : "circle.dashed")
                .foregroundStyle(configuration.isOn ? .green : .gray)
                .font(.system(size: 20))
                .contentTransition(.symbolEffect)
                .animation(.linear, value: configuration.isOn)
        }
        .buttonStyle(.plain)
        .contentShape(.circle)
    }
}

extension ToggleStyle where Self == ChecklistToggleStyle {
    static var checklist: ChecklistToggleStyle { .init() }
}
```

* Next recipe: We want to make a custom control that allows us to focus on the control by using the `Tab` key, and then switch between different emoji reactions using the arrow keys
    * First, we need to make the control accept focusable by adding the `.focusable()` modifier with no arguments
        * This has unwanted behavior - it gives us focus on click, which we don't want
        * We fix this by changing our modifier to only have activate focus: `.focusable(interactions: .activate)`
    * Next, we notice that the focus ring around the control is rectangular
        * We add a `.contentShape(.capsule)` above our `.focusable` modifier to set the focus ring shape
    * We can use the `onMoveCommand { direction in ... }` modifier to provide an action to perform in response to a platform-appropriate move command, like when arrow keys are pressed on a Mac keyboard, or the directional edges are tapped on an Apple TV remote
        * Control content should flip horizontally for folks using a right-to-left language, make sure your move command action uses the Environment's `layoutDirection` to account for this
    * To handle focus  watchOS, use the `digitalCrownRotation` modifier instead of the `onMoveCommand` modifier
        * Use the `isFocused` environment value to draw the familiar green border around the control when it has focus

```swift
struct RatingPicker: View {
    @Environment(\.layoutDirection) private var layoutDirection
    @Binding var rating: Rating?

    #if os(watchOS)
    @State private var digitalCrownRotation = 0.0
    #endif

    var body: some View {
        EmojiContainer { ratingOptions }
            .contentShape(.capsule)
            .focusable(interactions: .activate)
            #if os(macOS)
            .onMoveCommand { direction in
                selectRating(direction, layoutDirection: layoutDirection)
            }
            #endif
            #if os(watchOS)
            .digitalCrownRotation($digitalCrownRotation, from: 0, through: Double(Rating.allCases.count - 1), by: 1, sensitivity: .low)
            .onChange(of: digitalCrownRotation) { oldValue, newValue in
                if let rating = Rating(rawValue: Int(round(digitalCrownRotation))) {
                    self.rating = rating
                }
            }
            #endif
    }

    private var ratingOptions: some View {
        ForEach(Rating.allCases) { rating in
            EmojiView(rating: rating, isSelected: self.rating == rating) {
                self.rating = rating
            }
        }
    }

    #if os(macOS)
    private func selectRating(
        _ direction: MoveCommandDirection, layoutDirection: LayoutDirection
    ) {
        var direction = direction

        if layoutDirection == .rightToLeft {
            switch direction {
            case .left: direction = .right
            case .right: direction = .left
            default: break
            }
        }

        if let rating {
            switch direction {
            case .left:
                guard let previousRating = rating.previous else { return }
                self.rating = previousRating
            case .right:
                guard let nextRating = rating.next else { return }
                self.rating = nextRating
            default:
                break
            }
        }
    }
    #endif
}

private struct EmojiContainer<Content: View>: View {
    @Environment(\.isFocused) private var isFocused
    private var content: Content

    #if os(watchOS)
    private var strokeColor: Color {
        isFocused ? .green : .clear
    }
    #endif

    init(@ViewBuilder content: @escaping () -> Content) {
        self.content = content()
    }

    var body: some View {
        HStack(spacing: 2) {
            content
        }
            .frame(height: 32)
            .font(.system(size: 24))
            .padding(.horizontal, 8)
            .padding(.vertical, 6)
            .background(.quaternary)
            .clipShape(.capsule)
            #if os(watchOS)
            .overlay(
                Capsule()
                    .strokeBorder(strokeColor, lineWidth: 1.5)
            )
            #endif
    }
}

private struct EmojiView: View {
    var rating: Rating
    var isSelected: Bool
    var action: () -> Void

    var body: some View {
        ZStack {
            Circle()
                .fill(isSelected ? Color.accentColor : Color.clear)
            Text(verbatim: rating.emoji)
                .onTapGesture { action() }
                .accessibilityLabel(rating.localizedName)
        }
    }
}

enum Rating: Int, CaseIterable, Identifiable {
    case meh
    case yummy
    case delicious

    var id: RawValue { rawValue }

    var emoji: String {
        switch self {
        case .meh:
            return "😕"
        case .yummy:
            return "🙂"
        case .delicious:
            return "🥰"
        }
    }

    var localizedName: LocalizedStringKey {
        switch self {
        case .meh:
            return "Meh"
        case .yummy:
            return "Yummy"
        case .delicious:
            return "Delicious"
        }
    }

    var previous: Rating? {
        let ratings = Rating.allCases
        let index = ratings.firstIndex(of: self)!

        guard index != ratings.startIndex else {
            return nil
        }

        let previousIndex = ratings.index(before: index)
        return ratings[previousIndex]
    }

    var next: Rating? {
        let ratings = Rating.allCases
        let index = ratings.firstIndex(of: self)!

        let nextIndex = ratings.index(after: index)
        guard nextIndex != ratings.endIndex else {
            return nil
        }

        return ratings[nextIndex]
    }
}
```

* Final recipe: focusable grid view that focuses on `Tab`, is navigable with the arrow keys, and selects an item on `Return`
    * First step is to make the grid `.focusable`
    * Use `.focusEffectDisabled()` to turn off the automatic focus ring
    * Use `SelectionShapeStyle` for borders and other indicators that a view is selected
        * Automatically adapts to my chosen accent color, and it turns gray when none of its ancestor views have focus, like when focus moves from the grid to the sidebar
    * Want to do is hook up a main menu command for marking the selected recipe as a favorite
        * Use `.focusedValue(\.selectedRecipe, $selection)` to do this, passing in a binding to the selection for menu commands to update as needed
    * Use the `onMoveCommand` modifier to support arrow key selection
    * Use the `onKeyPress(.return) { ... }` modifier (new in iOS 17 and macOS Sonoma) to capture if the user wants to respond to the `Return` key being entered when an image is selected
        * Return `.ignored` if the action did not perform when the key(s) were pressed to let the action propagate up the view hierarchy
        * Can also use `onKeyPress(characters: .alphanumerics, phases: .down)` to implement type selection, to quickly scroll down to and select a recipe by typing the first letter of its name

```swift
struct ContentView: View {
    @State private var recipes = Recipe.examples
    @State private var selection: Recipe.ID = Recipe.examples.first!.id
    @Environment(\.layoutDirection) private var layoutDirection

    var body: some View {
        LazyVGrid(columns: columns) {
            ForEach(recipes) { recipe in
                RecipeTile(recipe: recipe, isSelected: recipe.id == selection)
                    .id(recipe.id)
                    #if os(macOS)
                    .onTapGesture { selection = recipe.id }
                    .simultaneousGesture(TapGesture(count: 2).onEnded {
                        navigateToRecipe(id: recipe.id)
                    })
                    #else
                    .onTapGesture { navigateToRecipe(id: recipe.id) }
                    #endif
            }
        }
        .focusable()
        .focusEffectDisabled()
        .focusedValue(\.selectedRecipe, $selection)
        .onMoveCommand { direction in
            selectRecipe(direction, layoutDirection: layoutDirection)
        }
        .onKeyPress(.return) {
            navigateToRecipe(id: selection)
            return .handled
        }
        .onKeyPress(characters: .alphanumerics, phases: .down) { keyPress in
            selectRecipe(matching: keyPress.characters)
        }
    }

    private var columns: [GridItem] {
        [ GridItem(.adaptive(minimum: RecipeTile.size), spacing: 0) ]
    }

    private func navigateToRecipe(id: Recipe.ID) {
        // ...
    }

    private func selectRecipe(
        _ direction: MoveCommandDirection, layoutDirection: LayoutDirection
    ) {
        // ...
    }

    private func selectRecipe(matching characters: String) -> KeyPress.Result {
        // ...
        return .handled
    }
}

struct RecipeTile: View {
    static let size = 240.0
    static let selectionStrokeWidth = 4.0

    var recipe: Recipe
    var isSelected: Bool

    private var strokeStyle: AnyShapeStyle {
        isSelected
            ? AnyShapeStyle(.selection)
            : AnyShapeStyle(.clear)
    }

    var body: some View {
        VStack {
            RoundedRectangle(cornerRadius: 20)
                .fill(.background)
                .strokeBorder(
                    strokeStyle,
                    lineWidth: Self.selectionStrokeWidth)
                .frame(width: Self.size, height: Self.size)
            Text(recipe.name)
        }
    }
}

struct SelectedRecipeKey: FocusedValueKey {
    typealias Value = Binding<Recipe.ID>
}

extension FocusedValues {
    var selectedRecipe: Binding<Recipe.ID>? {
        get { self[SelectedRecipeKey.self] }
        set { self[SelectedRecipeKey.self] = newValue }
    }
}

struct RecipeCommands: Commands {
    @FocusedBinding(\.selectedRecipe) private var selectedRecipe: Recipe.ID?

    var body: some Commands {
        CommandMenu("Recipe") {
            Button("Add to Grocery List") {
                if let selectedRecipe {
                    addRecipe(selectedRecipe)
                }
            }
            .disabled(selectedRecipe == nil)
        }
    }

    private func addRecipe(_ recipe: Recipe.ID) { /* ... */ }
}

struct Recipe: Hashable, Identifiable {
    static let examples: [Recipe] = [
        Recipe(name: "Apple Pie"),
        Recipe(name: "Baklava"),
        Recipe(name: "Crème Brûlée")
    ]

    let id = UUID()
    var name = ""
    var imageName = ""
}
```

* Next, we want to _focus_ on how we can use the grid on tvOS
    * By default, tvOS uses the `lift` hover effect when showing focus on different views
        * We can now use the `highlight` effect in tvOS 17, which adds a perspective shift and a specular shine to the focused item
    * Finally, we add focus sections so we can easily swipe between the category buttons and the image grid

```swift
struct ContentView: View {
    var body: some View {
        HStack {
            VStack {
                List(["Dessert", "Pancake", "Salad", "Sandwich"], id: \.self) {
                    NavigationLink($0, destination: Color.gray)
                }
                Spacer()
            }
            .focusSection()

            ScrollView {
                LazyVGrid(columns: [GridItem(), GridItem()]) {
                    RoundedRectangle(cornerRadius: 5.0)
                        .focusable()
                }
            }
            .focusSection()
        }
    }
}
```
