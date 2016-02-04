---

layout:     post
title:      我为什么写博客
category:   []
tags:       [新鲜事]
published:  True
date:       2016-2-4


--- 


## 我为什么写博客

##### It's something cool

对，我觉得写博客很 Cool，尤其是有这样一个自己的网站。

偶然间发现了 Melodie 同学的[博客](http://melodiezhang.com/about/)，简洁的设计和深入思考的文字令我很崇拜，所以有了建立自己博客的念头。虽然全民博客的时代早已经过去，但博客本身却仍是网络中信息的一种基础的组织形式。随意的碎片化的东西被卷入了社交网络，剩下有价值的东西，变成每天无数次 google 的结果中的部分条目。那些厉害的人把自己的见解与总结写下来，与大家分享。

##### 写作可以促进思考
> 思考是我们常常挂在嘴边却实际上没几个人真正做到的一件事；多数情况下我们持有的某些观点只是环境所灌输的，我们急于下结论，忽略假设、理由、证据、逻辑推理的整个过程，即所谓的批判性思考（ critical thinking ）。博客是一个很好地能够静下心记录和反刍思考的地方，而我写博客是为了训练自己成为一个**终身的学习者和思考者（ To be a lifelong learner and thinker ）**。
> 
>  Melodie

想法中认为理解的东西，可能只是你认定的理解。当它们被写下来的时候，就会发现其中的不严谨\不完备\不够结构化，书写的过程就是在不断克服这些东西。而且写给大众 ( 而不是局限为自己的笔记 ) 会更加强化这个作用。博客的存在也会更加督促自己把一些东西书写下来。


## 简单地搭建指导

博客使用 [Github Pages](https://pages.github.com) (GP) 来搭建。因为很喜欢 Melodie 的样式，所以就 ( 经过允许并做了细微修改后 ) 直接拿来用了。


Github Pages 是一个静态网页 hoster。只需要把编写好的静态网页放到自己的 Github 仓库的 gh-pages 分支 ( 需要设为默认分支 ) 中，就可以通过 `[username].github.io/[repository-name]` 访问到。同时，它使用 [Jekyll](http://jekyllrb.com) 提供了一定的动态内容生成的功能，适合简单 blog 使用。Jekyll 相当于一个编译器，把混杂[代码](http://jekyllrb.com/docs/posts/#displaying-angit-index-of-posts)的 HTML 转换成静态的 HTML。所以，博客的文章需要使用 Git 进行管理。样式确定好后，写新文章主只需写 Markdown 文件即可。

使用 GP 搭建自己的博客，只需要按照 [Jekyll 的格式](http://jekyllrb.com/docs/structure/)编写好 Web 文件 ( HTML CSS )，上传到仓库即可。可以 fork 一个已有的仓库以简化这个过程。需要注意，Jekyll 默认使用 kramdown 作为 Markdown Parser ，而 [kramdown](http://kramdown.gettalong.org/syntax.html) 与标准 Markdown 并不完全相同。比如，header 前需要空一行， `##` 与标题的内容间要有一个空格，否则不能识别。


配上一个独立的域名才够帅气。域名从 namecheap.com 购买，只因为便宜。GP 本身支持独立域名，需要在仓库中添加一个名为 CNAME 的文件，其内容要使用的域名。在域名提供商的网站上配置一条 A Record，并把内容设为 `192.30.252.153`。详细配置请看[这里](https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/)。按我的理解，后者是把域名定位到 Github 的服务器，而前者是告诉 Github 的服务器这个域名要转到我这里。













