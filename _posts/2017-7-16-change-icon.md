---
layout:     post
title:      如何动态更换 iOS App 图标
category:   []
tags:       [小品]
published:  True
date:       2017-7-16

---

在 iOS 10.3 之前，App 图标是固定，只能通过发新版来更新。但现在有些 App 可以让用户自己选取图标，并及时生效。这是因为 10.3 里引入了一个新的 API，它允许在 App 运行的时候，通过代码为 app 更换 icon。


## 快速上手

虽然提供了更换的功能，但更换的 icon 是有限制的。它需要开发者提前预置在工程里，并做好相应配置。更改 icon 的时候，只能在有限的选项中进行选择。

具体方法很简单：

配置 `info.plist` 文件，添加对应的 alternate icons 的信息。

调用 `UIApplication.setAlternateIconName(_:completionHandler:)` 方法设置即可。其中这里的 name 就是上一步中设定的配置名称（如 anotherIcon）。
info.plist 的配置方式如下:

```xml
<key>CFBundleIcons</key>
<dict>
	<key>CFBundleAlternateIcons</key>
	<dict>
		<key>我是一个图标</key>
		<dict>
			<key>CFBundleIconFiles</key>
			<array>
				<string>文件名</string>
			</array>
			<key>UIPrerenderedIcon</key>
			<false/>
		</dict>
	</dict>
	<key>CFBundlePrimaryIcon</key>
	<dict>
		<key>CFBundleIconFiles</key>
		<array>
			<string>AppIcon60x60</string>
		</array>
	</dict>
</dict>
```

## 详细配置

虽然步骤很简单，但实际使用起来却有诸多模糊之处。官方一共有三处描述， 但都比较简略:

- [setAlternateIconName 方法文档](https://developer.apple.com/documentation/uikit/uiapplication/2806818-setalternateiconname)
- [About Info.plist Keys and Values](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009247)
- [Human Interface Guidelines: App Icon](https://developer.apple.com/ios/human-interface-guidelines/graphics/app-icon/)

下面介绍一些需要注意的地方:

#### 省略 CFBundlePrimaryIcon

虽然文档 1 中写着 「You must declare your app's primary and alternate icons using the CFBundleIcons key of your app's Info.plist file. 」，但经测试，CFBundlePrimaryIcon 可以省略掉。在工程配置 `App Icons and Launch Image` - `App Icons Source` 中使用 asset catalog（默认配置），删除 CFBundlePrimaryIcon 的配置也是没有问题的。

省略这个配置的好处是，避免处理 App icon 的尺寸。现在的工程中，大家一般都使用 asset catalog 进行 icon 的配置，而一个 icon 对应有很多尺寸的文件。省略 CFBundlePrimaryIcon 就可以沿用 Asset 中的配置。

如果想设置回默认 icon，在 `setAlternateIconName` 中传入 nil 即可。

#### 图片存放位置和引用名称

Alternate Icon 的图片不能放在 asset catalog 中，而应该直接放在工程里。

info.plist 的配置中，图片的文件名应该尽量不带 @2x/@3x 后缀扩展名，而让它自动选择 。



#### 提供多种尺寸的 icons

文档 3 中说明:

>Like your primary app icon, each alternate app icon is delivered as a collection of related images that vary in size. When the user chooses an alternate icon, the appropriate sizes of that icon replace your primary app icon on the Home screen, in Spotlight, and elsewhere in the system. To ensure that alternate icons appear consistently throughout the system—the user shouldn't see one version of your icon on the Home screen and a completely different version in Settings, for example—provide them in the same sizes you provide for your primary app icon (with the exception of the large App Store icon). See App Icon Sizes.


即需要为 AlternateIcon 提供多种尺寸的 image。具体的尺寸在引用文字里的链接中可以找到，一个 icon 最多可能需要十几种尺寸。

对于 info.plist 中的每个 icon 配置，CFBundleIconFiles 的值是一个数组，我们可以在其中填入这十几种规格的图片名称。经测试:

文件的命名没有强制的规则，可以随意取，数组中的文件名也不关心先后顺序。总之把对应的文件名填进去即可，它会自动选择合适分辨率的文件（比如在 setting 中显示 icon 时，它会找到提供的数组中分辨率为 29pt 的那个文件）。


## 演示

demo 在这里 [alternate-icon-demo](https://github.com/leavez/alternate-icon-demo)

![animation](/images/2017-7-16-change_icon/animation.gif)

（demo 中特意为 Setting App 设置了不同的图片，以证明图片选择的方）