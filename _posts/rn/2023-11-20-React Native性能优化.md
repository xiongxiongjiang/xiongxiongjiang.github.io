---
title:      React Native性能优化
date:       2023-11-20
tags:       ["rn"]
category:   React Native
math:       true
---



### 1. RAM Bundles 和内联引用优化

> 如果你有一个较为庞大的应用程序，你可能要考虑使用`RAM`(Random Access Modules，随机存取模块）格式的 bundle 和内联引用。这对于具有大量页面的应用程序是非常有用的，这些页面在应用程序的典型使用过程中可能不会被打开。通常对于启动后一段时间内不需要大量代码的应用程序来说是非常有用的。例如应用程序包含复杂的配置文件屏幕或较少使用的功能，但大多数会话只涉及访问应用程序的主屏幕更新。我们可以通过使用`RAM`格式来优化`bundle`的加载，并且内联引用这些功能和页面（当它们被实际使用时）。

#### 内联引用

内联引用(require 代替 import)可以延迟模块或文件的加载，直到实际需要该文件。一个基本的例子看起来像这样：

优化前

```tsx
import React, { Component } from 'react';
import { Text } from 'react-native';
// ... import some very expensive modules

// You may want to log at the file level to verify when this is happening
console.log('VeryExpensive component loaded');

export default class VeryExpensive extends Component {
  // lots and lots of code
  render() {
    return <Text>Very Expensive Component</Text>
  }
}
```

优化后

```tsx
import React, { Component } from 'react';
import { TouchableOpacity, View, Text } from 'react-native';

let VeryExpensive = null;

export default class Optimized extends Component {
  state = { needsExpensive: false };

  didPress = () => {
    if (VeryExpensive == null) {
      VeryExpensive = require('./VeryExpensive').default;
    }

    this.setState(() => ({
      needsExpensive: true,
    }));
  };

  render() {
    return (
      <View style={{ marginTop: 20 }}>
        <TouchableOpacity onPress={this.didPress}>
          <Text>Load</Text>
        </TouchableOpacity>
        {this.state.needsExpensive ? <VeryExpensive /> : null}
      </View>
    );
  }
}
```

即便不使用 RAM 格式，内联引用也会使启动时间减少，因为优化后的代码只有在第一次 require 时才会执行。

#### RAM Bundles 

在 iOS 上使用 RAM 格式将创建一个简单的索引文件，React Native 将根据此文件一次加载一个模块。在 Android 上，默认情况下它会为每个模块创建一组文件。你可以像 iOS 一样，强制 Android 只创建一个文件，但使用多个文件可以提高性能，并降低内存占用。

在 Xcode 中启用 RAM 格式，需要编辑 build phase 里的"Bundle React Native code and images"。在`../node_modules/react-native/scripts/react-native-xcode.sh`中添加 `export BUNDLE_COMMAND="ram-bundle"`

在 Android 上启用 RAM 格式，需要编辑 android/app/build.gradle 文件。在`apply from: "../../node_modules/react-native/react.gradle"`之前修改或添加`project.ext.react`

>***Note***: If you are using [Hermes JS Engine](https://github.com/facebook/hermes), you do not need RAM bundles. When loading the bytecode, `mmap` ensures that the entire file is not loaded.
>
>注意：如果您使用 Hermes JS Engine，则不需要 RAM 捆绑包。加载字节码时，mmap 确保不会加载整个文件。

#### 预加载

> 调用`require`会造成额外的开销。因为当遇到尚未加载的模块时，`require`需要通过bridge来发送消息。这主要会影响到启动速度，因为在应用程序加载初始模块时可能触发相当大量的请求调用。幸运的是，我们可以配置一部分模块进行预加载。为了做到这一点，你将需要实现某种形式的内联引用。

### 2. 使用Hermes

从 0.68 版本开始，React Native 提供了新架构，在React native 0.70+的版本，创建新的项目会默认开启Hermes。

#### 旧架构

旧的架构曾经通过使用一个叫做`桥（Bridge）`的组件将所有必须从 JS 层传递到本地层的数据序列化来工作。桥可以被想象成一条总线，生产者层为消费者层发送一些数据。消费者可以读取数据，将其反序列化并执行所需的操作。

桥有一些固有的限制：

- **它是异步的**：某个层将数据提交给桥，再异步地"等待"另一个层来处理它们，即使有时候这并不是真正必要的。
- **它是单线程的**：JS 是单线程的，因此发生在 JS 中的计算也必须在单线程上进行。
- **它带来了额外的开销**：每当一个层必须使用另一个层时，它就必须序列化一些数据。另一层则必须对其进行反序列化。这里选择的格式是 JSON，因为它的简单性和人的可读性，但尽管是轻量级的，它也是有开销的。

#### 新架构的改进

新架构放弃了"桥"的概念，转而采用另一种通信机制：`JavaScript 接口（JSI）`。JSI 是一个接口，允许 JavaScript 对象持有对 C++ 的引用，反之亦然。

一旦一个对象拥有另一个对象的引用，它就可以直接调用该对象的方法。例如一个 C++ 对象现在可以直接调用一个 JavaScript 对象在 JavaScript 环境中执行一个方法，反之亦然。

这个想法可以带来几个好处：

- **同步执行**：现在可以同步执行那些本来就不应该是异步的函数。
- **并发**：可以在 JavaScript 中调用在不同线程上执行的函数。
- **更低的开销**：新架构不需要再对数据进行序列化/反序列化，因此可以避免序列化的开销。
- **代码共享**：通过引入 C++，现在有可能抽象出所有与平台无关的代码，并在平台之间轻松共享它。
- **类型安全**：为了确保 JS 可以正确调用 C++ 对象的方法，反之亦然，因此增加了一层自动生成的代码。这些代码必须通过 Flow 或 TypeScript 类型化的 JS 规范来生成。

这些优势是[TurboModule](https://reactnative.cn/docs/the-new-architecture/pillars-turbomodules)系统的基础，也是进一步增强功能的跳板。例如，我们有可能开发出一种新的渲染器，它的速度更快，性能更强：[Fabric](https://reactnative.cn/architecture/fabric-renderer)及其[Fabric 组件](https://reactnative.cn/docs/the-new-architecture/pillars-fabric-components)。

#### [TurboModules 新的原生模块体系 ](https://reactnative.cn/docs/the-new-architecture/pillars-turbomodules)

Turbo Native Modules 与 Native Modules 相比，存在以下[优势](https://reactnative.cn/docs/the-new-architecture/why)：

- 各个平台的强类型接口声明是一致的；
- 您可以使用 C++ 编写模块或迁移其它平台的原生代码，以此避免在跨平台重复实现模块；
- 模块支持懒加载，可以加快 App 启动速度；
- 通过替换 Bridge 为 JSI（使用原生代码编写的 JavaScript 接口），提升 JavaScript 与原生代码的通讯效率。

#### [Fabric 渲染器和组件](https://reactnative.cn/docs/the-new-architecture/pillars-fabric-components)
#### [Codegen](https://reactnative.cn/docs/the-new-architecture/pillars-codegen)

它通过 JavaScript 的静态类型化，生成新架构所需的 C++ 模板。

### 3.视图拍平
这里不是一个用户主动优化的行为，但是值得一看：[视图拍平](https://reactnative.cn/architecture/view-flattening)

在编写代码的时候，我们也应该注意免布局嵌套太深.

### 4.移除console.log 语句

> 有个[babel 插件](https://babeljs.io/docs/plugins/transform-remove-console/)可以帮你移除所有的`console.*`调用。首先需要使用`yarn add --dev babel-plugin-transform-remove-console`来安装，然后在项目根目录下编辑（或者是新建）一个名为·.babelrc`的文件，在其中加入

```javascript
{
  "env": {
    "production": {
      "plugins": ["transform-remove-console"]
    }
  }
}
```

### 5.合理使用Hook

#### useMemo

`useMemo` 用于缓存计算结果，只有当依赖项发生变化时才重新计算。这在需要进行性能开销较大的计算时非常有用。

#### useCallback

`useCallback` 用于缓存函数实例，只有当依赖项发生变化时才重新创建。这在将回调函数传递给子组件时非常有用，防止子组件因函数引用变化而不必要地重新渲染。

### 6.避免使用匿名函数

在 React Native 中，渲染组件时避免使用匿名函数通常是一种很好的做法。原因是匿名函数可能会导致不必要的重新渲染，从而对应用的性能产生负面影响。

当你在组件的 render 方法中定义一个匿名函数时，每次组件渲染时它都会创建一个新的函数对象。这意味着即使该函数具有相同的行为和输出，它也会被视为新函数并导致重新渲染。

为了避免这个问题，请将函数写在 render 方法之外，并将它们作为 props 提供给组件。这样，函数对象只会生成一次，您的组件就不会过度重新渲染。

```tsx
import React from 'react';
import { TouchableOpacity, Text } from 'react-native';

const MyComponent = () => {
  const handleClick = () => {
    // 在这里处理点击事件
  }

  return (
    < TouchableOpacity onPress={handleClick} >
      <Text >Click me</ Text>
    </TouchableOpacity >
  );
}

export default MyComponent;
```

### 7.延迟加载大型组件

如果一个包含大量代码/依赖项的组件在最初渲染应用程序时不太可能被使用，您可以使用 React 的 [`lazy`](https://react.dev/reference/react/lazy) API 来推迟加载其代码，直到它首次呈现。通常，您应该考虑延迟加载应用程序中的屏幕级组件，这样添加新屏幕到您的应用程序就不会增加其启动时间。

使用lazy api：

```js
{
  "compilerOptions": {
    "module": "esnext",
  },
}
```

```tsx
const BigScreen = React.lazy(() => import('./screens/BigScreen'))
```





### 参考来源

- [The Ultimate Guide to Optimize React Native App Performance in 2024](https://www.bacancytechnology.com/blog/react-native-app-performance)
- [性能综述](https://reactnative.cn/docs/performance)
- [Optimizing JavaScript loading](https://reactnative.cn/docs/optimizing-javascript-loading)
- [使用新的 Hermes 引擎](https://reactnative.cn/docs/hermes)
- [为何要设计新架构](https://reactnative.cn/docs/the-new-architecture/why)

