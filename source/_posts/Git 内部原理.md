---
title: Git 内部原理
date: 2021/6/10 11:47:0
tags: [Git]
categories: [Git]
---

这篇博文主要用于记录博主在学习 Git 官方教程时的整个过程和一些心得体会，具体学习的章节是《Git 内部原理》。

<!--more-->

# 底层命令与上层命令
我们平常使用的 git 命令一般都是对用户友好的上层（porcelain）命令，然而，由于 Git 最初只是一套面向版本控制系统的工具集，所以它还包含了一部分用于完成底层工作的子命令，这部分命令一般称为底层（plumbing）命令。

# 目录结构
当在一个新目录或已有目录执行 `git init` 时，Git 会创建一个 `.git` 目录。 这个目录包含了几乎所有 Git 存储和操作的东西。如果想备份或复制一个版本库，只需把这个目录拷贝至另一处即可。新初始化的 `.git` 目录的典型结构如下：

```
config
description
HEAD
hooks/
info/
objects/
refs/
```

description 文件仅供 GitWeb 程序使用，我们无需关心。config 文件包含项目特有的配置选项。info 目录包含一个全局性的排除（global exclude）文件，用于放置那些不希望被记录在 `.gitignore` 文件中的忽略模式（ignored patterns）。hooks 目录包含客户端或服务端的钩子脚本（hook scripts）。

剩下的四个条目很重要：HEAD 文件、（尚未创建的）index 文件，和 objects 目录、refs 目录。它们都是 Git 的核心组成部分。objects 目录存储所有的数据内容；refs 目录存储指向数据（分支、远程仓库和标签等）的提交对象的指针；HEAD 文件指向目前被检出的分支；index 文件保存暂存区信息。

# Git 对象
Git 的核心部分是一个简单的键值对数据库。我们可以向 Git 仓库中插入任意类型的内容（文本或文件），它会返回一个唯一的键，通过该键可以在任意时刻再次取回该内容。底层命令 `git hash-object` 可以实现该效果：将任意数据保存到 `objects` 目录，并返回指向该数据对象的唯一的键。

```
$ git init test
Initialized empty Git repository in /tmp/test/.git/
$ cd test
$ find .git/objects
.git/objects
.git/objects/info
.git/objects/pack
```

可以看到，Git 对 objects 目录进行了初始化，并创建了 pack 和 info 子目录，但均为空。接着，我们用 `git hash-object` 创建一个新的数据对象并将它手动存入 Git 数据库中：

```
$ echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

其中，`-w` 选项指示该命令不光要返回键，还要将该对象写入数据库中。`--stdin` 选项则指示该命令从标准输入读取内容，如果不指定此选项，则须在命令尾部给出需要存储的文件的路径。比如：

```
$ echo 'test content' > test.txt
$ git hash-object -w test.txt
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

该命令输出的是一个长度为 40 个字符的哈希值，该校验和通过将待存储的数据外加一个头部信息（header）一起做 SHA-1 校验运算得到。现在我们可以查看 Git 是如何存储数据的：

```
$ find .git/objects -type f
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

再次查看 objects 目录，可以在其中找到一个与新内容对应的文件。这就是 Git 存储内容的方式：一个文件对应一条内容，以该内容加上特定头部信息一起做 SHA-1 校验和来为文件命名。校验和的前两个字符用于命名子目录，余下的 38 个字符则用作文件名。

一旦将内容存储在了对象数据库中，那么就可以通过 `cat-file` 命令从 Git 仓库取回数据（如果直接使用 cat 命令查看写入的内容，会发现全是乱码，这是因为写入的东西是经过压缩的）。指定 `-p` 选项可指示该命令自动判断内容的类型，并为我们显示大致的内容：

```
$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
test content
```

接下来我们尝试对一个文件进行简单的版本控制。首先，创建一个新文件并将其内容存入数据库：

```
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30
```

接着，向文件里写入新内容，并再次将其存入数据库：

```
$ echo 'version 2' > test.txt
$ git hash-object -w test.txt
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
```

对象数据库记录下了该文件的两个不同版本，当然之前我们存入的第一条内容也还在：

```
$ find .git/objects -type f
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

接下来我们只需要通过 `cat-file` 传入不同的键，就可以取回该文件的第一个版本和第二个版本：

```
$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30
version 1
$ git cat-file -p 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
version 2
```

然而，记住文件的每一个版本所对应的 SHA-1 值并不现实；另一个问题是，在这个（简单的版本控制）系统中，文件名并没有被保存。上述类型的对象我们称之为数据对象（blob object）。利用 `git cat-file -t` 命令，可以让 Git 告诉我们其内部存储的任何对象类型，只要给定该对象的 SHA-1 值：

```
$ git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
blob
```