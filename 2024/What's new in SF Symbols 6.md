# [**What's new in SF Symbols 6**](https://developer.apple.com/videos/play/wwdc2024-10188)

---

### **Animation presents**

* Animations introduced in [**What's new in SF Symbols 5**](../2023/What's%20new%20in%20SF%20Symbols%205.md) session from WWDC 2023
* Creating animated symbols covered in [**Create animated symbols**](../2023/Create%20animated%20symbols.md) session from WWDC 2023
* Animation presets
    * Adding Wiggle, Rotate, and Breathe
    * Updates to Replace and Variable Color
* Wiggle is directional
    * Horizontal
    * Vertical
    * Clockwise/Counterclockwise
    * At a custom angle
    * Many symbols have a built-in preferred direction
        * Can always be customized
* Rotate
    * Some symbols rotate entirely
    * Other symbols only rotate certain parts
        * Use the `By Layer` rotation option to rotate only part of a symbol
* Breathe
    * Smoothly increases and decreases the presence of a symbol
    * Similar to Pulse animation
        * Pulse conveys ongoing activity by changing its opacity
        * Breathe animates opacity *and* size
* Replace
    * A preset where a symbol is swapped with another
    * Magic Replace added this year
        * A smart transition between two symbols with related shapes
            * Slashes draw on and off, badges appear/disappear
        * If a symbol can not be replaced with magic replace, a standard replace animation is the fallback
            * Can still control what direction should be used for the fallback
        * Re-export custom SF Symbols 5 using symbol components to get Magic Replace functionality
* Variable Color
    * Closed loop animations repeat smoothly (when start state matches end state)

### **SF Symbols app**

* All new presets available in the animation inspector
    * Can choose animation directions there as well
* More control over how to repeat animations
    * Once
    * Repeat with delay
        * Customizable delay time
    * Continuous
* If you have a symbol that was previously annotated with a layer that opts into Pulse, Breathe will leverage that annotation when Pulses is selected
    * No additional steps needed
* For rotate, you can set the anchor to rotate around
    * By default, the center of the symbol (not the layer)
    * Snap to points button can be used to align with points for centering
    * Can inspect points and manually type in a coordinate
    * Must set the anchor point in all three weights
        * System will then interpolate the values for all other weights/scales

### **New symbols**

* Expanded automotive category
* New figures for activities
* New localized symbols, including scripts like Greek, Cyrillic, and several Indic numeral systems
* Progress indicators, haptics, home and widget relate symbols
* Over 800 new symbols
* Beta app found at [**developer.apple.com/sf-symbols**](https://developer.apple.com/sf-symbols)
