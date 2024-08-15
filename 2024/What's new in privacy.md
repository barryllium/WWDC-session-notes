# [**What's new in privacy**](https://developer.apple.com/videos/play/wwdc2024-10123)

---

* Privacy pillars
    * Data minimization
    * On-device processing
    * Transparency and control
    * Security protections

### **New pickers**

* FinanceKit
    * Offers access to a new category of on-device data from sources like Apple Card, Savings for Card, and Apple Cash
    * Transaction picker - access a set of existing transaction records, not a complete and updating history
    * Ongoing access control
        * When app requests access to all financial data, the OS presents a list of all eligible accounts
        * Can choose which accounts to share, and for each account, the earliest data to share
    * [**Meet FinanceKit**](https://developer.apple.com/videos/play/wwdc2024-2023) session
* Image Playgrounds
    * Access to the system-provided image generation capabilities also used in the Image Playground app
* AccessorySetupKit
    * Gives app Bluetooth and Wi-Fi access to relevant accessories
    * Combines 3 formerly required permissions:
        * Bluetooth pairing prompt
        * Wi-Fi network join prompt
        * Bluetooth accessory pairing confirmation
    * Improved accessory management
        * Accessory naming
        * Accessory assets
        * Accessory cleanup
        * Per-app scoped permissions
    * [**Meet AccessorySetupKit**](https://developer.apple.com/videos/play/wwdc2024-10203) session

### **Upgraded protections**

* Private Wi-Fi
    * Private wi-fi controls for MAC address rotation on iOS
    * Bringing MAC address protection to macOS
    * Private Wi-Fi addresses now rotate on iOS and macOS
        * iOS will use random, per-network MAC addresses
        * When `Rotate Wi-Fi Address` is on for a network, the MAC address will change approximately every two weeks
        * Addresses for forgotten networks will always rotate after at most 24 hours
    * For public networks, the Rotate Wi-Fi Address setting will default to a static rotating address
    * In iOS 18, in Wi-Fi settings, `Private Wi-Fi Address` is now `Rotate Wi-Fi Address`
    * Now provides MAC address rotation in addition to random initialization
    * Default state and behavior of the toggle is based ont eh type of the network a device is connected to
* macOS extensions
    * macOS now provides a system notification when extensions are installed
    * Extensions can still be disabled in `Login Items & Extensions`
    * Now support for additional types of extensions:
        * Spotlight importers
        * Dock tiles
        * Smart card reader devices
        * Color panels
        * Media extensions
        * Cron (disabled by default)
    * Login items and extensions now live in a single place in macOS System Settings - `General -> Login Items & Extensions`
    * Directory Services plug-in, legacy QuickLook plug-ins, and `com.apple.loginitems.plist` are no longer supported
* App group container protection
    * Brings protections of sandboxing to groups of apps, and apps that aren't ready to sandbox all of their data yet
    * If one app tries to access another app's container, a prompt is presented to allow or deny access
        * This now extends to shared containers for a group of multiple apps
    * Protected group containers can now be used to protect a subset of the data for an app
    * Best practices
        * Declare app groups entitlement
        * Format identifiers correctly
        * Use only `containerURL(forSecurityApplicationGroupIdentifier:)` to get the path to the shared container
    * [**App Groups Entitlement**](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_security_application-groups) documentation

### **Permission changes**

* Sharing Contacts
    * iOS 18 provides the ability to allow only specific contacts
        * New flow is automatic
    * Contact Access Button
        * Incremental access
        * Seamlessly blends into your UI
        * Shows contact search result
        * Only requires one tap
    * [**Meet the Contact Access Button**](./Meet%20the%20Contact%20Access%20Button.md) session
* Bluetooth
    * Updated Bluetooth prompt now shows a map that indicates the current location of the device, as well as a sample of associate devices
    * Clear usage string explains how your app will use Bluetooth
    * iOS 18 shows the updated prompt without the need to adopt any new APIs
* Local Network
    * macOS Sequoia has a new prompt for Local Network Access
    * Request access in a contextual moment
    * Provide a descriptive usage string in Info.plist
        * Must provide a Local Network Usage description if you have a Bonjour Services key or the Networking Multicast entitlement, or access will be blocked
    * For use with
        * Bonjour browsing or advertising
        * Custom multicast
        * Custom broadcast
        * Unicast connections (local network)
    * [**Support local network privacy in your app**](https://developer.apple.com/videos/play/wwdc2020/10110/) session from WWDC 2020

### **New platform capabilities**

* Locked and hidden apps
    * Contents of app are not accessible without authenticating with Face ID, Touch ID, or passcode first
    * Includes all entry points, such as tapping on an action from the share sheet
    * Contents of the app will not appear in other places across the system like search or notifications
* Automatic passkey upgrades
    * Apps can automatically upgrade existing accounts to use passkeys when signing in
    * Recommended to add support for passkeys
    * Passkey upgrades are automatic
    * Works with existing login flows
    * [**Streamline sign-in with passkey upgrades and credential managers**](./) session
* Private caller ID
    * Open source resources for configuring your server will be available late 2024
    * [**Getting up-to-date calling and blocking information for your app**](https://developer.apple.com/documentation/sms_and_call_reporting/getting_up-to-date_calling_and_blocking_information_for_your_app) documentation
