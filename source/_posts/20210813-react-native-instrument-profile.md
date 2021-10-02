---
title: react-native-instrument 
date: 2021-08-13 22:34:33
categories:
  - iOS
---

<!--more-->

Sometimes, you may encounter heavy performance issue in RN page, like laggy and unresponsive pages. And you do want to take a deep look at how many data are passed through JSBridge and what's going on there. You may found some people leverage `spy` function in `MessageQueue.js` to check the traffic volume through JSBridge. [Blog here](https://callstack.com/blog/react-native-how-to-check-what-passes-through-your-bridge/ ). It looks cool and helpful. However, in real life, you could see the your app performance is dragged down dramatically, even totally unresponsive when using this debug tool. So I was trying to find out a more efficient way to monitor traffic volume in JS bridge. Luckily, I saw a native performance monitoring logger inside react native framework, which appears in almost every crucial points in the workflow of JS-Native method calling. Then, I realized, maybe, I can use it to achieve the goal to spy bridge messages efficiently since it is in C++ realm and I can store all the data I collect in memory or use `SignPost`  to make data visualized in Instrument.

## NativeModulePerfLogger in React Native

In some crucial classes in react-native,  you can see `NativeModulePerfLogger` is used to collect execution data. For example, in `RCTNativeModule:invokeInner` function, which is to dispatch function calling from JS to native, there are stubs using  `NativeModulePerfLogger`  in namespace `BridgeNativeModulePerfLogger`.

```c++
 if (context == Async) {
    BridgeNativeModulePerfLogger::asyncMethodCallExecutionStart(moduleName, methodName, (int32_t)callId);
    BridgeNativeModulePerfLogger::asyncMethodCallExecutionArgConversionStart(moduleName, methodName, (int32_t)callId);
  }
  
  if (context == Sync) {
      BridgeNativeModulePerfLogger::syncMethodCallExecutionStart(moduleName, methodName);
    } else {
      BridgeNativeModulePerfLogger::asyncMethodCallExecutionArgConversionEnd(moduleName, methodName, (int32_t)callId);
    }
    
  if (context == Sync) {
      BridgeNativeModulePerfLogger::syncMethodCallExecutionEnd(moduleName, methodName);
      BridgeNativeModulePerfLogger::syncMethodCallReturnConversionStart(moduleName, methodName);
    } else {
      BridgeNativeModulePerfLogger::asyncMethodCallExecutionEnd(moduleName, methodName, (int32_t)callId);
    }
```

If you look into `BridgeNativeModulePerfLogger`, it basically calls functions in  `NativeModulePerfLogger`  to collect data. Basically,  `NativeModulePerfLogger` is a virtual class, a platform-agnostic interface to do performance logging for native modules and tubomodules.  We can inherit  `NativeModulePerfLogger` and implement all its pure virtual functions. These functions are for initializing Native Module object, JS require , sync method calls, async method calls,pre-processing async method call batch, async method call execution etc. 
You may want to check out more details about these functions in the source file in [Github](https://github.com/facebook/react-native/blob/57aa70c06cba3597725f7447943613e8905ae11d/ReactCommon/reactperflogger/reactperflogger/NativeModulePerfLogger.h#L18). With the help of `NativeModulePerfLogger`, we can inject our own logger implementation, collecting performance data without modifying react-native SDK heavily.  

## SignPost

`os_signpost` is delivered by Apple for developers to collect performance data for visualization on iOS 10 and above. It allows you to place markers which can be displayed in instrument so that it is easier for developers to discern where the bottleneck is. The good is that Apple also provides C based APIs so I can basically use them in my c++ class which implements  `NativeModulePerfLogger`. 

![image-2021010`8180102653](image-20210108180102653.png)

I finally wrote [react-native-bridge-instrument](https://github.com/sueLan/react-native-bridge-instrument) to combine the power of  `NativeModulePerfLogger` and `SignPost` and help your monitor function callings happening in JS bridge much more efficiently.

For example, I place a markers when an async method calling starts; then place another related marker at the moment when this method calling ends. Instrument will help us wrap up all these data and visualize them for us well. 

```c++
void Cxx ::asyncMethodCallArgConversionStart(
                                                    const char *moduleName,
                                                    const char *methodName) {
  os_signpost_interval_begin(async_method_call_logger, OS_SIGNPOST_ID_EXCLUSIVE, function_name(__func__), "%s %s", moduleName, methodName);
};

void CxxNativeModulePerfLogger::asyncMethodCallArgConversionEnd(
                                                    const char *moduleName,
                                                    const char *methodName) {
  os_signpost_interval_end(async_method_call_logger, OS_SIGNPOST_ID_EXCLUSIVE, function_name(__func__), "%s %s", moduleName, methodName);
};
```

## How to use it

Because I implemented `CxxNativeModulePerfLogger` in C. Basically, you have to write a method in `mm` file to use these C++ functions.

```
#include <ReactNativeBridgeInstrument/CxxNativeModulePerfLogger.h>
// make sure `CXXFLAGS += -std=c++14` or 14 above in your Xcode building settings
#include <memory>

- (void)initializeInstrument
{
  facebook::react::CxxNativeModulePerfLogger logger = facebook::react::CxxNativeModulePerfLogger();
  facebook::react::BridgeNativeModulePerfLogger::enableLogging(std::make_unique<facebook::react::CxxNativeModulePerfLogger>(logger));
}

```

To test it, I integrate these `CxxNativeModulePerfLogger` to [`RNTester`](https://github.com/facebook/react-native/tree/main/packages/rn-tester) and run `RNTester` locally, then use Instrument to collect performance data. 

1. Open the instrument,  choose `Blank` template in Instrument ![image-20210926173249306](https://tva1.sinaimg.cn/large/008i3skNgy1guu5vkkpu7j61880aemyj02.jpg)

2. Then , add `os_signpost`  to current template,  and start recording. 

![image-20210926173031427](https://tva1.sinaimg.cn/large/008i3skNgy1gv18fu3r64j628009c40202.jpg)

3. pay attention to our customed logger Volume. There are 4 parts, 

- `async_method_call`:  collect data for asynchronous methods running through JSBridge
- `js_require`  : used to calculate time interval to initialize modules 
- `module` : used to calculate time interval to create `RCTModuleData` 
- `sync_method_call` :  collect data for synchronous methods running through JSBridge

![image-20210926174136104](https://tva1.sinaimg.cn/large/008i3skNgy1gv18fv7bi9j61ht0u0wjn02.jpg)

For example, after scrolling a FlatList, I checked `async_method_call` volume. I saw `createView` , `setChildren` and `managedChildren` mthods  in `UIManager`  are called frequently. In `1.91` second, `createView` called for 2886 times, costing `1.16` seconds. Although the average duration is about `402.27us`  , which may looks like it is not a big deal; considering its frequency, this method finally tops first in our analysis panel. In this `summary` panel,  you can see `total`, `min`, and `avg`  time intervals for a specific method. 

![image-20210926175209847](https://tva1.sinaimg.cn/large/008i3skNgy1gv18hjuyerj61jk0u0ti402.jpg)

Also, you may would like to check the time interval to load module data. 

![image-20210926180112597](https://tva1.sinaimg.cn/large/008i3skNgy1gv18hn21d0j61g70u0agn02.jpg)
