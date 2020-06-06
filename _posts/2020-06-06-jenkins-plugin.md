---
layout:     post
title:      开发 Jenkins 插件：在 Hello World 之后
category:   []
tags:       [tutor]
published:  True
date:       2020-06-06
---

在看完 Jenkins 官网的插件[教学](https://www.jenkins.io/doc/developer/tutorial/)之后，对实际开发 Jenkins 插件还是一头雾水。在实际东拼西凑出几个使用的插件之后，在这里分享一下，hello world 之后该怎么实现我们的功能。

Jenkins 是由 Java 实现，与 Ruby 等动态语言不同，我们无法直接 hook 或直接重写已有的方法来向系统注入功能。为了实现可扩展的能力，Jenkins 在设计上就实现了很多「接口」。比如 Job 设置中，执行 shell 脚本的步骤，就是由一个名为 `builder` 的类的子类实现。我们也可以写一个 `builder` 子类来实现另一种 build 步骤。Jenkins 本身预留了大量类似的接口，它自身的很多功能也是以此类的方式来实现。与此同时，这也意味着插件可以更改的能力是受限的。

这种接口在 Jenkins 中称为 Extension Point，[Extension Points defined in Jenkins Core](https://www.jenkins.io/doc/developer/extensions/jenkins-core/) 列出了所有的  Extension Points。比如上文提到的 `Builder` 可以用来添加自定义执行步骤（执行 shell 的子类叫 `Shell`），`NodeProperty<Node>` 可以为机器添加自定义属性，`QueueDecisionHandler` 可以定制队列，`LabelAssignmentAction` 可以为机器添加动态 label 等。

但单纯实现子类，并不会让系统发现并加载你的代码。每个 Extension Point 都实现了一个叫 `ExtensionPoint` 的 interface。但我们要做的不是要直接实现 interface 的方法，而是为我们的子类添加 `@Extension` 的注解（Annotation）。这样 Jenkins 在启动的时候，就会自动把这个类的功能加载到系统中。

```java
@Extension
public class MyHandler extends QueueDecisionHandler {
}
```

但并不是所有 Extension Point 都是简单对子类添加 `@Extension` 即可。需要添加 UI 元素的 Extension Point，需要对 `DescriptorImpl` 添加注解，而不是对子类本身。

那么如何扩展 UI 呢？首先，Jenkins 并不支持对 web UI 的任意改造：比如删除页面中的某些元素，在首页下面添加自定义内容等。此方面，它仅仅支持在页面的绝对位置（覆盖）添加 HTML 内容。另外一个方面，在添加的 Job / Node 等设置时，一定需要 UI 元素来让用户操作（比如选择或输入数据），并与我们的代码有交互，Jenkins 对此提供了比较完善的支持。

具体 UI 样式由 [Jelly](https://en.wikipedia.org/wiki/Apache_Jelly) 文件来实现，是一种 xml 描述。Jelly 文件需要存放在 resource 目录下的对应路径里。例如下面代码中的 `MyBuilder` 的 UI 就应该在 resource 下建一个 `MyBuilder/config.jelly` 文件。Jelly 的使用可以参见 [Basic guide to Jelly usage in Jenkins](https://wiki.jenkins.io/display/JENKINS/Basic+guide+to+Jelly+usage+in+Jenkins)，这里不赘述（其实我也没有弄太明白 😂）。代码侧，每个可以实现 UI 的 Extension Point 都符合 `Describable` 接口，一般需要在子类里实现一个符合该接口的内部类。UI 出现的位置，是由 Extension Point 的种类决定的。比如 `Builder` 就会出现在 Job 设置中的 Build 段，而 `BuildWrapper` 则出现在 Build Environment 段。

```java
public class MyBuilder extends Builder {
  
    @DataBoundConstructor
    public MyBuilder(MyJobSetting setting, String script) {
      // ...
    }
  
    @Extension
    public static class DescriptorImpl extends BuildStepDescriptor<Builder> {
    // BuildStepDescriptor 即为 Describable 的一个抽象类
        @Override
        public String getDisplayName() { // UI 元素中显式的名字
            return "my builder";
        }
    }
}
```

在 UI 与代码的数据流动中，代码中的 getter 供给 jelly 文件中 `filed` 字段同名的 UI 元素的内容；在从 web 界面保存设置时，jelly 中对应的字段会寻找标有 `@DataBoundConstructor` 的初始化方法，并供给同名参数，生成我们的子类。如果一个字段不是基本类型，则需要该类型实现 `Describable` 接口并实现自己的 jelly 文件。

如何适配多节点的 Jenkins？（多节点指的用多台机器作为 salve）Jenkins master 和 slave 的关系远不止 ssh 上去执行代码的层次，他们之间的通信是在 java 层面上的。其大致原理为，执行的代码以闭包的形式给出，序列化后传到 slave 的 JVM 上，执行完后再把结果传回来。具体到代码上，是实现一个 `MasterToSlaveCallable<T,U>` 的子类，然后通过 `channel.call()` 来执行。但 Jenkins 中也包含了一种隐式的远程执行，如  `hudson.FilePath`  就可以在无感知的情况下进行 slave 机器上的文件操作。更多信息参考 [Making your plugin behave in distributed Jenkins](https://wiki.jenkins.io/display/JENKINS/Making+your+plugin+behave+in+distributed+Jenkins)。

至此，大致概念已经介绍清楚。如果你仍然困惑该怎么下手，最好的办法是模仿。[Plugin Cookbook](https://wiki.jenkins.io/display/JENKINS/Plugin+Cookbook) 列出了几个典型的功能和对应实现的例子，可以照猫画虎。Extension Point 的文档中，也列举了每个接口的实际插件例子。开发中经常查阅的几个文档汇总在这里：

- [Wiki / Extend Jenkins ](https://wiki.jenkins.io/display/JENKINS/Extend+Jenkins) 为官方的插件文档总汇，算是最系统的资料了
- [Extension Points](https://www.jenkins.io/doc/developer/extensions/jenkins-core/)
- [Javadoc](https://wiki.jenkins.io/display/JENKINS/Extend+Jenkins) 可以查阅具体 Jenkins API 的使用方法
- [Jelly Tag](https://reports.jenkins.io/core-taglib/jelly-taglib-ref.html#form:nested) 为 Jelly UI 中使用到的元素文档

在实际开发中还会遇到一些其他问题。比如我不太熟悉 Java，但 IDE 现场教学，帮了很多忙。其实 Java 是一门很传统的语言，各种元素大家都很熟悉，很容易入门。Maven 依赖版本冲突问题，也是花了很多时间。Maven 默认是按照最短依赖路径来处理版本冲突的，但官方插件工程里的模板里设置了有冲突即报错。其中 pom 文件中的 `jenkins.version ` 字段，它带来了该版本的 jenkins core 的所有依赖。所以会出现 dependency 中只引了一个库但出现版本冲突的问题。

额外说一下为什么要改造 Jenkins。从界面就可以看出来，jenkins 非常古老（不过仍在持续开发）。对于普通的 CI/CD 需求，Gitlab CI 等类似品可以非常好得满足需求，且天然的内置在 MR 流程中。但也因为和代码结合得太多，不适合独立的离线任务。而 Jenkins 正好能满足此类需求，灵活，且功能全面。 由于它设计上支持功能扩展，能以较低成本的实现定制化需求。这就是写 Jenkins 插件的理由。不过我们也需要忍受一些它的缺点，如作为一个老式的单机程序，无法靠多机器来扩展 master，单机的服务稳定性也难以保证，这些问题都会在规模扩大时凸显出来。

