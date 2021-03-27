---
title: Understand How MLeaksFinder and FBRetainCycleDetector automatically detect memory leaks

date: 2021-03-19 21:37:38
tags:
---


## Background 

Memory is important resource in iOS. If a application uses too much memory, exceeding the limit based on the device, the iOS system will kill our App, by sending `SIGKILL` signal. Besides, minimizing memory usage not only decreases application’s memory footprint, but also reduce the amount of CPU time it consumes. These are mentioned in several WWDC sessions. 

- [WWDC: performance and power optimization](https://developer.apple.com/videos/play/wwdc2011/312/)
- [Advanced Memory Management Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)
- https://developer.apple.com/videos/play/wwdc2018/219
- https://developer.apple.com/videos/play/wwdc2018/416/
- https://developer.apple.com/videos/play/wwdc2020/10078/?time=256

 Obviously, it is important to keep memory under control. In our daily life, we usually use Xcode memory debugger tool and instruments to detect memory leaks. Basically, lots of manual work. A better to integrate memory leak detection into internal test phase or regression phase. The earlier we detect the issue, the more time we got to fix it. The less efforts we put in checking memory leaks, the more likely we keep our app away from memory leaks. Using `MLeaksFinder` and `FBRetainCycleDetector` is a good solution. 

## What is MLeaksFinder for? 

`MLeaksFinder` is an light-weight tool from `WeChat` team, `Tencent`. It automatically finds leaks in some specific objects. When leaks happening, it will present an alert showing the leaked object and backtrace. 

## What does MLeaksFinder do under the hood? 

The basic idea is to set a timer when the object is about to be released. When the timer is triggered, checked if the reference to the object is still valid. If it is, it turns out this object is leaked. Then, it uses this leaked object as seed object to `FBRetainCycleDetector` to figure out the retain cycle using. Actually, I found lots of articles introducing `MLeakdsFinder` are Chinese and outdated. While its source code is a bit easy to read.  

Basing on this idea, `MLeaksFinder` has several categories for [these classes](https://github.com/Tencent/MLeaksFinder/tree/master/MLeaksFinder): 
- NSObject+MemoryLeak
- UIApplication+MemoryLeak
- UINavigationController+MemoryLeak
- UIPageViewController+MemoryLeak
- UISplitViewController+MemoryLeak
- UITabBarController+MemoryLeak
- UITouch+MemoryLeak
- UIView+MemoryLeak
- UIViewController+MemoryLeak


Take `UIViewController` as an example, it swizzles `viewDidDisappear:` method, then checks if current view controller have been popped by `UINavigationController`. Because the view controller isn't necessarily popped from view controller stack when `viewDidDisappear:` called. Maybe, another view controller just has been pushed into the view controller stack, cover it and showing on the screen.

```c++
- (void)swizzled_viewDidDisappear:(BOOL)animated {
    [self swizzled_viewDidDisappear:animated];
    
    if ([objc_getAssociatedObject(self, kHasBeenPoppedKey) boolValue]) {
        [self willDealloc];
    }
}
```

`kHasBeenPoppedKey` tag here is set by [UINavigationController, code here](https://github.com/Tencent/MLeaksFinder/blob/master/MLeaksFinder/UINavigationController%2BMemoryLeak.m#L56). 

![image-20210319225704137](image-20210319225704137.png)
As this pic demonstrates, if the view controller was released, the reference the block captured 2 seconds ago is `nil`. If this view controller isn't released, `strongSelf` here would be a valid base address to it. Then `MLeakFinder` will show an alter to warning users. 

We have talked about view controller, how about views?  Well, in `willDealloc` method in `UIViewController`, MLeaksFinder will run self.view's `willDealloc`; then check `subviews` Array. Basically, the view tree in this view controller will be traversed through and checked. 

```c++
@implementation UIView (MemoryLeak)
​
- (BOOL)willDealloc {
    if (![super willDealloc]) {
        return NO;
    }
    
    [self willReleaseChildren:self.subviews];
    
    return YES;
}
​
@end
```

![image-20210314194009331](image-20210314194009331.png)

If you enable the `FBRetainCycleDetector` through macro, the current leaked object will be the seed object for FBRetainCycleDetector, which will detect the retain cycle. 

## What is FBRetainCycleDetector for? 

Facebook has a dedicated article about the [FBRetainCycleDetector](https://engineering.fb.com/2016/04/13/ios/automatic-memory-leak-detection-on-ios/)
> Finding retain cycles in Objective-C is analogous to finding cycles in a directed acyclic graph in which nodes are objects and edges are references between objects (so if object A retains object B, there exists reference from A to B). Our Objective-C objects are already in our graph; all we have to do is traverse it with a depth-first search.

So, in order to traverse the directed graph, how to get neighbors of each node? How to get objects each node references? For each node in the graph, it could be either an object or block.

## References in object
### strong ivars 

For objects, `FBRetainCycleDetector` get its **ivar list** from the object. 

> The first thing we can do is grab the layout of all an object's instance variables (the “ivar layout”). For a given object, an ivar layout describes where we should look for other objects that it references.

```
const uint8_t *fullLayout = class_getIvarLayout(aCls);

Ivar *class_copyIvarList(Class cls, unsigned int *outCount)

```

[`FBClassStrongLayout` here](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Layout/Classes/FBClassStrongLayout.mm#L184)
1. get ivar list for current, its superclass, all the way up to its ancestor
2. cache ivar list in a map, `<Class, NSArray<FBObjectReference>>`

For the following class, there are 4 strong references to others, 2 weak reference.   


```c++
@interface _RCDTestClassWithMixedWeakAndStrongProperties : NSObject
@property (nonatomic, strong) NSObject *object1;
@property (nonatomic, strong) NSObject *object2;
@property (nonatomic, strong) NSObject *object3;
@property (nonatomic, weak) NSObject *object4;
@property (nonatomic, strong) NSObject *object5;
@property (nonatomic, weak) NSObject *object6;
@end
```

using  `class_copyIvarList`, we can see its Ivar list. Each pointer is 8-byte in memory in 64-bit device, we an see the `offset` for first ivar to the class base address is `8 bytes`; the second ivar is `16 bytes`, the third one is `24bytes`, etc. 

<img src="image-20210320160316683.png" width="330" height="600">

Then, use `class_getIvarLayout` to get ivar layout 

```objective-c
  const uint8_t *fullLayout = class_getIvarLayout(aCls);
```

The full layout is 

```
"\x03\x11"
```
- `\x03` indicates that there are zero non-strong ivar and 3 strong ivar, `_object1`, `_object2`, `_object3`. 
- `x11` claims that there comes 1 non-strong ivar,  weak `_object4` in above declaration. And then follows 1 strong ivar `object5`

```c++
static NSIndexSet *FBGetLayoutAsIndexesForDescription(NSUInteger minimumIndex, const uint8_t *layoutDescription) {
  NSMutableIndexSet *interestingIndexes = [NSMutableIndexSet new];
  NSUInteger currentIndex = minimumIndex;

  while (*layoutDescription != '\x00') {
    int upperNibble = (*layoutDescription & 0xf0) >> 4;
    int lowerNibble = *layoutDescription & 0xf;

    // Upper nimble is for skipping
    currentIndex += upperNibble;

    // Lower nimble describes count
    [interestingIndexes addIndexesInRange:NSMakeRange(currentIndex, lowerNibble)];
    currentIndex += lowerNibble;

    ++layoutDescription;
  }

  return interestingIndexes;
}
```

Parsing layout get a index set for strong ivars. 

```
<NSMutableIndexSet: 0x7fb7aea8ef40>[number of indexes: 4 (in 2 ranges), indexes: (1-3 5)]
```

Filter out the fourth and sixth ivar. 
<img src="image-20210320161003483.png" width="330" height="600">


There are other interesting cases in the [FBClassStrongLayoutTests.mm](https://github.com/facebook/FBRetainCycleDetector/blob/master/FBRetainCycleDetectorTests/FBClassStrongLayoutTests.mm), the ivar type could be structure or block, and it could be weak as well. 

### References to associated objects 

`FBRetainCycleDetector` hooks the calls, `objc_setAssociatedObject` and `objc_removeAssociatedObjects`. Then it store objects and a set of pointers to their associated objects into a global map. 

```c++
using ObjectAssociationSet = std::unordered_set<void *>;
using AssociationMap = std::unordered_map<id, ObjectAssociationSet *>;
```


### Block and captured object 

What attracts me most is the capability in FBRetainCycleDetector to detect leaked blocks and its reference.  [Amazing method to get references from the block](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Layout/Blocks/FBBlockStrongLayout.m#L79) and strong reference layout in block.  

>  What we can use is [application binary interface for blocks](http://clang.llvm.org/docs/Block-ABI-Apple.html) (ABI). It describes how the block will look in memory. If we know that the reference we are dealing with is a block, we can cast it on a fake structure that imitates a block. After casting the block to a C-struct we know where objects retained by the block are kept.

**ABI for block**

First of all, let's take a look at the Block Literal. 

For a block like this

```c++
^ { printf("hello world\n"); }
```

It will be compiled into 

```c++
struct __block_literal_1 {
    void *isa; // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_1 *); // 🌟🌟🌟 The invoke function pointer is set to a function that takes the Block structure as its first argument and the rest of the arguments (if any) to the Block and executes the Block compound statement.
    struct __block_descriptor_1 *descriptor;
};

void __block_invoke_1(struct __block_literal_1 *_block) {
    printf("hello world\n");
}

static struct __block_descriptor_1 {
    unsigned long int reserved;
    unsigned long int Block_size; // the size of the following Block literal structure.
} __block_descriptor_1 = { 0, sizeof(struct __block_literal_1) }
```

and 

```c++
struct __block_literal_1 _block_literal = {
     &_NSConcreteStackBlock,
     (1<<29), <uninitialized>,
     __block_invoke_1,
     &__block_descriptor_1
};
```
This is the initialization of the block literal structure.  

#### What if the block has reference to others?

1. Case 1: Variables are imported as `const` copies.

```objective-c
int x = 10;
void (^vv)(void) = ^{ printf("x is %d\n", x); }
x = 11;
vv();
```

It will be compiled into 

```c++
struct __block_literal_2 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_2 *);
    struct __block_descriptor_2 *descriptor;
    const int x; // const copy variable x 
};

void __block_invoke_2(struct __block_literal_2 *_block) {
    printf("x is %d\n", _block->x);
}

static struct __block_descriptor_2 {
    unsigned long int reserved;
    unsigned long int Block_size;
} __block_descriptor_2 = { 0, sizeof(struct __block_literal_2) };
```

```c++
struct __block_literal_2 __block_literal_2 = {
      &_NSConcreteStackBlock,
      (1<<29), <uninitialized>,
      __block_invoke_2,
      &__block_descriptor_2,
      x
 };
```
We can see the variable `x` is appended at the end of `__block_literal_2` structure.  

2. Case 2: Variables of `__block` storage class are imported as a pointer to an enclosing data structure. see [more here]([Imported  copy of  reference](https://clang.llvm.org/docs/Block-ABI-Apple.html#id5))

From the above cases, we can see in the  descriptor structure `__block_descriptor_1`,  the `Block_size` field is sizeof(struct ` __block_literal_1`) . This is a very import field.  `FBRetainCycleDetector` uses it to get the number of pointers inside

```c++
  void (*dispose_helper)(void *src) = blockLiteral->descriptor->dispose_helper;
  const size_t ptrSize = sizeof(void *);

  // Figure out the number of pointers it takes to fill out the object, rounding up.
  const size_t elements = (blockLiteral->descriptor->size + ptrSize - 1) / ptrSize;

```

Let's take a look at a test case here. Supposed a block captures an object from out scope.

```c++
 NSObject *object = [NSObject new];
  
  void (^block)() = ^{
    __unused NSObject *someObject = object;
  };
  
  NSArray *retainedObjects = FBGetBlockStrongReferences((__bridge void *)(block));
```

The value of `blockLiteral->descriptor->size` is 40, indicating the block 40 bytes in memory; in ARM64 device, the pointer size a `8` bytes.  So it needs 5 pointers to fill out the fake object. 

```
(lldb) p blockLiteral->descriptor->size
(unsigned long) $0 = 40
```

>  We create an object that pretends to be a block we want to investigate. Because we know the block’s interface, we know where to look for references this block holds. In place of those references our fake object will have “release detectors.” Release detectors are small objects that are observing release messages sent to them. These messages are sent to strong references when an owner wants to relinquish ownership. We can check which detectors received such a message when we deallocate our fake object. Knowing which indexes said detectors are in the fake object, we can find actual objects that are owned by our original block.

Create detector for each of the pointer in the faked object. 

```c++
// Create a fake object of the appropriate length.
  void *obj[elements];
  void *detectors[elements];

  for (size_t i = 0; i < elements; ++i) {
    FBBlockStrongRelationDetector *detector = [FBBlockStrongRelationDetector new];
    obj[i] = detectors[i] = detector;
  }
```
Now faked object `obj` contains 5 references to 5 `FBBlockStrongRelationDetector` instances.  
```
(void *[]) obj = ([0] = 0x00007fc07b41c370, [1] = 0x00007fc07b41f4b0, [2] = 0x00007fc07b42c080, [3] = 0x00007fc07b422820, [4] = 0x00007fc07b42e420)
```

![image-20210320170900864](image-20210320170900864.png)

Then, it try to dispose the faked object. 

```c++
 @autoreleasepool {
    dispose_helper(obj);
  }
```
The disposing actually trigger sending `release` message to `FBBlockStrongRelationDetector`, in which `release` message has been overridden and set `_strong` ivar to `YES`
```
FBBlockStrongRelationDetector
// set _strong as YES when received release message
- (oneway void)release
{
  _strong = YES;
}
```

![image-20210320172426371](image-20210320172426371.png)

Finally get the index of the strong reference of current block by figuring out in which `FBBlockStrongRelationDetector`, `_strong` is `YES`.  

```
<NSMutableIndexSet: 0x7fc07b41f560>[number of indexes: 1 (in 1 ranges), indexes: (4)]
```


### Detect cycle

To detect the cycle of objects, it is doing DFS over graph of objects.[ code here](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Detector/FBRetainCycleDetector.mm#L89).  

![image-20201008115603415](image-20201008115603415.png)

## Impact on memory footprint 

- **MLeaksFinder** is light-weight. It has **few** impact on memory footprint 
- **FBRetainCycleDetector has impact** on the memory footprint**.** The upside is that **MLeaksFinder** triggers **DFS in FBRetainCycleDetector** on when the user click `Retain Cycle` button in the alter. After the Alert is dismissed, most of the memory usage will be gone. 


## Summary:

- **FBRetainCycleDetector** is quite powerful. It can even detect leaks related to Blocks. But it is a bit slow since it uses `DFS` algorithm to traverse the object tree. Besides, there is potential risks of data race in [associated manager](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Associations/FBAssociationManager.mm#L136). 
- **MLeaksFinder** is simple but tricky. So once it detects the leaked object, it use **FBRetainCycleDetector to detect the retain cycle.** Then it shows the **alter.**
- We can use `MLeaksFinder` to detect some seed objects. and provide **FBRetainCycleDetector**  with these candidate objects from which it will start detection.