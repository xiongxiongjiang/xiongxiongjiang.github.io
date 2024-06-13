---
title: AutoreleasePool
date: 2020-02-09
tags: ["ios"]
category: iOS
math:       true
---

## 定义

> AutoreleasePool（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。在正常情况下，创建的变量会在超出其作用域的时候release，但是如果将变量加入AutoreleasePool，那么release将延迟执行。



## AutoreleasePool创建和释放

- App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。
- 第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。
- 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。
- 在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

也就是说AutoreleasePool创建是在一个RunLoop事件开始之前(push)，AutoreleasePool释放是在一个RunLoop事件即将结束之前(pop)。 AutoreleasePool里的Autorelease对象的加入是在RunLoop事件中，AutoreleasePool里的Autorelease对象的释放是在AutoreleasePool释放时。



## AutoreleasePool实现

把以下代码通过clang转换：

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        NSLog(@"Hello, World!");
    }
    return 0;
}
```

源码：

```c++
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_3c_txgk_n8s2_q_g55r9vkxhfbr0000gn_T_main_df2595_mi_0);
    }
    return 0;
}
```

可以看出，@autoreleasepool{}是通过声明一个__AtAutoreleasePool类型的对象实现的。

__AtAutoreleasePool：

```c++
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

根据构造函数和析构函数的特点（自动局部变量的构造函数是在程序执行到声明这个对象的位置时调用的，而对应的析构函数是在程序执行到离开这个对象的作用域时调用）。所以可以把main中代码转换成：

```c++
int main(int argc, const char * argv[]) {

    /* @autoreleasepool */ {
        void *atautoreleasepoolobj = objc_autoreleasePoolPush();
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_3c_txgk_n8s2_q_g55r9vkxhfbr0000gn_T_main_df2595_mi_0);
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

一个autorelease pool的执行过程可以这么理解：push() -> your code -> [obj autorelease] -> pop()

再来看看push和pop的实现：

```c++
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

可以看到，是通过AutoreleasePoolPage实现的。

## AutoreleasePoolPage

AutoreleasePoolPage类：

```c++
class AutoreleasePoolPage {
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

1. `id *next` 指向了下一个能存放autorelease对象地址的区域
2. `parent` 父节点 指向前一个page
3. `child` 子节点 指向下一个page 
4. `POOL_BOUNDARY`是一个边界对象 nil，之前的源代码变量名是 `POOL_SENTINEL`哨兵对象,用来区别每个page即每个 `AutoreleasePoolPage`边界



AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的形式组合而成的栈结构（分别对应结构中的parent指针和child指针）。

parent和child就是用来构造双向链表的指针。parent指向前一个page, child指向下一个page。 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表，后来的autorelease对象在新的page加入。

AutoreleasePoolPage的类方法push()：

```c++
static inline void *push() {
   return autoreleaseFast(POOL_BOUNDARY);
}
```

autoreleaseFast：

```c++
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
```

hotPage为当前使用的page，根据代码，有三种情况：

- 有hotPage且hotPage没满，往hotPage栈中加入obj
- 有hotPage但满了，创建一个新的page并将obj添加到新的page中
- 没有hotPage，创建一个新的page并将obj添加到新的page中

## 需要手动使用AutoreleasePool的场景

- 如果您编写的程序不是基于UI框架，例如命令行工具。
- 如果您编写一个创建许多临时对象的循环。您可以在循环内使用autorelease池块在下一次迭代之前释放这些对象。在循环中使用自动释放池块有助于减少应用程序的最大内存占用。
- 如果生成一个辅助线程。一旦线程开始执行，您必须创建自己的自动释放池块;否则，应用程序将泄漏对象。(有关详细信息，请参见自动释放池块和线程。)

## 小结

- 自动释放池是一个个 AutoreleasePoolPage 组成的一个page是4096字节大小,每个 AutoreleasePoolPage 以双向链表连接起来形成一个自动释放池
- 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
- pop 时是传入边界对象,然后对page 中的对象发送release 的消息



[Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI)

[AutoreleasePool详解和runloop的关系](https://www.jianshu.com/p/d3d3196f5edb)