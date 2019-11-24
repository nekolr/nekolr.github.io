---
title: IDEA 导出可执行 jar
date: 2017/7/26 11:12:0
tags: [IDEA,Java,问题集]
categories: [问题集]
---
STEP1：File -> Project Structure -> Artifacts		
		
<!--more-->		

STEP2：Add -> JAR -> From modules with dependencies
![add](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/V9.png)

STEP3：设置 Main Class，设置 MANIFEST.MF 目录。这里注意 META-INF/MANIFEST.MF 目录不要使用默认的，而是选择项目目录。		
![manifest.mf](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/2w.png)

STEP4：Build -> Build Artifacts