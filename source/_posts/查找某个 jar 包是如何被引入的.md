---
title: 查找某个 jar 包是如何被引入的
date: 2019/3/8 16:30:0
tags: [Java,IDEA,问题集]
categories: [问题集]
---

公司项目使用 Maven 进行包依赖管理，一个项目少说得有上百个包，如何找到某个包是如何被引入的呢？可以有两种方式来定位。  

<!--more-->  

第一种就是通过开发工具查找。  

我使用的是 IDEA。点开 Maven Projects（一般在 IDEA 右侧），然后选择某个 Module，然后右键选择 Show Dependencies，IDEA 会打开一个视图。  

![依赖视图](https://img.nekolr.com/images/2019/06/19/2x3.png)  

使用 `Ctrl + F` 搜索需要寻找的包名，然后就可以看到该包是通过哪个其他的 jar 包引入的。  

![定位包](https://img.nekolr.com/images/2019/06/19/VYD.png)  

第二种方式就是通过 Maven 命令去查找。  

使用 `mvn dependency:tree -Dverbose -Dincludes=com.google.code.gson:gson` 命令查找。这里的 `dependency:tree` 表示通过树状显示所有的依赖项，`Dverbose` 表示显示所有的引用，包括因为多次引用而重复的。`Dincludes` 表示被引用的包。