---
layout:     post
title:      Swift 二进制库的兼容问题
category:   []
tags:       [技术]
published:  True
date:       2019-11-24
---

Swift 发布二进制库，可没有那么简单。

不同于 ObjC，编译出二进制后（.a / framework）即完成了大部分工作，Swift 还有很多需要额外注意的问题，如多编译器版本产物不兼容问题，二进制之间的兼容性问题。而且个人在尝试过程中，也发现了一些与此相关的编译器 bug，相关的经验，值得分享一下。

## 编译器版本兼容

多编译器版本产物不兼容问题，随着 Swift 5.1 实现的 Module Stability 在理论上已经解决，在[上一篇](2019/08/17/swiftinterface)中有详细介绍。但在实际测试中发现，部分库生成的 swift interface 文件，被 xcode import 的时候会提示错误。比如常用的 RxSwift Alamofire 等。这已经被官方确认为 [bug](https://forums.swift.org/t/generated-swiftinterface-has-wrong-content/28543)，至 xcode 11.2 问题仍然没有修复。这不禁令我对 swift 的质量产生了怀疑，明明是版本的主要特性。Swift 5.2 主要[目标](https://swift.org/blog/5-2-release-process/)是提升性能和稳定性，希望能真正解决问题。

## 二进制兼容

Swift 5.0 实现了 ABI 稳定，但这不代表各种情况的二进制可以混用。官方发布了一篇博客，详细说明了这个事情 [ABI Stability and More](https://swift.org/blog/abi-stability-and-more/) 。

### ABI 稳定的含义

ABI 稳定，其直接理解就是「类型的布局，函数的调用约定」等规范的确定。这就好比前后端工程师定好了接口数据的规范，之后两人互不联系独立开发也能跑得通，而不是依赖于一个人把前后端都写了。这里的工程师就是不同版本的编译器。

> app built with one version of the Swift compiler will be able to talk to a library built with another version -- *ABI Stability and More*

换种角度，ABI 稳定意味着，对于相同的代码，使用不同版本编译器编译出的产物是可互相替换的。

### 二进制兼容

但如果代码本身发生变化了呢？

在不重新编译依赖链下游的二进制库的情况下，直接替换为改变代码后的编译产物，是不一定兼容的。一个具体的例子：

```swift
# module A
public struct A {
    # public var v2 = "Hi, I'm new!"
    public var v1 = "The first instance variable"
}

# module B, 依赖 A
func main {
   print(A().v1)
}
```

这里打印的结果应该是 `The first instance variable`, 但使 `var v2` 那行生效后，不重新编译 B 的情况下，打印的结果将是 `v2` 的内容 `Hi, I'm new!`。如果 `v2` 与 `v1` 是不同的类型，将会出现 crash。

Swift 为了解决这个问题，引入了一个名为「library evolution」的模式 [SE-260](https://github.com/apple/swift-evolution/blob/master/proposals/0260-library-evolution.md)。开启后，对于结构体 fields 的更改（添加/删除/重排列）和对枚举添加新 case，将不会破坏 ABI 。对于上述例子，在 A 库开启该功能后，B 运行将会得到正常的值。其基本原理是，把对 struct 和 enum 的调用都会变成 indirect 的方式。类型的 size，fields 的布局，只能在运行时才被决定。

需要注意的是，这种兼容性问题不是 swift 语言特有的。对于 C 的结构体，同样会有此类的问题。开发者一般会通过对结构体添加保留字段来实现未来的可更改性。而 ObjC 的很少见此类问题，是因为它很少使用 c 的类型作为共有接口。而在类的实例变量上之所以没有出现这个问题，是因为实现了「non-fragile Objective-C runtime」的特性。

对于一个以二进制库的方式进行组件化的工程，其每个组件都需要开启 Library Evolution 的模式，因为每个组件都可能会被其他二进制组件所引用。没有开启的组件，一旦更改影响 ABI 的代码，将需要重新编译所有依赖它的组件。换种角度，如果一个库以二进制形式发布，且本身依赖了其他二进制库，除了它自己需要开启 Library Evolution 的模式，它所依赖的库也需要递归开启该模式。

### 开启 Library Evolution 的方式

Swift interface 与 Library Evolution 都由 xcode 的 BUILD_LIBRARY_FOR_DISTRIBUTION 选项开启，它是 xcode 11 中新引入的。虽然官方论坛中说，Swift interface 的实现只是暂时依赖 Library Evolution 的功能，但两个功能都是二进制发布所需要的，只有一个开关也没有问题。

### 开启  Library Evolution 的代价

开启 library evolution 会有一定的代价，下面分别从原理、限制和 bug 的角度来介绍。

从原理上来看，很多机制由编译时移到了运行时，会影响代码运行效率。此处我没有做详细测试。但 swift 标准库从 5.0 时就内部优先开启了这个功能（否则标准库嵌到系统里后 Apple 就无法改标准库的代码了），而我们对效率的要求应该低于标准库。同时对于开启该模式的代码，提供了 `@frozen` 关键字来人工标明那些不需要扩展能力的结构体，以恢复原始性能。

在实际使用中，开启该模式后，有一些代码写法会在编译时提示要求使用 iOS 13。这都是与 @objc 有关内容，我所遇到的有：一个库开启了该模式，在依赖链下游的库中，

- 对它的类实现 @objc 子类
- 对它的类使用 extension 实现 @objc 的方法（这在 UIKit 的 protocol 中经常会遇到）
- 对它的类实现子类，并添加 @objc 方法，且方法中使用父类的类型作为参数。

根据论坛中的[说法](https://forums.swift.org/t/xcframework-requires-to-target-ios-13-for-inter-framework-dependencies-with-objective-c-compatibility-tested-with-xcode-11-beta-7/28539)，这些功能是实现的局限。估计是需要在 swift 运行时中有一些对应的更改，所以只在 swift 5.1 （iOS 13）运行时里才可以运行。

除此之外，还会有一些编译时没有报错，但运行时 crash 或结果不正确的情况。可以从官方 jira 中查看 `Library Evolution` 标签获取具体[例子]([https://bugs.swift.org/browse/SR-11719?jql=labels%20%3D%20LibraryEvolution](https://bugs.swift.org/browse/SR-11719?jql=labels %3D LibraryEvolution))。

# 总结

虽然发布 swift 二进制库所需的语言特性都已实现，但其实现质量令人堪忧，暂时还无法正常使用。对于二进制组件化的工程，Library Evolution 是一个绕不过的功能，即使在 bug 都修复后，它对某些 `@objc` 写法要求 iOS 13 的问题，也会是一个难以处理的问题。



