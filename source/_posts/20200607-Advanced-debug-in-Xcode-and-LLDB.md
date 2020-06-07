---
title: Advanced debug in Xcode and LLDB
date: 2020-06-07 17:53:24
categories: 
  - iOS
tags:
  - Debug
  - LLDB
---

# Advanced debug in Xcode and LLDB 

https://developer.apple.com/videos/play/wwdc2018/412 

This session is awesome! 

## Configure behaviors to dedicate a tab for debugging

In `Prefernece` -> `Behaviors` ,  we can custom the debugging tab for ourselves. For example, choose `Show tab named` , fill in the name and choose `active window` . We will have a independent debug tab. 

![image-20200601222857985](412/image-20200601222857985.png)



<img src="media/15902152252875/image-20200607112605302.png" alt="image-20200607112605302" style="zoom:50%;" />

## LLDB expressions can modify program state

By using  auto-continuing breakpoints with debugger commands to inject code live, you can inject expression, change state or logic without compiling the project.  

**How to do it? **

- Click `Edit BreakPoint...`

- Click `add action` ; the default option is `Debugger Command`  we can write some `LLDB` command here

- Choose the `Automatically continue`, it will not pause when trigger this breakpoint. 

  

  <img src="media/15902152252875/image-20200607111532596.png" alt="image-20200607111532596" style="zoom:50%;" />



<img src="media/15902152252875/image-20200601224246210.png" alt="image-20200601224246210" style="zoom:50%;" />



## Symbolic Breakpoint 

`Symbolic Breakpoint` is one of my favorite tools to debug issues caused by others' framework.

In the  `Breakpoint navigator` , choose `Symbolic Breakpoint` .  

<img src="412/image-20200607113135963.png" style="zoom:50%;" />

Fill in any signature of the `Objective-c` method you want. 

<img src="412/image-20200607113105829.png" style="zoom:50%;" />

## “po $arg1” ($arg2, etc) in assembly frames to print function arguments

<img src="412/image-20200601224622911.png" style="zoom:50%;" />

In this case,  we use `Symbolic Breakpoint` to hit the `setText:` in `UILabe`, but we don't have the source code of `UIKit`. When the method is hit, we are in an assembly frame, we can use `$arg` to inspect the parameters.  `$arg1` is `self` pointer. `$arg2` is the  `selector` of the method. Others are  arguements to the method. [doc for objc_msgSend](https://developer.apple.com/documentation/objectivec/1456712-objc_msgsend#parameters)

<img src="media/15902152252875/image-20200601224723843.png" style="zoom: 50%;" />



**Noted**:  we have to do typecast for the `$arg2` , which is a `Slector` . 

<img src="media/15902152252875/image-20200601224918951.png" alt="image-20200601224918951" style="zoom:50%;" />

## Create dependent breakpoints using “breakpoint set --one-shot true”

If a Breakpoint is frequently hit, Like the above breakpoint `[UILabel setText:]`,  usually we will edit `conditions` for this breakpoint. And then, this `Breakppint` will be hit only when the expression for the condition is true.  However, when we don't a property to make the `condition expression`.   There is another way.  We can add action `one-shot symbolic breakpoint` in `a specific breakpoint`, where it will be hit only once in the proper time. 

>  The one shot breakpoint is a temporary breakpoint that only exists until it's triggered and then it's automatically deleted.



<img src="media/15902152252875/image-20200607115734569.png" alt="image-20200607115734569" style="zoom:67%;" />

- Set  a break point in line 96, where we might don't want a break, but we can `configure this breakpoint` to actually set the `symbolic breakpoint` in UI label set text . 
- choose `automatically continue` 

- Then add action , `breakpoint set --one-shot true --name "[UILabel setText:]"`.  This `one-shot symbolic breakpoint` is activated only after this breakpoint in line 96 is hit.  



How magical the dependent breakpoints are!



## Skip lines of code 

By `dragging` Instruction Pointer, the green handler in the following picture,  we can skip lines of code. It means, these lines of  code will not be executed. 

![image-20200607123358909](media/15902152252875/image-20200607123358909.png)



Or we can use `action` int the breakpoint to skip this line of code for us. 

<img src="media/15902152252875/image-20200607123639829.png" alt="image-20200607123639829" style="zoom:50%;" />



After doing that, we can add `expression` to add new expressions, like calling other functions. 

<img src="media/15902152252875/image-20200607123845152.png" alt="image-20200607123845152" style="zoom:50%;" />



## Pause when variables are modified by using watchpoints



- filter the variable we want using `fitler` 
- click `Watch attempts` 

<img src="412/image-20200607124534515.png" style="zoom:50%;" />

Then, we create a `Watchpoint` , which can be seen in the `breakpoint navigator`

<img src="412/image-20200607124622132.png" style="zoom:50%;" />

## Evaluate Obj-C code in Swift frames with `expression -l objc -O -- <expr>`

In swift frames, we can't use `pointers` or private func as we do in  obj-c frames. So here comes ``expression -l objc -O -- <expr>`.  This can help use them as we do in obj-c frames.  

```
// In swift
expression -l objc -O -- [`self.view`  recursiveDescription]

// In Obj-c 
[self.view  recursiveDescription]
```

By using the  `command alis` , we can short cut it. 

```
command alis poc expression -l -objc -O --
```

![image-20200607141219843](media/15902152252875/image-20200607141219843.png)



## Flush view changes to the screen using “expression CATransaction.flush()”

#### unsafeBitCast

We can use `unsafeBitCast`  in Swift.  To do the type cast, we have to provide the correct type. 

Print its property or change its property: 

![image-20200607142019406](media/15902152252875/image-20200607142019406.png)

Use `CATransaction.flush` to apply the view module changes to the screen's frame buffer. 

## Add custom LLDB commands using aliases and scripts. Alias examples:

- download nudge LLDB script provided by the Apple. https://developer.apple.com/videos/play/wwdc2018/412/ 
- Add to `~/.lldbinit`
- Add  custom alias in the lldbinit

![image-20200607143100209](media/15902152252875/image-20200607143100209.png)

```
command alias poc expression -l objc -O --
command alias flush expression -l objc -- (void)[CATransaction flush]
```

## Customizing Data Formatters

https://developer.apple.com/videos/play/wwdc2019/429