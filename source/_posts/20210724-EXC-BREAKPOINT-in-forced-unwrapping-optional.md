---
title: EXC_BREAKPOINT when forced unwrapping optional in Swift
date: 2021-07-24 23:47:33
tags:
  - Swift
categories:
  - iOS
---

When unforced unwrapping optional in Swift, we will see the exception type in crash report is `Exception Type:  EXC_BREAKPOINT (SIGTRAP)` instead of `EXC_BAD_ACCESS` in Objective-C. That intrigued me. So I did some investigations on it. 

As Mike Ash mentioned in [the article about swift memory layout](https://www.mikeash.com/pyblog/friday-qa-2014-08-01-exploring-swift-memory-layout-part-ii.html)

> Swift optionals are represented just like Objective-C: a plain address when pointing to an object, and all zeroes for nil.

He gave an example like this, there are 4 variables

```swift
    let obj: NSObject? = NSObject()
    let nilobj: NSObject? = nil
    let explicitobj: NSObject! = NSObject()
    let explicitnilobj: NSObject! = nil
```
Basically, they lay out in memory like this: 
```
    806040c1957f0000
    0000000000000000
    70e440c1957f0000
    0000000000000000
```
Optional variable either contains a valid object address or all zero, so-call `nil` in Swift, which is different from `nil` in OC. While, `0000000000000000` in address space actually is an illegal region to access, iOS system raises an exception if you trying to do so. 

### PAGEZERO  in Mach-O Executable 

> __PAGEZERO: On 32-bit systems, this is a single page (4 KB) of memory, with all of its access permissions revoked. On 64-bit systems, this corresponds to the entire 32-bit address space — i.e. the first 4 GB. This is useful for trapping NULL pointer references (as NULL is really “0”), or integer-as-pointer references (as all values up to 4,095 in 32-bit, or 4 GB in 64-bit, fall within this page). Because access permissions — read, write, and execute — are all revoked, any attempt to dereference memory addresses that lie within this page will trigger a hardware page fault from the MMU, which in turn leads to a trap, which the kernel can trap. The kernel will convert the trap to a C++ exception or a POSIX signal for a bus error (SIGBUS).      ---- Mac OS and iOS internals 

When the executable is loaded into memory, the first `4G` memory region for this process is `non-executable, non-writable, non-readable`. Trying to access to this region of memory is illegal. 

### Example 

I wrote a demo to access  underlying value of an optional  by adding an exclamation point (`!`) to the end of the optional’s name. 

```
class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
        let image = UIImage(named: "aMissingIcon")!
        print(image.size)
    }
}
```

### Analyse Crash Report 

### Exception Type 

```
Exception Type:  EXC_BREAKPOINT (SIGTRAP)
Exception Codes: 0x0000000000000001, 0x000000010074596c
Termination Signal: Trace/BPT trap: 5
Termination Reason: Namespace SIGNAL, Code 0x5
Terminating Process: exc handler [12523]
Triggered by Thread:  0
```

#### EXC_BREAKPOINT (SIGTRAP)  

`EXC_BREAKPOINT (SIGTRAP)  ` is a trace trap interrupted the process. It gives an attached debugger, if any, a chance to interrupt the process at a specific point in its execution.  So when iOS system raises this exception since my process is trying to `access`  `0x00000000`, Xcode debugger can pause the process and show me this fatal error. 

![image-20210217115027905](./image-20210217115027905.png)

> On ARM processors, this appears as `EXC_BREAKPOINT (SIGTRAP). `On `x86_64` processors, this appears as `EXC_BAD_INSTRUCTION (SIGILL)`.

The Swift runtime uses trace traps for specific types of unrecoverable errors—see [Addressing Crashes from Swift Runtime Errors](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/addressing_crashes_from_swift_runtime_errors) for information on those errors. 

### Backtrace in Crashed Thread 

```
Thread 0 Crashed:
0   InvalideMemoryDemo            	0x000000010074596c 0x100740000 + 22892
1   InvalideMemoryDemo            	0x00000001007458d8 0x100740000 + 22744
2   InvalideMemoryDemo            	0x000000010074598c 0x100740000 + 22924
```

### Disassembling executable in Hopper 

Go to file offset `22892`

![image-20210217121336204](./image-20210217121336204.png)

 `BRK` is Breakpoint instruction to cause a Software Breakpoint Instruction exception.

Assembly code 

![image-20210217121006944](./image-20210217121006944.png)

Compiler already did optimizations for us.  It will use `cbz` command to compare value in register `x19` with zero. If it is zero, then go to label `loc_10000596c`, which causes a Software Breakpoint Instruction exception.

![image-20210217121336204](./image-20210217121336204.png)

From Thread State in in the crash report, we got `x19: 0x0000000000000000`. So here it goes to  `loc_10000596c` to raise `EXC_BREAKPOINT` exception.  That is what we see in the crash report. 

![image-20210217121128060](./image-20210217121128060.png)

seems, nothing fancy. 

- https://www.mikeash.com/pyblog/friday-qa-2014-08-01-exploring-swift-memory-layout-part-ii.html 
- https://www.objc.io/issues/6-build-tools/mach-o-executables/#sections
- https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/understanding_the_exception_types_in_a_crash_report#3582420
- https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/addressing_crashes_from_swift_runtime_errors
- https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID330