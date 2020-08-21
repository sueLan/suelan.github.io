---
title: iOS main in Assembly
date: 2020-08-18 09:15:32
categories: 
  - iOS
tags:
  - Debug
---

### How to see Assemble code in Xcode 

`debug` -> `Product -> Action`->  `Assemble`  "main.m"

```objective-c
12 int main(int argc, char * argv[]) {
13    NSString * appDelegateClassName;
14    @autoreleasepool {
15        // Setup code that might create autoreleased objects goes here.
16        appDelegateClassName = NSStringFromClass([AppDelegate class]);
17    }
18    return UIApplicationMain(argc, argv, nil, appDelegateClassName);
19 }
```

It produces a file with 541 lines of code with lots of assembler directives for debugger.  

###  Assembler directives 

```assembly
	.section	__TEXT,__text,regular,pure_instructions
	.ios_version_min 11, 0	sdk_version 13, 6
	.file	1 "/Users/rongyan.zheng/Downloads/AssemblyMain" "AssemblyMain/main.m"
	.globl	_main                   ; -- Begin function main
	.p2align	2
```

These are assembler directives, not assembly code. The `.section` directive specifies into which section the following will go.  

Next, the `.globl` directive specifies that `_main` is an external symbol.`.p2align	2` aligns  the current location in the file to a specified boundary, here it is 2^2, 4 bytes.  



Then, here comes our `main`  assemble label. 

```assembly
_main:                                  ; @main
Lfunc_begin0:
	.loc	1 12 0                  ; AssemblyMain/main.m:12:0
	.cfi_startproc
; %bb.0:

```



> The `.cfi_startproc` directive is used at the beginning of most functions. CFI is short for Call Frame Information. A *frame* corresponds loosely to a function. When you use the debugger and *step in* or *step out*, you’re actually stepping in/out of call frames. In C code, functions have their own call frames, but other things can too. The `.cfi_startproc` directive gives the function an entry into `.eh_frame`, which contains unwind information – this is how exception can unwind the call frame stack. The directive will also emit architecture-dependent instructions for CFI. It’s matched by a corresponding `.cfi_endproc` further down in the output to mark the end of our `main()` function.  -- https://www.objc.io/issues/6-build-tools/mach-o-executables/ 



### SP and stack

```assembly
	sub	sp, sp, #48             ; =48
```

It sets up a call frame on the stack. in AArch64 the stack-pointer must be 128-bit aligned; here is 48*8-bit.

![image-20200807093341883](image-20200807093341883.png)

About stack and stack pointer, see more in. 

- [Using the Stack in AArch32 and AArch64](https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/using-the-stack-in-aarch32-and-aarch64)
- [Using the Stack in AArch64: Implementing Push and Pop](https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/using-the-stack-in-aarch64-implementing-push-and-pop)



### Addressing Mode

As a beginners, we may want to know  how the addresses caculated from the figures within the square brackets

Here, we may have know something about  `addressing modes`. There are several addressing modes that define how the address is formed according to [document s about ARMv8 instructions set](https://developer.arm.com/architectures/learn-the-architecture/armv8-a-instruction-set-architecture/loads-and-stores-addressing)  

- **Base register** - The simplest form of addressing is a single register. Base register is an X register that contains the full, or absolute, virtual address of the data being accessed, as you can see in this figure:

  <img src="image-20200807150953454.png" alt="image-20200807150953454"  />

- **Offset addressing modes** - An offset can be applied optionally to the base address, as you can see in this figure:

  <img src="image-20200807151043180.png" alt="image-20200807151043180"/>

  In the preceding figure, X1 contains the base address and `#12` is a byte offset from that address. This means that the accessed address is` X1+12`. The offset can be either a constant or another register. This type of addressing might be used for structs, for example. The compiler maintains a pointer to the base of struct using the offset to select different members.

- **Pre-index addressing modes** - In the instruction syntax, pre-index is shown by adding an exclamation mark `!` After the square brackets, as this figure shows: 

  ![image-20200807151621841](image-20200807151621841.png)

  Pre-indexed addressing is like offset addressing,` except that the base pointer is updated as a result of the instruction`. In the preceding figure, `X1` would have the value X1+12 after the instruction has completed.

- **Post-index addressing modes** - With post-index addressing, the value is loaded from the address in the base pointer, and then the pointer is updated, as this figure shows: 

  ![image-20200807151851210](image-20200807151851210.png)

  Post-index addressing is useful for popping off the stack. The instruction loads the value from the location pointed at by the stack pointer, and then moves the stack pointer on to the next full location in the stack.

So, here comes the next instruction in our `main` function: 

```assembly
stp	x29, x30, [sp, #32] 
```

`stp` pushes X29 and X30 onto the stack, which means this instruction will store values from `x29` and `x30` to memory where the address is `sp+32`. And in ARMv8, `X29` is for frame pointer, `X30` is used as the Link Register and can be referred to as LR.

![image-20200807103821841](image-20200807103821841.png)

### Store parameters on the Stack

```assembly
add	x29, sp, #32            ; =32
```

set the value of `x29` as `sp` + `#32`

```assembly
stur	wzr, [x29, #-4]
stur	w0, [x29, #-8]
str	x1, [sp, #16]
```

store value from `wzr` to `x29 + #-4`, which means write `x29 + #-4` using 0; 

>  The zero registers, ZXR and WZR, always read as 0 and ignore writes.

store value  from `w0` to `x29 + #-8;`

store value from `x1` to `sp + #16`.

![image-20200807103920889](image-20200807103920889.png)

Most A64 instructions operate on registers. The architecture provides 31 general purpose registers. Each register can be used as a 64-bit X register (X0..X30), or as a 32-bit W register (W0..W30).   W0 is the bottom 32 bits of X0.

<img src="image-20200807155505092.png" alt="image-20200807155505092" style="zoom:50%;" />

The choice of X or W determines the size of the operation. Using X registers will result in 64-bit calculations, and using W registers will result in 32-bit calculations. 

#### Registers for parameters 

The Arm architecture has some restrictions on how general purpose registers are used.  

`X0-X7` is the parameter/result registers.  x0-x7, are used to pass argument values into a subroutine and to return result values from a function.  The first argument is passed into `X0`, the second argument is in `X1`.  

In our case, `main` function takes two arguments, so `w0` and `X` are used. 

#### Registers for return 

Which register is used for return result is determined by the type of that result:

- If the type, T, of the result of a function is such that

  `void func(T arg)`,  the result is returned in the same registers used as passing arguments. For exmaple

![image-20200807175855759](image-20200807175855759.png)

- Otherwise,  the caller shall reserve a block of memory of sufficient size and alignment to hold the result. The address of the memory block shall be passed as an additional argument to the function in X8.`XR` |`X8`.

  

```assembly
Ltmp0:
	.loc	1 13 16 prologue_end    ; AssemblyMain/main.m:13:16
	mov	x8, #0                    ; copy #0 to x8
	str	x8, [sp, #8]              ; 
	.loc	1 14 22                 ; AssemblyMain/main.m:14:22
```

Then,  store value from `x8` to `sp + #8`.  

![image-20200807154518025](image-20200807154518025.png)

### Branch instructions and Function calls 

So let's see the next instruction. 

```assembly
bl	_objc_autoreleasePoolPush
```

> Ordinarily, a processor executes instructions in program order. This means that a processor executes instructions in the same order that they are set in memory. One way to change this order is to use branch instructions. Branch instructions change the program flow and are used for loops, decisions and function calls.
>
> The A64 instruction set also has some conditional branch instructions. These are instructions that change the way they execute, based on the results of previous instructions.       --  https://developer.arm.com/architectures/learn-the-architecture/armv8-a-instruction-set-architecture/program-flow

#### Unconditional branch instructions 

>  There are two types of unconditional branch instructions; `B` which means Branch and `BR` which means Branch with Register. 

The unconditional branch instruction `B <label`> performs a direct, PC-relative, branch to <label>.

In our case, the `labe` is  `_objc_autoreleasePoolPush`; `bl	_objc_autoreleasePoolPush` means we will jump to `_objc_autoreleasePoolPush` routine. 

#### Conditional branch instructions 

The conditional branch instruction B.<cond> <label> is the conditional version of the B instruction. The branch is only taken if the condition <cond> is true. The range is limited to +/- 1MB.

#### [Labels for PC-relative addresses](https://www.keil.com/support/man/docs/armasm/armasm_dom1359731174549.htm)

> A label can represent the PC value plus or minus the offset from the PC to the label. Use these labels as targets for branch instructions, or to access small items of data embedded in code sections.



```assembly
	adrp	x8, _OBJC_SELECTOR_REFERENCES_@PAGE
	add	x8, x8, _OBJC_SELECTOR_REFERENCES_@PAGEOFF        ;@selector(class)
	adrp	x9, _OBJC_CLASSLIST_REFERENCES_$_@PAGE 
	add	x9, x9, _OBJC_CLASSLIST_REFERENCES_$_@PAGEOFF     ; objc_cls_ref_AppDelegate
```

- `adrp` is to Address of 4KB page at a PC-relative offset

  I drag the final executable into `hopper`,   the address for `_OBJC_SELECTOR_REFERENCES_@PAGE` is `#0x100009000   `

  ```assembly
  000000010000621c         adrp       x8, #0x100009000                            ; 0x1000093e0@PAGE
  0000000100006220         add        x8, x8, #0x3e0                              ; 0x1000093e0@PAGEOFF, &@selector(class)
  0000000100006224         adrp       x9, #0x100009000                            ; 0x1000093f0@PAGE
  0000000100006228         add        x9, x9, #0x3f0                              ; 0x1000093f0@PAGEOFF, objc_cls_ref_AppDelegate
  000000010000622c         ldr        x9, [x9]                                    ; objc_cls_ref_AppDelegate,_OBJC_CLASS_$_AppDelegate
  0000000100006230         ldr        x1, [x8]                                    ; "class",@selector(class)
  ```

  

```assembly
	ldr	x9, [x9]                ; load data into x9 from the value in x9.
	ldr	x1, [x8]                ; load data into x1 from the value in x8, which is the address of @selector(class)
	str	x0, [sp]                ; 8-byte Folded Spill
	mov	x0, x9                  ; move the addreess in x9 into x0, which is the address of objc_cls_ref_AppDelegate
	bl	_objc_msgSend           ; call msgSend, [AppDelegate class]
	bl	_NSStringFromClass      ; call NSStringFromClass 
```

#### The return value and auto release 

```assembly
  mov	x29, x29	; marker for objc_retainAutoreleaseReturnValue
	bl	_objc_retainAutoreleasedReturnValue
  ldr	x8, [sp, #8]            ; load data into x8 from memory [sp, #8] 
	str	x0, [sp, #8]            ; store data from x0 into [sp, #8], which is the result of _NSStringFromClass
	mov	x0, x8                  ; move data from x8 to x0
	bl	_objc_release
	ldr	x0, [sp]                ; 8-byte Folded Reload
	bl	_objc_autoreleasePoolPop
```



`str	x0, [sp, #8] ` means store data from x0 into [sp, #8]. As we mentioned before, for functions with this type `NSString *NSStringFromClass(Class aClass)`, the `x0` is used to store the return result. 

### Pass parameters

```assembly
	ldur	w0, [x29, #-8]      ; load data into w0 from [x29, #-8]; which is int argc
	ldr	x1, [sp, #16]         ; load data into x1 from [sp, #16], where argv is stroed
	ldr	x3, [sp, #8]          ; laod data into x3 from [sp, #8], which is the result of  the result of _NSStringFromClass
```

```assembly
	mov	x8, #0
	mov	x2, x8                ; nil
```



```assembly
	bl	_UIApplicationMain    ; call _UIApplicationMain
	stur	w0, [x29, #-4]      
	add	x8, sp, #8              ; =8
	mov	x0, x8
	mov	x8, #0
	mov	x1, x8
	bl	_objc_storeStrong
	ldur	w0, [x29, #-4]
	ldp	x29, x30, [sp, #32]     ; 16-byte Folded Reload, reset 
	add	sp, sp, #48             ; =48, reset 
	ret
```



`ldp	x29, x30, [sp, #32]`  this is for reset the `fp` and linker regiseter 

`add	sp, sp, #48 ` means, the call frame for this function is gone. 



![image-20200810095955537](image-20200810095955537.png)