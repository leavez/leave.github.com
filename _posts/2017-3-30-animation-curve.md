---

layout:     post
title:      一个隐藏的动画曲线
category:   []
tags:       [小品]
published:  True
date:       2017-3-20

--- 

## 摘要

本文从代码的角度探讨了 iOS 键盘弹出的动画曲线的一些特点，并给出了精确模拟这个动画曲线的方法。同时也介绍了如何逐帧分解 iOS 动画，以及如何通过数据集拟合 Bezeir 曲线的控制点（Control Point）。

## 一个隐藏的参数

### 好看的曲线

![三种动画曲线](/images/2017-3-30-animation_curve/start_animation.gif)

图中第一个动画有一个很好看的动画曲线，看起来很有灵性。这条动画曲线不是 iOS 中默认的那四种，也不是其他[常见的曲线](http://easings.net/zh-cn)。它是 iOS 中一个隐藏的动画参数，没有囊括在 UIViewAnimationCurve 中。

这条曲线并不陌生，它被键盘弹出动画所采用。在实现输入框与键盘同步弹出的场景里，我们会监听键盘弹出的通知。系统在通知的 userInfo 中提供了这个动画的曲线[^1]，它的值为 7。

在工程中，这样可以应用这条曲线[^2]:

```swift
UIView.animate(withDuration: 0.25, animations: {
    UIView.setAnimationCurve(UIViewAnimationCurve(rawValue: 7)!)
    // ...
}
```

### 动画时间

但你会惊奇地发现，一旦使用了这个曲线，动画时间将无法改变，即无论 duration 传了什么值，动画的速度和 CompletionBlock 的调用时间都是一样的。

在键盘弹出的通知中， `UIKeyboardAnimationDurationUserInfoKey` 给出的值是 0.25。但根据实际测试，这个值并不准确。

我们使用 CATransaction 获取来获取动画结束的时刻，并打印用时。（这个方法可以为任何动画提供 CompletionBlock 的调用，很实用。更多 CATransaction 的用法参见[这里](http://calayer.com/core-animation/2016/05/17/catransaction-in-depth.html) )

```swift
let begin = CFAbsoluteTimeGetCurrent()
CATransaction.begin()
CATransaction.setCompletionBlock { 
	print( CFAbsoluteTimeGetCurrent() - begin ) // 0.505300045013428
}
// animation
// UIView.animate(withDuration ...  )
CATransaction.commit()
```

可以得到，这个动画的 duration 是 0.5s。

尝试几种方式改变动画的时间，都没有达到最完美的效果[^5]。既然隐藏的参数带来许多不可控的行为，那么我们希望可以拟合出这个曲线，应用在 CATimingFunction 中，以获得更多的控制空间。

## Rebuid it!

### 导出动画进度

为了拟合出动画曲线，我们需要知道「动画完成的百分比 -  时间」的关系。动画的过程不像 scrollViewDidScroll: 方法一样有一个回调，比较难处理。这里我们使用一个不太常见的方式来处理这个问题: 让动画停下了，手动控制动画进度。

CAMediaTiming 是 CAAnimation 和 CALayer 都实现的一个协议。动画中与时间有关系的属性，最终都与这个协议有关[^3]。这里只使用协议中的两个属性: `speed` 和 `timeOffset`。`speed` 控制动画的速度，当它设为 0 的时候，动画就停止了。`timeOffset` 表示时间的进度，单位为秒，（可以认为）取值范围为 [0, duration] 。

在开始执行一个动画后，我们设置 `view.layer.speed = 0` 使动画暂停，然后手动控制 `timeOffset` 以控制动画的进度。我们等差遍历 `timeOffset` ，然后在每一次遍历中，获取 `view.layer.presentation()` 的属性，就可以得到动画百分比与时间的关系[^4]。

```swift

class ViewController: UIViewController {
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        let v = UIView()
        let startFrame = CGRect(x: 0, y: 0, width: 100, height: 100)
        let finalFrame = CGRect(x: 0, y: 1000, width: 100, height: 100)
        self.v.frame = startFrame
        UIView.animate(withDuration: 0.5) {
            UIView.setAnimationCurve(UIViewAnimationCurve(rawValue: 7)!)
            self.v.frame = finalFrame
        }

        var yPrecentage: [CGFloat] = []
        Helper.dumpAnimation(of: self.v.layer, duration: 0.5, callBack: { i in
            let layer = self.v.layer
            let p = (layer.presentation()?.frame.origin.y ?? 0) / 1000.0
            yPrecentage.append(p)
            print(i, p)
        }) {}
    }
}

struct Helper {
    /// call this method after commiting animation
    static func dumpAnimation(of layer:CALayer, duration: TimeInterval, callBack:@escaping (Int)->Void, completion:@escaping ()->Void) {
        layer.speed = 0
        let total = 1000
        var index = 0
        perform(afterDelay: 0.01, forTimes: total, action: {
            layer.timeOffset = Double(index) / Double(total) * duration
            callBack(index)
            index += 1
        }) { completion() }
    }

    private static func perform(afterDelay:TimeInterval,
                         forTimes: Int,
                         action:@escaping ()->Void,
                         completion:@escaping ()->Void )
    {
        guard forTimes >= 0 else {  completion(); return }
        action()
        DispatchQueue.main.asyncAfter(deadline: .now() + afterDelay) { 
            self.perform(afterDelay: afterDelay, forTimes: forTimes - 1, action: action, completion: completion)
        }
    }
}
```

导出的数据[在此](/images/2017-3-30-animation_curve/keyboard_curve_data.txt)。绘制得到曲线如下:

![键盘弹出的动画曲线。可以看出，它的进入较缓的起始，然后在大约一半时间内完成了 90%，然后有一个很缓慢的结束](/images/2017-3-30-animation_curve/keyboard_curve.png)

### Animation Timing Function

iOS 动画的曲线，即 timingFunction，是由 cubic bezier 曲线表示。一个任意的三阶 Bezier 曲线可以由四个控制点表示。但对于 timingFunction，它的起点是 (0, 0) ， 终点是 (1, 1) ，所以只剩下 2 个点，对于二维曲线即为 4 个参数。

### 通过数据集拟合 Bezier 曲线

使用 http://nbviewer.jupyter.org/gist/anonymous/5688579 里面描述的方法，求出给定数据集的 least squares fitting，给出 control point。（这是找到的唯一一个对应这个问题的且代码可以运行的回答，其他相关资料都不对题）

对上面得到的数据，计算出的结果是:

```
[(0, -0.106),  (0.333, 1.313),  (0.667, 0.909),  (1, 1) ]
```

![红色曲线为拟合的结果](/images/2017-3-30-animation_curve/fitting.png)

可以看出，对于导出数据的最佳 Bezier 拟合并不是我们想要的结果，在起始和结束部分有较大的误差。这也表示这条隐藏的动画曲线并不是 Cubic Bezier 曲线。由于 CAMediaTimingFunction 只能使用 Bezier 曲线来表示，timingFunction 的拟合已经无法实现。

## Aha, CASpringAnimation !

试图找到更多关于曲线的信息，通过如下方式打印出 UIView Animation 中实际生成的 CAAnimation

```swift
let anis = view.layer.animationKeys()?.map { view.layer.animation(forKey: $0) }
print(anis)
```

正常的 UIView Animation，输出如下（已删掉了不必要的信息）:

```
<CABasicAnimation:0x60000003f0c0; timingFunction = easeInEaseOut; duration = 1; keyPath = position> 
```

当使用了隐藏动画曲线后:

```
<CASpringAnimation:0x6000002373a0; timingFunction = linear; duration = 0.5; velocity = 0; damping = 500; stiffness = 1000; mass = 3; keyPath = position>
```

原来这是一个 SpringAnimation，从这个角度看，它已经不是一个动画曲线了。

常见的 SpringAnimation 都是在终点值附近波动几下，所以很难看出来它是一个 SpringAnimation。在阻尼震动中，在终点值附近波动的是欠阻尼状态，即日常所说的弹簧效果。当阻尼系数大于某个值时，震动不会在终点值附近波动，而是缓慢靠近终点值，这种情况叫[过阻尼](https://zh.wikipedia.org/wiki/阻尼)。而我们的动画曲线就是处于过阻尼的状态。

手动创建 CASpringAnimation，来模拟隐藏动画曲线的效果:

```
let ani = CASpringAnimation(keyPath: "position")
ani.damping = 500
ani.stiffness = 1000
ani.mass = 3
ani.duration = 0.5
```

对于一个 1000 pt 的平移动画，两者的标准差为 0.000035，几乎完全相等。我们精确复制了这条隐藏的动画曲线。

回到我们的最初的问题，如何调整动画时长?

对于 CASpringAnimation 直接设置 duration 并不能达到目的，因为当其他几个物理参数设定后，动画的时间就由物理公式确定下来了。duration 的值可以决定在物理过程中，取多长一段。

一个可行的方式是改变质量 mass，并设定 `ani.duration = ani.settlingDuration` 。当质量为 1 时，duration 为 0.3 s，对于 1000 pt 的平移动画，标准差为 0.011。质量为 30 时，duration 为 1.6s，标准差为 0.003。可认为完全达到了我们的目的。

![三种速度的演示](/images/2017-3-30-animation_curve/slow.gif)




[^1]: 从 notification.userInfo 中提取 `UIKeyboardAnimationCurveUserInfoKey` 和`UIKeyboardAnimationDurationUserInfoKey`
[^2]: 虽然是私有 API，但是已经在多个线上的 APP 中使用过，没有问题。
[^3]: 一篇很好的讲解文章 http://ronnqvi.st/controlling-animation-timing/
[^4]: 这个方法来自于右划返回的一种实现方式，参见 http://www.iosnomad.com/blog/2014/5/12/interactive-custom-container-view-controller-transitions Appendix A
[^5]: `CATransaction.setAnimationDuration`， 没有作用。`view.layer.duration`，导致 view 不显现（原因未知）。`view.layer.speed`，在设置比 1 小的值时: 动画时间会变长。但只是把「帧率」降低，「帧数」并没有增加，表现的行为是一帧一帧缓慢跳跃，没有实际意义。比 1 大 时: 有效。例如 speed = 2, 时间会缩小一半。帧率提高不会使动画变得断断续续，但并没有达到最佳效果。