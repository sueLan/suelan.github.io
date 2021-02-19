---
title: Dive into CFRunLoop
date: 2021-02-13 20:20:54
categories: 
  - iOS
---
## Background 

Sometimes, you may want to collect on-device performance metrics in main thread to know how our App performs and help you find more clues to  analyse performance issue. [MetricKit](MetricKit) is a useful utility framework to achieve that. It starts accumulating reports for your app after being called for the first time and delivers reports at most once per day. The reports contain the metrics from the past 24 hours and any previously undelivered daily reports. Then, you can go to `Xcode->Organizer->Metric` panel to check these info. However, you may want your in-house App Performance Monitoring framework to gain more controls on when to collect metrics, or how to upload it, or collect what you want. Earlier days, `Tencent` launched a [iOS framework called matrix](https://github.com/Tencent/matrix) to monitor App performance metrics. When I explored this library, I saw they use `CFRunLoop` to detect hitch in the thread, like main thread. This intrigued me. So I investigated `CFRunLoop` to learn more. 


## What is RunLoop in iOS? 

Before talking about RunLoop in iOS, we may have to know something about event loop and thread.  In [OS/360 Multiprogramming with a Variable Number of Tasks](https://en.wikipedia.org/wiki/OS/360_and_successors#MVT) (MVT) in 1967, threads made an early appearance under the name of "tasks". A **thread** in computer science  is short for a *thread of execution*. Once the tasks in one Thread is all done, the thread finishes its job and exits. Sometimes, we need a way to keep it alive and handling events.  Then,  comes the event loop. The psudo code for event loop is like this:

```js
function loop
    initialize()
    while message != quit
        message := get_next_message()
        process_message(message)
    end while
end function
```

In Wikipedia, event loop is a programming construct or [design pattern](https://en.wikipedia.org/wiki/Software_design_pattern) that waits for and dispatches [events](https://en.wikipedia.org/wiki/Event-driven_programming) or [messages](https://en.wikipedia.org/wiki/Message_Passing_Interface) in a [program](https://en.wikipedia.org/wiki/Computer_program).  In this event loop, it keeps `waiting events -> receive events -> handle events` until the exit condition is met. 

In Apple's doc,  this kind of event loop is implemented by `CFRunLoop` in low-level. In cocoa, the object is an instance of `NSRunLoop` There is exactly one run loop per thread. 

> Run loops are part of the fundamental infrastructure associated with threads. A *run loop* is an event processing loop that you use to schedule work and coordinate the receipt of incoming events. The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.

> A CFRunLoop object monitors sources of input to a task and dispatches control when they become ready for processing.  

It can handle 

- user input devices

- port objects 

- network connections

- periodic or time-delayed events

- asynchronous callbacks

  ![image-20210128230441389](image-20210128230441389.png)

Apple provides two APIs to get runloop object 

- `CFRunLoopGetMain()` // the main CFRunLoop object
- `CFRunLoopGetCurrent()` // CFRunLoop object for the current thread

## RunLoop Mode 

> A *run loop mode* is a collection of input sources and timers to be monitored and a collection of run loop observers to be notified. Each time you run your run loop, you specify (either explicitly or implicitly) a particular “mode” in which to run.During that pass of the run loop, only sources associated with that mode are monitored and allowed to deliver their events.   --- [doc](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)

A run loop mode contains a set of `CFRunLoopSource`, a list of `CFRunLoopTimer` and `CFRunLoopObservers`. They are all inputs for runloop. 

![image-20210128223829153](image-20210128223829153.png)



## Inputs 

Three kinds of inputs can be monitored by a run loop 

- CFRunLoopSource 
- CFRunLoopTimer
- CFRunLoopObservers

### Source  - CFRunLoopSource

[CFRunLoopSource](https://developer.apple.com/documentation/corefoundation/cfrunloopsource-rhr)

> A CFRunLoopSource object is an abstraction of an input source that can be put into a run loop. Input sources typically generate asynchronous events, such as messages arriving on a network port or actions performed by the user.

```c++
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
	      CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```

`_context` is a `union` type. A union looks like a structure, but  it will use the memory space for just one of the fields in its definition. So the `_context` is either an `CFRunLoopSourceContext`    structure or `CFRunLoopSourceContext1` structure.

#### Two categories

As it is mentioned in this [doc](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23), we mainly care about two categories, port-base input sources, source1,  and non-port-based input sources, source0.

- Version 0 sources, so named because the `version` field of their context structure is 0, are managed manually by the `application`.
  - When a source is ready to fire, some part of the `application`, perhaps code on a separate` thread` waiting for an event, must call [`CFRunLoopSourceSignal(_:)`](https://developer.apple.com/documentation/corefoundation/1543700-cfrunloopsourcesignal) to tell the run loop that the source is ready to fire. 
- Version 1 sources are managed by the run loop and kernel. 
  - These sources use `Mach ports` to signal when the sources are ready to fire. 
  - A source is automatically `signaled by the kernel` when a message arrives on the source’s Mach port.

![image-20210125183432020](image-20210125183432020.png)



#### `bits` field

It seems that `bits` field is used to mark the status of the `CFRunLoopSouceRef` .  

```c++
CF_INLINE Boolean __CFRunLoopSourceIsSignaled(CFRunLoopSourceRef rls) {
    return (Boolean)__CFBitfieldGetValue(rls->_bits, 1, 1);
}

CF_INLINE void __CFRunLoopSourceSetSignaled(CFRunLoopSourceRef rls) {
    __CFBitfieldSetValue(rls->_bits, 1, 1, 1);
}

CF_INLINE void __CFRunLoopSourceUnsetSignaled(CFRunLoopSourceRef rls) {
    __CFBitfieldSetValue(rls->_bits, 1, 1, 0);
}
```

[CFRunLoopSourceSignal](https://developer.apple.com/documentation/corefoundation/1543700-cfrunloopsourcesignal) is used to signals a `version 0 source` , marking it as ready to fire. It actually updated the `bits` in the  `CFRunLoopSouceRef` structure. 

```c++
void CFRunLoopSourceSignal(CFRunLoopSourceRef rls) {
    CHECK_FOR_FORK();
    __CFRunLoopSourceLock(rls);
    if (__CFIsValid(rls)) {
	__CFRunLoopSourceSetSignaled(rls);
    }
    __CFRunLoopSourceUnlock(rls);
}
```

### CFRunLoopTimer 

#### What is CFRunLoopTimer

[Doc for CFRunLoopTimer](https://developer.apple.com/documentation/corefoundation/cfrunlooptimer-rhk)

> A CFRunLoopTimer object represents a specialized run loop source that fires at a preset time in the future. Timers can fire either only once or repeatedly at fixed time intervals.

There are two conditions for a timer to be fired: 

- one of the run loop modes to which the timer has been added is running 
- the timer's firing time has passed

> If a timer’s firing time occurs while the run loop is in a mode that is not monitoring the timer or during a long callout, the timer does not fire until the next time the run loop checks the timer. Therefore, the actual time at which the timer fires potentially can be a significant period of time after the scheduled firing time.

### RunLoopObserver

#### How to use observer 

1. We can use these two APIs to create RunLoopObserver and associated it with handlers. 

- CFRunLoopObserverCreate(_:_:_:_:_:_:)
- CFRunLoopObserverCreateWithHandler(_:_:_:_:_:)

2. add the observer into the runloop 

   ```objective-c
     
       CFRunLoopObserverRef runloopObserver = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, kCFRunLoopBeforeWaiting, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {   
        // handler code here  
     });
   
       CFRunLoopAddObserver(CFRunLoopGetMain(), runloopObserver, kCFRunLoopDefaultMode);
   ```

3. Observe specific RunLoop Activity 

## RunLoop Activity 

> The run loop stages in which an observer is scheduled are selected when the observer is created with [`CFRunLoopObserverCreate`](https://developer.apple.com/documentation/corefoundation/1541546-cfrunloopobservercreate?language=objc).       -[doc](https://developer.apple.com/documentation/corefoundation/cfrunloopactivity?language=objc)

There are several kinds of RunLoop Activity for CFRunLoop. You can associate run loop observers with these RunLoopActivity

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0), // The entrance of the run loop, before entering the event processing loop. This activity occurs once for each call to CFRunLoopRun() and CFRunLoopRunInMode(_:_:_:).
    kCFRunLoopBeforeTimers = (1UL << 1), // Inside the event processing loop before any timers are processed.
    kCFRunLoopBeforeSources = (1UL << 2), // Inside the event processing loop before any sources are processed.
    kCFRunLoopBeforeWaiting = (1UL << 5), 
    kCFRunLoopAfterWaiting = (1UL << 6),  // Inside the event processing loop after the run loop wakes up, but before processing the event that woke it up. This activity occurs only if the run loop did in fact go to sleep during the current loop.
    kCFRunLoopExit = (1UL << 7), // The exit of the run loop, after exiting the event processing loop. This activity occurs once for each call to CFRunLoopRun() and CFRunLoopRunInMode(_:_:_:).
    kCFRunLoopAllActivities = 0x0FFFFFFFU 
};
```

https://developer.apple.com/documentation/corefoundation/cfrunloopactivity 

###  Run Loop Sequence of Events

According to [apple doc](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1),  when runloop running  in a thread, it processes pending events and generates notifications for attached observers.  Briefly, it works as the follow diagram shows. 

![image-20210209142244931](image-20210209142244931.png)

The implementation is in `CFRunLoopRunSpecific` and `__CFRunLoopRun` in `CFRunloop.c` . 

### Use case in App Performance Monitoring  

 In [Tencent matrix](https://github.com/Tencent/matrix), it leverages the Run Loop notifications to record timestamp when these notifications sent. 

1. create and add RunLoopObserver to current RunLoop [CFRunLoopAddObserver](https://github.com/Tencent/matrix/blob/c7fd99237af189fb060f90d1272350db19182dbf/matrix/matrix-iOS/Matrix/WCCrashBlockMonitor/CrashBlockPlugin/Main/BlockMonitor/WCBlockMonitorMgr.mm#L831)

2. [record timestamp](https://github.com/Tencent/matrix/blob/c7fd99237af189fb060f90d1272350db19182dbf/matrix/matrix-iOS/Matrix/WCCrashBlockMonitor/CrashBlockPlugin/Main/BlockMonitor/WCBlockMonitorMgr.mm#L858) in callback function invoked when the observer runs

So, I did a small experiments. I added a RunLoop Observer to the runloop in main thread. Then calculate the time gap between `kCFRunLoopBeforeTimers` notification in two continuous loop. 

```c++
// 1. create runloop observer 
CFRunLoopObserverRef beginObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, LONG_MIN, &myRunLoopCallback, &context);
// 2.  add observer to runloop in the main thread
CFRunLoopAddObserver([[NSRunLoop mainRunLoop] getCFRunLoop], beginObserver, kCFRunLoopCommonModes);
```

```c++
// 3. implemented callback for RunLoop observer 
static void myRunLoopCallback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
     switch (activity) {
        case kCFRunLoopEntry:
             self.isRunloopRunning = YES;
             break;       
        case kCFRunLoopBeforeTimers:
             NSLog(@"[RY]kCFRunLoopBeforeTimers called %@", @(getCurrentMilliTimestamp() - monitor.runloopMilliTimestamp));
             self.runloopMilliTimestamp = getCurrentMilliTimestamp();
             self.isRunloopRunning = YES;
             break;
        case kCFRunLoopBeforeSources:
             self.isRunloopRunning = YES;
             break;
        case kCFRunLoopBeforeWaiting:
             self.isRunloopRunning = NO;
             break;
        case kCFRunLoopAfterWaiting:
             self.isRunloopRunning = YES;
             break;
        case kCFRunLoopExit:
             self.isRunloopRunning = NO;
             break;b
        default:
            break;
    }
}
```

Theoretically,  the time diff between  two continuous `kCFRunLoopBeforeTimers` notification should be within `16.67ms` to achieve smooth user experience in main thread, which means `RunLoop` runs 60 times per second. In the following log, one frame takes about `72ms` to executed. 

```
 [RY]kCFRunLoopBeforeTimers called 3.628173828125
 [RY]kCFRunLoopBeforeTimers called 17.784912109375
 [RY]kCFRunLoopBeforeTimers called 0.041015625
 [RY]kCFRunLoopBeforeTimers called 1.23388671875
 [RY]kCFRunLoopBeforeTimers called 72.05419921875
 [RY]kCFRunLoopBeforeTimers called 5.138916015625
 [RY]kCFRunLoopBeforeTimers called 0.072021484375
ers called 1.296875
 [RY]kCFRunLoopBeforeTimers called 0.070068359375
 [RY]kCFRunLoopBeforeTimers called 0.035888671875
 [RY]kCFRunLoopBeforeTimers called 0.051025390625
 [RY]kCFRunLoopBeforeTimers called 0.057861328125
 [RY]kCFRunLoopBeforeTimers called 0.01806640625
 [RY]kCFRunLoopBeforeTimers called 0.260009765625
 [RY]kCFRunLoopBeforeTimers called 0.03515625
 [RY]kCFRunLoopBeforeTimers called 0.43212890625

```

- https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1
- https://blog.ibireme.com/2015/05/18/runloop/
- https://opensource.apple.com/tarballs/CF/
- https://developer.apple.com/documentation/corefoundation
- https://github.com/apple/swift-corelibs-foundation/