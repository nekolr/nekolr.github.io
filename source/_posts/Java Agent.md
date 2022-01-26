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
Instrument 是 JDK 5 提供的一个新特性，用一句话来总结它的主要作用就是：实现了 JVM 级别的 AOP。通过这个特性，开发者可以构建一个独立于应用程序的代理程序，用来监测和协助运行在 JVM 上的应用。

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

> 这里提到的 agent 程序和 java agent 不是同一概念。我们在使用 IDE 进行开发时，如果仔细观察，在控制台中会发现类似的命令：`java.exe -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:62290,suspend=y,server=n`，这个动态链接库 jdwp 同样也是一个 JVMTIAgent，它实现了程序调试相关的功能。

## java agent
java agent 的功能则是由一个叫做 instrument 的 JVMTIAgent 实现的，它由 JDK 内置提供，在 Linux 下对应的动态库是 `libinstrument.so`，在 Windows 下是 `instrument.dll`。由于它实现了 Agent_OnLoad 和 Agent_OnAttach 函数，因此可以在 JVM 启动时加载，也可以在运行时动态加载。其中，启动时加载还可以通过类似 `-javaagent:agent.jar` 的方式来间接加载 instrument agent。

对于开发人员来说，如果希望 agent 在目标 JVM 启动时加载，只需要编写一个类，然后实现以下方法：

```java
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);
```

如果希望目标 JVM 在运行时加载 agent，则需要实现以下方法：

```java
public static void agentmain(String agentArgs, Instrumentation inst);
public static void agentmain(String agentArgs);
```

上述方法中，JVM 会先寻找对应的第一个方法，如果没有找到则会去寻找对应的第二个方法。其中 agentArgs 是 premain 函数或 agentmain 函数得到的程序参数，由 `-javaagent` 指定。inst 是一个 `Instrumentation` 实例，由 JVM 传入，我们可以通过该参数进行类增强等操作。

接下来需要将这个 agent 打包成一个 jar 文件，同时 jar 文件中还要包含一个 `MANIFEST.MF` 描述文件，文件中需要指定 Premain-Class 或 Agent-Class 属性。

```
Manifest-Version: 1.0
Premain-Class: org.example.AgentApplication
Agent-Class: org.example.AgentApplication
```

## 从源代码解析启动时加载
在创建 JVM 时，JVM 会进行启动参数的解析，我们这里重点关注 `-agentlib`、`-agentpath` 和 `-javaagent`。

```cpp
// -agentlib and -agentpath
} else if (match_option(option, "-agentlib:", &tail) ||
      (is_absolute_path = match_option(option, "-agentpath:", &tail))) {
  if(tail != NULL) {
    const char* pos = strchr(tail, '=');
    size_t len = (pos == NULL) ? strlen(tail) : pos - tail;
    char* name = strncpy(NEW_C_HEAP_ARRAY(char, len + 1, mtInternal), tail, len);
    name[len] = '\0';

    char *options = NULL;
    if(pos != NULL) {
      options = strcpy(NEW_C_HEAP_ARRAY(char, strlen(pos + 1) + 1, mtInternal), pos + 1);
    }
#if !INCLUDE_JVMTI
    if (valid_hprof_or_jdwp_agent(name, is_absolute_path)) {
      jio_fprintf(defaultStream::error_stream(),
        "Profiling and debugging agents are not supported in this VM\n");
      return JNI_ERR;
    }
#endif // !INCLUDE_JVMTI
    add_init_agent(name, options, is_absolute_path);
  }
// -javaagent
} else if (match_option(option, "-javaagent:", &tail)) {
#if !INCLUDE_JVMTI
  jio_fprintf(defaultStream::error_stream(),
    "Instrumentation agents are not supported in this VM\n");
  return JNI_ERR;
#else
  if(tail != NULL) {
    char *options = strcpy(NEW_C_HEAP_ARRAY(char, strlen(tail) + 1, mtInternal), tail);
    add_init_agent("instrument", options, false);
  }
#endif // !INCLUDE_JVMTI
// -Xnoclassgc
}
```

> ref: hotspot/src/share/vm/runtime/arguments.cpp

```cpp
// -agentlib and -agentpath arguments
static AgentLibraryList _agentList;
static void add_init_agent(const char* name, char* options, bool absolute_path)
  { _agentList.add(new AgentLibrary(name, options, absolute_path, NULL)); }
```

> ref: hotspot/src/share/vm/runtime/arguments.hpp

这里我们只需要关注 `add_init_agent` 方法，该方法将解析好的参数放入了一个 `AgentLibraryList` 类型的链表中，以备后续使用。

接下来我们回到创建 JVM 的方法，在这个方法中，我省略了部分代码，重点关注解析参数后的加载过程即可。

```cpp
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  // ...
  // 解析参数
  jint parse_result = Arguments::parse(args);
  if (parse_result != JNI_OK) return parse_result;
  // ...
  // Launch -agentlib/-agentpath and converted -Xrun agents
  if (Arguments::init_agents_at_startup()) {
    create_vm_init_agents();
  }
  // ...
}
```

```cpp
void Threads::create_vm_init_agents() {
  extern struct JavaVM_ main_vm;
  AgentLibrary* agent;
  JvmtiExport::enter_onload_phase();
  // agents 方法从 _agentList 链表中取出一个元素
  for (agent = Arguments::agents(); agent != NULL; agent = agent->next()) {
    OnLoadEntry_t  on_load_entry = lookup_agent_on_load(agent);
    if (on_load_entry != NULL) {
      // 调用 Agent_OnLoad 方法
      jint err = (*on_load_entry)(&main_vm, agent->options(), NULL);
      if (err != JNI_OK) {
        vm_exit_during_initialization("agent library failed to init", agent->name());
      }
    } else {
      vm_exit_during_initialization("Could not find Agent_OnLoad function in the agent library", agent->name());
    }
  }
  JvmtiExport::enter_primordial_phase();
}
```

```cpp
// Find the Agent_OnLoad entry point
static OnLoadEntry_t lookup_agent_on_load(AgentLibrary* agent) {
  // Agent_OnLoad
  const char *on_load_symbols[] = AGENT_ONLOAD_SYMBOLS;
  return lookup_on_load(agent, on_load_symbols, sizeof(on_load_symbols) / sizeof(char*));
}
```

> ref: hotspot/src/share/vm/runtime/thread.cpp

可以看到，在加载过程中，又调用了 `lookup_agent_on_load` 方法，该方法的主要作用是加载 agent 对应的动态链接文件。我们回忆刚才分析的代码，也就是说，当指定了 `-javaagent` 参数时，这里会加载 `instrument` 这个动态链接文件，最终还会调用它的 Agent_OnLoad 方法，因此，我们接下来要分析 Agent_OnLoad 方法。

```cpp
JNIEXPORT jint JNICALL
Agent_OnLoad(JavaVM *vm, char *tail, void * reserved) {
    initerror = createNewJPLISAgent(vm, &agent);
    if ( initerror == JPLIS_INIT_ERROR_NONE ) {
        /*
        * Parse <jarfile>[=options] into jarfile and options
        */
        if (parseArgumentTail(tail, &jarfile, &options) != 0) {
          fprintf(stderr, "-javaagent: memory allocation failure.\n");
          return JNI_ERR;
        }
        attributes = readAttributes(jarfile);
        premainClass = getAttribute(attributes, "Premain-Class");
        /*
        * Convert JAR attributes into agent capabilities
        */
        convertCapabilityAtrributes(attributes, agent);
        /*
        * Track (record) the agent class name and options data
        */
        initerror = recordCommandLineData(agent, premainClass, options);
    }
}
```

> ref: jdk/src/share/instrument/InvocationAdapter.c

以上是精简后的代码，大概的流程就是：先创建一个 JPLISAgent，然后将 ManiFest 文件中设定的一些参数解析出来， 比如 Premain-Class 等。在创建了 JPLISAgent 之后，还会调用 initializeJPLISAgent 方法对这个 Agent 进行初始化操作。

```cpp
JPLISInitializationError
initializeJPLISAgent(   JPLISAgent *    agent,
                        JavaVM *        vm,
                        jvmtiEnv *      jvmtienv) {
    /* now turn on the VMInit event */
    if ( jvmtierror == JVMTI_ERROR_NONE ) {
        jvmtiEventCallbacks callbacks;
        memset(&callbacks, 0, sizeof(callbacks));
        callbacks.VMInit = &eventHandlerVMInit;

        jvmtierror = (*jvmtienv)->SetEventCallbacks( jvmtienv,
                                                     &callbacks,
                                                     sizeof(callbacks));
        check_phase_ret_blob(jvmtierror, JPLIS_INIT_ERROR_FAILURE);
        jplis_assert(jvmtierror == JVMTI_ERROR_NONE);
    }
}
```

在该方法中，我们只关注 `callbacks.VMInit = &eventHandlerVMInit;` 这行代码，这里设置了一个 VMInit 事件的回调函数，表示**在 JVM 初始化的时候**会回调 eventHandlerVMInit 函数。

```cpp
void JNICALL
eventHandlerVMInit( jvmtiEnv *      jvmtienv,
                    JNIEnv *        jnienv,
                    jthread         thread) {
    /* process the premain calls on the all the JPL agents */
    if ( environment != NULL ) {
        jthrowable outstandingException = preserveThrowable(jnienv);
        success = processJavaStart( environment->mAgent,
                                    jnienv);
        restoreThrowable(jnienv, outstandingException);
    }
}
```

```cpp
jboolean
processJavaStart(   JPLISAgent *    agent,
                    JNIEnv *        jnienv) {
    jboolean    result;
    result = initializeFallbackError(jnienv);
    if ( result ) {
        // 创建一个 sun.instrument.InstrumentationImpl 实例
        result = createInstrumentationImpl(jnienv, agent);
        jplis_assert(result);
    }
    // 关闭 VMInit handler 同时设置 ClassFileLoadHook 事件的回调函数
    if ( result ) {
        result = setLivePhaseEventHandlers(agent);
        jplis_assert(result);
    }
    // 加载 java agent 同时调用它的 premain 方法
    if ( result ) {
        result = startJavaAgent(agent, jnienv,
                                agent->mAgentClassName, agent->mOptionsString,
                                agent->mPremainCaller);
    }
    return result;
}
```

可以看到，这个 VMInit 事件发生时，虚拟机的主要操作是实例化了一个 `sun.instrument.InstrumentationImpl`，然后设置了 ClassFileLoadHook 事件的回调函数，最后加载 java agent 同时调用它的 premain 方法。其中，InstrumentationImpl 是 `java.lang.instrument.Instrumentation` 接口的实现类。此时我们很容易就会想到 premain 方法中的 inst 参数，没错，在调用 premain 方法时，由虚拟机传入的 inst 参数就是它。

## 从源代码解析运行时加载
与启动时加载 Agent 相比，运行时加载 Agent 显得更有吸引力，因为运行时加载 Agent 给我们提供了很强的动态性，我们可以在需要的时候加载 Agent 来进行一些工作。`tools.jar` 中提供了一个 `com.sun.tools.attach.VirtualMachine` 类，通过它可以实现虚拟机在运行时动态加载 agent。以下代码来自美团技术博客。

```java
private void attachAgentToTargetJVM() throws Exception {
    List<VirtualMachineDescriptor> virtualMachineDescriptors = VirtualMachine.list();
    VirtualMachineDescriptor targetVM = null;
    for (VirtualMachineDescriptor descriptor : virtualMachineDescriptors) {
        if (descriptor.id().equals(configure.getPid())) {
            targetVM = descriptor;
            break;
        }
    }
    if (targetVM == null) {
        throw new IllegalArgumentException("could not find the target jvm by process id:" 
        + configure.getPid());
    }
    VirtualMachine virtualMachine = null;
    try {
        virtualMachine = VirtualMachine.attach(targetVM);
        virtualMachine.loadAgent("{agent}", "{params}");
    } catch (Exception e) {
        if (virtualMachine != null) {
            virtualMachine.detach();
        }
    }
}
```

接下来我们回到创建虚拟机的过程，在上面我们分析过了这个过程的部分操作，其中我们忽略了一个操作：

```cpp
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  // ...
  os::signal_init();
  // ...
}
```

```cpp
void os::signal_init() {
  if (!ReduceSignalUsage) {
    // ...
    const char thread_name[] = "Signal Dispatcher";
    Handle string = java_lang_String::create_from_str(thread_name, CHECK);
    // ...
    { MutexLocker mu(Threads_lock);
      JavaThread* signal_thread = new JavaThread(&signal_thread_entry);
      // ...
    }
    // Handle ^BREAK
    os::signal(SIGBREAK, os::user_handler());
  }
}
```

这个方法创建了一个名为：Signal Dispatcher 的线程，这个线程的入口方法为：signal_thread_entry。

```cpp
static void signal_thread_entry(JavaThread* thread, TRAPS) {
  while (true) {
    int sig;
    {
      sig = os::signal_wait();
    }
    if (sig == os::sigexitnum_pd()) {
       // Terminate the signal thread
       return;
    }
    switch (sig) {
      case SIGBREAK: {
        if (!DisableAttachMechanism && AttachListener::is_init_trigger()) {
          continue;
        }
        // ...
      }
    }
  }
}
```

在这个方法中，如果 Signal Dispatcher 线程接收到 `SIGBREAK` 信号时，就执行接下来的代码。

```cpp
bool AttachListener::is_init_trigger() {
  if (init_at_startup() || is_initialized()) {
    return false;
  }
  char fn[PATH_MAX+1];
  sprintf(fn, ".attach_pid%d", os::current_process_id());
  int ret;
  struct stat64 st;
  RESTARTABLE(::stat64(fn, &st), ret);
  if (ret == -1) {
    snprintf(fn, sizeof(fn), "%s/.attach_pid%d",
             os::get_temp_directory(), os::current_process_id());
    RESTARTABLE(::stat64(fn, &st), ret);
  }
  if (ret == 0) {
    // simple check to avoid starting the attach mechanism when
    // a bogus user creates the file
    if (st.st_uid == geteuid()) {
      init();
      return true;
    }
  }
  return false;
}
```

> ref: hotspot\src\os\linux\vm\attachListener_linux.cpp

上面这部分代码是 linux 平台下的实现，可以看到，在该方法中会先检查 JVM 是否已经启动了 Attach Listener，如果没有，会在 `/tmp` 目录下创建一个叫做 `.attach_pid{pid}` 的文件，然后执行 AttachListener 的 init 函数。

```cpp
void AttachListener::init() {
  // ...
  const char thread_name[] = "Attach Listener";
  Handle string = java_lang_String::create_from_str(thread_name, CHECK);

  { MutexLocker mu(Threads_lock);
    JavaThread* listener_thread = new JavaThread(&attach_listener_thread_entry);
    // ...
  }
}
```

与 Signal Dispatcher 线程类似，这里也创建了一个线程：Attach Listener。

```cpp
static void attach_listener_thread_entry(JavaThread* thread, TRAPS) {
  for (;;) {
    AttachOperation* op = AttachListener::dequeue();
    if (op == NULL) {
      return;
    }
    if (strcmp(op->name(), AttachOperation::detachall_operation_name()) == 0) {
      AttachListener::detachall();
    } else {
      // find the function to dispatch too
      AttachOperationFunctionInfo* info = NULL;
      for (int i=0; funcs[i].name != NULL; i++) {
        const char* name = funcs[i].name;
        assert(strlen(name) <= AttachOperation::name_length_max, "operation <= name_length_max");
        if (strcmp(op->name(), name) == 0) {
          info = &(funcs[i]);
          break;
        }
      }
      // ...
      if (info != NULL) {
        // dispatch to the function that implements this operation
        res = (info->func)(op, &st);
      } else {
        st.print("Operation %s not recognized!", op->name());
        res = JNI_ERR;
      }
    }
  }
}
```

在这个入口函数中，首先通过 dequeue 方法拉取操作，这个拉取的方法在不同的平台有不同的实现，在 linux 下，Attach Listener 线程会监听某个端口，通过 accept 方法来接收一个连接，然后从连接中读取请求并封装成一个 AttachOperation 类型的对象，然后到对应的操作列表中去匹配，最后执行相应的函数。以下是这个操作列表：

```cpp
static AttachOperationFunctionInfo funcs[] = {
  { "agentProperties",  get_agent_properties },
  { "datadump",         data_dump },
  { "dumpheap",         dump_heap },
  { "load",             JvmtiExport::load_agent_library },
  { "properties",       get_system_properties },
  { "threaddump",       thread_dump },
  { "inspectheap",      heap_inspection },
  { "setflag",          set_flag },
  { "printflag",        print_flag },
  { "jcmd",             jcmd },
  { NULL,               NULL }
};
```

经过上面的分析，我们应该能够隐约知道 VirtualMachine 的 attach 方法的大概逻辑，下面通过源代码来验证一下。

```java
public static VirtualMachine attach(String var0) throws AttachNotSupportedException, IOException {
    if (var0 == null) {
        throw new NullPointerException("id cannot be null");
    } else {
        List var1 = AttachProvider.providers();
        if (var1.size() == 0) {
            throw new AttachNotSupportedException("no providers installed");
        } else {
            AttachNotSupportedException var2 = null;
            Iterator var3 = var1.iterator();
            while(var3.hasNext()) {
                AttachProvider var4 = (AttachProvider)var3.next();
                try {
                    return var4.attachVirtualMachine(var0);
                } catch (AttachNotSupportedException var6) {
                    var2 = var6;
                }
            }
            throw var2;
        }
    }
}
```

```java
public VirtualMachine attachVirtualMachine(String var1) throws AttachNotSupportedException, IOException {
    this.checkAttachPermission();
    this.testAttachable(var1);
    return new LinuxVirtualMachine(this, var1);
}
```

```java
LinuxVirtualMachine(AttachProvider var1, String var2) throws AttachNotSupportedException, IOException {
    super(var1, var2);
    int var3;
    try {
        var3 = Integer.parseInt(var2);
    } catch (NumberFormatException var25) {
        throw new AttachNotSupportedException("Invalid process identifier");
    }
    this.path = this.findSocketFile(var3);
    if (this.path == null) {
        File var4 = this.createAttachFile(var3);
        try {
            int var5;
            if (isLinuxThreads) {
                try {
                    var5 = getLinuxThreadsManager(var3);
                } catch (IOException var24) {
                    throw new AttachNotSupportedException(var24.getMessage());
                }
                assert var5 >= 1;
                sendQuitToChildrenOf(var5);
            } else {
                sendQuitTo(var3);
            }
            var5 = 0;
            long var6 = 200L;
            int var8 = (int)(this.attachTimeout() / var6);
            do {
                try {
                    Thread.sleep(var6);
                } catch (InterruptedException var23) {
                }
                this.path = this.findSocketFile(var3);
                ++var5;
            } while(var5 <= var8 && this.path == null);
            if (this.path == null) {
                throw new AttachNotSupportedException("Unable to open socket file: target process not responding or HotSpot VM not loaded");
            }
        } finally {
            var4.delete();
        }
    }
    checkPermissions(this.path);
    int var27 = socket();
    try {
        connect(var27, this.path);
    } finally {
        close(var27);
    }
}
```

```java
private String findSocketFile(int var1) {
    File var2 = new File("/tmp", ".java_pid" + var1);
    return !var2.exists() ? null : var2.getPath();
}
```

attachVirtualMachine 方法在不同的平台有不同的实现，上面的代码是 linux 平台下的实现。大体逻辑是，首先检查 `/tmp` 目录下是否存在 `java_pid{pid}` 文件。如果已经存在了，则说明 Attach 机制已经准备就绪，可以接受客户端的命令了，这个时候客户端就可以通过 connect 方法连接到目标 JVM 进行命令的发送。如果 `java_pid{pid}` 文件还不存在，则通过 sendQuitTo 方法向目标 JVM 发送一个 `SIGBREAK` 信号，让它初始化 Attach Listener 线程并准备接受客户端连接。可以看到，发送了信号之后客户端会循环等待 `java_pid{pid}` 这个文件，之后再通过 connect 连接到目标 JVM 上。

## instrument 实例
在上面我们分析了 agent 技术的实现，在实际使用中，我们只需要编写 premain 或者 agentmain 方法，然后在其中通过 Instrument API 来完成类的动态修改即可。Instrument 接口的 addTransformer 方法可以添加一个类转换器（也就是 ClassFileTransformer 接口），该接口只有一个方法：transform，当类被加载时，虚拟机就会调用它进行类的转换。

下面我们通过 Byte Buddy 这个开源库来写一个简单的实例，这个 java agent 能够实现打印指定包中所有方法的执行耗时。

```java
public class MyAgent {

    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("this is a java agent");
        System.out.println("arguments: " + agentArgs);

        AgentBuilder.Transformer transformer = (builder, typeDescription, classLoader, javaModule) -> builder
                // 拦截所有方法
                .method(ElementMatchers.any())
                // 指定拦截器
                .intercept(MethodDelegation.to(ExecuteTimeInterceptor.class));

        new AgentBuilder
                .Default()
                // 根据包名前缀拦截类
                .type(ElementMatchers.nameStartsWith("org.example.agent.demo"))
                .transform(transformer)
                .installOn(inst);
    }
}
```

```java
public class ExecuteTimeInterceptor {
    @RuntimeType
    public static Object intercept(@Origin Method method, @SuperCall Callable<?> callable) throws Exception {
        long start = System.currentTimeMillis();
        try {
            // 执行原始方法
            return callable.call();
        } finally {
            System.out.println(method.getName() + ":" + (System.currentTimeMillis() - start) + " ms");
        }
    }
}
```

与加载时不同，运行时需要通过 redefineClasses 方法进行类的重定义，同时使用该方法不能添加、删除或者重命名字段和方法，也不能修改方法的签名或者类的继承关系。

# 参考
> [Java 动态调试技术原理及实践](https://tech.meituan.com/2019/11/07/java-dynamic-debugging-technology.html)