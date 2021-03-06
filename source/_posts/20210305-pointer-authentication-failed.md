---
title: Crash - Pointer Authentication Failures or invalid memory accesses
date: 2021-03-05 23:34:51
tags:
  - iOS
categories:
  - React Native 
---

Recently, a colleague came to me for a crash. It is quite tricky. So i took a note here. The exception section is as follow:

```
Exception Type:  EXC_BAD_ACCESS (SIGSEGV)
Exception Subtype: KERN_INVALID_ADDRESS at 0x0000ec033bb40320 -> 0x000000033bb40320 (possible pointer authentication failure)
VM Region Info: 0x33bb40320 is not in any region.  Bytes after previous region: 2612265761  Bytes before following region: 53759180000
      REGION TYPE                 START - END      [ VSIZE] PRT/MAX SHRMOD  REGION DETAIL
      MALLOC_NANO              283fb8000-2a0000000 [448.3M] rw-/rwx SM=ZER  
--->  GAP OF 0xd20000000 BYTES
      commpage (reserved)      fc0000000-1000000000 [  1.0G] ---/--- SM=NUL  ...(unallocated)

Termination Signal: Segmentation fault: 11
Termination Reason: Namespace SIGNAL, Code 0xb
Terminating Process: exc handler [8919]
Triggered by Thread:  14
```
From this exception type, we can see `KERN_INVALID_ADDRESS at 0x0000ec033bb40320 -> 0x000000033bb40320 (possible pointer authentication failure)` . You may wonder what it is. Well, according to Apple doc 

>  When pointer authentication fails, the system invalidates the failing pointer by setting a high-order bit. Subsequent use of the pointer results in a segmentation fault. The crash report contains a message that includes the value of the pointer both after and before invalidation:

But there is another much more common crash caused by this kind: 

> Be aware that other invalid memory accesses, where high-order bits are erroneously set, can also look like pointer authentication failures.

In exception section in the crash report, there are 3 messages matching 

- The exception type is `EXC_BAD_ACCESS (SIGSEGV) `,  it is a segment violation signal from system to kill your process
- The `subtype` is  `KERN_INVALID_ADDRESS at 0x0000ec033bb40320 -> 0x000000033bb40320 (possible pointer authentication failure)`.  We can see here the failing pointer is `0x0000ec033bb40320`, after system invalidates the higher bits of this pointer, it becomes `0x000000033bb40320`.
- The converted address `0x33bb40320` isn't in any valid memory region. 

Now we know the invalid address is `0x0000ec033bb40320`. Then we actually can see it is in `x0` register in the Thread State. `Thread State` in crash report actually records the state of thread's register when crash happened. 
See more about thread state in [doc](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/investigating_memory_access_crashes#3562197). In Objective-C , c or Swift, register 0 holds address for current object when function calling. 

```
Thread 14 crashed with ARM Thread State (64-bit):
    x0: 0x0000ec033bb40320   x1: 0x000000016b692b58   x2: 0x000000016b692b50   x3: 0x000000016b692b30
    x4: 0x0000000000008903   x5: 0x0000000000000020   x6: 0x0000000000000000   x7: 0x0000000000000000
    x8: 0x003f4281a5942364   x9: 0x8ef94d448c3b009a  x10: 0x00000000000007c8  x11: 0x007f00013e1bd800
   x12: 0x000000000000004d  x13: 0x000000013e1bdcc0  x14: 0x000000000000035f  x15: 0x00000001f3e4d8b8
   x16: 0x00000001dc3bfbc0  x17: 0x00000001908b9474  x18: 0x0000000000000000  x19: 0x000000016b692b30
   x20: 0x000000016b692b50  x21: 0x000000016b692b58  x22: 0x0000000280e10320  x23: 0x000000028392f600
   x24: 0x0000000108367ae8  x25: 0x0000000000000000  x26: 0x000000016b6930e0  x27: 0x0000000000000114
   x28: 0xffffffffffffffff   fp: 0x000000016b692b20   lr: 0x00000001076903ac
    sp: 0x000000016b692ad0   pc: 0x00000001076900d0 cpsr: 0x20001000
   esr: 0x92000004 (Data Abort) byte read Translation fault

```

So we can switch gears and take a deeper look into the stacktrace in the crashed thread to find out the current instance. At the top of the stacktrace, it is `loadApplication` function from `Instance` class. Since the invalid address is in `x0`, which usually holds the pointer to current object. I suspect something goes wrong with `Instance`  instance. 

```c++
Thread 14 name:
Thread 14 Crashed:
0   xxx                       	0x00000001076900d0 facebook::react::Instance::loadApplication(std::__1::unique_ptr<facebook::react::RAMBundleRegistry, std::__1::default_delete<facebook::react::RAMBundleRegistry> >, std::__1::unique_ptr<facebook::re... + 40 (Instance.cpp:61)
1   xxx                       	0x00000001076903ac facebook::react::Instance::loadScriptFromString(std::__1::unique_ptr<facebook::react::JSBigString const, std::__1::default_delete<facebook::react::JSBigString const> >, std::__1::basic_string<char,... + 200 (Instance.cpp:95)
2   xxx                       	0x00000001076903ac facebook::react::Instance::loadScriptFromString(std::__1::unique_ptr<facebook::react::JSBigString const, std::__1::default_delete<facebook::react::JSBigString const> >, std::__1::basic_string<char,... + 200 (Instance.cpp:95)
3   xxx                       	0x0000000106434fb4 __51-[RCTCxxBridge executeApplicationScript:url:async:]_block_invoke + 668 (RCTCxxBridge.mm:1429)
4   xxx                       	0x0000000106439414 std::__1::__function::__value_func<void ()>::operator()() const + 20 (functional:1873)
5   xxx                       	0x0000000106439414 std::__1::function<void ()>::operator()() const + 20 (functional:2548)
6   xxx                       	0x0000000106439414 facebook::react::tryAndReturnError(std::__1::function<void ()> const&) + 40 (RCTCxxUtils.mm:72)
7   xxx                       	0x000000010642e5e8 -[RCTCxxBridge _tryAndHandleError:] + 100 (RCTCxxBridge.mm:276)
8   xxx                       	0x0000000106434cb0 -[RCTCxxBridge executeApplicationScript:url:async:] + 156 (RCTCxxBridge.mm:1412)
9   xxx                       	0x0000000106434aa4 -[RCTCxxBridge enqueueApplicationScript:url:onComplete:] + 112 (RCTCxxBridge.mm:1392)
10  xxx                       	0x0000000106432710 -[RCTCxxBridge executeSourceCode:sync:] + 228 (RCTCxxBridge.mm:1005)
11  xxx                       	0x000000010642f374 __21-[RCTCxxBridge start]_block_invoke_2 + 96 (RCTCxxBridge.mm:390)
12  libdispatch.dylib             	0x000000019050824c _dispatch_call_block_and_release + 32 (init.c:1454)
13  libdispatch.dylib             	0x0000000190509db0 _dispatch_client_callout + 20 (object.m:559)
14  libdispatch.dylib             	0x000000019051aa68 _dispatch_root_queue_drain + 656 (inline_internal.h:2548)
15  libdispatch.dylib             	0x000000019051b120 _dispatch_worker_thread2 + 116 (queue.c:6777)
16  libsystem_pthread.dylib       	0x00000001dc3c57d8 _pthread_wqthread + 216 (pthread.c:2227)
17  libsystem_pthread.dylib       	0x00000001dc3cc76c start_wqthread + 8
```

Then, i checked the backtrace again.  It actually happens in the phase of Bridge initialization.  `[RCTCxxBridge start]`. 

```c++
- (void)start
{
  RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"-[RCTCxxBridge start]", nil);

  [[NSNotificationCenter defaultCenter]
    postNotificationName:RCTJavaScriptWillStartLoadingNotification
    object:_parentBridge userInfo:@{@"bridge": self}];

  // Set up the JS thread early
  _jsThread = [[NSThread alloc] initWithTarget:[self class]
                                      selector:@selector(runRunLoop)
                                        object:nil];
  _jsThread.name = RCTJSThreadName;
  _jsThread.qualityOfService = NSOperationQualityOfServiceUserInteractive;
#if RCT_DEBUG
  _jsThread.stackSize *= 2;
#endif
  [_jsThread start];

  dispatch_group_t prepareBridge = dispatch_group_create();

  [_performanceLogger markStartForTag:RCTPLNativeModuleInit];

  [self registerExtraModules];
  // Initialize all native modules that cannot be loaded lazily
  (void)[self _initializeModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];
  [self registerExtraLazyModules];

  [_performanceLogger markStopForTag:RCTPLNativeModuleInit];

  // This doesn't really do anything.  The real work happens in initializeBridge.
  _reactInstance.reset(new Instance);

  __weak RCTCxxBridge *weakSelf = self;

  // Prepare executor factory (shared_ptr for copy into block)
  std::shared_ptr<JSExecutorFactory> executorFactory;
  if (!self.executorClass) {
    if ([self.delegate conformsToProtocol:@protocol(RCTCxxBridgeDelegate)]) {
      id<RCTCxxBridgeDelegate> cxxDelegate = (id<RCTCxxBridgeDelegate>) self.delegate;
      executorFactory = [cxxDelegate jsExecutorFactoryForBridge:self];
    }
    if (!executorFactory) {
      executorFactory = std::make_shared<JSCExecutorFactory>(nullptr);
    }
  } else {
    id<RCTJavaScriptExecutor> objcExecutor = [self moduleForClass:self.executorClass];
    executorFactory.reset(new RCTObjcExecutorFactory(objcExecutor, ^(NSError *error) {
      if (error) {
        [weakSelf handleError:error];
      }
    }));
  }

  // Dispatch the instance initialization as soon as the initial module metadata has
  // been collected (see initModules)
  dispatch_group_enter(prepareBridge);
  [self ensureOnJavaScriptThread:^{
    [weakSelf _initializeBridge:executorFactory];
    dispatch_group_leave(prepareBridge);
  }];

  // Load the source asynchronously, then store it for later execution.
  dispatch_group_enter(prepareBridge);
  __block NSData *sourceCode;
  [self loadSource:^(NSError *error, RCTSource *source) {
    if (error) {
      [weakSelf handleError:error];
    }

    sourceCode = source.data;
    dispatch_group_leave(prepareBridge);
  } onProgress:^(RCTLoadingProgress *progressData) {
#if RCT_DEV && __has_include("RCTDevLoadingView.h")
    // Note: RCTDevLoadingView should have been loaded at this point, so no need to allow lazy loading.
    RCTDevLoadingView *loadingView = [weakSelf moduleForName:RCTBridgeModuleNameForClass([RCTDevLoadingView class])
                                       lazilyLoadIfNecessary:NO];
    [loadingView updateProgress:progressData];
#endif
  }];

  // Wait for both the modules and source code to have finished loading
  dispatch_group_notify(prepareBridge, dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0), ^{
    RCTCxxBridge *strongSelf = weakSelf;
    if (sourceCode && strongSelf.loading) {
      [strongSelf executeSourceCode:sourceCode sync:NO];
    }
  });
  RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"");
}
```

This code comes from `React Native 0.61` , what this function basically does is to 

1. Init JSThread and . If you are insterested in `CFRunloop` as well, can take a look at my another [article](https://suelan.github.io/2021/02/13/20210213-dive-into-runloop-ios/)
2. Initialize native modules 
3. create instance of  `facebook::react::Instance` class, and initialization in `JSThread`

```c++
// In RCTCxxBridge
dispatch_group_enter(prepareBridge);
  [self ensureOnJavaScriptThread:^{
    [weakSelf _initializeBridge:executorFactory];
    dispatch_group_leave(prepareBridge);
  }];

// In Instance
void Instance::initializeBridge(
    std::unique_ptr<InstanceCallback> callback,
    std::shared_ptr<JSExecutorFactory> jsef,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<ModuleRegistry> moduleRegistry) {
  callback_ = std::move(callback);
  moduleRegistry_ = std::move(moduleRegistry);
  jsQueue->runOnQueueSync([this, &jsef, jsQueue]() mutable {
    nativeToJsBridge_ = folly::make_unique<NativeToJsBridge>(
        jsef.get(), moduleRegistry_, jsQueue, callback_);

    std::lock_guard<std::mutex> lock(m_syncMutex);
    m_syncReady = true;
    m_syncCV.notify_all();
  });

  CHECK(nativeToJsBridge_);
}
```

4. Load JavaScript Bundle 

5. execute JavaScript source code in background thread, which is the crash thread. 

   

Obviously, there are potential risk of multi-thread write issue for `Instance` object. It would be accessed from 

- `com.facebook.react.JavaScript` thread. You can search `+[RCTCxxBridge runRunLoop]` in your crash report

- Background thread created by `Dispatch`

- The thread to setup JavaScript Bridge, where you call

  `_bridge = [[RCTBridge alloc] initWithDelegate:self                               launchOptions:launchOptions];`

And in my friend's case, they could init `RCTBridge`  for more than once, which could rise the possibility of this crash. I have a [video](https://www.youtube.com/watch?v=GPs3oPFSdo0) for this. 


### Ref
- https://developer.apple.com/documentation/security/preparing_your_app_to_work_with_pointer_authentication
- https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/analyzing_a_crash_report