---
title: 打包 JavaFX 程序
date: 2018/10/31 17:12:0
tags: [JavaFX]
categories: [JavaFX]
---

公司有一个项目在发布更新时，一直是由开发人员手工选择修改过的源代码文件编译后打包发给实施人员，然后由实施人员到测试环境去替换，如果修改的文件较多，开发人员一个个去查找会比较麻烦，还容易出错，所以就开发了一个小工具，帮助开发人员自动生成增量更新包。  

<!--more-->  

这个小工具大体的思路是，根据 `svn log` 命令生成一个 changelog 文件，读取这个文件来收集被修改过的文件，然后去本地已经编译过的项目中寻找对应的文件，然后打包。这个小工具已经发布到了 Github 上，链接是：<https://github.com/nekolr/sirius-inc>。在写完大体逻辑后，又使用 JavaFX Scene Builder 构建了程序的界面。不同于之前的 Swing，JavaFX 的布局和逻辑代码完全分离，代码更清晰也更容易编写。前面的过程都还算顺利，在最终打包阶段遇到了不小的麻烦，不过也一点点解决掉了，在这里记录一下打包的过程。  

首先在 IDEA 中打开项目的 Project Structure，选择 Artifacts 新建一个 JavaFX Application 构件。  

![new JavaFX Application](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/10/31/LgX.png)  

在这里将 sirius-inc_main 和 sirius-inc_test 都加入到 jar 包下。  

![加入到 jar 包中](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/10/31/4e4.png)  

这里需要注意的是，如果源代码目录下有 resources 资源目录，则需要手工将该目录加入到 jar 包下。  

![加入资源目录](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/10/31/l2q.png)  

选择到 Java FX 标签页，配置主程序入口和本地包类型（如 exe）。  

![javafx 配置](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/10/31/dMM.png)  

这些都配置好后，选择 Build 中的 Build Artifacts 来生成构件，等待完成后就可以在 bundles 目录下的项目目录中找到 exe 可执行文件。  

> 通过这种方式在生成 exe 的同时，还会生成 Java 运行环境，所以整个目录的体积较大。  

在打包完成后，其实又遇到了一个新的问题，那就是程序的图标不能自定义。一开始尝试使用 launch4j 来打包，虽然可以自定义图标，但是不知道什么原因，程序只有外壳，逻辑无法执行。最后没办法，只能使用 eclipse 配合 e(fx)clipse、ant 来打包，这里同样记录一下过程。  

首先 eclipse 要安装 e(fx)clipse 插件，然后通过 eclipse 新建一个 JavaFX 项目，eclipse 会在项目目录中生成一个 build.fxbuild 文件，打开该文件，填写配置。  

![build.fxbuild](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/11/01/Rl6.png)  

配置好后，点击右侧的 Generate ant build.xml only 来生成 build 目录和文件。  

在 build 目录下新建一个 package 目录，并在 package 目录下创建 macos 和 windows 目录，这两个目录用来放置程序的图标，图标名称要与 build.fxbuild 文件中的程序名相同。  

修改 build.xml 文件，在 `<path id="fxant">` 的文件列表中添加一行 `<file name="${basedir}"/>` 用来包含资源文件，如图标。  

![指定资源文件](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/11/01/AGN.png)  

修改程序的版本。  

![修改版本](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/11/01/9by.png)  

修改完毕后，右键 build.xml 文件 Run as Ant build 来打包。等待打包完毕后，会在 build/deploy 目录生成可执行文件，目录结构与通过 IDEA 打包生成的结构一致，唯一不同的是程序的图标已经变成自定义的了。  

