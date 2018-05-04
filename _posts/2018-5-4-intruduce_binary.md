---
layout:     post
title:      加快 CocoaPods 项目编译时间：Pod 预编译的傻瓜式解决方案
category:   []
tags:       [技术]
published:  True
date:       2018-5-4

---

## 问题

CocoaPods 以源代码的形式集成第三方库，这些库在开发的过程中经常会重新编译。当引用的库数量很多的时候，这个编译时间就会非无法忽略变得非常讨厌。毫无意义，并且降低了工作效率。

加快编译的一个方式是把第三方库全部预编译好，再集成到工程里。但这个过程非常繁琐，要手动编译库建私有仓库，版本更新时又要重做。虽然已有一些方案来解决这个事情，但在前两个问题上处理的都不够好，无法完全做到自动化。

#### 为什么不用 Carthage

Carthage 可以集成二进制库，为什么不用呢？有以下几个原因：

- Pod 是一个简单的易用的组织文件\管理依赖的形式。它可以使用私有 Pod 或 本地 Pod，Podspec 和 Podfile 的配置就等于 xcproject 繁琐的项目文件。
- Carthage 对于静态库的支持也不是很全面。动态库的问题太多了。
- Carthage 无法继承源代码，但 Pod 可以用个各种方式进行二进制化
- 并不是所有库都支持 Carthage

## 方案

你也许会感叹 CocoaPods 自己提供了这个功能该多好：pod install 之后得到的不是源代码，而是编译好的 framework。

为了实现这个功能，我写了 [cocoapods-binary](https://github.com/leavez/cocoapods-binary) 这个插件。它提供了一个 Podfile 的一个配置，通过简单的开关，在 pod insatll 的过程中进行 pod 库的预编译，生成 framework，并自动集成在 pod project 中。完全自动化，什么都不用做。

详细用法：

1. 安装插件 `gem install cocoapods-binary`
2. 在你的 Podfile 做上面加上 `plugin 'cocoapods-binary'` ，并在想要二进制化的 pod 后面加上 `:binary => true` ( 或者直接使用 `all_binary!` 让全部生效 )
3. pod install

这样，需要预编译的库就已经变成了编译好的 framework，在开发的时候就不用反复编译，加快编译时间。更详细用法请跳转 [GitHub](https://github.com/leavez/cocoapods-binary).

它完全自动化，只需添加简单配置。在库升级的时候也不需要额外的操作。对于反复 pod install 也做了同普通 pod 一样的缓存机制，不会反复编译。

#### 实现方式

因为想提供一个傻瓜式的解决方案，所以并不想改变用户的使用习惯，不使用额外的独立脚本。所以我们就必须改造 CocoaPods 的功能。

CocoaPods 本身提供了插件的机制。插件主要有两个用途，一是提供了如 `pod try` 这样的命令，二是为 install 添加修改功能。一个 CocoaPods 的插件就是一个 ruby gem （ruby 个包管理工具。我始终觉得 CocoaPods 就是仿造 Gems 写的）。plugin 的模板可通过 `pod plugins create` 创建（[详见](https://github.com/CocoaPods/cocoapods-plugins)）。

cocoapods-binary 的主要原理就是在 pre install 的 hook 中，独立执行另外一个 pod install。这个独立的 install 根据 podfile 中的配置过滤出要二进制化的 pod，生成 pod project，再使用 xcodebuild 编译出 framework。接着在正常的 install 过程中，通过运行时更改 CocoaPod 的代码，使得在集成的时候，对于指定的库使用的刚才编译好的 framework，而非源代码。这些 framework 作为该 pod 库的 vendored_framework 来实现引用。

思路很简单。这么做几乎无差别地支持了动态库和静态库，并且以 CocoaPods 的原生方式解决了预编译库的缓存与重复编译问题。对于主工程没有任何变化（前提是之前也使用了 `use_framework!`, use_framework 的更改不属于本文的范畴）。