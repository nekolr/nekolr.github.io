---
title: 一次 Git 踩坑记录
date: 2018/9/14 10:10:0
tags: [Git]
categories: [Git]
---
在不同的操作系统下，文本文件使用的换行符是不一样的。UNIX/Linux 使用的是 LF，对应的十六进制编码为 0x0A，而 DOS/Windows 使用的是 CRLF，对应的十六进制编码为 0x0D0A。一般的开源项目，源代码都是使用 UNIX/Linux 风格保存的，这么做是为了统一文本风格，使源代码不论在哪个平台下，显示都是一致的。这在多人协作开发，尤其是团队成员使用的平台不同时尤为重要。  

由于我使用的开发平台是 Windows，因此就犯了提交的代码文本风格全部为 CRLF 的错误。这些代码在 UNIX/Linux 平台使用 vim 打开时，就会是以下的样子。  

![错误的文本风格](https://img.nekolr.com/images/2018/09/14/Lkp.png)

可以看到，所有应该是回车的地方都多出来了 ^M（如果使用 vim 打开不是这样的，可以输入 :e ++ff=UNIX 调整显示）。  

那么该如何统一规范呢？首先说第一种情况。  

## Git 仓库中源代码为 CRLF 风格
此时为了统一成 LF 风格，我们需要借助一些工具来将仓库中的所有代码文本中的 ^M 替换掉。在 Linux 中有一个工具特别好用：dos2UNIX。  
```
dos2UNIX [options] [file ...] [-n infile outfile ...]
```

```
-k：保持输出文件的日期不变 
-q：安静模式，不提示任何警告信息
-V：查看版本
-c：转换模式，模式有：ASCII, 7bit, ISO, Mac, 默认是：ASCII
-o：写入到源文件
-n：写入到新文件
```

由于 dos2UNIX 替换文件只能通过枚举的方式，所以再借助 `xargs` 命令来实现批量替换。比如：  
```
:: 查找 demo 目录下的所有后缀为 java 的文件替换
find demo/ -name "*.java" | xargs dos2UNIX -o

:: 查找 demo 目录下的所有的文件替换
find demo/ -type f | xargs dos2UNIX -o
```

将所有文件替换好后提交到 Git 仓库中。  

## Git 仓库中源代码为 LF 风格
这样的情况肯定是好的，接下来我们该做的就是规范我们的提交。  

一种方式是依赖于 Git 客户端提供的自动转换功能。  
```
git config --global core.autocrlf true
```

这样设置，在提交代码时，会将自动 CRLF 转换成 LF，在 checkout 回本地时，会自动将 LF 转换成 CRLF。  

如果你使用的是 Linux 或者是 Mac，可以进行如下的设置：  
```
git config --global core.autocrlf input
```

这样在提交代码时会自动将 CRLF 转换成 LF，而在 checkout 时则不会进行转换。  

另一种方式就比较严格了。首先关闭 Git 客户端的自动转换，同时设置 IDE 的换行符为 UNIX/Linux 风格。比如在 IDEA 下：  

![UNIX 风格换行](https://img.nekolr.com/images/2018/09/14/4kd.png)

## 统一规范（仅参考）

- 关闭 Git 客户端自动转换。
- 设置 IDE 换行为 UNIX/Linux 风格。
- 设置项目编码为 UTF-8 without BOM。  

![utf-8](https://img.nekolr.com/images/2018/09/14/lqv.png)

- 如果是 maven 项目，自定义合适的配置。  

![maven](https://img.nekolr.com/images/2018/09/14/dy8.png)

- 设置 Tab 为四个空格。  

## 参考

> [GitHub 第一坑：换行符自动转换](https://github.com/cssmagic/blog/issues/22)