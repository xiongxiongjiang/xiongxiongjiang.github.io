---
title:      React Native使用TurboModule
date:       2024-07-17
tags:       ["rn"]
category:   React Native
math:       true
---

## 准备工作

安卓上，修改 `android/gradle.properties`，把newArchEnabled设置为true

```
- newArchEnabled=false
+ newArchEnabled=true
```

iOS上，进入ios目录，执行：

```
RCT_NEW_ARCH_ENABLED=1 && pod install
```

在根目录下，创建tm文件夹，最后你的目录应该是这个样子：

```
.
├── __tests__
├── android
├── ios
├── node_modules
├── src
└── tm
```



## 1. 定义 JavaScript 规范

NativeSampleModule.ts


```typescript
// 两点要求
// 1. 文件必须使用 Native<MODULE_NAME> 命名
// 2. 代码中必须要输出 TurboModuleRegistrySpec 对象

import type {TurboModule} from 'react-native/Libraries/TurboModule/RCTExport'
import {TurboModuleRegistry} from 'react-native'

// 此接口必须命名为 Spec
export interface Spec extends TurboModule {
  readonly reverseString: (input: string) => string
}

export default TurboModuleRegistry.get<Spec>(
  'NativeSampleModule',
) as Spec | null
```

## 2. 配置 Codegen 以生成脚手架

package.json

```json
{
  ...
  "author": "xurong xurong@xurong.tech (https://github.com/xiongxiongjiang)",
  "license": "MIT",
  "homepage": "https://github.com/xiongxiongjiang/RNPlayground#readme",
  "summary": "A react native playground",
  ...
  "codegenConfig": {
    "name": "AppSpecs",
    "type": "all",
    "jsSrcsDir": "tm",
    "android": {
      "javaPackageName": "tech.xurong.rn"
    }
  }
}
```

### iOS: 创建 `podspec` 文件

在tm目录下，创建AppTurboModules.podspec文件：

```ruby
require "json"

package = JSON.parse(File.read(File.join(__dir__, "../package.json")))

Pod::Spec.new do |s|
  s.name            = "AppTurboModules"
  s.version         = package["version"]
  s.summary         = package["description"]
  s.description     = package["description"]
  s.homepage        = package["homepage"]
  s.license         = package["license"]
  s.platforms       = { :ios => "12.4" }
  s.author          = package["author"]
  s.source          = { :git => package["repository"], :tag => "#{s.version}" }
  s.source_files    = "**/*.{h,cpp}"
  s.pod_target_xcconfig = {
    "CLANG_CXX_LANGUAGE_STANDARD" => "c++17"
  }
  install_modules_dependencies(s)
end
```

然后`ios/Podfile` 文件中，在 `use_react_native!(...)` 部分之后：

```ruby
if ENV['RCT_NEW_ARCH_ENABLED'] == '1'
  pod 'AppTurboModules', :path => "./../tm"
end
```

### 安卓：在tm目录下，创建CMakeLists.txt

```
cmake_minimum_required(VERSION 3.13)
set(CMAKE_VERBOSE_MAKEFILE on)

add_compile_options(
        -fexceptions
        -frtti
        -std=c++17)

file(GLOB tm_SRC CONFIGURE_DEPENDS *.cpp)
add_library(tm STATIC ${tm_SRC})

target_include_directories(tm PUBLIC .)
target_include_directories(react_codegen_AppSpecs PUBLIC .)

target_link_libraries(tm
        jsi
        react_nativemodule_core
        react_codegen_AppSpecs)
```

在`android/app/build.gradle`文件的末尾：

```
android {
   externalNativeBuild {
       cmake {
           path "src/main/jni/CMakeLists.txt"
       }
   }
}
```

## 3. 注册本地模块

### iOS

修改```AppDelegate.m``` 【经过测试，不修改也可以，可能是官方有什么改动但是没有更新文档】

关于```RCTTurboModuleManagerDelegate```，可以参考 [在 iOS 上启用 TurboModule](https://reactnative.cn/docs/next/new-architecture-app-modules-ios#1-provide-a-turbomodulemanager-delegate)

**注意：这里#import <NativeSampleModule.h> 是会报错的，需要在iOS目录下执行```RCT_NEW_ARCH_ENABLED=1 pod install```**才会在Development Pods

```objc
#import "AppDelegate.h"

#import <React/CoreModulesPlugins.h>
#import <ReactCommon/RCTTurboModuleManager.h>
#import <NativeSampleModule.h>

+ @interface AppDelegate () <RCTTurboModuleManagerDelegate> {}
+ @end

// ...

᠆ (Class)getModuleClassFromName:(const char *)name
{
  return RCTCoreModulesClassProvider(name);
}

+ #pragma mark RCTTurboModuleManagerDelegate

+ - (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:(const std::string &)name
+                                                       jsInvoker:(std::shared_ptr<facebook::react::CallInvoker>)jsInvoker
+ {
+   if (name == "NativeSampleModule") {
+     return std::make_shared<facebook::react::NativeSampleModule>(jsInvoker);
+   }
+   return nullptr;
+ }
```

### 安卓

1. 创建文件夹 `android/app/src/main/jni`
2. 从[node_modules/react-native/ReactAndroid/cmake-utils/default-app-setup](https://github.com/facebook/react-native/tree/main/packages/react-native/ReactAndroid/cmake-utils/default-app-setup)复制`CMakeLists.txt`和`Onload.cpp`到 `android/app/src/main/jni` 文件夹中。

具体代码：

CMakeLists.txt

```
# Copyright (c) Meta Platforms, Inc. and affiliates.
#
# This source code is licensed under the MIT license found in the
# LICENSE file in the root directory of this source tree.

# This CMake file is the default used by apps and is placed inside react-native
# to encapsulate it from user space (so you won't need to touch C++/Cmake code at all on Android).
#
# If you wish to customize it (because you want to manually link a C++ library or pass a custom
# compilation flag) you can:
#
# 1. Copy this CMake file inside the `android/app/src/main/jni` folder of your project
# 2. Copy the OnLoad.cpp (in this same folder) file inside the same folder as above.
# 3. Extend your `android/app/build.gradle` as follows
#
# android {
#    // Other config here...
#    externalNativeBuild {
#        cmake {
#            path "src/main/jni/CMakeLists.txt"
#        }
#    }
# }

cmake_minimum_required(VERSION 3.13)

# Define the library name here.
project(appmodules)

# This file includes all the necessary to let you build your application with the New Architecture.
include(${REACT_ANDROID_DIR}/cmake-utils/ReactNative-application.cmake)

# App needs to and and link against tm (TurboModules folder)
add_subdirectory(${REACT_ANDROID_DIR}/../../../tm/ tm_build)
target_link_libraries(${CMAKE_PROJECT_NAME} tm)
```

OnLoad.cpp：

```c++
/*
 * Copyright (c) Meta Platforms, Inc. and affiliates.
 *
 * This source code is licensed under the MIT license found in the
 * LICENSE file in the root directory of this source tree.
 */

// This C++ file is part of the default configuration used by apps and is placed
// inside react-native to encapsulate it from user space (so you won't need to
// touch C++/Cmake code at all on Android).
//
// If you wish to customize it (because you want to manually link a C++ library
// or pass a custom compilation flag) you can:
//
// 1. Copy this CMake file inside the `android/app/src/main/jni` folder of your
// project
// 2. Copy the OnLoad.cpp (in this same folder) file inside the same folder as
// above.
// 3. Extend your `android/app/build.gradle` as follows
//
// android {
//    // Other config here...
//    externalNativeBuild {
//        cmake {
//            path "src/main/jni/CMakeLists.txt"
//        }
//    }
// }

#include <DefaultComponentsRegistry.h>
#include <DefaultTurboModuleManagerDelegate.h>
#include <fbjni/fbjni.h>
#include <react/renderer/componentregistry/ComponentDescriptorProviderRegistry.h>
#include <rncli.h>
#include <NativeSampleModule.h>

namespace facebook {
namespace react {

void registerComponents(
    std::shared_ptr<ComponentDescriptorProviderRegistry const> registry) {
  // Custom Fabric Components go here. You can register custom
  // components coming from your App or from 3rd party libraries here.
  //
  // providerRegistry->add(concreteComponentDescriptorProvider<
  //        AocViewerComponentDescriptor>());

  // By default we just use the components autolinked by RN CLI
  rncli_registerProviders(registry);
}

std::shared_ptr<TurboModule> cxxModuleProvider(
    const std::string &name,
    const std::shared_ptr<CallInvoker> &jsInvoker) {
  if (name == "NativeSampleModule") {
    return std::make_shared<facebook::react::NativeSampleModule>(jsInvoker);
  }
  return nullptr;
}

std::shared_ptr<TurboModule> javaModuleProvider(
    const std::string &name,
    const JavaTurboModule::InitParams &params) {
  // Here you can provide your own module provider for TurboModules coming from
  // either your application or from external libraries. The approach to follow
  // is similar to the following (for a library called `samplelibrary`):
  //
  // auto module = samplelibrary_ModuleProvider(moduleName, params);
  // if (module != nullptr) {
  //    return module;
  // }
  // return rncore_ModuleProvider(moduleName, params);

  // By default we just use the module providers autolinked by RN CLI
  return rncli_ModuleProvider(name, params);
}

} // namespace react
} // namespace facebook

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *) {
  return facebook::jni::initialize(vm, [] {
    facebook::react::DefaultTurboModuleManagerDelegate::cxxModuleProvider =
        &facebook::react::cxxModuleProvider;
    facebook::react::DefaultTurboModuleManagerDelegate::javaModuleProvider =
        &facebook::react::javaModuleProvider;
    facebook::react::DefaultComponentsRegistry::
        registerComponentDescriptorsFromEntryPoint =
            &facebook::react::registerComponents;
  });
}
```



## 4. 编写原生代码来完成模块的实现

### 实现

在tm目录下，创建：

NativeSampleModule.h

```c++
#pragma once

#if __has_include(<React-Codegen/AppSpecsJSI.h>) // CocoaPod headers on Apple
#include <React-Codegen/AppSpecsJSI.h>
#elif __has_include("AppSpecsJSI.h") // CMake headers on Android
#include "AppSpecsJSI.h"
#endif
#include <memory>
#include <string>

namespace facebook::react {

class NativeSampleModule : public NativeSampleModuleCxxSpec<NativeSampleModule> {
 public:
  NativeSampleModule(std::shared_ptr<CallInvoker> jsInvoker);

  std::string reverseString(jsi::Runtime& rt, std::string input);
};

} // namespace facebook::react
```

NativeSampleModule.cpp

```
#include "NativeSampleModule.h"

namespace facebook::react {

NativeSampleModule::NativeSampleModule(std::shared_ptr<CallInvoker> jsInvoker)
    : NativeSampleModuleCxxSpec(std::move(jsInvoker)) {}

std::string NativeSampleModule::reverseString(jsi::Runtime& rt, std::string input) {
  return std::string(input.rbegin(), input.rend());
}

} // namespace facebook::react
```

### 运行codegen

ios：

```
RCT_NEW_ARCH_ENABLED=1 pod install
```

安卓：

```
yarn android
```

## 使用

```typescript
import React, {useState} from 'react'
import {Button, Text, View} from 'react-native'
import {styles} from './TurboModulesScreenStyles'
import Header from '~/components/Header'
import NativeSampleModule from '../../../tm/NativeSampleModule'

const TurboModulesScreen: React.FC = () => {

  const [result, setResult] = useState<string>()

  const handleAdd = async () => {
    const value = await NativeSampleModule?.reverseString('the quick brown fox jumps over the lazy dog')
    setResult(value)
  }

  return (
    <View style={styles.container}>
      <Header title='🚀 TurboModules 使用'></Header>
      <Text style={{marginLeft: 20, marginTop: 20}}>
        {result ?? ''}
      </Text>
      <Button
        title="Reverse String"
        onPress={handleAdd}
      />
    </View>
  )
}

export default TurboModulesScreen
```



etc：在最新的react native 模板中，babel.config.js 里面的配置为

```
presets: ['module:metro-react-native-babel-preset'],
```

这样运行后会报错：

Codegen didn't run for xxx. This will be an error in the future. Make sure you are using @react-native/babel-preset when building your JavaScript code.

需要修改配置为：

```
presets: ['module:@react-native/babel-preset'],
```

