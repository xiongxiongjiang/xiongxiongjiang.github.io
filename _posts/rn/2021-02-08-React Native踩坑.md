---
title:      React Native踩坑
date:       2021-02-08
tags:       ["ios", "android", "rn"]
category:   React Native
math:       true
---

### 安卓报错：Unable to load script.Make sure you are either running a Metro server or that your bundle 'index.android.bundle' is packaged correctly for release
**场景:**   
触发这个报错的场景很多，比如已经用xcode拉起过Metro server，或者在电脑运行过别的安卓，等等
**解决方法:**   
- 假如已经运行了iOS，先找到终端，杀死当前的Metro server进程，并且执行:
  ```sh
  react-native start
  ```
- 假如跑过别的安卓，在终端执行:
  ```sh 
  adb reverse tcp:8081 tcp:8081
  ```
  
  再点击RELOAD就可以了

---
### 安卓上TouchableOpacity点击穿透
**场景:**   
假设有两个View，ViewA 和 ViewB。ViewA作为视图的容器，ViewB作为悬浮层使用absolute布局。   
**问题:**   
在**安卓**中会出现bug。如果ViewB超出了ViewA的范围，点击ViewB会不生效并且会穿透到点击ViewB下面的视图。   
**解决方法:**   
增加一层ViewC作为ViewA和ViewB的父容器，并且ViewA和ViewB中的TouchableOpacity，都使用[react-native-gesture-handler](https://github.com/software-mansion/react-native-gesture-handler)

```ts
import {TouchableOpacity} from 'react-native-gesture-handler'
```
---
### iOS下使用RCT_REMAP_METHOD，RCTPromiseResolveBlock中断
**场景:**   
在RN层拉起内购，iOS原生层处理内购并回调，但是由于内购弹窗弹出让通信中断导致内购失败   
**解决方法:**   
不使用```resolve()```进行回调，RN对原生层使用```RCT_REMAP_METHOD```，原生层使用```RCTEventEmitter```发通知给RN层进行刷新UI等等

---
### iOS报错Undefined symbol: _swift_getOpaqueTypeConformance
**场景:**  
RN上创建swift target，会报Undefined symbol: _swift_getOpaqueTypeConformance   
**解决方法:**   
在对应的target上，找到LIBRARY_SEARCH_PATHS，把"$(TOOLCHAIN_DIR)/usr/lib/swift-5.0/$(PLATFORM_NAME)"删除就行。

---

### xcode报main.jsbundle does not exist。
**解决方法:**
在package.json中加入这段脚本：
```sh 
 "build:ios": "react-native bundle --entry-file='index.js' --bundle-output='./ios/main.jsbundle' --dev=false --platform='ios'"
```

### RN卡在启动页
**原因**
- 切换了分支，构建缓存，sdk等冲突
- 手机和电脑不在同一个WiFi导致   
- 网络不好

**解决方法:**   
- 删除app并重新安装

- 切换手机wifi到电脑同一个wifi

- 检查手机是否开启了vpn

- 忽略wifi重连

### dev环境下，无法热更新，修改代码UI不刷新
**解决方法**
- 摇一摇手机，点击reload



### 启动时报错 Failed to construct transformer:  Error: error:0308010C:digital envelope routines::unsupported

```
Failed to construct transformer:  Error: error:0308010C:digital envelope routines::unsupported
    at new Hash (node:internal/crypto/hash:71:19)
    at Object.createHash (node:crypto:133:10)
		...
    at process.processTicksAndRejections (node:internal/process/task_queues:95:5) {
  opensslErrorStack: [ 'error:03000086:digital envelope routines::initialization error' ],
  library: 'digital envelope routines',
  reason: 'unsupported',
  code: 'ERR_OSSL_EVP_UNSUPPORTED'
}
error Cannot read properties of undefined (reading 'transformFile').
TypeError: Cannot read properties of undefined (reading 'transformFile')
```

原因： 当前电脑的node版本太高，而react-native的版本太低的原因。

解决方法：

- 方法一： 升级到最新版的react-native

- 方法二：安装一个低版本的node：

    ```shell
    n 14.18.1
    ```



### 安卓手机连接电脑后，android studio上不显示

重启电脑
