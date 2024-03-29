# [**Get the most out of Xcode Cloud**](https://developer.apple.com/videos/play/wwdc2022/110374/)

### **Xcode Cloud overview**

* Continuous integration and delivery service
* Brings together TestFlight, App Store Connect, and Testing in an online service
* [**Meet Xcode Cloud**](https://developer.apple.com/videos/play/wwdc2021/10267/) session from 2021

---

### **Review an existing Workflow**

Dashboard

* Overview shows when build was started, and how long each stage took in the build process

![](images/xcodeCloud/dashboard.png)
![](images/xcodeCloud/usage.png)

---

### **Xcode Cloud usage dashboard**

Usage Overview

![](images/xcodeCloud/usage_overview.png)

Usage Trends

![](images/xcodeCloud/usage_trends.png)

You can see how much time each of the workflows is using

![](images/xcodeCloud/usage_workflows.png)

---

### **Best Practices**

Check out [**Explore Xcode Cloud Workflows**](https://developer.apple.com/videos/play/wwdc2021/10268/) session from 2021

**Avoid unintended builds**

You can define start conditions based on:

* Branch changes
* Pull request changes
* Tag changes
* On a schedule for a branch

![](images/xcodeCloud/start_conditions.png)
![](images/xcodeCloud/release_branch.png)
 
You can exclude files/folders from triggering a build

![](images/xcodeCloud/exclude_folder.png)
 
**Select reasonable test destinations**

Make sure you are not testing on unnecessary test devices

All the devices | Just the recommended set
--------------- | ------------------------
![](images/xcodeCloud/all_devices.png) | ![](images/xcodeCloud/recommended_devices.png)

**Skip builds**

* put `[ci skip]` tag in your commit message to skip a CI build

**Optimize custom scripts and tests**

* Check out the [**Customize your advanced Xcode Cloud workflows**](https://developer.apple.com/videos/play/wwdc2021/10269/) session from 2021
* [**Author fast and reliable tests for Xcode Cloud**](./Author%20fast%20and%20reliable%20tests%20for%20Xcode%20Cloud.md) session

---

**Revisit optimized build**

Making the changes above did the following to the original workflow:

* build duration was 1m less
* usage was 4 minutes less
* Integration builds trended -7% time

For teams, check out the [**Deep Dive into Xcode Cloud for Teams**](./Deep%20dive%20into%20Xcode%20Cloud%20for%20teams.md) session
