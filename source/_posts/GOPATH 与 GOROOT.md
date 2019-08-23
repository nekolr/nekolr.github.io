---
title: GOPATH 与 GOROOT
date: 2019/4/15 15:12:0
tags: [Golang]
categories: [Golang]
---
在 Windows 平台下，我一般会下载软件的 archive 文件然后解压使用，这次在初学 Golang 时就遇到了一个小坑。  

<!--more-->

Golang 的环境变量中有几个比较重要的：GOROOT 和 GOPATH，其中 GOROOT 其实就是 Golang 的安装路径，而 GOPATH 是 Golang 的工作目录，也就是平常开发使用的所有项目的根目录。在设置环境变量时，新建 `$GOROOT` 和 `$GOROOT` 环境变量，值分别为 `D:\go` 和 `D:\goworkspace`，同时添加一条 `$PATH` 值为 `%GOROOT%\bin`，这样就可以使用 go 命令了。  

接下来将 Golang 官方的包管理工具 dep 安装到 `%$GOROOT%\bin` 中，此时就可以开始编程惯例 Hello World 了。  

在 `$GOPATH` 下新建 src 目录，然后在 src 目录下新建一个项目目录为 example，同时使用 `dep init` 命令初始化，这会在该目录中生成一个 vendor 目录，一个 Gopkg.lock 文件和一个 Gopkg.toml 文件。在项目目录下新建一个 helloworld.go 文件，代码如下：  

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    fmt.Println("Hello World!")
    time.Sleep(5 * time.Second)
}
```

接下来一定要在项目目录 `example` 下使用 `go install` 命令，这样才会将编译生成的 exe 二进制文件复制到 `$GOPATH\bin` 目录中。