# **Get to know Developer Mode**

**What is Developer Mode**

* A new mode for iOS 16 and watchOS 9 that enables developer workflows
* disabled by default and requires explicit enrollment
* Persists across reboots and system updates
* Setup can be automated
* Why Developer Mode?
	* Improves security for vast majority of users that aren't devs
		* Developer features are being abused in targeted attacks
		* Most people don't need to use developer features by default
* Developer Mode is only required for local development; it's not required for
	* TestFlight
	* Enterprise distribution
	* App Store


**Using Developer Mode**

* Turn on if you need to
	* Run/install development signed applications (including signed using a Personal Team)
	* Debug/instrument your applications
	* Automate testing
* To turn on
	* Connect device to Xcode for the menu item to appear
		* iOS 16 beta releases have this menu visible at all times
	* Installing an application without Xcode will also make it visible
	* Developer Mode controls live in Settings -> Privacy & Security
* Turning on Developer mode requires you to restart your device
	* On restart, you will be prompted again to turn on Developer Mode
	* Setting will persist after confirmation


**Automation Flows**

* Devices w/o passcodes can be automatically enrolled into Developer Mode
* macOS Ventura ships with `devmodectl` that can do this for you:
	* Single device one-off mode
	* Streaming mode