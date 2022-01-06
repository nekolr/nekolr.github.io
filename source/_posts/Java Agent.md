---
title: Java Agent
date: 2021/8/15 16:32:0
tags: [Java]
categories: [Java]
---

说到 Java Agent，有必要提一提 AOP，也就是面向切面编程思想。AOP 作为 OOP（面向对象编程思想）的一个补充，在技术实现上是没有规定和约束的。在 Java 中，最常见的实现方式就是 Filter 的责任链模式和 Spring AOP 的代理模式。

 <!--more-->

当然，AOP 也不一定非得像 Spring AOP 那样，在运行时通过动态生成代理对象来织入增强。一个 Java 类的生命周期，从编码开始，还需要经历编译、加载、连接、初始化、使用和卸载这些阶段。而在使用之前，每个阶段我们都可以对类进行增强。

# 编码阶段
在编码阶段，我们可以通过静态代理的方式对目标对象进行增强，但是缺点也很明显，就是不够灵活，属于一锤子买卖。

```java
public interface Subject {
    void operation();
}

public class SubjectImpl implements Subject {
    @Override
    public void operation() {
        System.out.println("operation...");
    }
}

public class SubjectProxy implements Subject {
    private Subject target;

    public SubjectProxy(Subject target) {
        this.target = target;
    }

    @Override
    public void operation() {
        System.out.println("before...");
        target.operation();
        System.out.println("after...");
    }
}
```

# 编译阶段
在编译阶段，我们可以通过使用特殊的编译器，在将源文件编译成字节码的时候进行增强，编译后的字节码本身就包含了增强的内容，这就是编译期织入 CTW（Compile-Time Weaving）。典型的工具库就是：AspectJ。

AspectJ 提供了两种切面的编写方式，其中一种是使用 AspectJ 特有的语法；另一种是使用注解。

```java
public aspect SubjectAspect {
    // 切点 pointcut
    pointcut doBefore():execution(void SubjectImpl.operation(..));
    pointcut doAfter():execution(void SubjectImpl.operation(..));
    pointcut doAround():execution(void SubjectImpl.operation(..));
    
    // 通知 advice
    before(): doBefore() {
        System.out.println("before...");
    }

    after(): doAfter() {
        System.out.println("after...");
    }

    // around 和 before、after 不要同时出现，编译会报错
    void around(): doAround() {
        System.out.println("around before...");
        proceed();
        System.out.println("around after...");
    }
}
```

```java
@Aspect
public class SubjectAspect {

    @Pointcut("execution(* org.example.SubjectImpl.*(..))")
    public void pointcut() {}

    @Before("pointcut()")
    public void doBefore() {
        System.out.println("before...");
    }

    @After("pointcut()")
    public void doAfter() {
        System.out.println("after...");
    }
}
```

虽然使用 AspectJ 的特有语法来描述切面会更加灵活，但是由于它不兼容 java 语法，在 IDE 中需要安装插件来支持它的语法，再加上需要额外的学习成本，因此这种方式实际上使用的并不多，通常还是采用兼容 java 语法的注解来定义切面。

在定义好了切面以后，还需要使用 AspectJ 特定的编译器来编译代码。在 AspectJ 1.9.7 版本中，可以直接通过下载的 jar 包进行安装，安装后的程序包含 `ajc` 命令。当然也可以通过安装 IDE 插件或者使用构建工具（比如 Maven）的插件来编译。

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.14.0</version>
    <configuration>
        <complianceLevel>1.8</complianceLevel>
        <source>1.8</source>
        <target>1.8</target>
        <showWeaveInfo>true</showWeaveInfo>
        <verbose>true</verbose>
        <Xlint>ignore</Xlint>
        <encoding>UTF-8</encoding>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>test-compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

下面就是通过 `ajc` 编译器编译后的代码，可以看到增强的部分直接被织入到了目标方法前后。

```java
public class SubjectImpl implements Subject {
    public SubjectImpl() {
    }

    public void operation() {
        try {
            SubjectAspect.aspectOf().doBefore();
            System.out.println("operation...");
        } catch (Throwable var2) {
            SubjectAspect.aspectOf().doAfter();
            throw var2;
        }

        SubjectAspect.aspectOf().doAfter();
    }
}
```

> aspectjrt.jar 包的主要作用是提供运行时环境，包括一些注解、静态方法等，在使用 AspectJ 时一般都需要引入。aspectjtools.jar 主要提供的是赫赫有名的 `ajc` 编译器，通常这个包会被封装到 IDE 插件或者自动化构建工具的插件中。aspectjweaver.jar 主要提供了一个 java agent 用于在类加载期间织入切面（LTW）。

# 加载阶段
由于类的加载本质上就是类加载器将从文件、网络或者其他渠道获取的字节流解析成 Class 对象的过程，因此我们只需要通过某种方式修改字节流，就可以实现类的增强。也就是说，在类加载阶段，我们只需要自定义一个类加载器，在类加载器读取字节流之前，利用一些字节码增强工具（比如：ASM、Javassist 等）对类进行增强，最后将增强后的字节流解析为 Class 对象即可。

下面使用 Javassist 简单演示一下在类加载阶段实现类增强的步骤。首先写一个类，添加一个方法 `test`，方便我们后续对该方法进行增强。

```java
public class SomeThing {

    public void test () {
        System.out.println("test...");
    }
}
```

然后写一个方法，该方法接收字节流，内部通过 Javassist 的 API 对字节流进行修饰。

```java
public class EnhanceMethod {

    public static byte[] printCost(byte[] classBytes) throws Exception {
        CtClass ctClass = ClassPool.getDefault().makeClass(new ByteArrayInputStream(classBytes));
        CtMethod ctMethod = ctClass.getMethod("test", "()V");
        ctMethod.addLocalVariable("cost", CtClass.longType);
        ctMethod.insertBefore("cost = System.currentTimeMillis();");
        ctMethod.insertAfter("cost = System.currentTimeMillis() - cost; " +
                "System.out.println(\"total cost: \" + cost);");
        return ctClass.toBytecode();
    }

}
```

接下来自定义一个类加载器，在读到原始类文件的流之后，调用该方法替换流。

```java
public class MyClassLoader extends ClassLoader {

    private String filePath;

    public MyClassLoader(String filePath) {
        this.filePath = filePath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String fileUrl = filePath;
        try {
            Path path = Paths.get(new URI(fileUrl + name + ".class"));
            byte[] bytes = Files.readAllBytes(path);
            bytes = EnhanceMethod.printCost(bytes);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.findClass(name);
    }
}
```

最后就是使用自定义的类加载器加载原始类文件，然后调用对象的 `test` 方法即可。

```java
public class Main {
    public static void main(String[] args) throws Exception {
        String filePath = "file:/d:/code/myAgent/";
        MyClassLoader myClassLoader = new MyClassLoader(filePath);
        Class clazz = myClassLoader.loadClass("SomeThing");
        Object obj = clazz.newInstance();
        Method method = clazz.getMethod("test");
        method.invoke(obj);
    }
}
```

# Instrument
Instrument 是 JDK 5 提供的一个新特性，用一句话来总结它的作用就是：实现了 JVM 级别的 AOP。通过这个特性，开发者可以构建一个独立于应用程序的代理程序，用来监测和协助运行在 JVM 上的应用，甚至能够替换和修改某些类的定义。

## JVMTI
Instrument 的底层实现依赖于 JVMTI（JVM Tool Interface），它是 JVM 暴露出来为了方便用户扩展的接口集合。JVMTI 是基于事件驱动的，具体来说就是，JVM 在执行过程中触发了某些事件就会调用对应事件的回调接口（如果有的话），这些接口可以供开发者去扩展自己的逻辑。

## JVMTIAgent
JVMTIAgent 其实就是一个动态链接库。它利用 JVMTI 暴露出来的接口实现了一些特殊的功能，一般情况下，它会实现如下的一个或者多个接口。

```c
JNIEXPORT jint JNICALL
Agent_OnLoad(JavaVM *vm, char *options, void *reserved);

JNIEXPORT jint JNICALL
Agent_OnAttach(JavaVM* vm, char* options, void* reserved);

JNIEXPORT void JNICALL
Agent_OnUnload(JavaVM *vm);
```

VM 是通过启动函数来启动 agent 的。如果 agent 是在 VM 启动时加载的，也就是说 agent 是在 java 命令中通过 `-agentlib` 指定的，那么 VM 就会在启动过程中去执行这个 agent 里的 Agent_OnLoad 函数来启动该 agent。如果 agent 不是在 VM 启动时加载的，而是在 VM 处于运行过程中时，先 attach 到目标进程上，然后向目标进程发送 load 命令来加载的，此时 VM 会在加载过程中会调用这个 agent 里的 Agent_OnAttach 函数来启动该 agent。而 Agent_OnUnload 函数会在 agent 卸载时被调用，一般很少实现它。

> 这里提到的 agent 程序和 javaagent 不是同一概念。我们在使用 IDE 进行开发时，如果仔细观察，在控制台中会发现类似的命令：`java.exe -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:62290,suspend=y,server=n`，这个动态链接库 jdwp 同样也是一个 JVMTIAgent，它实现了程序调试相关的功能。

## javaagent
javaagent 的功能则是由一个叫做 instrument 的 JVMTIAgent 实现的，它由 JDK 内置提供，在 Linux 下对应的动态库是 `libinstrument.so`，在 Windows 下是 `instrument.dll`。由于它实现了 Agent_OnLoad 和 Agent_OnAttach 函数，因此可以在 JVM 启动时加载，也可以在运行时动态加载。其中，启动时加载还可以通过类似 `-javaagent:agent.jar` 的方式来间接加载 instrument agent。

对于开发人员来说，要使用 javaagent 的功能，只需要编写一个类，然后实现以下方法：

```java
public static void premain(String agentArgs, Instrumentation inst);
```

其中 agentArgs 是 premain 函数得到的程序参数，由 `-javaagent` 指定。inst 是一个 `Instrumentation` 实例，由 JVM 传入。然后还需要再添加一个 `MANIFEST.MF` 文件，并在文件中指定 Premain-Class。

```
Manifest-Version: 1.0
Premain-Class: org.example.AgentApplication
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

在 `premain` 方法中，我们拿到了 inst 实例，这个实例由 JVM 实例化并传入。它提供了很多有用的方法，一般常用的方法有以下几个：

```java
void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
void addTransformer(ClassFileTransformer transformer);
void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;
```