---
layout:     post
title:      评论区的几种形式
category:   []
tags: [设计]
published: True
date: 2015-11-1
summary:  前段时间的工作涉及到「评论」的设计，期间对评论这一模式有了新的理解。最近刚好又读到 How to display threaded discussions on the web，一篇很有启发性的针对讨论流展示的总结。这里通过一些案例，记录下在此过程中的收获。根据评论回复的对象可区分两种评论区的机制：1. 回复的对象是某个人，不区分具体回复的...
 
---   

  
前段时间的工作涉及到「评论」的设计，期间对评论这一模式有了新的理解。最近刚好又读到 [How to display threaded discussions on the web](http://www.elezea.com/2015/09/how-to-display-threaded-discussions-on-the-web/)，一篇很有启发性的针对讨论流展示的总结。这里通过一些案例，记录下在此过程中的收获。

### 两种评论区机制
根据评论回复的对象可区分两种评论区的机制：

**1. 回复的对象是某个人，不区分具体回复的是哪一条**  
适用于评论者之间的互动较少、结构简单的情况。基于这一机制的评论区，比较像多人在微信、QQ 一类的即时通讯工具中的群聊，回复类似于 @ 功能，不能针对某个人说的某句话回复。评论区多为扁平结构，即所有评论按某种顺序（时间序或其他）平铺展示。Dribbble、知乎目前采用的这种方式。

**2. 回复的对象是某条评论**  
当鼓励在评论区进行针对性讨论，强调评论区的社交属性时，这种机制更加适用。
这篇文章主要讨论该前提下的结构设计。
  
### 三种基本结构
#### 一层
平铺显示所有评论。由于可能缺失「回复」上下文，有些会提供「查看对话」或「评论引用」功能。一些例子：

**▼ 微博的评论**
![image](http://7xldlp.com1.z0.glb.clouddn.com/blog/image/weibo.png)

**▼ 网易新闻跟帖** 
![image](http://7xldlp.com1.z0.glb.clouddn.com/blog/image/netease.png)  
  
**▼ 果壳的「引用」**
![image](http://7xldlp.com1.z0.glb.clouddn.com/blog/image/guokr.png)  

#### 两层
两级及以上的回复评论都显示在第二层，在展示上更强调第一级的非回复性评论。

**▼ Facebook**
![image](http://7xldlp.com1.z0.glb.clouddn.com/blog/image/facebook2.png)
  
#### N 层
逻辑关系容易理解，但展示起来更复杂，采用这种方式的评论区一般会收起部分的子评论，提供查看更多回复的入口。如果回复评论的层级很多，传统的阶梯式展现形式会导致阅读体验不佳。

▼ **Reddit**   
![image](http://7xldlp.com1.z0.glb.clouddn.com/blog/image/reddit2.png)  
 
▼ **Quora**  
![image](http://7xldlp.com1.z0.glb.clouddn.com/blog/image/quora2.png)

从「一层」到「N 层」，评论区的讨论氛围随着结构的丰富依次上升。虚拟世界的「评论区」好比一个「空间」，其形式当然不止以上几种，就像一处「空间」有着无限可能性。不同的产品特质催生不同形式的评论区，在设计时，需要考虑我们希望这里将发生什么样的交流，是更重视人们对主体内容的评价，还是个体之间的相互探讨？据此重新思考对交流「空间」的定义。
  

  

