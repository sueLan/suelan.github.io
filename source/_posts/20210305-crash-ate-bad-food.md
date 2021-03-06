---
title:  Crash - ate bad food (0x8badf00d)
date: 2021-03-05 23:33:37
categories:
  - iOS
---

Recently, my colleague came to me for a crash report. It is a case where system watch dog kills our app for blocking `main thread` for too long time. It seems many people encounter issues of this sorts from time to time. However, there are a few articles about it. I hope this article could help you. 

## Exception Type 

As we know, a crash report records app's state when it crashed. It is a crucial and first-hand resource to help us figure out what was happening when a crash happened in user's device. When i to a crash report. I always try to extract as much information as I can. Apple actually has dedicated article about how to [analyze a crash report](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/analyzing_a_crash_report). 

In this case, firstly, we can take a look at exception type, which is `EXC_CRASH (SIGKILL)`. `EXC_CRASH (SIGKILL)` indicates the operating system terminated the process. See more [here](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/understanding_the_exception_types_in_a_crash_report#3582412). 

The crash report also contains a `Termination Reason` field with a code that explains the reason for the crash. Here, that code is `0xdead10cc`:

- `0x8badf00d` (pronounced “ate bad food”). The operating system’s watchdog terminated the app. See [Addressing Watchdog Terminations](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/addressing_watchdog_terminations).

```
Exception Type:  EXC_CRASH (SIGKILL)
Exception Codes: 0x0000000000000000, 0x0000000000000000
Exception Note:  EXC_CORPSE_NOTIFY
Termination Reason: Namespace SPRINGBOARD, Code 0x8badf00d
Termination Description: SPRINGBOARD, scene-update watchdog transgression: application<.......>:307 exhausted real (wall clock) time allowance of 10.00 seconds | ProcessVisibility: Background | ProcessState: Running | WatchdogEvent: scene-update | WatchdogVisibility: Background | WatchdogCPUStatistics: ( | "Elapsed total CPU time (seconds): 39.500 (user 39.500, system 0.000), 39% CPU", | "Elapsed application CPU time (seconds): 21.283, 21% CPU" | )
Triggered by Thread:  0
```



### WatchDog

As we know, iOS has watchdog oto ensure application not drag down the system responsiveness. If your app has poor performance, watchdog will check then kill you app to make sure the shared resource well allocated.  It is dev's duty to maintain system responsiveness. The following table showing some cases for watchdog check, it comes from this [session](https://developer.apple.com/videos/play/wwdc2011/312/),

<img src="image-20210305194510328.png" alt="image-20210305194510328.png" style="zoom:50%;" />

As for scenarios, Slow launch will cause OS to abort your app. `20 sec`; resume and suspend is also very important `10sec`; smooth scrolling `6s` is also a key point. Besides, I also have seen `blocking main thread`  will cause watchdog kill an app, like this 

```
transgression: application<com.wangya.test2>:21021 exhausted real (wall clock) time allowance of 19.99 seconds 
Triggered by Thread:  0
```



The `Termination Description`  in my crash report shows our app `exhausted real (wall clock) time allowance of 10.00 seconds`.  Besides, the `ProcessVisibility` shows it is `Background` and `ProcessState` is `Running`.  It seems the crashed happened when it tried to resumed from background. But what caused it? 

### Backtrace 

So we have to go to the stack trace in the crashed report. Unfortunately, he gave me a partially symbolicated crash report with `CoreData` related method remaining hexadecimal memory address. From the frame in our app, i know operations related to `CoreData` happening in `main thread`. Then, I have to symbolicate the crash report. 

### Symbolicate 

Apple actually has doc to [symbolicate the Crash Report in Xcode](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/adding_identifiable_symbol_names_to_a_crash_report#3403794)  or [symbolicate the crash report with the command line](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/adding_identifiable_symbol_names_to_a_crash_report#3403800). 

> To symbolicate in Xcode, click the Device Logs button in the [Devices and Simulators window](https://help.apple.com/xcode/mac/current/#/dev85c64ec79), then drag and drop the crash report file into the list of device logs. And, crash reports must have the `.crash` file extension. 

![img](https://help.apple.com/xcode/mac/current/en.lproj/Art/dw_view_crash_logs.png)

 As Apple's doc, [Acquire Device Symbol Information](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/adding_identifiable_symbol_names_to_a_crash_report#3403795) says, I have to collect symbols for system frameworks like `CoreData` from the device where crashed happened.

> The symbols for system frameworks are specific to the operating system release and the CPU architecture of the device. For example, the symbols for an iPhone running iOS 13.1.0 aren’t the same as the symbols for the same iPhone running iOS 13.1.2. 

Unfortunately, I don't have  the device on hand to get the symbol information matching the operating system version recorded in the crash report. How to solve it? Well, I still have a way to work around. Let's start to do that.

1. Check the os version and hardware model from the crash report. OS Version: `iPhone OS 13.3.1 (17D50); Hardware model: `Hardware Model:      iPhone11,8`

2. download [IPSW](https://en.wikipedia.org/wiki/IPSW), iPod Software from  https://www.theiphonewiki.com/wiki/Firmware/iPhone/11.x or https://ipsw.me/; or can decrypte `ipsw` by yourself following this [guide](https://gist.github.com/tzmartin/812027b879f6ff4459af)

3. `IPSW` file actually is `zip` file, we can change the file extension `ipsw` to `zip`. Then unzip it. You can see a big disk image there.

   ![](image-20210305201936593.png)

    Double click it. It will be mounted in filesystem. Then we can see it showing in the `Finder` 

   ![](image-20210305202238201.png)

4. Then go to `/System/Library/Caches/com.apple.dyld` folder. Drag this file to `Hopper`

   ![](image-20210305202347747.png)

5. Since all executable dylib are shipped into this whole file, we have to filter out `CoreData` in the search bar. 

   ![](image-20210305202507076.png)

6. Get `offset` from frame in the backtrace in the crash report.  Using `go to file offset`  in Hopper, `option + g` to jump to that line of assembly code. I actually have a [talk](https://www.youtube.com/watch?v=GPs3oPFSdo0) about how to use exception info, thread state, offset in backtrace in crash report  to analyze crash report. 

   ![](image-20210305204358148.png)

   take `line 29` as an example, it goes to a `bl` instruction. As we know, in Objective-C, `x1` stores the sector. So we can quickly figure out the selector is 

   [executeRequest:withContext:error:](https://developer.apple.com/documentation/coredata/nspersistentstorecoordinator/1468872-executerequest?language=objc)

   ![](image-20210305204545965.png)

   Hmm, although it is a little bit tedious, we finanly get more info about what is happening here. 

   Basically, when resuming from background, our app triggers a data request operation in `CoreData`. It is executed in `main thread` and make the main thread wait this transaction finished. While,  it is a bit of slow to execute this SQL-related transaction here and blocks `main thread` for extended period of time. Then when watchdog come to check, it terminates our app in order to maintain the system responsiveness. 

![](image-20210305210726254.png)

As apple suggests, the watchdog terminates apps that block the main thread for a significant time.  There are some cases of this sort, like 

- Synchronous networking
- Processing large amouts of data, such as large JSON files or 3D models
- Triggering lightweight migration for a large [Core Data](https://developer.apple.com/documentation/coredata) store synchronously
- Analysis requests with [Vision](https://developer.apple.com/documentation/vision)

This time, we ate bad food. 

### See More

- [Addressing Watchdog Terminations](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/addressing_watchdog_terminations)

- [Investigating Memory Access Crashes](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/investigating_memory_access_crashes)

- [Analyzing a Crash Report](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/analyzing_a_crash_report)
- [0x8badf00d in main thread](https://stackoverflow.com/questions/59827727/ios-frequently-getting-exhausted-real-wall-clock-time-allowance-of-10-00-seco)

