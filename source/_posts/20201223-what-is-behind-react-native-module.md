---
title: Dive into react native module
date: 2020-12-23 21:50:17
tags:
  - Dive into React Native
categories: 
  - [iOS, React Native]
---

[Native Modules](https://reactnative.dev/docs/native-modules-ios) is one of the key part of React Native which enables JavasScript to call methods in iOS or Android native implementation.

 It is quite interesting to explore the source code about native modules in react native repo. In this article, I would like to talk something about how React Native register, initialize native modules and what is behind the method calling from JavaScript side to iOS Objective-C modules. 

First of all, let's take look at [RCTBridgeModule`](https://github.com/facebook/react-native/blob/23717e48aff3d7fdaea30c9b8dcdd6cfbb7802d5/React/Base/RCTBridgeModule.h#L63) , which is a protocol provides interface needed to register a bridge module.   

```objective-c
#define RCT_EXPORT_MODULE(js_name)  
#define RCT_EXPORT_METHOD(method)
+ (NSString *)moduleName;

@optional
+ (BOOL)requiresMainQueueSetup;
```

## Export and Register Modules

`RCT_EXPORT_MODULE` is in `RCTBridgeModule` protocol, You can use this macro to export your iOS module. 

> Place this macro in your class implementation to automatically register your module with the bridge when it loads. The optional js_name argument. It will be used as the JS module name. If omitted, the JS module name will match the Objective-C class name.

For example, `RCT_EXPORT_MODULE` is used in [RNCAsyncStorage.m#L404, react-native-async-storage](https://github.com/react-native-async-storage/async-storage/blob/bfd76c7508bcc35d88f6b6c8d2fd525466f77ba0/ios/RNCAsyncStorage.m#L404).

This macro will be replaced by the following code in the preprocessor phase when you are compiling a source file. 

```objective-c
RCT_EXTERN void RCTRegisterModule(Class); 
+ (NSString *)moduleName { return @#js_name; } 
+ (void)load { RCTRegisterModule(self); }
```

In Objc world, `load` method is invoked whenever a class is added to the Objective-C runtime at the very beginning of app launch. At this time, it invokes `RCTRegisterModule` function to add the class object into the  `RCTModuleClasses` array. `RCTModuleClasses` is an array containing a list of registered classes.

```objective-c
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
    RCTModuleClassesSyncQueue =
        dispatch_queue_create("com.facebook.react.ModuleClassesSyncQueue", DISPATCH_QUEUE_CONCURRENT);
  });

  RCTAssert(
      [moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
      @"%@ does not conform to the RCTBridgeModule protocol",
      moduleClass);

  // Register module
  dispatch_barrier_async(RCTModuleClassesSyncQueue, ^{
    [RCTModuleClasses addObject:moduleClass];
  });
}
```

When adding current module class into `RCTModuleClasses` array,  it uses `dispatch_barrier_async`  to dispatch this task to a concurrent queue, `RCTModuleClassesSyncQueue`. Using  `dispatch_barrier_async` makes sure that 
one async block executing at a time in `RCTModuleClassesSyncQueue`. 

As we mentioned just now,  `RCTModuleClasses` is a global array containing a list of registered classes.

```
Printing description of RCTModuleClasses:
<__NSArrayM 0x7fe4dc5d6bb0>(
....
RNSVGTSpanManager,
RNSVGUseManager,
RCTBaseTextViewManager,
RCTTextViewManager,
RNLinearTextGradientViewManager,
RCTVirtualTextViewManager,
RNVirtualLinearTextGradientViewManager,
RCTAccessibilityManager,
RCTActionSheetManager,
RCTActivityIndicatorViewManager,
RCTAlertManager,
.....
)
```

## Register and Initialize Modules 

1. **Modules in `RCTModuleClasses` are registered and initialized in [start](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L364)  phase in `RCTCxxBridge.mm`**.  In [initializeModules:withDispatchGroup:lazilyDiscovered](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L854), react native firstly registers all these `automatically-exported` modules.[code here](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L854); then store them  into array `_moduleClassesByID`, a bunch of pointers to `Class` objects, and `_moduleDataByID` referring to a bunch of `RCTModuleData` object. [code here](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L727). The back trace for module registering is as follow: 

  ![image-20201001142132537](image-20201001142132537.png)

2. **If the module needs to be set up in `main` thread. It will be guaranteed running in Main thread**. [code here](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L877) and [here](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L935).  It makes sense that [react native doc](https://reactnative.dev/docs/native-modules-ios#implementing--requiresmainqueuesetup) suggests us return `NO` in the `requiresMainQueueSetup` method.
  
> If your module does not require access to UIKit, then you should respond to `+ requiresMainQueueSetup` with `NO`.

3. When the `RNBridge` is initialized. It inits the `RCTCxxBridge` and `starts` it as current bridge.`RNBridge` in Objc realm basically is a wrapper of `RCTCxxBridge.mm`. At that time, RN [registers extra modules](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L738). 

Just as what  [*Lorenzo S.*](https://medium.com/u/b25eae7ad28?source=post_page-----9bb82659792c--------------------------------) shared in this  [talk](https://www.youtube.com/watch?v=7gm0owyO8HU) . The initialization of all modules in `RCTCxxBridge` start phase slows down its launch time. The more native module you have, the longer time it takes to initialize these modules even you won't use them on the first page. 


## Store modules

In the registration phase, in [`[RCTCXXBridge  _registerModulesForClasses:lazilyDiscovered:]`](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L681) , react native stores a list of registered classes into  a dictionary `_moduleDataByName`. The key is the `moduleName`, the value is the `RCTModuleData`. 

```
  NSMutableDictionary<NSString *, RCTModuleData *> *_moduleDataByName;
```

Remember, in registering phase, The classes of modules ` are stored in `_moduleClassesByID`. While, `_moduleDataByID` stores `RCTModuleData`.

```
  // Native modules
  NSMutableArray<RCTModuleData *> *_moduleDataByID;
  NSMutableArray<Class> *_moduleClassesByID;
```

### RCTModuleData

Now, you may wondering what is `RCTModuleData`? While, it is just the data structure to hold data for react native module. 

![image-20201215174307790](image-20201215174307790.png)

`RCTBridgeModuleProvider` in `RCTModuleData` is for initializing an `instance` of `RCTBridgeModule`. Then, `RCTModuleData` retain this `instance`.  In initialization phase, `RCTModuleData`  creates an instance for the `module class`  by `RCTBridgeModuleProvider` . What `RCTBridgeModuleProvider` does is to provide an instance using`[moduleClass new]`.  [Code ref](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/Base/RCTModuleData.mm#L104).


```objective-c
- (instancetype)initWithModuleClass:(Class)moduleClass
                             bridge:(RCTBridge *)bridge
{
  return [self initWithModuleClass:moduleClass
                    moduleProvider:^id<RCTBridgeModule>{ return [moduleClass new]; }
                            bridge:bridge];
}
```

### Get Module by name or class 

`RCTBridge` bridge is a simple wrapper for the `RCTCxxBridge` , to make the methods in `RCTCxxBridge.mm` available for other Objective-C objects.  You can access to these two methods to get the module from Objective-C objects. 

```objc
// RCTBridge 
- (id)moduleForName:(NSString *)moduleName
- (id)moduleForClass:(Class)moduleClass
```

They will look up the `_moduleDataByName` dictionary to find out the target  `RCTModuleData` and get its `instance`. [Source code here](https://github.com/facebook/react-native/blob/e125f12c01262c11d70c1015139d5f72c5576042/React/CxxBridge/RCTCxxBridge.mm#L529)

## Module-related Class

You might be interested in the relationship between these classes.  

![](module_diagram.png)

## [RCTNativeModule](https://github.com/facebook/react-native/blob/master/React/CxxModule/RCTNativeModule.mm) and ModuleRegistry

In `RCTNativeModule.mm`, `RCTNativeModule`, this c++ class inherits from the base class, `NativeModule`, which holds the info for modules.
The constructor method in `RCTNativeModule` takes in two parameters, `RCTBridge` object and `RCTModuleData` object. 

### Invoke method in RCTNativeModule

The `invoke` method in `RCTNativeModule` 

1. firstly get the `methodName` by `methodId`

2. get the execution queue for current module 

3. construct a block, which calls [`invokeInner`](https://github.com/facebook/react-native/blob/306a8adade432ae5c12f0a13130c4e599d64c642/React/CxxModule/RCTNativeModule.mm#L135). `invokeInner` [calls](https://github.com/facebook/react-native/blob/306a8adade432ae5c12f0a13130c4e599d64c642/React/CxxModule/RCTNativeModule.mm#L181)  [`invokeWithBridge:module:arguments:`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/Base/RCTModuleMethod.mm#L519) in the `RCTModuleMethod.mm`.

   ```c++
     invokeInner(weakBridge, weakModuleData, methodId, std::move(params), callId, isSyncModule ? Sync : Async);
   ```

![image-20201029152216662](image-20201029152216662.png)

4. if current module is executed in `JSThread`, then block for invoking the method is executed synchronously. But if methods in current module should be executed in other threads, react native will dispatch the previous block into the related queue. 

 ```c++
   dispatch_queue_t queue = m_moduleData.methodQueue;
   const bool isSyncModule = queue == RCTJSThread;
   if (isSyncModule) {
     block();
     BridgeNativeModulePerfLogger::syncMethodCallReturnConversionEnd(moduleName, methodName);
   } else if (queue) {
     BridgeNativeModulePerfLogger::asyncMethodCallDispatch(moduleName, methodName);
     dispatch_async(queue, block);
   }
 ```

### NativeModuleProxy 

- In the Runtime initialization phase, `RCTxxBridge.start->Instance.initializeBridge->NativeToJSBridge.initializeRuntime->JSIExecutor.initializeRuntime`,

 an `NativeModuleProxy` object is created and set as `nativeModuleProxy` property to the `global` object in JavaScript runtime context. 

```c++
void JSIExecutor::initializeRuntime() {
 ...
  runtime_->global().setProperty(
      *runtime_,
      "nativeModuleProxy",
      Object::createFromHostObject(
          *runtime_, std::make_shared<NativeModuleProxy>(nativeModules_)));
 ...
 }
```

- In `NativeModule.js`, JavaScript side can get a bunch of `native modules` from this `nativeModuleProxy` property from `global` object. 

```js
let NativeModules: {[moduleName: string]: $FlowFixMe, ...} = {};
if (global.nativeModuleProxy) {
  NativeModules = global.nativeModuleProxy;
```

- In the iOS side, the `NativeModuleProxy` class holds a weak pointer for `JSINativeModules` , which actually holds a list of registered native modules. 

```c++
  std::weak_ptr<JSINativeModules> weakNativeModules_; 
```

![image-20201217183117953](image-20201217183117953.png)

## Export Method 

The `RCT_EXPORT_METHOD` macro is used to export method in Objective realm, and will be replaced by the following code

```objective-c
#define RCT_REMAP_METHOD(js_name, method)       \
  _RCT_EXTERN_REMAP_METHOD(js_name, method, NO) \
  
+(const RCTMethodInfo *)RCT_CONCAT(__rct_export__, RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) 
  {                                                                                                          
    static RCTMethodInfo config = {#js_name, #method, is_blocking_synchronous_method};                       
    return &config;                                                                                          
  }
```

Basically, what it does is to construct a `RCTMethodInfo` structure.  

```objective-c
typedef struct RCTMethodInfo {
  const char *const jsName;
  const char *const objcName;
  const BOOL isSync;
} RCTMethodInfo;
```

The  [calculateMethods](https://github.com/facebook/react-native/blob/23717e48aff3d7fdaea30c9b8dcdd6cfbb7802d5/React/Base/RCTModuleData.mm#L294)  extracts exported methods and get the `IMP` of these methods.  `IMP` actually is a c function which takes in `class` and `selector` as its parameters. And then, `RCTMethodInfo` and `moduleClass` are used to construct `RCTModuleMethod`, which confirms to `RCTBridgeMethod`.

### RCTBridgeMethod 

The `RCTModuleData` also contains information about exported methods.

```objective-c
/**
 * Returns the module methods. Note that this will gather the methods the first
 * time it is called and then memoize the results.
 */
@property (nonatomic, copy, readonly) NSArray<id<RCTBridgeMethod>> *methods;

/**
 * Returns a map of the module methods. Note that this will gather the methods the first
 * time it is called and then memoize the results.
 */
@property (nonatomic, copy, readonly) NSDictionary<NSString *, id<RCTBridgeMethod>> *methodsByName;
```

The [getter methods](https://github.com/facebook/react-native/blob/23717e48aff3d7fdaea30c9b8dcdd6cfbb7802d5/React/Base/RCTModuleData.mm#L373) for these two will generate a list of `RCTBridgeMethod` objects. `RCTBridgeMethod` is a protocol. Any class confirms to this protocol has to implement `JSMethodName`,  `functionType` and `invokeWithBridge: module:arguments:` function. 

```objective-c
@protocol RCTBridgeMethod <NSObject>

@property (nonatomic, readonly) const char *JSMethodName;
@property (nonatomic, readonly) RCTFunctionType functionType;

- (id)invokeWithBridge:(RCTBridge *)bridge module:(id)module arguments:(NSArray *)arguments;

@end
```

So when react native triggers these two getter method? Well, in the `RCTUIManager`

```
RCT_EXPORT_METHOD(dispatchViewManagerCommand
                  : (nonnull NSNumber *)reactTag commandID
                  : (id /*(NSString or NSNumber) */)commandID commandArgs
                  : (NSArray<id> *)commandArgs)
```

Beside, `RCTModuleData` also sets up and holds the thread `methodQueue` for the native module.  This `dispatch_queue_t` object is also retained in [RCTBridgeModule.h](https://github.com/facebook/react-native/blob/e54ead6556f813fc622fd5e1f27ab2b41fa4f213/React/Base/RCTBridgeModule.h#L155), which is used to call all exported methods.

```objc
- (void)setUpMethodQueue
{
  if (_instance && !_methodQueue && _bridge.valid) {
    RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"[RCTModuleData setUpMethodQueue]", nil);
    BOOL implementsMethodQueue = [_instance respondsToSelector:@selector(methodQueue)];
    if (implementsMethodQueue && _bridge.valid) {
      _methodQueue = _instance.methodQueue;
    }
    if (!_methodQueue && _bridge.valid) {
      // Create new queue (store queueName, as it isn't retained by dispatch_queue)
      _queueName = [NSString stringWithFormat:@"com.facebook.react.%@Queue", self.name];
      _methodQueue = dispatch_queue_create(_queueName.UTF8String, DISPATCH_QUEUE_SERIAL);

      // assign it to the module
      if (implementsMethodQueue) {
        @try {
          [(id)_instance setValue:_methodQueue forKey:@"methodQueue"];
        } @catch (NSException *exception) {
          RCTLogError(
              @"%@ is returning nil for its methodQueue, which is not "
               "permitted. You must either return a pre-initialized "
               "queue, or @synthesize the methodQueue to let the bridge "
               "create a queue for you.",
              self.name);
        }
      }
    }
    RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"");
  }
}
```

## How does JS invoke a method in Native?

I set a breakpoint at   [`[RCTModuleMethod invokeWithBridge:module:arguments:]`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/Base/RCTModuleMethod.mm#L519).

![image-20201217171417579](image-20201217171417579.png)

1. in initialization phase, `nativeFlushQueueImmediate` property is set in the global object of the JavaScript execution context.. [code reference here](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/jsiexecutor/jsireact/JSIExecutor.cpp#L90). `    global.nativeFlushQueueImmediate(queue)` is called in `enqueueNativeCall` from the Javascript side. [code ref.](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/Libraries/BatchedBridge/MessageQueue.js#L318)  Before calling, it makes sure the last Flush is 5 milliseconds ago;  then `nativeFlushQueueImmediate` is triggered.  

```js
 if (
      global.nativeFlushQueueImmediate &&
      now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS
    ) {
      const queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      // 🌟
      global.nativeFlushQueueImmediate(queue);
    }
```

2. [`call` function in JSCRuntime](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/jsi/JSCRuntime.cpp#L1151) is invoked when there is a method call from JavaScrip side, then the `host function` is triggered. In this case, the host function is `nativeFlushQueueImmediate` in the C++ realm.

```js
JSValueRef call(
        JSContextRef ctx,
        JSObjectRef function,
        JSObjectRef thisObject,
        size_t argumentCount,
        const JSValueRef arguments[],
        JSValueRef *exception) {
  HostFunctionMetadata *metadata =
          static_cast<HostFunctionMetadata *>(JSObjectGetPrivate(function));
  JSCRuntime &rt = *(metadata->runtime);
  ...
     // 🌟🌟
     metadata->hostFunction_(rt, thisVal, args, argumentCount));
  ...
}
```

3. See the following C++ lambda is used to create the host function `nativeFlushQueueImmediate`. In side this  C++ lambda,  we can see it calls [callNativeModules](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/jsiexecutor/jsireact/JSIExecutor.cpp#L106).

```c++
 runtime_->global().setProperty(
      *runtime_,
      "nativeFlushQueueImmediate",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "nativeFlushQueueImmediate"),
          1,
          [this](
              jsi::Runtime &,
              const jsi::Value &,
              const jsi::Value *args,
              size_t count) {
            if (count != 1) {
              throw std::invalid_argument(
                  "nativeFlushQueueImmediate arg count must be 1");
            }
            callNativeModules(args[0], false);
            return Value::undefined();
          }
      )
  );
```

Then [ `callNativeModules` in `JSToNativeBridge`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/jsiexecutor/jsireact/JSIExecutor.cpp#L390) is invoked.

4. The [callNativeModules in `JSToNativeBridge`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/cxxreact/NativeToJsBridge.cpp#L54) firstly parses the `JSON` data to get the `moduleIds`, `methodIds` and `params`, etc.  [source code here](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/cxxreact/MethodCall.cpp#L38).  After constructing `MethodCall` structure, which holds `moduleId`, `methodId`, `arguments`, `callId`, [`callNativeModules` in `ModuleRegistry.cpp`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/ReactCommon/cxxreact/ModuleRegistry.cpp#L220)  is called.
   
```c++
   // This is always populated
  std::vector<std::unique_ptr<NativeModule>> modules_;

  void ModuleRegistry::callNativeMethod(
    unsigned int moduleId,
    unsigned int methodId,
    folly::dynamic &&params,
    int callId) {
  if (moduleId >= modules_.size()) {
    throw std::runtime_error(folly::to<std::string>(
        "moduleId ", moduleId, " out of range [0..", modules_.size(), ")"));
  }
  modules_[moduleId]->invoke(methodId, std::move(params), callId);
  }
```

`moduleId` is used to look up the `NativeModule` object in a list of modules.  `NativeModule` list is set up in `CxxBridge` initialization phases, which basically is a vector holding module information in C++ realm, transformed from `_moduleDataByID` array in Objc realm. As we know `_moduleDataByID` is a list of  `RCTModuleData` holding registered `RCTBridgeModule` and its instance. 

5. It goes to `invoke` in the [`RCTNativeModule.mm`]( https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/CxxModule/RCTNativeModule.mm#L106), which has module info in its private variable `RCTModuleData` object. 

6. The[`invoke`](https://github.com/facebook/react-native/blob/306a8adade432ae5c12f0a13130c4e599d64c642/React/CxxModule/RCTNativeModule.mm#L74) function in the [`RCTNativeModule.mm`]( https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/CxxModule/RCTNativeModule.mm#L106) calls [`invokeInner`](https://github.com/facebook/react-native/blob/306a8adade432ae5c12f0a13130c4e599d64c642/React/CxxModule/RCTNativeModule.mm#L135). Inside ` invokeInner`, it get the `RCTBridgeMethod` object from `RCTModuleData` object using `methodId` . [code](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/CxxModule/RCTNativeModule.mm#L155)


```objective-c
  id<RCTBridgeMethod> method = moduleData.methods[methodId]
```

Then, it [calls](https://github.com/facebook/react-native/blob/306a8adade432ae5c12f0a13130c4e599d64c642/React/CxxModule/RCTNativeModule.mm#L181)  [`invokeWithBridge:module:arguments:`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/Base/RCTModuleMethod.mm#L519) in the `RCTModuleMethod.mm`. 

![image-20201029152216662](image-20201029152216662.png)


```c++
id result = [method invokeWithBridge:bridge module:moduleData.instance arguments:objcParams];
```

[code ref](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/CxxModule/RCTNativeModule.mm#L181)

<img src="image-20201029162107074.png" alt="image-20201029162107074" style="zoom:50%;" />



7. The  [`invokeWithBridge:module:arguments:`](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/Base/RCTModuleMethod.mm#L519)  uses [ NSInvocation ](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/Base/RCTModuleMethod.mm#L566) to send message to the related `module` object. 

>  An `NSInvocation` object contains all the elements of an Objective-C message: a target, a selector, arguments, and the return value.

​
7.1.  [parse MethodSignature](https://github.com/facebook/react-native/blob/17a8737ecb68f61a14f54bf0e2efeb618f97d326/React/Base/RCTModuleMethod.mm#L204) to init `NSInvocation`.

```objective-c
_selector = NSSelectorFromString(RCTParseMethodSignature(_methodInfo->objcName, &arguments));
....  
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
...
// set  selector  
invocation.selector = _selector;  
```

7.2, trigger the method in the module  

```objective-c
 [_invocation invokeWithTarget:module];
```

In a nutshell,  when JavaScript side calls a method in a specific native module, the indices of the module and method are passed to Objc through JSCRuntime. These information is serialized as JSON object. By looking up the table which holds the information about registered modules and methods, React Native can invoke the specific method in a specific module in the Objective-C realm.  There are some [restrictions](https://github.com/react-native-community/discussions-and-proposals/blob/master/proposals/0002-Turbo-Modules.md#motivation) in this workflow. 

![image-20201001143632128](image-20201001143632128.png)

> 1. Native Modules are specified in [a package](https://github.com/facebook/react-native/blob/1e8f3b11027fe0a7514b4fc97d0798d3c64bc895/local-cli/link/__fixtures__/android/0.20/MainActivity.java#L37) and are eagerly initialized. The startup time of React Native increases with the number of Native Modules, even if some of those Native Modules are never used. 
> 2. There is no simple way to check if the Native Modules that JavaScript calls are actually included in the native app. With over the air updates, there is no easy way to check if a newer version of JavaScript calls the right method with the correct set of arguments in Native Module.
> 3. Native Modules are always singleton and their lifecycle is typically tied to the lifetime of the bridge. This issue is compounded in brownfield apps where the React Native bridge may start and shut down multiple times.
> 4. During the startup process, Native Modules are typically specified in [multiple packages](https://github.com/facebook/react-native/blob/b938cd524a20c239a5d67e4a1150cd19e00e45ba/ReactAndroid/src/main/java/com/facebook/react/CompositeReactPackage.java). We then [iterate](https://github.com/facebook/react-native/blob/407e033b34b6afa0ea96ed72f16cd164d572e911/ReactAndroid/src/main/java/com/facebook/react/ReactInstanceManager.java#L1132) over the list [multiple times](https://github.com/facebook/react-native/blob/617e25d9b5cb10cfc6842eca62ff22d39eefcf7b/ReactAndroid/src/main/java/com/facebook/react/bridge/NativeModuleRegistry.java#L46) before we finally give the bridge a [list of Native Modules](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/CatalystInstanceImpl.java#L124). This does not need to happen at runtime.
> 5. The actual methods and constants of a Native Module are [computed during runtime](https://github.com/facebook/react-native/blob/master/ReactCommon/cxxreact/ModuleRegistry.cpp#L81) using [reflection](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/JavaModuleWrapper.java#L75). 

