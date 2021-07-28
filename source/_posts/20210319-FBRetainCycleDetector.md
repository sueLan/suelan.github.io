---
title: Understand How MLeaksFinder and FBRetainCycleDetector automatically detect memory leaks

date: 2021-03-19 21:37:38
categories:
  - iOS
---


# Background 

Memory is important resource in iOS. If a application uses too much memory, exceeding the limit based on the device, the iOS system will kill this App by sending `SIGKILL` signal. Besides, minimizing memory usage not only decreases application’s memory footprint, but also reduces the amount of CPU time it consumes. These are mentioned in several WWDC sessions. 

- [WWDC: performance and power optimization](https://developer.apple.com/videos/play/wwdc2011/312/)
- [Advanced Memory Management Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)
- https://developer.apple.com/videos/play/wwdc2018/219
- https://developer.apple.com/videos/play/wwdc2018/416/
- https://developer.apple.com/videos/play/wwdc2020/10078/?time=256

Obviously, it is important to keep memory under control. In our daily life, we usually use Xcode memory debugger tool and instruments to detect memory leaks. Basically, lots of manual works. Maybe, a better way is to integrate memory leak detection into internal test phase or regression phase. The earlier we detect the issue, the more time we got to fix it. The less efforts we put in checking memory leaks manually, the more likely we spend less time to optimal memory issues and keep our app away from memory leaks. So, using `MLeaksFinder` and `FBRetainCycleDetector` in dev or test phase sounds like a good idea. 

# What is MLeaksFinder for? 

As we know, `MLeaksFinder` is an light-weight tool from `WeChat` team, `Tencent`, which automatically finds leaks in some specific objects. When leaks happening, it will present an alert showing the leaked object and backtrace. 

# How does MLeaksFinder work? 

The basic idea for `MLeaksFinder` is to set a timer when the object is about to be released. When the timer is triggered, checked if the reference to the object is still valid. If it is, this object is leaked. Then, it uses this leaked object as seed object for `FBRetainCycleDetector` to figure out the retain cycle using DFS algorithm to traversal object graph. You may see this brief introduction in some Chinese tech articles. While, I found lots of articles introducing `MLeakdsFinder` are outdated. Since its source code is a bit easy to read, let's just start to explore it. 

`MLeaksFinder` has several categories for [these classes](https://github.com/Tencent/MLeaksFinder/tree/master/MLeaksFinder): 
- NSObject+MemoryLeak
- UIApplication+MemoryLeak
- UINavigationController+MemoryLeak
- UIPageViewController+MemoryLeak
- UISplitViewController+MemoryLeak
- UITabBarController+MemoryLeak
- UITouch+MemoryLeak
- UIView+MemoryLeak
- UIViewController+MemoryLeak


Take `UIViewController` as an example, it swizzles `viewDidDisappear:` method, then checks if current view controller has been popped by `UINavigationController`. Why need this check? Because the view controller isn't necessarily popped from view controller stack when `viewDidDisappear:` called. Maybe, another view controller just has been pushed into the view controller stack, cover it and showing on the screen.

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
As this pic demonstrates, if the view controller was released, the reference the block captured 2 seconds ago is `nil`. If this view controller isn't released, `strongSelf` here would be a valid base address to it. Then `MLeakFinder` will show an alter to warn users. 

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

If you enable the `FBRetainCycleDetector` through macro, the current leaked object will be used as  seed object for FBRetainCycleDetector, which will detect the retain cycle. 

# What is FBRetainCycleDetector for? 

Facebook has a dedicated article about the [FBRetainCycleDetector](https://engineering.fb.com/2016/04/13/ios/automatic-memory-leak-detection-on-ios/)
> Finding retain cycles in Objective-C is analogous to finding cycles in a directed acyclic graph in which nodes are objects and edges are references between objects (so if object A retains object B, there exists reference from A to B). Our Objective-C objects are already in our graph; all we have to do is traverse it with a depth-first search.

So, in order to traverse the directed graph, how to get neighbors of each node? How to get objects each node references? For each node in the graph, it could be either an object or block.

# References in object

## strong ivars 

For objects, `FBRetainCycleDetector` get its **ivar list** from the object. 

> The first thing we can do is grab the layout of all an object's instance variables (the “ivar layout”). For a given object, an ivar layout describes where we should look for other objects that it references.

```objective-c
const uint8_t *fullLayout = class_getIvarLayout(aCls);

Ivar *class_copyIvarList(Class cls, unsigned int *outCount)

```

[`FBClassStrongLayout` here](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Layout/Classes/FBClassStrongLayout.mm#L184)
1. Because `class_copyIvarList` won't include instance variables declared by superclasses. This method has to get ivar list for current, its superclass, all the way up to its ancestoiicoder
2. get strong ivar by analyzing ivar layout
3. cache ivar list in a map, `<Class, NSArray<FBObjectReference>>`


Let's understand it deeper by taking an example. For the following class, there are 4 strong references to others, 2 weak reference.

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

```c++
  const uint8_t *fullLayout = class_getIvarLayout(aCls);
```

Basically, the value of `fullLayout` is 

```
"\x03\x11"
```

- In  hexadecimal figure `\x03`, the high bits represents the number of `non-strong` ivar, the lower bits represents the number of `strong ivar`. `\x03` indicates that there are zero non-strong ivar and 3 strong ivar, `_object1`, `_object2`, `_object3` in this case. 
- `x11` claims that there comes 1 non-strong ivar,  weak `_object4` in above declaration; and then follows 1 strong ivar `object5`

The following method is to parse ivar layout according to the above rule and get a set of `NSRange` for index and length for strong ivars in this class. One range is `1 to 3` and the other is `5`. 

```c++
static NSIndexSet *FBGetLayoutAsIndexesForDescription(NSUInteger minimumIndex, const uint8_t *layoutDescription) {
  NSMutableIndexSet *interestingIndexes = [NSMutableIndexSet new];
  NSUInteger currentIndex = minimumIndex;

  while (*layoutDescription != '\x00') {
    // how many non-strong ivar 
    int upperNibble = (*layoutDescription & 0xf0) >> 4; 
    // how many strong ivar
    int lowerNibble = *layoutDescription & 0xf;

    // currentIndex is to track the first idx of strong ivar currently being analyzed from hexodecimal value
    // Upper nimble is for skipping `non-strong` ivar
    currentIndex += upperNibble;

    // Lower nimble describes count of the strong ivar 
    [interestingIndexes addIndexesInRange:NSMakeRange(currentIndex, lowerNibble)];
    currentIndex += lowerNibble;

    ++layoutDescription;
  }

  return interestingIndexes;
}
```

| Idx  | Weak/strong |         |
| ---- | ----------- | ------- |
| 1    | strong      | object1 |
| 2    | strong      | object2 |
| 3    | strong      | object3 |
| 4    | weak        | object4 |
| 5    | strong      | object5 |
| 6    | weak        | object6 |
For the above case, the ivar layout is `"\x03\x11"`

|      | upperNibble | lowerNibble | currentIndex | NSRange |
| ---- | ----------- | ----------- | ------------ | ------- |
| x03  | 0           | 3           | 1            | {1, 3}  |
| x11  | 1           | 1           | 5            | {5, 1}  |



Parsing ivar layout to filter out the 4th and 6th ivar and get a set of index range for strong ivar. The result are two ranges, {1, 3} and {5, 1} 

```
<NSMutableIndexSet: 0x7fb7aea8ef40>[number of indexes: 4 (in 2 ranges), indexes: (1-3 5)]
```

<img src="image-20210320161003483.png" width="375" height="500">


There are other interesting cases in the [FBClassStrongLayoutTests.mm](https://github.com/facebook/FBRetainCycleDetector/blob/master/FBRetainCycleDetectorTests/FBClassStrongLayoutTests.mm), the ivar type can be structure or block, and it can be weak as well. 

## References to associated objects 

`FBRetainCycleDetector` hooks the calls, `objc_setAssociatedObject` and `objc_removeAssociatedObjects`. Then it store objects and a set of pointers to strongly referred associated objects into a global map. 

```c++
using ObjectAssociationSet = std::unordered_set<void *>;
using AssociationMap = std::unordered_map<id, ObjectAssociationSet *>;
```

Using  `OBJC_ASSOCIATION_RETAIN` and `OBJC_ASSOCIATION_RETAIN_NONATOMIC` to trace strong references only

```c++
   if (policy == OBJC_ASSOCIATION_RETAIN ||
          policy == OBJC_ASSOCIATION_RETAIN_NONATOMIC) {
        _threadUnsafeSetStrongAssociation(object, key, value);
   ...
```

## Block and captured objects

What attracts me most is the capability in `FBRetainCycleDetector` to detect leaked blocks and its reference.  [Amazing method to get references from the block](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Layout/Blocks/FBBlockStrongLayout.m#L79) and strong reference layout in block.  

>  What we can use is [application binary interface for blocks](http://clang.llvm.org/docs/Block-ABI-Apple.html) (ABI). It describes how the block will look in memory. If we know that the reference we are dealing with is a block, we can cast it on a fake structure that imitates a block. After casting the block to a C-struct we know where objects retained by the block are kept.

** ABI for block**

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

You can use `clang -rewrite-objc` to convert the Objective-C code into cpp implementation .  
```
clang -rewrite-objc xxxxx.m
```

## What if the block has reference to others?

### Imported const copy variables

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
    // const copy variable x is here 
    const int x; 
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

### Imported const copy of Block reference

In the following case, block `existingBlock` is captured by `vv`.  

- a Block requires `copy/dispose` helpers in block descriptor if it imports any block variables

```c++
void (^existingBlock)(void) = ...;
void (^vv)(void) = ^{ existingBlock(); }
vv();

struct __block_literal_3 {
   ...; // existing block
};

struct __block_literal_4 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_4 *);
    struct __block_literal_4 *descriptor;
    struct __block_literal_3 *const existingBlock;
};

// the invoke function for __block_literal_4
void __block_invoke_4(struct __block_literal_2 *_block) {
   __block->existingBlock->invoke(__block->existingBlock);
}

// copy helper is needed 
void __block_copy_4(struct __block_literal_4 *dst, struct __block_literal_4 *src) {
     //_Block_copy_assign(&dst->existingBlock, src->existingBlock, 0);
     // will increase the reference counting for existingBlock
     _Block_object_assign(&dst->existingBlock, src->existingBlock, BLOCK_FIELD_IS_BLOCK);
}

void __block_dispose_4(struct __block_literal_4 *src) {
     // was _Block_destroy
     // will decrease the reference counting for existingBlock
     _Block_object_dispose(src->existingBlock, BLOCK_FIELD_IS_BLOCK);
}

static struct __block_descriptor_4 {
    unsigned long int reserved;
    unsigned long int Block_size;
    void (*copy_helper)(struct __block_literal_4 *dst, struct __block_literal_4 *src);
    void (*dispose_helper)(struct __block_literal_4 *);
} __block_descriptor_4 = {
    0,
    sizeof(struct __block_literal_4),
    __block_copy_4,
    __block_dispose_4,
};
```

```c++
struct __block_literal_4 _block_literal = {
      &_NSConcreteStackBlock,
      (1<<25)|(1<<29), <uninitialized>
      __block_invoke_4,
      & __block_descriptor_4
      existingBlock,
};
```

### Importing `__block` variables into Blocks

- Variables of `__block` storage class are imported as a pointer to an enclosing data structure. see [more here]([Imported  copy of  reference](https://clang.llvm.org/docs/Block-ABI-Apple.html#id5))

```c++
int __block i = 2;
functioncall(^{ i = 10; });
```
would be compiled into:

```
struct _block_byref_i {
    void *isa;  // set to NULL
    struct _block_byref_voidBlock *forwarding;
    int flags;   //refcount;
    int size;
    void (*byref_keep)(struct _block_byref_i *dst, struct _block_byref_i *src);
    void (*byref_dispose)(struct _block_byref_i *);
    int captured_i;  // the capture variable
};


struct __block_literal_5 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_5 *);
    struct __block_descriptor_5 *descriptor;
    struct _block_byref_i *i_holder;
};
 
void __block_invoke_5(struct __block_literal_5 *_block) {
   _block->forwarding->captured_i = 10;
}

void __block_copy_5(struct __block_literal_5 *dst, struct __block_literal_5 *src) {
     //_Block_byref_assign_copy(&dst->captured_i, src->captured_i);
     _Block_object_assign(&dst->captured_i, src->captured_i, BLOCK_FIELD_IS_BYREF | BLOCK_BYREF_CALLER);
}

void __block_dispose_5(struct __block_literal_5 *src) {
     //_Block_byref_release(src->captured_i);
     _Block_object_dispose(src->captured_i, BLOCK_FIELD_IS_BYREF | BLOCK_BYREF_CALLER);
}

static struct __block_descriptor_5 {
    unsigned long int reserved
    unsigned long int Block_size;
    void (*copy_helper)(struct __block_literal_5 *dst, struct __block_literal_5 *src);
    void (*dispose_helper)(struct __block_literal_5 *);
} __block_descriptor_5 = { 0, sizeof(struct __block_literal_5) __block_copy_5, __block_dispose_5 };
```

and 

```c++
struct _block_byref_i i = {( .isa=NULL, .forwarding=&i, .flags=0, .size=sizeof(struct _block_byref_i), .captured_i=2 )};
struct __block_literal_5 _block_literal = {
      &_NSConcreteStackBlock,
      (1<<25)|(1<<29), <uninitialized>,
      __block_invoke_5,
      &__block_descriptor_5,
      &i,
};
```

- `copy_helper` and `dispose_helper` helper functions are added
- a structure `_block_byref_i` is generated to store  `__block` variable; see `captured_i` in `_block_byref_i`

### import `__block` object 

```c++
void func() {
    __block NSObject *obj = [[NSObject alloc] init];
    void (^blk)(void) = ^() {
        obj = nil;
    };
}
```
-  structure `__Block_byref_obj_0` holds reference to `NSObject *obj` pointer. 
-  need copy/dispose helper function
```c++
struct __Block_byref_obj_0 {
  void *__isa;
  __Block_byref_obj_0 *__forwarding;
  int __flags;
  int __size;
  void (*__Block_byref_id_object_copy)(void*, void*);
  void (*__Block_byref_id_object_dispose)(void*)
  NSObject *obj; // capture __block NSObject *obj 
};

static void __func_block_func_0(struct __func_block_impl_0 *__cself) {
  __Block_byref_obj_0 *obj = __cself->obj; // bound by ref
  (obj->__forwarding->obj) = __null;
}

static void __func_block_copy_0(struct __func_block_impl_0*dst, struct __func_block_impl_0*src) {_Block_object_assign((void*)&dst->obj, (void*)src->obj, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __func_block_dispose_0(struct __func_block_impl_0*src) {_Block_object_dispose((void*)src->obj, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __func_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __func_block_impl_0*, struct __func_block_impl_0*);
  void (*dispose)(struct __func_block_impl_0*);
} 

```

## Block_size

From the above cases, we can see in the  descriptor structure `__block_descriptor_2`,  the `Block_size` field is sizeof(struct ` __block_literal_2`) . This is a very import field.  `FBRetainCycleDetector` uses it to get the number of pointers inside

```c++
  void (*dispose_helper)(void *src) = blockLiteral->descriptor->dispose_helper;
  const size_t ptrSize = sizeof(void *);

  // Figure out the number of pointers it takes to fill out the object, rounding up.
  const size_t elements = (blockLiteral->descriptor->size + ptrSize - 1) / ptrSize;

```

Let's take a look at a test case here. Supposed a block captures an object from outside.

```c++
 NSObject *object = [NSObject new];
  
  void (^block)() = ^{
    __unused NSObject *someObject = object;
  };
  
  NSArray *retainedObjects = FBGetBlockStrongReferences((__bridge void *)(block));
```

The block literal is like this

```
struct BlockLiteral {
  void *isa;  // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
  int flags;
  int reserved;
  void (*invoke)(void *, ...);
  struct BlockDescriptor *descriptor;
  // imported variables
  const void *someObject 
};
```
![](block_literal_x.png)
```
(lldb) p blockLiteral->descriptor->size
(unsigned long) $0 = 40
```

- The value of `blockLiteral->descriptor->size` is 40, indicating the block 40 bytes in memory;
- `int` is 32 bit, `flags` and `reserved` will be put together into one word, 8 bytes in 64bit processor device. 
- In ARM64 device, the pointer size a `8` bytes.  
- So it needs 5 pointers to fill out the fake object. 


>  We create an object that pretends to be a block we want to investigate. Because we know the block’s interface, we know where to look for references this block holds. In place of those references our fake object will have “release detectors.” Release detectors are small objects that are observing release messages sent to them. These messages are sent to strong references when an owner wants to relinquish ownership. We can check which detectors received such a message when we deallocate our fake object. Knowing which indexes said detectors are in the fake object, we can find actual objects that are owned by our original block.

Create detector for each of the pointer in the faked object. 

```c++
// Create a fake object of the appropriate length.
  void *obj[elements];
  void *detectors[elements];

  for (size_t i = 0; i < elements; ++i) {
    // new detectors to detect whether the pointer inside obj is strong or not
    FBBlockStrongRelationDetector *detector = [FBBlockStrongRelationDetector new];
    obj[i] = detectors[i] = detector;
  }
```
Now faked object `obj` contains 5 references to 5 `FBBlockStrongRelationDetector` instances. These 5 detectors are newly created to detect whether the pointer inside obj is strong or not. They are not the original block object in your code, but with same memory layout and reference retain policy.  

```
(void *[]) obj = ([0] = 0x00007fc07b41c370, [1] = 0x00007fc07b41f4b0, [2] = 0x00007fc07b42c080, [3] = 0x00007fc07b422820, [4] = 0x00007fc07b42e420)
```

![image-20210320170900864](image-20210320170900864.png)

Then, try to dispose the faked object. 

```c++
 @autoreleasepool {
    dispose_helper(obj);
  }
```
The disposing of the fake object actually triggers `releasing` of those detectors if they are strongly referred by the fake object only. In FBBlockStrongRelationDetector, `release` message has been overridden and set `_strong` ivar to `YES` to mark the related strong reference in the `blockLiteral`

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


## Detect cycle

To detect the cycle of objects, it is doing DFS over graph of objects.[ code here](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Detector/FBRetainCycleDetector.mm#L89).  

![image-20201008115603415](image-20201008115603415.png)

# Impact on memory footprint 

- **MLeaksFinder** is light-weight. It has **few** impact on memory footprint 
- **FBRetainCycleDetector has impact** on the memory footprint**.** The upside is that **MLeaksFinder** triggers **DFS in FBRetainCycleDetector** on when the user click `Retain Cycle` button in the alter. After the Alert is dismissed, most of the memory usage will be gone. 


# Summary:

- **FBRetainCycleDetector** is quite powerful. It can even detect leaks related to Blocks. But it is a bit slow since it uses `DFS` algorithm to traverse the object tree. Besides, there is potential risks of data race in [associated manager](https://github.com/facebook/FBRetainCycleDetector/blob/1ff2adee84a6ee94a1ae82526104a188774eb90a/FBRetainCycleDetector/Associations/FBAssociationManager.mm#L136). 
- **MLeaksFinder** is simple but tricky. So once it detects the leaked object, it use **FBRetainCycleDetector to detect the retain cycle.** Then it shows the **alter.**
- We can use `MLeaksFinder` to detect some seed objects. and provide **FBRetainCycleDetector**  with these candidate objects from which it will start detection.