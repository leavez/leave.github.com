---
layout:     post
title:      更友好的 warning 消除方案
category:   []
tags:       [技术]
published:  True
date:       2018-1-6

---


对于编译型语言，开启编译器的「Treat Warnings as Errors」是非常有益的。它把 warning  当做错误，会中断编译，强制我们修复问题。在没有开启这个功能的时候，warning 会随着开发不断积累增多。当数量很多的时候，新增的 warning 不容易被发现，从而掩盖问题。 对于一个自律的独立开发者，这个问题是可以避免的，但多人协作的情况下，需要「Treat Warnings as Errors」这样强制的功能来保障。

但是它有一个问题: 对于开发和调试不太友好。我们会经常编译一些未完成的代码，这个时候非常容易出现 warning。比如调试时注释掉了几行代码，因此上一行出现了一个「未使用的变量」，打断编译，无法向下进行。此时必须手动（递归的）解决掉 warning， 而这没有任何实际意义。影响效率，非常讨厌。

## 在 commit 时禁止 warning

我们需要更灵活的方式。在 commit 的时候禁止有 warning 的代码提交是一个很好的选择。开发的时候任你胡搞，只要提交的代码完好就可以了。虽然 Xcode 没有类似的功能直接使用，但自己实现原理也不复杂: 通过 git 的 pre-commit hook，检查是否存在 warning，如果有，则中断 commit 即可。

### 如何检查代码存在 warning？

Xcode 编译的时候会把 log 保存成文件。通过添加 run script，打印 `$BUILD_ROOT`，可以找到当次编译的生成文件目录。目录结构如下

```
├── Build
│   ├── Intermediates.noindex
│   └── Products (this is $BUILD_ROOT)
├── Index
│   ├── DataStore
│   └── UniDB
├── Logs
│   ├── Build
│   ├── Debug
│   └── Issues
└── TextIndex
```

其中 `Logs/Issues` 目录下保存了 xcode 界面中 Issues 展示的信息，也就是我们要找的 warning。

log 以 `.xcactivitylog` 的文件存储，它其实就是一个 gzip 文件，解压后就可以读了。log 的组织有些混乱，每个 log 似乎是以 `\r` 分割。用最简单的方式过滤出 带有 warning 行，即可知道是否存在 warning。

我们把检查结果存放在 `your_project_dir/.warning_checker/last_result` 。

### 添加 commit hook

[git hook](https://git-scm.com/book/en/v2/Customizing-Git-Git-HooksZ) 是 git 自带的功能，我们只需要在 `your_project_dir/.git/hooks` 添加固定名称的脚本（可以使用任何脚本语言），git 在对应的时刻就会执行。比如 `pre-commit` 会在 commit 前执行，如果返回值不等于 0，则中断 commit。中断的时候在 stderr 中输出原因，会被 git 展示出来（无论是命令行， 还是 GUI 程序）。

我们在 pre commit hook 中读取上一步中的检查结果，如果存在 warning，则打印出对应原因，返回错误，使 commit 中断。实现我们在 commit 时候禁止 warning 的目的。

### 集成

在每次 build 的时候执行脚本。一个方式是在 Build Phases 中添加 Run Script 来执行，但它不能后台执行。虽然每次脚本并不耗费多少时间，但还是有感知。另一个方式是添加到 scheme 的 build post-action 中，可以后台执行，不影响 build & run 的流程。

另一个问题是如何添加部署 hook。hook 文件并不在 git 的管理范围内，只在本机生效。我们可以在 warning 脚本里添加写入 hook 的代码，这样 hook 会随着脚本的执行添加到每一个使用者的仓库。

完成的代码见 [github](https://github.com/leavez/LazyWarningChecker)。除了上面所说的内容，代码里添加了对 warning log 的解析与规则的配置功能。

## 逐步地清除 warning

每一个想开启「Treat Warnings as Errors」功能的项目，都会面临这一个问题: 如何把已有的上百个 warning 清除掉？

问题还是要手动解决，但上面的脚本可以给我们一些帮助:

- 选择性地禁止提交 warning。这样可以分批次分类型地开启禁止 warning（比如最先禁止 unsed variable），以防止不断有新的 warning 产生。
- 找到每个 warning 的 blame，指定给产生问题的人来处理。

warning log 中包含文件路径、警告内容和对应的编译器 flag，我们可以做很多事情。对于前者，只需要在脚本中定制检查规则即可，对于「未使用的变量」，可以用 `-Wunused-variable` 的编译器 flag 来过滤。对于后者，因为已知文件路径和行数，可以通过 git 读取到 blame。

```python
command = "git blame -L%s,+1 %s -p" % (lineNumber, path)
result = subprocess.check_output(command, shell=True)
```





