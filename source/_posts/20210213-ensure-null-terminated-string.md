---
title: Ensure Null-Terminated JS Bundle in React Native 
date: 2021-02-13 23:05:59
tags:
  - Dive into React Native
categories: 
  - React Native
---

## Description

There is an expensive copy operation for data buffer when loading and parsing JS Bundle in React Native.  In my case, I have a `4.8` MB JS bundle. Xcode instrument profiler shows it takes up 15.81 MB trying to copy a `NSData` object in  `ensureNullTerminated` function in `NSDataBigString` class.

![image](instrument-profile.png)

This is the brief process for `RCTBridge` loading JS bundle
![image](image-load.png)

In Objc, `NSData` is a static byte buffer that bridges to Data, which is a byte buffer in a contiguous region of memory. Here, `NSDataBigString` class holds `NSData` object constructed from JS bundle. 

![image](nsdata.png)

Inside react-native framework, it has to treat this data buffer in a contiguous region of memory as a [null-terminated byte string (NTBS)](https://en.cppreference.com/w/c/string/byte#:~:text=A%20null%2Dterminated%20byte%20string,(the%20terminating%20null%20character).&text=For%20example%2C%20the%20character%20array,%22cat%22%20in%20ASCII%20encoding.) . 

> A null-terminated byte string (NTBS) is a sequence of nonzero bytes followed by a byte with value zero (the terminating null character). Each byte in a byte string encodes one character of some character set. For example, the character array {'\x63','\x61','\x74','\0'} is an NTBS holding the string "cat" in ASCII encoding.

While, this data buffer doesn't end with `\0` character. Besides, In Objective-C, `NSData` is an immutable object. In order to add the `Null` byte, `\0` character, firstly, it has to copy the original immutable `NSData` object and construct a mutable object; then append `\0` to the data buffer.


## React Native version:
Run `react-native info` in your terminal and copy the results here.
react-native: v0.63.3
device: iPhone 5s 
OS version: 12.4.8 

## Steps To Reproduce
Provide a detailed list of steps that reproduce the issue.

1. open `RNTester` in `react-native` repo. `react-native/packages/rn-tester`
2. find `ensureNullTerminated`  and set a breakpoint there 
3. run the `RNTester` 

## Snack, code example, screenshot, or link to a repository:

![image](code-snippet.png)

## Solution 

- Add `\0` character ahead of time when building JS bundle. By doing this, you can avoid this copy operation during bundle loading in App launching phase. 

## Ref 

- https://github.com/facebook/react-native/issues/30145

