---
title: React Native: ExpoKit模式配置Push Notification 
date: 2019-06-10 18:44:33
categories:
 - React Native 
---

Expo sdk已经有一套Push Notifications的方案，接入方便, 但需要服务端配合接入对应sdk.而expo-server-sdk中默认使用Google Cloud Messaging 和 APNS； 因为Android国内无法用谷歌的服务，再加上没有服务端对应开发语言的sdk, 我们最后还是选了传统的方式。

## Expo Eject 
用expo创建的RN项目已经不再有ios/ & android/的工程配置文件夹。 所以我们要先用 `expo eject`命名生成iOS跟Android的工程配置文件, 。

`expo eject`会改变开发流程，比如，要运行一个项目，不止要`expo start`， 还要用Xcode或者AndroidStudio Run一个native项目。 所以真的要通读这个文档，仔细考虑要不要这么做[Ejecting to ExpoKit - Expo Documentation](https://docs.expo.io/versions/latest/expokit/eject/#__next)

1. 首先，我们要check一下app.json对不对https://docs.expo.io/versions/latest/distribution/building-standalone-apps/#2-configure-appjson

2. 接着，运行`expo eject`命令，会有如下提示, 我选第三个，ExpoKit。 ExpoKit是Objective-C 和Java库， 将native开发跟expo平台开发结合起来。

```
It's recommended to commit all your changes before proceeding,
so you can revert the changes made by this command if necessary.

? How would you like to eject your app?
  Read more: https://docs.expo.io/versions/latest/expokit/eject/
  React Native: I'd like a regular React Native project.
❯ ExpoKit: I'll create or log in with an Expo account to use React Native and the Expo SDK.
  Cancel: I'll continue with my current project structure.
```
你会发现多了两个目录
```
rn-project/
  - ios/
  - android/
```
## 运行Native工程

React Native packager方面，要像以前那样，运行`expo start`。 
接着配置native工程。以iOS为例: 

1. 进入ios文件夹, pod install (如果遇到undefined method'native_target', 请参考[Issues · expo/expo · GitHub](https://github.com/expo/expo/issues/2401d), 将cocoapods降到1.5.3版本。 
2. 打开xcwordspace文件，正常buid & run iOS项目

 app启动的时候，它会自动请求并加载JS bundle。如果我们修改了js文件，再保存，hot-update也生效的.
 
## 配置 PushNotificationIOS 
[官方教程PushNotificationIOS · React Native](https://facebook.github.io/react-native/docs/pushnotificationios) 很是麻烦，还容易报一堆这个那个找不到的错误。

```
RCTPushNotificationManager.h not found
React/RCTEventEmitter.h file not found
```
其实它本质上是xcodeproject要手动link RCTPushNotification lib，我们大可以直接修改Podfile文件，如下: 
- 最后一行，添加`RCTPushNotification`
- 再run一下`pod install`
- 重新`build`.

```
  pod 'React',
    :path => "../node_modules/react-native",
    :inhibit_warnings => true,
    :subspecs => [
      "Core",
      "ART",
      "RCTActionSheet",
      "RCTAnimation",
      "RCTCameraRoll",
      "RCTGeolocation",
      "RCTImage",
      "RCTNetwork",
      "RCTText",
      "RCTVibration",
      "RCTWebSocket",
      "DevSupport",
      "CxxBridge",
      "RCTPushNotification"
    ]
```