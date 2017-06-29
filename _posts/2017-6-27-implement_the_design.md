---

layout:     post
title:      文本对齐，如何像素般精确还原设计稿
category:   []
tags:       [技术]
published:  True
date:       2017-6-27

---



## 问题

在工程师实现设计稿的时候，文本框的对齐是一个经常遇到且棘手的问题。明明已经遵照设计师的标注实现，但结果却与设计稿有很大差异。

> 「label 为什么有这么大的上下边距呢？」
>
> 「行距是 1.2 倍但是效果完全不一样！」

这时候只能靠手工一点一点试，而且由于 app 开发不能像 web 一样及时生效，很浪费时间，且不够精确。

我们的目标是: 只需按照标注 coding，即可像素般准确实现设计稿中的文字对齐与行间距样式。为了解决这个问题，我们需要先明确一些概念。

## 文本的度量

文字的排版不只是把方块字依次排列起来即可。对于拉丁字母，f 与 g 以什么方式上下对齐？为了准确的描述，字体中有以下几个概念:

![Metric of Font](/images/2017-6-text-design/metric.PNG)

我们只看纵向:

- baseline: 相当于坐标原点。大部分的拉丁字母底部与此对齐，汉字的中下部与此对齐（这是设定）
- ascent, descent: 相当于字体可绘制区域的上下最大值。根据自己的观察，ascent 并不一定是最高字符的高度（比如上图的 f），在大部分字体中，ascent 会比最高的字符还要高一些，上面会有个空间。 desecnt 同理。（descent 为负值）
- leading: 即行间距。但这个行间距与平时所说的行间距并不是一个东西。在文本编辑器中，选择不同字体的时候，视觉上的每行的距离并不是一样的。有可能就是 leading 不同。这个值可能为 0。（对于 iOS 上的 SF 系列字体，它的值就是 0 ）
- line height: 即行高。它的值定义为 ascent + descent + leading。（descent 为负值，所以准确的写应该取绝对值）。这也是我们最关心的一个值。

这些值都是字体的属性，是字体的设计者制定的，不可变，不同的字体会不一样。

平时用来表示字体大小的「字号」并不对应上图中的任何值，也就没有一个直接的几何意义。字号准确的说法是 point size。对于一个 point size 是 15 的 SFUI 字体，它的 line height 为 17.900390625,  约为 point size 的 1.2 倍。所以对于这个字体的一行文字，它的行高为 17.900390625。如果硬要显示在 15.0 高度的矩形内，g 和 f 应该会显示不全。

### 行间距

line height 所代表的高度只是一行文字的高度。可以把一行文字看做以 line height 为高的矩形，多行文字就是这些矩形纵向排列。矩形的间距就是通常我们说的行间距:

![多行文本](/images/2017-6-text-design/linespacing.png)

而通常所说的「行间距」「行高」「line height mutiple」 这样的词语，描述的就是这个间距的大小。

- 「行间距」: 直接对应间距的值
- 「行高」: line height + 间距。可以认为是，除了首行与尾行，每行实际所占的高度
- line height mutiple: 即是「 x 倍行高」中的数值。`line_height_mutiple = 「行高」/ line_height`

## 不同平台的实现效果

### iOS

使用 autolayout 的一行 UIlabel 的高度即为所使用字体的 line height。但 autolayout 中，view 的 frame 的小数点精度会对齐到像素精度。所以 15 号字体的 label 高度为 18.0 point 。

对于多行文字的行间距，可以通过 attributedString 中的 paragraph style 来控制。paragraph style 可以设定如下值:

- lineHeightMultiple: 同上面所说的。
- minimumLineHeight/maximumLineHeight: 即「行高」

这两个值都会改变行高，只是写法不同而已。但使用它们控制行间距有一个问题，如果行高大于字体的 line height，那么多余的空间将会放在这行的上面: baseline 所在的位置是矩形底边 + ( leading + descent ) 的位置。一个常见的情况，圆形的 avatar 与右侧的 label 顶端对齐，如果使用 lineHeightMultiple，那么为了达到视觉上的对齐，avatar 与 label 的 frame.y 就会不一样。不是很理想。（在使用 insets 或 background color 的时候就会很麻烦）

- lineSpacing: 即行间距。

使用 lineSpacing 只会在每行之间添加间距。在首行与尾行外侧并没有额外的空白（当然，line height 里所带的空白仍然存在）。比较符合我们行间距的设定，不存在上面提到的问题。但不同 point size 为了有同样的效果，需要设定不同的 lineSpacing，不如 lineHeightMultiple 使用方便。

### Sketch App

Sketch 是常见的 UI 设计工具。sketch 中的一个单行文本框的高度同 ios 一样，即精确到整数的 line height。（以前并不是，至于从什么版本开始我也不清楚。）（但文本框的高度可以设为小于 line height，而且并不会截断文字显示。此时文字会居中对齐，但在 iOS 中如此操作，文字会顶端对齐，截断下面。）

sketch 对于多行文本设定非常简单，只有一个 `lineHeight` 值，对应于上文中的「行高」。设定  `lineHeight` 的效果是文字的每一行都变高了，原有的文字在每行内居中对齐。这个效果与 iOS 中使用 minimumLineHeight 是不一样的，后者是向下对齐。而与 iOS 中使用 lineSpacing 不同，首行和尾行外侧也会多出同样的空白边距。

## 解决方案

可以看出，不同平台对于这些参数的实现形式是不一样的，这就造成了开头所说的问题。为了解决这个问题，需要统一两个平台的效果:

我们这样约定显示规则: （这种约定不是唯一的选择，但这样设定更方便使用和理解）

- 使用相同的字体: SF UI（或 SF Pro）
- 单行文本: 文本框高度等于 font 的 line height
- 多行文本: 只在行与行之间加入 spacing，同使用 iOS lineSpacing 的效果。并使用 multiple 来描述 spacing 的大小。

通过以下的方式可以实现上述效果:

- 字体:
  - iOS : 使用系统默认字体 SF UI，可以使用 systemFontOfSize 方法获得（systemFontOfSize:weight: 方法也可以，字重没有影响）。注意不能直接使用 PingFang SC 字体，它与 SF UI 的 line height 不同。
  - sketch: 使用 SF UI Text 或 SF UI Display，两者在横向上都会有误差，原因后面会说明。同样不能直接使用 PingFang SC，虽然在使用 SF UI 的中文会 fall back 到 PingFang SC, 但两者的 line height 不同，PingFang SC 会更大一点。
- 单行文本:
  - iOS: UILabel，使用 autolayout（view 高度精确到像素）。或者其他等同效果的 view，内部使用 textKit 的都有一样的效果1。UILabel 在（只有一行，有中文，paragraph style 中的 line spacing 不为零）的情况下有个小 bug，view 的高度会比 line height 大。解决办法是去掉 lineSpacing，或者设定 view 高度等于 font.lineHeight。
  - sketch: 对于纯英文的情况下，使用 text layer 的默认高度即可，不要手动更改行高。如果有中文，view 高度会比 SF 字体的 line height 高。需要使用插件设定文本的 line height multiple 为 1（后面介绍）。
- 多行文本:
  - iOS: 行间距的设置使用 lineSpacing 属性实现。设定 lineSpacing = font.lineHeight * (multiple - 1.0 ) 。其中存在取整的问题需要注意2。
  - sketch: 使用插件直接设置 line height multiple。比如想要 1.2 倍行高，直接输入 1.2。

因为 sketch 无法直接达到想要的效果，为此写了一个插件 [sketch-engineer-friendly-text](https://github.com/leavez/sketch-engineer-friendly-text)。它可以通过输入的 multiple 值和字体的本身的 line height 自动计算行高。并把文本上下多余的「行间距」切掉。

这样，我们统一了两个平台的显示效果，工程师在开发的时候只需直接遵照设计图上的参数，即可快速准确实现文字的显示样式。

## 实际效果

通过这样的方式，实现了一个 [demo](https://github.com/leavez/Sketch-iOS-text-solution) ，结果如下:

![结果对比（原图很大，可以看到细节）](/images/2017-6-text-design/result.png)

![细节放大](/images/2017-6-text-design/result_detail.png)

其中黑色和绿色部分是 ios 的结果， 白色和紫色的 sketch 反色的结果。左侧是一个文本框的全部，两者文字对齐，文本框大小一样，在纵向完全一致。右侧的放大版，g 和 r 虽然在纵向上有差异，但 sketch 中的 g 比 iOS 中的上下都要小，可能是字体的原因。baseline 仍然是完全一致的。

![中文的情况](/images/2017-6-text-design/compareChineseCrop.png)

中文的对比结果，在此看[原图](/images/2017-6-text-design/compareChinese.png)。在 30px 字体上，完全一致。在 60px 字体上，sketch 会偏下 1px，暂时没有解决，但也可以接受。 

至于横向上的差异，原因[在此](https://developer.apple.com/ios/human-interface-guidelines/visual-design/typography#font-usage-and-tracking)，已经超出了本文的范围。

至此，我们完成了目标。

# Appendix I - Sketch 插件的使用方法

 [sketch-engineer-friendly-text](https://github.com/leavez/sketch-engineer-friendly-text) 如何使用:

![](/images/2017-6-text-design/plugin.png)

选中所要更改的 text layer，选择菜单中的 「set line height multiple ..」, 输入想要的值（比如 1.2 ）。该 text layer 的行高就会设定成正确的值，并且会使用一个 mask 把 text layer 的多余边距切掉。

text layer 会与 mask 放在同一个 group 内。菜单中的功能可以直接应用在 text layer 上，也可以应用在 group[^group] 上 （插件会自动寻找内部的 text layer 并更改）。

如果更改了文字内容，使得 text layer 的大小发生变化，可以使用「fix text margin for iOS」来重新设定 group 和 mask 的大小。

插件支持同时选中多个进行操作。

# Appendix II - 如何写 Sketch 插件

sketch 插件可以理解为通过 sketch 执行的脚本。

插件文件本身是一个 bundle，也就是一个文件夹。最基本的配置是一个配置文件和一个脚本文件。配置文件可以控制 sketch 菜单栏上的 plugin 选项。当用户点击菜单的选项之后，会触发对应的脚本。

脚本使用 CocoaScript 书写。它只是 JS 的一个扩展，可以调用 Cocoa 的 API。从脚本回调的 context 中可以得到工作区域中的以 object 表示的树形元素。通过改变 object 的属性，即可改变 sketch 中的内容。比如设置 layer 的 frame，可以改变它的大小。

sketch 官方提供一个 native js 的 api [文档](http://developer.sketchapp.com/reference/api/)，方便使用但功能较少。另外一种方式是直接使用 cocoa object，即 sketch 源代码中的 ObjC 对象。CocoaScript 把原生的 object 都桥接成 JS object 给我们使用，并把我们写的 JS 代码内容在 cocoa 中执行（很像 JSPatch）。 以这种方式写插件，就像在给 sketch 代码开发功能，只不过使用的语言是 JS。可以调用任何 cocoa api，比如弹 alert，或者打开文件选择器。但是官方并没有给我们头文件和任何文档，只能参考 classdump 出来的[头文件](https://github.com/abynim/Sketch-Headers)，比较低效。

官方介绍 [http://developer.sketchapp.com/introduction/](http://developer.sketchapp.com/introduction/)

[^group]: 不是任何 group。是通过插件生成的包含一个 text layer 和 一个 mask 的 group。
[^lineSpacing]: 其实使用的时候会稍微复杂一点。sketch 似乎只能按照整数渲染，而 line height * multiple 会出现小数，设定给 sketch 的行高会被约成整数（虽然界面中仍然是小数，但渲染时会按照整数来渲染）。所以需要在代码中保持 line height + line spacing 为整数（sketch 一般以 2x 图画，所以这里是取到像素精确）。`let finalHeight = ceilToPixel( font.lineHeight * lineHeightMultiple ); style.lineSpacing = finalHeight - font.lineHeight;` 具体可以参照 demo。
[^textkit]:  如果使用 textkit，可能存在中英混排行高的问题，可以使用 [Neat](https://github.com/leavez/Neat) 来解决