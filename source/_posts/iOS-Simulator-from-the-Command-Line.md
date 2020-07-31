---
title: iOS Simulator from the Command Line
date: 2020-02-05 12:44:41
categories:
    - iOS 
tags:
    - Simulator
    - WWDC
    - Debug
---

`xcrun simctl` is command utils to control iOS simulator, just like  [adb](https://developer.android.com/studio/command-line/adb?hl=en) for Android. Sometimes, in CI server script, We need these simulator-integration command to interact with simulators and run test cases. 

 If we run `xcrun simctl help`, here are some subcommands. When successful, most of these commands exit with 0; when failed, most exit with a non-zero number. 

```
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
        ui                  Get or Set UI options
        push                Send a simulated push notification
        privacy             Grant, revoke, or reset privacy and permissions
        keychain            Manipulate a device's keychain
```
Just see I can do a lot of things with these command. If wanting to know more about a specific subcommand, we can use `xcrun simctl help [subcommand]` to seek help. Like, `xcrun simctl help boot` 


### Simulator info 

use `xcrun simctl list` to see all the simulator information. We can get a list of device types, a list of info of runtime names, a list of device names. 
  
```
== Device Types ==
iPhone 11 Pro (com.apple.CoreSimulator.SimDeviceType.iPhone-11-Pro)
iPhone 11 Pro Max (com.apple.CoreSimulator.SimDeviceType.iP

== Runtimes ==
iOS 13.3 (13.3 - 17C45) - com.apple.CoreSimulator.SimRuntime.iOS-13-3 

== Devices ==
-- iOS 13.3 --
iPhone 11 Pro Max (34C9AC6A-577D-4CEF-8B10-20011DCFBA27) (Shutdown) 
```

Use `xcrun simctl list device` to see the list of device info. Or use a device name to get paired devices' info, like name, uuid, and status. For example,  

```
xcrun simctl list devices "iPhone 11 Pro Max"
```

```
== Devices ==
-- iOS 12.4 --
-- iOS 13.3 --
    iPhone 11 Pro Max (34C9AC6A-577D-4CEF-8B10-20011DCFBA27) (Shutdown) 
-- tvOS 13.3 --
-- watchOS 6.1 --
-- Unavailale: com.apple.CoreSimulator.SimRuntime.iOS-13-0 --
    iPhone 11 Pro Max (893827B6-91E0-417C-A179-68343F695040) (Creating) (unavailable, runtime profile not found)
-- Unavailable: com.apple.CoreSimulator.SimRuntime.iOS-13-1 --
    iPhone 11 Pro Max (F4B573E0-4106-4FF2-ADA8-16DCC053026C) (Shutdown) (unavailable, runtime profile not found)
-- Unavailable: com.apple.CoreSimulator.SimRuntime.iOS-13-2 --
    iPhone 11 Pro Max (B63C96BD-1CE2-499B-8387-3B8AEBF50931) (Creating) (unavailable, runtime profile not found)
```

### Get device environment variable 

`xcrun simctl getenv` is for printing an environment variable from a running device. 

```
Usage: simctl getenv <device> <variable name>
```


Take the follow command as an example, the `SIMULATOR_UDID` environment variable contains the UDID of the simulated device. But it doesn't work for physical device. 

```
xcrun simctl getenv booted SIMULATOR_UDID
9362BF90-132F-4894-A057-CEBFEC9C1FB6
```

We can add more environment variables by ourselves in schema for debugging use. 
![](environment_variable.png)

[Launch Arguments & Environment Variables](https://nshipster.com/launch-arguments-and-environment-variables/)

### Create a custom simulator


`xcrun simctl create <name> <device type> <runtime>`
For example, if I would like to create a iPhone 11 Pro Max with iOS 13.3, I can use the follow command.

```
xcrun simctl create "ry" "iPhone 11 Pro Max" iOS13.3  
BE9A72F0-5793-447B-BEC4-63A73242BED5 
```



The `uuid` of the new simulator is `BE9A72F0-5793-447B-BEC4-63A73242BED5`, which is output in standard out. And errors comes to standard error. 

> Tips: you should use available runtime, or you will get an error of `invalid runtime: xxx`. 

If in shell scrip, we can capture the new device's name using environment variable:

```
NEW_DEVICE=$(xcrun simctl create "Test Phone" "iPhone XR" iOS13.0)
echo "🤖 Created ${NEW_DEVICE}"
🤖 Created BE9A72F0-5793-447B-BEC4-63A73242BED5
```

Another way to create a simulator using GUI is to go to `Window` -> `Devices and Simulators` 

![create](create.png)
   
### Boot a simulator 

Boot a device using `$uuid` 

`xcrun simctl boot BE9A72F0-5793-447B-BEC4-63A73242BED5` 

Use `simctl shutdown <device>` to shutdown a device. 

### Install/Uninstall app 

`xcrun simctl install <device> <path>`. We use this command to install an app on a device. 

```
➜  xcrun simctl install booted ./demo.app
```

 `booted` is the alias name of the booted device. We can also use `uuid` of a device.

 use ` simctl uninstall <device> <app bundle identifier>` to uninstall an app. like 
 `xcrun simctl uninstall booted com.rong.lan.CubeTransitionAnimationDemo`  

### Launch 

`xcrun simctl launch <device> <bundle> <arguments>`. 

Launch command is used to launch a application in your simulator device.

```
xcrun simctl launch booted com.apple.example -MyDefaultKey YES
```

`com.apple.example` is bundle id of the application. 

If we use `--console-pty`,  launch command will connect the standard output and error of the application to the terminal line.

```
➜  xcrun simctl launch --console-pty booted com.rong.lan.CubeTransitionAnimationDemo

com.rong.lan.CubeTransitionAnimationDemo: 98045
2020-02-09 10:38:28.594 3DCubeTransitionAnimationDemo[98045:931065] gesture end translation x -281.000000; velocity-1056.111584
2020-02-09 10:38:28.594 3DCubeTransitionAnimationDemo[98045:931065] timer current tx -281.000332
2020-02-09 10:38:28.611 3DCubeTransitionAnimationDemo[98045:931065] timer current tx -285.030635
2020-02-09 10:38:28.627 3DCubeTransitionAnimationDemo[98045:931065] timer current tx -289.060938
```


 Use `xcrun simctl terminate <device> <bundle>` to terminate an application by identifier on a device.
### Container Path 

`xcrun simctl simctl get_app_container <device> <app bundle identifier> [<container>]`  command is used to get the path of the installed app's container.  

```
container   Optionally specify the container. Defaults to app.
  app                 The .app bundle
  data                The application's data container
  groups              The App Group containers
  <group identifier>  A specific App Group container
```


For example

```
➜ xcrun simctl get_app_container booted com.rong.lan.CubeTransitionAnimationDemo 
~/Library/Developer/CoreSimulator/Devices/BE9A72F0-5793-447B-BEC4-63A73242BED5/data/Containers/Bundle/Application/135A7B46-DE38-47EC-9871-D1F3615E9CE5/3DCubeTransitionAnimationDemo.app
➜ xcrun simctl get_app_container booted com.rong.lan.CubeTransitionAnimationDemo data 
~/Library/Developer/CoreSimulator/Devices/BE9A72F0-5793-447B-BEC4-63A73242BED5/data/Containers/Data/Application/10D64231-20B5-4DFE-B1E9-277AD9C02C4F
```

### Spawn 

Spawn command will pause xspawn, a process inside the simulator environment. 

```
xcrun simctl spawn booted defaults write com.example.app ResetDatabase -bool YES
```
Here, we use `defaults` utils, because we already have a booted device `BE9A72F0-5793-447B-BEC4-63A73242BED5`, we don't have to specify the device. 

`com.example.app` is bundle id of my application. We reset `ResetDatabase` to YES. 
This is a handy way to change the user defaults for the application before its running.  

### Log Stream 

spwan command would work with log stream utility. We can pass a predicate and filter the log output. Here the predicate is `senderImagePath CONTAINS "nsurlsessiond"`. We can debug something wrong with URL session. 

```
xcrun simctl spawn booted log stream --predicate 'senderImagePath CONTAINS "nsurlsessiond"'
```
![spwan](spawn_log.png)

Also, we can use different predicates to filter what we want.  

```
xcrun simctl spawn booted log stream — predicate ‘processImagePath endswith “xxx”’
xcrun simctl spawn booted log stream — predicate ‘eventMessage contains “error” and messageType == info’

```


### Dignose 

In a auto-test system, if having some kind of issue, by using this command, you can not only collect logs on disk but also capture ephemeral logging and dump system state. 

```
➜  xcrun simctl diagnose -l                              
Writing to /private/tmp/simctl_diagnose_2020_02_09.10-10-56+0800
Getting Simulator component versions...
Collecting CoreSimulator logs...
Collecting device information (this may take some time)...
This operation will time-out after 300 seconds.
Gathering 15 crash reports...
Compressing Archive...
Successfully wrote simctl diagnose archive to '/private/tmp/simctl_diagnose_2020_02_09.10-10-56+0800.tar.gz'
```

![diagnose](diagnose.png)

### Clone 
Clone is a very powerful command. See details in [wwdc2019/418](https://developer.apple.com/videos/play/wwdc2019/418/).  

`xcrun simctl clone <device> <clone name>`.  You can copy your custom device using this command. 

```
// boot base simulator ry
➜ xcrun simctl boot ry   
➜ xcrun simctl install ry ./demo.app
// Must shutdown it before clone 
➜ xcrun simctl shutdown all 
➜ xcrun simctl clone ry ry-1  
➜ xcrun simctl clone ry ry-2 
➜ xcrun simctl boot ry-1 && xcrun simctl boot ry-2 
```


Two new devices are created with the same contents. 

![clone](clone.png)


### Push Notification to simulator

1. Create a `.apns` file, `ExamplePush.apns` 

```
{
    "Simulator Target Bundle": "com.facebook.flipper",
    "aps": {
        "alert": "This is a simulated notification!",
        "badge": 3,
        "sound": "default"
    }
}
```

[valid Apple Push Notification values](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/generating_a_remote_notification). Noted that the `Target Bundle` should be the same as the `bundle identifier`  

2. Drag and drop an APNs file onto the target simulator. 

> The file must be a JSON file with a valid Apple Push Notification Service payload, including the “aps” key. It must also contain a top-level “Simulator Target Bundle” with a string value matching the target application‘s bundle identifier. --  https://stackoverflow.com/a/60085404/4026902



Or, we use command lines. 

```
xcrun simctl push booted com.facebook.flipper ExamplePush.apns
```


```
➜  ~ xcrun simctl push 
Send a simulated push notification
Usage: simctl push <device> [<bundle identifier>] (<json file> | -)

        bundle identifier
             The bundle identifier of the target application
             If the payload file contains a 'Simulator Target Bundle' top-level key this parameter may be omitted.
             If both are provided this argument will override the value from the payload.
        json file
             Path to a JSON payload or '-' to read from stdin. The payload must:
               - Contain an object at the top level.
               - Contain an 'aps' key with valid Apple Push Notification values.
               - Be 4096 bytes or less.

Only application remote push notifications are supported. VoIP, Complication, File Provider, and other types are not supported.

```

### Manipulate a device's keychain 

```
➜  ~ xcrun simctl keychain
Manipulate a device's keychain
Usage: simctl keychain <device> <action> [arguments]

    add-root-cert [path]
        Add a certificate to the trusted root store.

    add-cert [path]
        Add a certificate to the keychain.

    reset
        Reset the keychain.
```

```
xcrun simctl keychain booted add-root-cert myCA.cer
```

### Privacy and Permissions 

These commands, introduced in Xcode 11.4,  could be very useful when you are developing some features and have to test the permissions.  

```
➜  ~ xcrun simctl privacy                                         
Grant, revoke, or reset privacy and permissions
Usage: simctl privacy <device> <action> <service> [<bundle identifier>]

        action
             The action to take:
                 grant - Grant access without prompting. Requires bundle identifier.
                 revoke - Revoke access, denying all use of the service. Requires bundle identifier.
                 reset - Reset access, prompting on next use. Bundle identifier optional.
             Some permission changes will terminate the application if running.
        service
             The service:
                 all - Apply the action to all services.
                 calendar - Allow access to calendar.
                 contacts-limited - Allow access to basic contact info.
                 contacts - Allow access to full contact details.
                 location - Allow access to location services when app is in use.
                 location-always - Allow access to location services at all times.
                 photos-add - Allow adding photos to the photo library.
                 photos - Allow full access to the photo library.
                 media-library - Allow access to the media library.
                 microphone - Allow access to audio input.
                 motion - Allow access to motion and fitness data.
                 reminders - Allow access to reminders.
                 siri - Allow use of the app with Siri.
        bundle identifier
             The bundle identifier of the target application.

Examples:
        reset all permissions: privacy <device> reset all
        grant test host photo permissions: privacy <device> grant photos com.example.app.test-host
```
For example,

```
➜  ~ xcrun simctl privacy booted grant location com.facebook.flipper
➜  ~ xcrun simctl privacy booted grant photos com.facebook.flipper
```

and use `revoke` to revoke the permissions. 


### Switch the appearance style in a device between light and dark.

```
xcrun simctl ui booted appearance dark
xcrun simctl ui booted appearance light
```

### StatsBar override 

`Usage: simctl status_bar <device> [list | clear | override <override arguments>]`

It seems it can support a lot of overrides variables in status bar. But I have figure out the scenarios to use them. 

```
➜  ~ xcrun simctl status_bar                                    
Set or clear status bar overrides
Usage: simctl status_bar <device> [list | clear | override <override arguments>]

Supported Operations:
    list
        List existing overrides.

    clear
        Clear all existing status bar overrides.

    override <override arguments>
        Set status bar override values, according to these flags.
        You may specify any combination of these flags (at least one is required):

        --time <string>
             Set the date or time to a fixed value.
             If the string is a valid ISO date string it will also set the date on relevant devices.
        --dataNetwork <dataNetworkType>
             If specified must be one of 'wifi', '3g', '4g', 'lte', 'lte-a', or 'lte+'.
        --wifiMode <mode>
             If specified must be one of 'searching', 'failed', or 'active'.
        --wifiBars <int>
             If specified must be 0-3.
        --cellularMode <mode>
             If specified must be one of 'notSupported', 'searching', 'failed', or 'active'.
        --cellularBars <int>
             If specified must be 0-4.
        --operatorName <string>
             Set the cellular operator/carrier name. Use '' for the empty string.
        --batteryState <state>
             If specified must be one of 'charging', 'charged', or 'discharging'.
        --batteryLevel <int>
             If specified must be 0-100.
```

```
// change the `dataNetwork` from `wifi` to `4g`
➜  ~ xcrun simctl status_bar booted override --dataNetwork '4g'
```

```
xcrun simctl status_bar booted override --time 15:42 --cellularBars 1 --dataNetwork '3g' --wifiMode 'failed' --batteryState 'charging'
```
![status bar](status_bar_test.png)

and use `clear` option to clear all the settings.

```
xcrun simctl status_bar booted clear
```

https://developer.apple.com/videos/play/wwdc2020/10647/

### Record Video 

We can run `xcrun simctl io --help` to see more. 

```
xcrun simctl io <device> recordVideo <file>
```

```
xcrun simctl io booted recordVideo video.mp4
```
This command records the content of the screen of the current booted simulator, and save it to the video.mp4 file.  To stop recording, we have to use `ctl+c` in the terminal. 

```
xcrun simctl io booted recordVideo  -f video.mp4
```
`recordVideo` cannot save recorded video output into a file that already exists unless we use  `-f` flag to override the existing file. 

```
xcrun simctl io booted recordVideo video.mp4 --codec h264 --mask ignored video.mp4
```
 - By default, codec type is `hevc`. 
 - Ignore the mask when the mask is rendered black because of compatibility issues. 


And we can also capture the out of the external display of the simulator. 
```
xcrun simctl io booted recordVideo --display external external.mp4 
```

![](record_video.gif)

### Other useful commands

```
// Open a URL in a device.
simctl openurl <device> <URL>
// add a photo or movie to the Photos library of the specified simulator 
xcrun simctl addmedia <device> <file1> <file2>
// Set up a device IO operation. screenshot or recordVideo 
xcrun simctl io <device> screenshot <output.png>

```

> Ref: 
> https://developer.apple.com/videos/play/wwdc2019/418/
> https://www.iosdev.recipes/simctl/
> https://nshipster.com/simctl/ 
> https://developer.apple.com/documentation/xcode_release_notes/xcode_11_4_release_notes
