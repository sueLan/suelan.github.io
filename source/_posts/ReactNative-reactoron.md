---
title: ReactNative开发-神器Reactoron
date: 2019-05-23 17:36:24
tags: 
 - React Native Dev
categories:
 - React Native

---
Reactoron能改善React Native开发体验。 
<!--more--> 
 
Reactoron这个开发工具，把我们输出的日志像twitter信息流一样保存起来，等我们需要的时候，可以回过头过滤找到日志，展开日志详情查看。提高ReactNative开发效率。 看日志会特别方便，体验也不错。

![image](https://github.com/infinitered/reactotron/raw/master/docs/images/quick-start-react-native/react-demo-native-reactotron.jpg)

项目地址： [GitHub - infinitered/reactotron: A desktop app for inspecting your React JS and React Native projects. macOS, Linux, and Windows.](https://github.com/infinitered/reactotron)

下载最新的release包，本地安装。

## 使用

### 1. 集成在React Native项目中

```sh
npm i --save-dev reactotron-react-native
// or
yarn add reactotron-react-native --dev
```

### 2. 配置文件

创建`ReactotronConfig.js`

基础用法：
```js
import Reactotron from 'reactotron-react-native'

Reactotron
  .configure() // controls connection & communication settings
  .useReactNative() // add all built-in react native plugins
  .connect() // let's connect!
```
高级配置: 

```js
import Reactotron from 'reactotron-react-native'

Reactotron
  .configure({
    name: "React Native Demo"
  })
  .useReactNative({     
      asyncStorage: {ignore: []},
      networking: {
          ignoreUrls: new RegExp(`http://127.0.0.1:19000/logs`)
      },
      editor: false,
      errors: {veto: (stackFrame) => false},
      overlay: false,})
  .connect();
```

#### asyncStorage 

参数类型：

```js
export interface AsyncStorageOptions {
    ignore?: string[];
}
```

`ignore`的值，传入key数组，`reactotron`不会展示这些key的存储数据

#### networking
参数类型： 
```js
export interface NetworkingOptions {
    ignoreContentTypes?: RegExp; // 如果response的Content-Type匹配上这个正则表达式，不展示response，
    ignoreUrls?: RegExp; //  要忽略的urls
}
```
例子：
```js networking({
  ignoreContentTypes: /^(image)\/.*$/i,
  ignoreUrls: /\/(logs|symbolicate)$/,
})
```


#### Error 
参数类型：
```js
export interface TrackGlobalErrorsOptions {
    veto?: (frame: any) => boolean;
}
```
`veto`函数，我们可以通过它指定我们不想看到的堆栈信息.这里的frame指stack frame.

比如： 如下忽略报错时所有react-native module里头的stack frame.
```js
Reactotron
  .configure()
  .use(trackGlobalErrors({
    veto: frame => frame.fileName.indexOf('/node_modules/react-native/') >= 0
   }))
  .connect()
```
### 3. improt

 在`App.js` 或者`index.js`文件头中引入：
 
```js
if(__DEV__) {
  import('./ReactotronConfig').then(() => console.log('Reactotron Configured'))
}
```

### 4. 打点

像用`console.log`一样，调用 `Reactotron.log`

ReactNative项目跑起来后，可以在Reactotron面板上看到： 

![9b1dbee4.png](https://github.com/infinitered/reactotron/raw/master/docs/images/quick-start-react-native/hello-1.jpg)

其他API： 
```js
Reactotron.log({ numbers: [1, 2, 3], boolean: false, nested: { here: 'we go' } })

Reactotron.warn('*glares*')
Reactotron.error('Now you\'ve done it.')
Reactotron.display({
  name: 'KNOCK KNOCK',
  preview: 'Who\'s there?',
  value: 'Orange.'
})

Reactotron.display({
  name: 'ORANGE',
  preview: 'Who?',
  value: 'Orange you glad you don\'t know me in real life?',
  important: true
})

```

## Redux的数据信息
[reactotron/plugin-redux.md at master · infinitered/reactotron · GitHub](https://github.com/infinitered/reactotron/blob/master/docs/plugin-redux.md)
![](https://github.com/infinitered/reactotron/raw/master/docs/images/redux/redux-keys-values.jpg)

[YouTube](https://www.youtube.com/watch?v=UiPo9A9k7xc)


