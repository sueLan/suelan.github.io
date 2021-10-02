---
title: How to use `SignPost `
date: 2021-10-02 20:05:59
categories:
  - iOS
---

`SignPost` is part of the `os_log` family delivered by Apple for iOS10 and above. It allows developers to place performance-focused time markers which can be displayed in Instrument for visualization. Or we can say, `SignPost` APIs enable us add some lightweight instrumentation to code for collection and visualization by performance analysis tooling. Now, here comes two basic concepts in `os_signpost` APIs. 

- `intervals `:  interesting periods of time. `os_signpost` interval begin and end matching  matching begins and ends is restricted to single threads.
- `events`: single points in time

## Instruments
As I said, we can use it with signpost logs to  
- Aggregate and analyze signpost data
- Visualize activity over time

Here comes a piece of code demonstrating how to use `OSLog`   

```swift
import os.signpost

// 1. create a container of related log messages. Each log contains a subsystem and a category, which you define and helpful when sorting and displaying them. Can see category in console.log  
let refreshLog = OSLog(subsystem: "com.example.your-app", category: "RefreshOperations")
 
// 2. A different signpost name for this different interval
os_signpost(.begin, log: refreshLog, name: "Refresh Panel")

for element in panel.elements {
  os_signpost(.begin, log: refreshLog, name: "Fetch Asset") fetchAsset(for: element)
  os_signpost(.end, log: refreshLog, name: "Fetch Asset")
}
os_signpost(.end, log: refreshLog, name: "Refresh Panel")
 
```

### Signpost Names

- The string literal identifies signpost intervals
- The name must match at .begin and .end

### Signpost IDs

Intervals with the same log handle and interval name can be in flight simultaneously. Signpost id is needed to correctly match begin signposts with end signposts. So we have to identify each interval with a signpost id. 

- Use signpost IDs to tell overlapping operations apart
- While running,use the same IDs for each pair of `.begin` and `.end`

```swift
let spid = OSSignpostID(log: refreshLog)
os_signpost(.begin, log: refreshLog, name: "Fetch Asset", signpostID: spid)

os_signpost(.end, log: refreshLog, name: "Fetch Asset", signpostID: spid)
```

### Making Signpost IDs

```swift
let spid = OSSignpostID(log: refreshLog, object: element)
```

- Signpost IDs are process-scoped
- Making from object is convenient if you have the same object at .begin and .end

## Organizing Signposts: A Hierarchy
You can use `subsystem`, `category` and `name` to organize signposts. 
```swift
log = OSLog(subsystem: "com.example.your-app", category: "RefreshOperations")
os_signpost(.begin, log: log, name: "Fetch Asset", signpostID: spid)
```

![image-20210108175744504](image-20210108175744504.png)

### Custom Metadata in Signpost Arguments

You can add more data when calling `os_signpost` 
- Add context to the .begin and .end
- Pass arguments with os_log format string literal
- Pass many arguments with different types
- Pass dynamic strings
- The format string is a fixed cost, so feel free to be descriptive!

```swift
os_signpost(.begin, log: log, name: "Compute Physics", "%d %d %d %d",
x1, y1, x2, y2)
```

### Signpost Events

```swift
os_signpost(.event, log: log, name: "Fetch Asset",
```
To add events between `.begin` and `.end`. They will become `Points of Interest ` displayed in instrument later. 

## Signposts Are Lightweight

- Built to minimize observer effect
- Built for fine-grained measurement in a short time span

## Enabling and Disabling Signpost Categories

```swift
OSLog.disabled
```

```swift
 
let refreshLog: OSLog
if ProcessInfo.processInfo.environment.keys.contains("SIGNPOSTS_FOR_REFRESH") {
   refreshLog = OSLog(subsystem: "com.example.your-app", category: "RefreshOperations") 
} else {
   refreshLog = .disabled
}
```

## Signposts in C

Well, if you want to use `os_signpost` in C, Apple also provides a series of APIs for C. 

![image-20210108180102653](image-20210108180102653.png)


### Points of Interest 

We may want to focus on timing of some important events. We can promote these events by using `points of interest`  as the category of `OSLog` .  Instrument then will look for this special category and display it in a seperated sections. 

![image-20210117174557365](image-20210117174557365.png)

We later can see these events displayed  and correlate these events with other performance data.  

In non-swift source code, we can use `OS_LOG_CATEGORY_POINTS_OF_INTEREST`.  And use `os_signpost(.event, ...)` to emit import events. 

```
#ifndef __swift__
#define OS_LOG_CATEGORY_POINTS_OF_INTEREST "PointsOfInterest"
#endif
```

![image-20210117224415205](image-20210117224415205.png)

In Instrument, we can use `Points of Interest` template. 

## How to use it in Instrument 

After opening the instrument, we can chose `Logging template` 

![image-20210117172127316](image-20210117172127316.png)

Or we can add `os_signpost`  here

![image-20210117170856013](image-20210117170856013.png)

What I like most in Instrument is that we can always zoom in and zoom out to select the region we insterest. 
![](demostration.gif)

### Analysis time of the intervals 

Choose `summary:Intervals` . We can see the summary of intervals. The count, duration, min duration, average duration, maximum duration of each signpost we collects. 

![image-20210117230659066](image-20210117230659066.png)

### Analysis the metadata 

`Summary:Metadata Statistics` 

#### Analysis Points of Interest 

By chosing the `Points of Interest` section, we can see the points of interest and its meta data. Besides, we can use points of interst together with other section, like Time Profile to check how is the performance like when these importance events happen. 

![image-20210117230047705](image-20210117230047705.png)

## Options 

The default recording mode is `immediate`, which means it records all the signposts immediately. This could bring overhead if your app emits thousands of signposts per second.  However, we can still work around that by changingto window mode and only collecting data for the last 5 seconds.  It is very useful to collect data to analyze stutters. 

Steps: 

1. long press the record button and select recording options 

   ![image-20210117182658245](image-20210117182658245.png)

2. Expand `Grobal Options`

![image-20210117182537413](image-20210117182537413.png)

3. Choose `Last`, filling 5 seconds. 
   ![image-20210117172616368](image-20210117172616368.png)

When recording in `Windowed Mode` , the Instrument shows a blank white screen with a hint `Recording in Windowed Mode`.

![image-20210117224703454](image-20210117224703454.png)

When stoping recording, the Instrument showing a progress view and analyzing the collected data. 

![image-20210117224757983](image-20210117224757983.png)

## Ref

- [Apple doc/logging](https://developer.apple.com/documentation/os/logging)
- [WWDC/Measuring Performance Using Logging](https://developer.apple.com/videos/play/wwdc2018/405/)