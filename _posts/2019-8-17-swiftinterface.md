---
layout:     post
title:      编译一个向后兼容的 swift 二进制库
category:   []
tags:       [技术]
published:  True
date:       2019-8-17
---

Swift 5.0 实现了 ABI 稳定，使用 Swift 5.0 编译器编译出的二进制文件，可以被未来的版本所使用。但在使用 5.0.1 编译器引用一个 5.0 的二进制文件时，熟悉的错误仍然出现：不同编译器的产物不能相互引用。这是因为 swift 5.0 还没有达到 Module Stability。

Module stability 代表着描述模块 API 的信息格式稳定。这个信息会在编译时使用，它表明了这个库所有的类和函数都是什么，如同 c 语言的 header 文件一样。swift 把这个信息存在一个名为 `.swiftmodule` 的二进制文件中。由于这个文件在不同编译器间不兼容，导致了上面的问题。Module Stability 是 swift 5.1 的主要目标。它实现了一个文字版接口文件的[方案](https://forums.swift.org/t/plan-for-module-stability/14551)，使用一个名为 .swiftinterface 的文本文件替换二进制的 .swiftmodule 文件，内容类似于 xcode 中 swift 文件的 generated interface。

但当我们激动地拿 xcode 11 beta （包含 swift 5.1 beta）编译出一个 framework 的时候，并没有生成所期待的 .swiftinterface 文件，而是同之前一样。为此经过一番探索，找到了实现的方式，也是本篇文章的主要内容。

### 添加参数

> UPDATE:
> 在 xcode 11 中，只需要在 build setting 中设置 `Build Libraries for Distribution` 为 YES ，即可自动生成所有东西。参考 [WWDC Session: Binary Frameworks in Swift](https://developer.apple.com/videos/play/wwdc2019/416/)

在 swift 5.1 编译器中，有一个叫做 `-emit-parseable-module-interface-path` 选项。通过此选项，可以在编译的时候在指定目录生成 `.swiftinterface` 文件。如

```
swiftc demo.swift -emit-parseable-module-interface-path ./a.swiftinterface
```

但实际上，使用该参数，需要同时启用另一个参数 `-enable-library-evolution`。它开启了 Library Evolution 的特性：

> This is the feature that allows you to change a framework in a backwards-compatible way without having to recompile a client application.

Library Evolution 为了达到向后兼容，在生成代码方式上会有些不同，同时对代码有少许额外的[限制](https://forums.swift.org/t/update-on-module-stability-and-module-interface-files/23337/5)。关于 Library Evolution 我并没有做更充分的了解，因为它不是我想要的。但目前使用 parseable module interface 却绕不过它，是一个技术限制（[出处](https://forums.swift.org/t/update-on-module-stability-and-module-interface-files/23337)）。（我能找到的）官方最新一次关于它的描述是在 4 月份，看来在 swift 5.1 里应该不会改变了。

所以正确的参数为：

```
swiftc -enable-library-evolution -emit-parseable-module-interface-path ./a.swiftinterface demo.swift
```

### 生成不同架构的文件

在 xcode 中，只需要在 build settings 中的 `other swift flags` 中添加 

```
-enable-library-evolution -emit-parseable-module-interface-path $(BUILT_PRODUCTS_DIR)/$(PLATFORM_PREFERRED_ARCH).swiftinterface
```

即可在编译时生成接口文件。但这种方式只生成了一个架构的接口文件。

.swiftinterface 需要和架构有关吗？.swiftmodule 是与架构有关的，但参考 c 语言的头文件，文字性的接口文件可以做到与架构平台无关。但经过测试，swift 却不是这样，它选择了一种「算完」的方式。如

```
#if arch(x86_64)
	public class A {}
#endif
```

在生成 arm64 架构的接口文件时，就不会包含 A 。所以我们需要为每个支持的架构生成接口文件。

参考 xcode 编译 framework 时的 log，其中有一个参数 

```
-emit-module-path some/path/to/file.swiftmodule
``` 

来生成 swiftmodule，与我们的操作类似，这说明方向是对的。只需要让 xcode 针对不同的架构，设置不同的参数即可。在 xcode 的 build settings 里，以及命令行中 xcodebuild 的参数中，都没有可直接针对架构设置不同参数的方法，使用环境变量也不可以。但 .xccofing 文件却提供了这样的能力。根据 [The Unofficial Guide to xcconfig files](https://pewpewthespells.com/blog/xcconfig_guide.html#CondVarArch) 中的说明，使用如下方式即可：

```
OTHER_SWIFT_FLAGS[arch=armv7] = $(inherited) -enable-library-evolution -emit-parseable-module-interface-path $(BUILT_PRODUCTS_DIR)/armv7.swiftinterface
OTHER_SWIFT_FLAGS[arch=armv7s] = $(inherited) -enable-library-evolution -emit-parseable-module-interface-path $(BUILT_PRODUCTS_DIR)/armv7s.swiftinterface
OTHER_SWIFT_FLAGS[arch=arm64] = $(inherited) -enable-library-evolution -emit-parseable-module-interface-path $(BUILT_PRODUCTS_DIR)/arm64.swiftinterface
OTHER_SWIFT_FLAGS[arch=i386] = $(inherited) -enable-library-evolution -emit-parseable-module-interface-path $(BUILT_PRODUCTS_DIR)/i386.swiftinterface
OTHER_SWIFT_FLAGS[arch=x86_64] = $(inherited) -enable-library-evolution -emit-parseable-module-interface-path $(BUILT_PRODUCTS_DIR)/x86_64.swiftinterface
```

xcode 可以在 project 设置中，project-info 中使用 xcconfig 文件（需要先把 xcconfig 文件放到工程中）。使用 xcodebuild 可以通过 `-xcconfig` 来指定文件。（注意，new build system 并不支持分架构的配置，需要同 `-UseModernBuildSystem=NO` 关闭）

### 如何引用

我们目前只是生成了 .swiftinterface 文件，是手动指定的目录，它仍不在 framework 中。据官方[所述](https://forums.swift.org/t/update-on-module-stability-and-module-interface-files/23337)：

> By default, the interface files will be used whenever the corresponding "compiled module" (.swiftmodule file) is either absent or unusable. Other than that, there's nothing special about using them; just have them show up in your search path wherever you'd currently use a .swiftmodule. (That includes both the "flat" layout "MyLibrary.swiftinterface" and the "target-specific" layout "MyLibrary.swiftmodule/x86_64-apple-macos.swiftinterface".)

即编译器优先使用二进制的 .swiftmodule 文件，其次会使用 .swiftinterface 文件。这也是 Xcode 生成的 framework 仍包含 .swiftmodule 文件的原因。如果 遇到 .swiftinterface 文件，xcode 会解析该文本文件，并把结果缓存下来。这也意味着，.swiftmodule 会比 .swiftinterface 更快。

其次，.swiftinterface 只要放在 search path 能找到任意地方就可以了。但为了 framework 的完整性，我们可以把它门放在 `DEMO.framework/Modules/DEMO.swiftmodule/*` 中，以架构作为名称。而原本该目录下的文件可以删掉。如果你的 .swiftinterface 与架构无关，也可以只保留一份文件，并存放在 `DEMO.framework/Modules/DEMO.swiftinferface`

使用 .swiftinferface 所以需要 xcode 11 及之后版本，但在它只存在于编译时，所以运行的 iOS 版本并没有限制。

至此，我们已经生成了一个支持 swift 5.1 (xcode11) 且向后兼容的二进制 framework。

------

参考资料：

[Plan for module stability](https://forums.swift.org/t/plan-for-module-stability/14551)

[Update on Module Stability and Module Interface Files](https://forums.swift.org/t/update-on-module-stability-and-module-interface-files/23337)

[Pitch: Library Evolution for Stable ABIs](https://forums.swift.org/t/pitch-library-evolution-for-stable-abis/23026)

