---
title: 使用 HSDB 调试 HotSpot
date: 2018/4/17 16:41:0
tags: [Java,JVM]
categories: [JVM]
---
环境说明：  
- OS：Windows 10
- JDK：Oracle JDK 1.8.0_144

<!--more-->

使用以下代码进行测试：  

```java
/**
 * HotSpot VM 运行时数据区测试类
 */
public class Test {

    static Test2 t1 = new Test2();
    Test2 t2 = new Test2();

    public void fn() {
        Test2 t3 = new Test2();
    }
}

class Test2 {

}
```

```java
public class Main {

    public static void main(String[] args) {
        Test test = new Test();
        test.fn();
    }
}
```

## 前置条件

首先需要确保环境变量 JAVA_HOME、PATH 和 CLASSPATH 配置成功。  

## 编译代码

![编译代码 ](https://img.nekolr.com/images/2018/04/17/Xzp.png)

## 启动 jdb

我们在使用 IDE 时，可以很方便地在代码中打上断点进行调试，如果不依赖 IDE，我们可以使用 Oracle JDK 自带的 `jdb` 工具来完成这项任务。  

![启动 jdb](https://img.nekolr.com/images/2018/04/17/PPk.png)

启动 `jdb` 时，设定 java 程序使用 `Serial GC` 和 10MB 的堆内存。  

![设置断点 ](https://img.nekolr.com/images/2018/04/17/y2j.png)

使用 `stop in` 命令在指定的 java 方法入口处设置断点。  

![run](https://img.nekolr.com/images/2018/04/17/zry.png)

使用 `run` 命令指定主类来启动 Java 程序。  

![next](https://img.nekolr.com/images/2018/04/17/x38.png)

使用 `next`、`step` 等命令来控制前进。  

## 使用 jps 查看进程 id

另起一个终端。  

![jps](https://img.nekolr.com/images/2018/04/17/rgG.png)

## 启动 HSDB

```

java -cp ".;%JAVA_HOME%/lib/sa-jdi.jar" sun.jvm.hotspot.HSDB

```

这时就会启动 HSDB 了。  

![HSDB](https://img.nekolr.com/images/2018/04/17/baN.png)

![attach](https://img.nekolr.com/images/2018/04/17/a6e.png)

![](https://img.nekolr.com/images/2018/04/17/O2w.png)

输入 Main 程序的进程 id。  

![进程信息 ](https://img.nekolr.com/images/2018/04/17/GPL.png)

这里是线程信息。  

![栈内存信息 ](https://img.nekolr.com/images/2018/04/17/Kbq.png)

选中主线程，然后选择 `Stack Memory` 一栏，会显示 main 线程的栈信息。  

## 参考

> [借 HSDB 来探索 HotSpot VM 的运行时数据 ](http://rednaxelafx.iteye.com/blog/1847971)