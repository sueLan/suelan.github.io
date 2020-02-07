---
title: iOS Simulator from the Command Line
date: 2020-02-04 16:44:41
categories:
    - iOS 
tags:
    - Simulator
---

`xcrun simctl` controls iOS simulators. 

When we run `xcrun simctl help`, here are some subcommands. 

```shell 
Subcommands:
        create              Create a new device.
        clone               Clone an existing device.
        upgrade             Upgrade a device to a newer runtime.
        delete              Delete spcified devices, unavailable devices, or all devices.
        pair                Create a new watch and phone pair.
        unpair              Unpair a watch and phone pair.
        pair_activate       Set a given pair as active.
        erase               Erase a device's contents and settings.
        boot                Boot a device.
        shutdown            Shutdown a device.
        rename              Rename a device.
        getenv              Print an environment variable from a running device.
        openurl             Open a URL in a device.
        addmedia            Add photos, live photos, videos, or contacts to the library of a device.
        install             Install an app on a device.
        uninstall           Uninstall an app from a device.
        get_app_container   Print the path of the installed app's container
        launch              Launch an application by identifier on a device.
        terminate           Terminate an application by identifier on a device.
        spawn               Spawn a process by executing a given executable on a device.
        list                List available devices, device types, runtimes, or device pairs.
        icloud_sync         Trigger iCloud sync on a device.
        pbsync              Sync the pasteboard content from one pasteboard to another.
        pbcopy              Copy standard input onto the device pasteboard.
        pbpaste             Print the contents of the device's pasteboard to standard output.
        help                Prints the usage for a given subcommand.
        io                  Set up a device IO operation.
        diagnose            Collect diagnostic information and logs.
        logverbose          enable or disable verbose logging for a device
        status_bar          Set or clear status bar overrides
```

Also, we can use `xcrun simctl help [subcommand]` to seek help for a specific subcommand. Like `xcrun simctl help boot` 


1. use `xcrun simctl list` to see all the simulators
  
```
== Device Types ==
iPhone 11 (com.apple.CoreSimulator.SimDeviceType.iPhone-11)
iPhone 11 Pro (com.apple.CoreSimulator.SimDeviceType.iPhone-11-Pro)
iPhone 11 Pro Max (com.apple.CoreSimulator.SimDeviceType.iPhone-11-Pro-Max)

== Runtimes ==
iOS 12.4 (12.4 - 16G73) - com.apple.CoreSimulator.SimRuntime.iOS-12-4 
iOS 13.3 (13.3 - 17C45) - com.apple.CoreSimulator.SimRuntime.iOS-13-3 
tvOS 13.3 (13.3 - 17K446) - com.apple.CoreSimulator.SimRuntime.tvOS-13-3 
watchOS 6.1 (6.1.1 - 17S445) - com.apple.CoreSimulator.SimRuntime.watchOS-6-1 

== Devices ==
-- iOS 13.3 --
iPhone 11 Pro (707DDBF0-13D4-4C2D-B87F-4276EB340E21) (Shutdown) 
iPhone 11 Pro Max (34C9AC6A-577D-4CEF-8B10-20011DCFBA27) (Shutdown) 
```

Use `xcrun simctl list device` to see the list of device info. Or use a device name to get paired devices with the same name  

`xcrun simctl list devices "iPhone 11 Pro Max"` 

```
== Devices ==
-- iOS 12.4 --
-- iOS 13.3 --
    iPhone 11 Pro Max (34C9AC6A-577D-4CEF-8B10-20011DCFBA27) (Shutdown) 
-- tvOS 13.3 --
-- watchOS 6.1 --
-- Unavailable: com.apple.CoreSimulator.SimRuntime.iOS-13-0 --
    iPhone 11 Pro Max (893827B6-91E0-417C-A179-68343F695040) (Creating) (unavailable, runtime profile not found)
-- Unavailable: com.apple.CoreSimulator.SimRuntime.iOS-13-1 --
    iPhone 11 Pro Max (F4B573E0-4106-4FF2-ADA8-16DCC053026C) (Shutdown) (unavailable, runtime profile not found)
-- Unavailable: com.apple.CoreSimulator.SimRuntime.iOS-13-2 --
    iPhone 11 Pro Max (B63C96BD-1CE2-499B-8387-3B8AEBF50931) (Creating) (unavailable, runtime profile not found)
```    

1. Boot a device using $uuid 

`xcrun simctl boot 34C9AC6A-577D-4CEF-8B10-20011DCFBA27` 

Sometimes, you may want to boot a specific device. This command is quite helpful. 
 


 https://www.iosdev.recipes/simctl/