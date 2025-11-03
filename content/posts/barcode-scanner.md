+++
date = '2025-10-18T11:57:54+02:00'
draft = false
title = 'Barcode scanner in BC - No camera!'
+++

> **Crucial**: As of right now (18.10.2025) the most recent android version of Business Central does not work with barcode scanning. The most recent *working* version I found was version `4-1-2281` from 2024! Every newer version would not pick up on broadcast intents!

# Introduction

The easiest way to create a functional barcode scanner integration using the default capabilities of Business Central is through the camera barcode scanner control add-in. While this works well enough, if you're working in a warehouse environment, you probably don't want to open the camera, scan one item, close it, and then check if the item was processed correctly. At this point, many suggest implementing a third-party module to help with warehousing and scanner integration—which is completely valid and works for most cases.

In this blog post, I want to talk about an intermediary option: scanning without a camera while still using BC standard functionality.

# What are we trying to achieve here?
The scenario is straightforward: we have a list of items to scan in Business Central and a hardware device with a built-in laser scanner. Instead of opening the camera to scan barcodes, we want to use a physical hardware button and laser scanner to capture barcodes with immediate feedback in BC.

# The best case scenario
The basic AL code setup for this is straightforward and covered in the MS Learn documentation. 
Microsoft provides this code sample as a starting point:
```al
page 50101 "Barcode Scanner"
{
    PageType = Card;
    ApplicationArea = All;
    Caption = 'Barcode Scanner Sample';

    layout
    {
        area(Content)
        {
            // Declare the user control based on the BarcodeScannerProviderAddIn control add-in.
            usercontrol(BarcodeControl; BarcodeScannerProviderAddIn) // Step 1
            {
                ApplicationArea = All;

                // The ControlAddInReady event is raised when the control add-in is ready to be used. 
                trigger ControlAddInReady(IsSupported: Boolean)
                begin
                    // If the barcode scanner is supported, request the barcode scanner.
                    if IsSupported then // Step 2
                        CurrPage.BarcodeControl.RequestBarcodeScannerAsync(); // Step 3
                end;

                // The BarcodeReceived event is raised when a barcode is received from the barcode scanner.
                trigger BarcodeReceived(Barcode: Text; Format: Text)
                begin
                    Message(Barcode); // Step 4
                end;
            }
        }
    }
}
```

As you can see, we use the control add-in `BarcodeScannerProviderAddIn`, the two triggers `ControlAddInReady` and `BarcodeReceived`, and the function `CurrPage.BarcodeControl.RequestBarcodeScannerAsync()`, and that's it! A clean and simple implementation covering everything we need to do on the BC side.

### Scanner settings

I've worked with three different devices from different manufacturers, and each had its own method for configuring these intents, so I can't provide universal step-by-step instructions. I worked most extensively with Zebra devices and their DataWedge application. However, there are a few key things you'll need to configure regardless of the device:

- Disable keyboard output when scanning a barcode—we'll catch the input directly in BC
- Set up the correct Android Intents, or discover which intents your device uses and configure them in BC (using the overload of `RequestBarcodeScannerAsync`)
- Set the delivery method to broadcast intent

## Android Intents Configuration

Now we need to configure the scanner device itself to communicate with BC. This is where understanding Android Intents becomes critical—and where things often get tricky.

You need to determine what intents your device sends when scanning a barcode. **This is crucial**: from experience, I strongly recommend finding documentation about these intents beforehand or contacting the manufacturer to ensure you have this information. Most devices allow you to configure the `Intent action` and `Intent category`, while the `Intent barcode string` and `Intent barcode type` are less commonly configurable.

Here are the default Business Central intent values:

|Scanner setting|Value|
|-|-|
|Intent delivery mode|broadcast intent|
|Intent action|com.businesscentral.barcode.receive_barcode|
|Intent category|com.businesscentral.barcode.receive_category|
|Intent barcode string|com.businesscentral.receive_barcode.barcode_string|
|Intent barcode type|com.businesscentral.receive_barcode.barcode_format|

Once you've configured these settings, you can run your initial test. Scan a barcode and check if the `BarcodeReceived` trigger is called! If it works, congratulations—you're done already! 

If not, let's dig deeper to find out what's actually happening and understand the technology behind this and how to troubleshoot.


# How to debug and troubleshoot

## The technology behind this

> **Disclaimer:** The information in this section is based purely on my own hands-on experience debugging and troubleshooting scanner integrations. I haven't conducted formal research. This is what I've learned through trial and error in real-world implementations. Take it as practical field knowledge.

### Android Intents

The most important part of this integration is understanding Android Intents. I first discovered these when the control add-in didn't work initially. You can think of Android Intents similarly to BC events:

**Intent delivery mode - broadcast intent:** This acts like an event publisher. Every time a barcode is scanned, the device sends a broadcast intent.

The broadcast intent carries all the other intent parameters listed in the table above. `Intent action` and `Intent category` are similar to the `<Event Publisher Object>` and `<Published Event Name>` in BC. When we call the `RequestBarcodeScannerAsync()` function, we're essentially registering a broadcast receiver that listens specifically for these two intents. The receiver only reacts to broadcast intents with the matching action and category.

Think of it like an AL event subscriber:
```al
[EventSubscriber(ObjectType::<Event Publisher Object Type>, <Event Publisher Object>, '<Published Event Name>', '<Published Event Element Name>', <SkipOnMissingLicense>, <SkipOnMissingPermission>)]
```

The remaining intent parameters carry the actual data:
- **`Intent barcode string`**: Contains the actual barcode text that was scanned
- **`Intent barcode type`**: Contains the barcode format (Code128, Code39, EAN-13, QR Code, etc.)

> **Important**: The event `BarcodeReceived` is only triggered if every intent is set up correctly.

### Troubleshooting Tools

#### Logcat
As mentioned earlier, broadcast intents are key to this integration. With Logcat, you can monitor when these broadcast intents are sent and view information about them. This is **extremely useful** for confirming whether a barcode is successfully scanned by the device and transmitted as a broadcast. You can also discover the intent action and category in these logs!

However, there's a limitation: the other intent parameters (`Intent barcode string` and `Intent barcode type`) are hidden and only show as `has extras` in the log—which is frustrating. I haven't found a way to view their actual values in Logcat.

To read these logs, you need to set up an ADB connection:
```bash
adb.exe logcat
```

![logcat_broadcast](/blog/broadcastintent.png)

#### ADB (Android Debug Bridge)

ADB was probably the most valuable tool I discovered during this journey of debugging two black boxes. It's a command-line tool for debugging Android devices and is surprisingly easy to set up.

**Setting up ADB:**

1. **Enable Developer Mode on your device:** Tap on "Build number" in your Android device settings 7 times to unlock Developer options
2. **Enable USB debugging:** Go to Developer options and enable USB debugging
3. **Download ADB:** Get the platform tools from [here](https://developer.android.com/tools/releases/platform-tools). It's just a zip file—extract it and you'll find `adb.exe`. No installation required!

**Why ADB is so powerful:**

Beyond viewing logs, ADB allows you to send your own broadcast intents directly to the device! This means you can test the Business Central integration without physically scanning anything—incredibly useful for debugging.

**Common ADB commands:**
```bash
# List connected devices
adb.exe devices

adb.exe install "dynamics-365-business-central-4-1-2281.apk"

# Send a test broadcast intent (simulates a barcode scan!)
adb.exe shell am broadcast -a com.businesscentral.barcode.receive_barcode -c com.businesscentral.barcode.receive_category --es com.symbol.datawedge.data_string "test" --es com.symbol.datawedge.label_type "qrcode"
```

#### Android Studio - Virtual Device Emulator

Android Studio allows you to quickly set up virtual Android devices on your PC. This is especially helpful when combined with the ADB broadcast command mentioned above. Together, these tools let you set up a fully functional Android environment on your computer and test the BC integration without needing physical scanner hardware. Virtual devices come with debugging mode enabled by default, and you can view Logcat output directly from Android Studio (there's a button in the bottom left panel).

**Setup tips:**
- You need to create a project to access the virtual device manager—just create an empty project
- The button to create virtual devices is in the top right toolbar (Device Manager icon)

![Android Studio](/blog/AndroidStudios.png)


#### Cordova Plugin

I need to at least mention Cordova. I discovered cordova in the JS of the control addin and I spend a lot of time troubleshooting in the wrong direction. The Business Central mobile app uses a Cordova plugin to handle various functions, including receiving Android Intents. When you open a page with the scanner control add-in, BC checks whether Cordova is available—if it's not.

**Important clarification:** The Cordova plugin implementation is entirely within the BC mobile app itself, not part of the mobile device's OS. You don't need to find a device with special "Cordova compatibility." The plugin uses standard Android APIs, and the only requirement is that the device runs at least Android 11.

To summarize: The BC mobile app uses cordova, but there is not much we need or can do about it's implementation. It's good to know it's there, but not much use beyond that.

#### Additional Notes and Discoveries

Some interesting things I learned during this investigation:

- **ProjectMadeira:** If you come across references to `ProjectMadeira` in logs or configurations, that's just Business Central
- **Control Add-in Source:** While debugging, I examined the JavaScript for the control add-in. Fair warning: the control add-in functionality is embedded in a minified, 28,000-line JavaScript amalgamation. Tread carefully, or you'll fall down a rabbit hole!
- **Broadcast Receiver Registration:** The Android BC app is where the broadcast receiver is registered
- **Dynamic vs. Static Receivers:** BC uses dynamic broadcast receivers (registered at runtime). Unlike static receivers, I wasn't able to get a list of registered dynamic receivers, which made debugging more challenging

## Zebra Device Setup

On Zebra devices, you'll find an app called **DataWedge**. This is where you configure the barcode scanner integration:

1. Create a new profile in DataWedge
2. Set `ProjectMadeira` as the associated application
3. Configure the following settings:
   - **Disable** keyboard output
   - **Enable** Intent output
   - Edit the Intent Action and Intent Category to match BC's expected values
   - Set Intent delivery to **Broadcast Intent**

Use these parameters in your control add-in for Zebra devices:
```al
CurrPage.BarcodeControl.RequestBarcodeScannerAsync(
    'com.businesscentral.barcode.receive_barcode',
    'com.businesscentral.barcode.receive_category',
    'com.symbol.datawedge.data_string',
    'com.symbol.datawedge.label_type'
);
```

You can find more details about Zebra's default intent outputs under "Single Decode Mode" in the [Zebra Intent Output documentation](https://techdocs.zebra.com/datawedge/15-0/guide/output/intent/).

## Links and Resources

- [MS Learn - Barcode Scanning in BC Mobile App](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-mobile-app-barcode-scanning#scenario-3-integrate-dedicated-barcode-scanners) - Official documentation (brief but helpful)
- [BC Mobile App Requirements](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-mobile-app-barcode-scanning) - Android 11+ requirement
- [Zebra DataWedge Intent Output Documentation](https://techdocs.zebra.com/datawedge/15-0/guide/output/intent/) - Detailed intent configuration for Zebra devices
- [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools) - Download ADB
