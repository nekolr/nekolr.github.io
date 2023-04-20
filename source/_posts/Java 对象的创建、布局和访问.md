---
title: Java 对象的创建、布局和访问
date: 2018/6/9 15:27:0
tags: [Java,JVM]
categories: [JVM]
---

本章主要讨论在 HotSpot 虚拟机的堆内存中，Java 对象的内存分配、内存布局和访问定位的全过程。

<!--more-->

# 对象的创建
虚拟机在遇到一条创建对象的指令（比如 new）时，首先会去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那么会先执行相应的类加载的过程。

在类加载检查通过后，虚拟机将会为新生对象分配内存。对象所需的内存大小在类加载完成之后便可完全确定，为对象分配内存空间的任务相当于把一块确定大小的内存从 Java 堆中划分出来。假如 Java 堆中的内存是绝对规整的，所有的内存都放到一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那么分配内存就仅仅是把指针向空闲空间那边移动一段与对象大小相等的距离。这种分配方式称为**指针碰撞（Bump the Pointer）**。如果堆中的内存不是规整的，那么虚拟机就必须维护一个列表，记录哪些内存块是可用的，在分配内存时从列表中寻找一块足够大的空间划分给对象实例，并更新列表，这种分配方式称为**空闲列表（Free List）**。选择哪种分配方式由 Java 堆是否规整决定，而 Java 堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。因此，在使用 Serial、ParNew 等带有 Compack 功能的收集器时，系统采用的分配算法是指针碰撞；而使用 CMS 这种基于标记-清除（Mark-Sweep）算法的收集器时，通常采用空闲列表。

在虚拟机中创建对象是很频繁的行为，即使是仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的，可能出现正在给对象 A 分配内存，指针还没来得及修改，对象 B 又同时使用了原来的指针来分配内存的情况。解决这个问题有两种方案，一种是对分配内存空间的动作进行同步处理，实际上虚拟机采用的是 CAS 配合失败重试的方式来保证更新操作的原子性。另一种是把内存分配按照线程划分在不同的空间中进行，即每个线程在 Java 堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB）。哪个线程要分配内存，就在哪个线程的 TLAB 上分配，只有在 TLAB 用完并重新分配新的 TLAB 时，才需要同步锁定。虚拟机是否使用 TLAB，可以通过 `-XX:+/-UseTLAB` 参数来设定。

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），如果使用 TLAB 则可以将初始化的操作提前至 TLAB 分配时进行。这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。接下来虚拟机要对对象进行必要的设置，例如这个对象是哪个类的实例，对象的哈希码，对象的 GC 分代年龄等信息，这些信息被存放在对象的对象头（Object Header）中。

在上面的工作都完成后，从虚拟机的角度看，一个新的对象已经产生，但从程序的角度看，对象的创建才刚刚开始，因为 `<init>` 方法还没有执行，所有的字段都为零。一般来说（由字节码中是否跟随 invokespecial 指令决定），执行 new 指令之后会接着执行 `<init>` 方法，这样一个真正可用的对象才算完全生产出来。

下面是 HotSpot 虚拟机的 `bytecodeInterpreter.cpp` 中的代码片段（JDK 6），这个解释器实现很少被实际使用，因为大部分平台都使用模板解释器，当代码通过 JIT 编译器执行时差异就更大了，不过通过它可以了解 HotSpot 的运作过程。

```cpp
u2 index = Bytes::get_Java_u2(pc+1);
constantPoolOop constants = istate->method()->constants();
// 确保常量池中存放的是已经解释的类
if (!constants->tag_at(index).is_unresolved_klass()) {
  // 断言确保是 klassOop 和 instanceKlassOop
  oop entry = (klassOop) *constants->obj_at_addr(index);
  assert(entry->is_klass(), "Should be resolved klass");
  klassOop k_entry = (klassOop) entry;
  assert(k_entry->klass_part()->oop_is_instance(), "Should be instanceKlass");
  instanceKlass* ik = (instanceKlass*) k_entry->klass_part();
  // 确保对象所属类型已经经过初始化阶段
  if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
    // 取对象长度
    size_t obj_size = ik->size_helper();
    oop result = NULL;
    // 记录是否需要将对象所有字段置为零值
    bool need_zero = !ZeroTLAB;
    // 是否在 TLAB 中分配对象
    if (UseTLAB) {
      result = (oop) THREAD->tlab().allocate(obj_size);
    }
    if (result == NULL) {
      need_zero = true;
      // 直接在 Eden 中分配对象
retry:
      HeapWord* compare_to = *Universe::heap()->top_addr();
      HeapWord* new_top = compare_to + obj_size;
      // cmpxchg 是 x86 中的 CAS 指令，这里是一个 C++ 方法，通过 CAS 方式分配空间，并发失败的话，
      // 转到 retry 中重试直至成功分配为止
      if (new_top <= *Universe::heap()->end_addr()) {
        if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
          goto retry;
        }
        result = (oop) compare_to;
      }
    }
    if (result != NULL) {
      // 如果需要，则为对象初始化零值
      if (need_zero ) {
        HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
        obj_size -= sizeof(oopDesc) / oopSize;
        if (obj_size > 0 ) {
          memset(to_zero, 0, obj_size * HeapWordSize);
        }
      }
      // 根据是否启用偏向锁来设置对象头信息
      if (UseBiasedLocking) {
        result->set_mark(ik->prototype_header());
      } else {
        result->set_mark(markOopDesc::prototype());
      }
      result->set_klass_gap(0);
      result->set_klass(k_entry);
      // 将对象引用入栈，继续执行下一条指令
      SET_STACK_OBJECT(result, 0);
      UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
    }
  }
}
```

# 指针压缩
OOPS 指的是普通对象指针（ordinary object pointers），可以认为是对象的引用。这些对象指针的长度通常跟机器的本地指针是一样的，一般 OOPS 在 32 位机上是 32 位长，在 64 位机上是 64 位长。64 位虚拟机出现后，OOPS 的尺寸也变成了 64 位，比之前大了一倍，这就导致对象的内存占用增加（对象的内存布局中有些区域存放的是内存地址值），更重要的是相同尺寸的 CPU Cache 要少存一倍的 OOPS。

从 JDK 1.6 开始，64 位虚拟机正式支持 `-XX:+UseCompressedOops` 参数，该参数可以压缩指针，节约对象的内存占用。还有一个 `-XX:+UseCompressedClassPointers` 参数是用来压缩对象头的类型指针的，在启用指针压缩时默认会开启。

由于 Java 默认是 8 字节对齐的内存，也就是一个对象占用的空间，必须是 8 字节的整数倍，不足的话就会填充到 8 字节的整数倍。因此实际上 Java 对象的内存地址都是 8、16 这种形式，也就是说根本不会定位到之间的比如 1 到 7 这些位置。在实现上，对象的内存地址值还是按照 0x0、0x1、0x2 进行存储，只不过当从堆中读取对象时，JVM 将其左移 3 位，相当于在地址末尾添加 3 个 0。例如内存地址 0x0、0x1、0x2 分别被转换为 0x0、0x8、0x10。这样只需要 32 位的内存地址就可以实际使用 32 GB（4GB x 8）的内存空间。

如果配置的最大堆内存超过 32 GB，那么指针压缩就会失效。当然此时可以通过参数 `-XX:ObjectAlignmentInBytes` 设置更大的字节对齐大小（8 的倍数，不能超过 128）来解决。比如设置为 24 字节，那么最大堆内存超过 96 GB 的情况下压缩指针才会失效。

使用指针压缩虽然可能会增加内存间隙，但是与开启的收益相比，这部分的损失完全可以接受。

# 对象的内存布局
对象在内存中存储的布局分为 3 块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

## 对象头
对象头通常包含两部分信息：Mark Word 和 Klass Word（类型指针）。

Mark Word 用于存储对象自身的运行时数据，如 HashCode 、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳（epoch）等。这部分数据的长度在 32 位和 64 位虚拟机（未开启压缩指针）中分别为 32bit 和 64bit。由于 Mark Word 的信息与对象自身定义的数据无关，考虑到虚拟机的空间效率，它被设计为非固定的数据结构，它会根据对象的状态复用存储空间。

Klass Word，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定该对象是哪个类的实例。并不是所有的虚拟机实现都必须在对象头存储类型指针（Java 程序通过栈上的 reference 数据来操作堆上的具体对象，由于虚拟机规范中只规定了一个指向对象的引用，并没有定义这个引用应该使用何种方式去定位、访问堆中的对象的具体位置，所以对象访问方式取决于虚拟机的实现，主流的访问方式有使用句柄和直接指针两种，而使用句柄的方式不需要在对象头中存储类型指针）。

另外如果对象是一个数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通 Java 对象的元数据信息确定 Java 对象的大小，但是从数组的元数据中却无法确定数组的大小。

![对象头与锁状态](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004171532/2020/04/17/ppa.png)

```
# 32 位虚拟机对象头（未启用指针压缩）

|----------------------------------------------------------------------------------------|--------------------|
|                                    Object Header (64 bits)                             |        State       |
|-------------------------------------------------------|--------------------------------|--------------------|
|                  Mark Word (32 bits)                  |      Klass Word (32 bits)      |                    |
|-------------------------------------------------------|--------------------------------|--------------------|
| identity_hashcode:25 | age:4 | biased_lock:1 | lock:2 |      OOP to metadata object    |       Normal       |
|-------------------------------------------------------|--------------------------------|--------------------|
|  thread:23 | epoch:2 | age:4 | biased_lock:1 | lock:2 |      OOP to metadata object    |       Biased       |
|-------------------------------------------------------|--------------------------------|--------------------|
|               ptr_to_lock_record:30          | lock:2 |      OOP to metadata object    | Lightweight Locked |
|-------------------------------------------------------|--------------------------------|--------------------|
|               ptr_to_heavyweight_monitor:30  | lock:2 |      OOP to metadata object    | Heavyweight Locked |
|-------------------------------------------------------|--------------------------------|--------------------|
|                                              | lock:2 |      OOP to metadata object    |    Marked for GC   |
|-------------------------------------------------------|--------------------------------|--------------------|
```

```
# 64 位虚拟机对象头（未启用指针压缩）

|------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (128 bits)                                        |        State       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                         |    Klass Word (64 bits)     |                    |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                       ptr_to_lock_record:62                         | lock:2 |    OOP to metadata object   | Lightweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor:62                   | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                     | lock:2 |    OOP to metadata object   |    Marked for GC   |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
```

```
# 64 位虚拟机对象头（启用指针压缩）

|--------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (96 bits)                                           |        State       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                           |    Klass Word (32 bits)     |                    |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                         ptr_to_lock_record                            | lock:2 |    OOP to metadata object   | Lightweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor                        | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                       | lock:2 |    OOP to metadata object   |    Marked for GC   |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
```

当对象在无锁状态时，HashCode 会懒初始化，它只会在调用 `System#identityHashCode(object)` 之后被初始化。而在其他状态时，HashCode 将不在 Mark Word 中存储，它会被存储到 monitor 中。GC 分代年龄长度只有四位，因此最大年龄为 15，超过该年龄对象会被转移到老年代。当对象在轻量级锁状态时，`ptr_to_lock_record` 指向锁记录，该记录在当前线程的栈帧中存储，锁记录中存储了锁对象的 Mark Word 的拷贝。线程获取轻量级锁的过程为使用 CAS 将对象的 Mark Word 更新为指向锁记录的指针，更新成功则表示获取到了轻量级锁。当对象升级为重量级锁时，`ptr_to_heavyweight_monitor` 指向 monitor。

## 实例数据
实例数据是对象真正存储有效信息的地方，包括程序代码中定义的各种类型的字段内容，无论是从父类继承的还是在子类中定义的都会记录下来，这部分的存储顺序受到虚拟机分配策略参数（FieldsAllocationStyle）和字段在源码中定义的顺序影响。

一个 Java 对象中的实例数据可能是基本类型或者引用类型。如果是 8 种基本类型，那么存储的就是值，并且大小是固定的；如果是引用类型，那么存储的就是地址值，它的大小与虚拟机位数和是否开启指针压缩有关。

类型 | 大小（字节）| 备注
-|-|-
boolean | 1 | 基本类型
byte | 1 | 基本类型
char | 2 | 基本类型
short | 2 | 基本类型
int | 4 | 基本类型
float | 4 | 基本类型
long | 8 | 基本类型
double | 8 | 基本类型
reference | 4 | 引用类型，64 位虚拟机开启指针压缩

为了验证该约定，并且方便接下来的实践，我们可以引入 OpenJDK 提供的 Java Object Layout 工具。

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

我们首先查看当前环境的一些信息：

```java
public static void main(String[] args) {
    System.out.println(VM.current().details());
}
```

```
# VM mode: 64 bits
# Compressed references (oops): 3-bit shift
# Compressed class pointers: 0-bit shift and 0x800000000 base
# Object alignment: 8 bytes
#                       ref, bool, byte, char, shrt,  int,  flt,  lng,  dbl
# Field sizes:            4,    1,    1,    2,    2,    4,    4,    8,    8
# Array element sizes:    4,    1,    1,    2,    2,    4,    4,    8,    8
# Array base offsets:    16,   16,   16,   16,   16,   16,   16,   16,   16
```

可以看到，当前环境是一个 64 位的虚拟机并且开启了指针压缩，我们使用参数 `-XX:-UseCompressedOops` 来取消指针压缩：

```
# VM mode: 64 bits
# Compressed references (oops): disabled
# Compressed class pointers: disabled
# Object alignment: 8 bytes
#                       ref, bool, byte, char, shrt,  int,  flt,  lng,  dbl
# Field sizes:            8,    1,    1,    2,    2,    4,    4,    8,    8
# Array element sizes:    8,    1,    1,    2,    2,    4,    4,    8,    8
# Array base offsets:    24,   24,   24,   24,   24,   24,   24,   24,   24
```

可以看到，引用类型的大小变为了 8 字节，同时 Array base offsets 也发生了变化。该数据表示的是数组的第一个元素的地址偏移量，在下面会详细解释。

创建一个自定义类 MyClass 并编写测试代码，运行时开启指针压缩：

```java
public class MyClass {
    private boolean bool;
    private short st;
    private int i;
    private byte b;
    private char c;
    private long l;
    private float f;
    private double d;
    private String s;
    private int arr[];
}
```

```java
public static void main(String[] args) {
    MyClass myClass = new MyClass();
    System.out.println(ClassLayout.parseInstance(myClass).toPrintable());
}
```

```
OFF  SZ               TYPE DESCRIPTION               VALUE
  0   8                    (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4                    (object header: class)    0x000d54b0
 12   4                int MyClass.i                 0
 16   8               long MyClass.l                 0
 24   8             double MyClass.d                 0.0
 32   4              float MyClass.f                 0.0
 36   2              short MyClass.st                0
 38   2               char MyClass.c                  
 40   1            boolean MyClass.bool              false
 41   1               byte MyClass.b                 0
 42   2                    (alignment/padding gap)   
 44   4   java.lang.String MyClass.s                 null
 48   4              int[] MyClass.arr               null
 52   4                    (object alignment gap)    
Instance size: 56 bytes
Space losses: 2 bytes internal + 4 bytes external = 6 bytes total
```

可以看到 Mark Word 的大小为 8 字节，Klass Word 的大小为 4 字节，引用的地址值大小为 4 个字节，同时对齐填充浪费了 6 个字节。接下来我们使用 `-XX:-UseCompressedClassPointers` 取消类型指针压缩：

```
OFF  SZ               TYPE DESCRIPTION               VALUE
  0   8                    (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   8                    (object header: class)    0x0000022b4a32d980
 16   8               long MyClass.l                 0
 24   8             double MyClass.d                 0.0
 32   4                int MyClass.i                 0
 36   4              float MyClass.f                 0.0
 40   2              short MyClass.st                0
 42   2               char MyClass.c                  
 44   1            boolean MyClass.bool              false
 45   1               byte MyClass.b                 0
 46   2                    (alignment/padding gap)   
 48   4   java.lang.String MyClass.s                 null
 52   4              int[] MyClass.arr               null
Instance size: 56 bytes
Space losses: 2 bytes internal + 0 bytes external = 2 bytes total
```

可以看到，Klass Word 的大小变为了 8 个字节，但是对齐填充只浪费了 2 个字节。接下来我们直接关闭指针压缩：

```
OFF  SZ               TYPE DESCRIPTION               VALUE
  0   8                    (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   8                    (object header: class)    0x0000019166eb0b08
 16   8               long MyClass.l                 0
 24   8             double MyClass.d                 0.0
 32   4                int MyClass.i                 0
 36   4              float MyClass.f                 0.0
 40   2              short MyClass.st                0
 42   2               char MyClass.c                  
 44   1            boolean MyClass.bool              false
 45   1               byte MyClass.b                 0
 46   2                    (alignment/padding gap)   
 48   8   java.lang.String MyClass.s                 null
 56   8              int[] MyClass.arr               null
Instance size: 64 bytes
Space losses: 2 bytes internal + 0 bytes external = 2 bytes total
```

可以看到，除了 Klass Word 的大小变为了 8 个字节外，引用类型的地址大小也变为了 8 个字节。

我们之前提到 Array base offsets 表示的是数组的第一个元素的地址偏移量，如何理解呢？请看下面的代码，运行时启用指针压缩：

```java
public static void main(String[] args) {
    System.out.println(VM.current().details());
    String[] arr = new String[2];
    System.out.println(ClassLayout.parseInstance(arr).toPrintable());
}
```

```
# VM mode: 64 bits
# Compressed references (oops): 3-bit shift
# Compressed class pointers: 0-bit shift and 0x800000000 base
# Object alignment: 8 bytes
#                       ref, bool, byte, char, shrt,  int,  flt,  lng,  dbl
# Field sizes:            4,    1,    1,    2,    2,    4,    4,    8,    8
# Array element sizes:    4,    1,    1,    2,    2,    4,    4,    8,    8
# Array base offsets:    16,   16,   16,   16,   16,   16,   16,   16,   16

OFF  SZ               TYPE DESCRIPTION               VALUE
  0   8                    (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4                    (object header: class)    0x0001dbc0
 12   4                    (array length)            2
 16   8   java.lang.String String;.<elements>        N/A
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

由于开启了指针压缩，Mark Word 占用 8 字节，Klass Word 占用 4 字节，由于是数组，还有 4 个字节用来存储数组的长度。也就是说，对于该数组而言（数组元素的内存分配是连续的），数组中的第一个元素的内存地址为：该对象的内存地址 + 16 字节。同时，该数组是一个长度为 2 的 String 数组，因此实例数据的大小为 2 个引用类型的地址值占用的大小。如果这里的数组是基本类型数组，那么实例数据的大小就是数组长度乘上基本类型的大小。接下来关闭指针压缩：

```
# VM mode: 64 bits
# Compressed references (oops): disabled
# Compressed class pointers: disabled
# Object alignment: 8 bytes
#                       ref, bool, byte, char, shrt,  int,  flt,  lng,  dbl
# Field sizes:            8,    1,    1,    2,    2,    4,    4,    8,    8
# Array element sizes:    8,    1,    1,    2,    2,    4,    4,    8,    8
# Array base offsets:    24,   24,   24,   24,   24,   24,   24,   24,   24

[Ljava.lang.String; object internals:
OFF  SZ               TYPE DESCRIPTION               VALUE
  0   8                    (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   8                    (object header: class)    0x000001f341b64248
 16   4                    (array length)            2
 20   4                    (alignment/padding gap)   
 24  16   java.lang.String String;.<elements>        N/A
Instance size: 40 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

与开启指针压缩相比，由于 Klass Word 变回了 8 个字节，所以需要对齐填充，可以看到这里对齐填充的部分紧跟在数组长度之后。

## 对齐填充
对齐填充不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。HotSpot 虚拟机自动内存管理系统要求对象的起始地址必须是 8 字节的整数倍，换句话说，就是对象的大小必须是 8 字节的整数倍。从上面的例子中我们也能够发现，对齐填充不光局限于对象的尾部，有时候也会在对象的字段中间插入填充间隙。

# 对象的访问定位
Java 程序需要通过栈上的 reference 数据来操作堆上具体的对象。由于 reference 类型在 Java 虚拟机规范中只规定了一个指向对象的引用，并没有定义这个引用应该通过何种方式去定位和访问，所以对象访问的方式也是取决于虚拟机的实现。目前主流的访问方式有**句柄**和**直接指针**两种，HotSpot 虚拟机采用的是直接指针的方式。

如果使用句柄访问，那么 Java 堆中将会划分出一块内存作为句柄池，reference 中存储的就是对象的句柄地址，句柄中包含了对象实例数据和类型数据各自具体的地址信息。

![句柄](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004171532/2020/04/17/4Vb.png)

如果使用直接指针，那么 Java 堆对象的内存布局中就必须考虑如何放置访问类型数据的相关信息，而 reference 中存储的直接就是对象的内存地址。

![直接指针](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004171532/2020/04/17/qYz.png)

这两种方式各有优势，使用句柄来访问的最大好处就是 reference 中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象非常常见）时只会改变句柄中实例数据的指针，而 reference 本身不需要修改。使用直接指针访问的最大好处就是速度块，它节省了一次指针定位的时间开销。由于对象的访问在 Java 中非常频繁，因此这类开销积少成多后也是一项非常可观的执行成本。

# 参考
> 《深入理解 Java 虚拟机:JVM 高级特性与最佳实践》

> [CompressedOops](https://wiki.openjdk.org/display/HotSpot/CompressedOops)