---
title: Java BitSet
date: 2019/8/28 15:00:0
tags: [Java]
categories: [Java]
---
在 Java 中，官方提供了一个 Bitmap 的简单实现，它就是 `java.util.BitSet`。

<!--more-->

# 构造函数
在 BitSet 中，使用一个 long 类型的数组作为 Bitmap 的数据结构。我们首先查看 BitSet 的两个构造函数。

```java
private final static int ADDRESS_BITS_PER_WORD = 6;
private final static int BITS_PER_WORD = 1 << ADDRESS_BITS_PER_WORD;
private long[] words;

/**
* 无参构造函数
*/
public BitSet() {
    // 1 << 6，结果是 1
    initWords(BITS_PER_WORD);
    // BitSet 的大小是否是用户指定的
    sizeIsSticky = false;
}

/**
* 指定初始化数组的长度（单位 bit） 
*/
public BitSet(int nbits) {
    // nbits can't be negative; size 0 is OK
    if (nbits < 0)
        throw new NegativeArraySizeException("nbits < 0: " + nbits);

    initWords(nbits);
    // BitSet 的大小是否是用户指定的
    sizeIsSticky = true;
}

private void initWords(int nbits) {
    words = new long[wordIndex(nbits-1) + 1];
}

private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
```

第一个构造函数默认会初始化一个长度为 1 的 long 型数组。第二个构造函数会根据指定的长度进行初始化。我们知道，Java 是没有 bit 这个基本类型的，而 long 类型的长度为 64 bit，那么如何将指定长度的 bit 数组转化为 long 类型的数组呢？答案就是将指定的长度除以 64，就可以得到 long 型数组的长度，即代码中的 `bitIndex/64`，换成位运算就是 `bitIndex >> 6`。

# set
```java
public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    int wordIndex = wordIndex(bitIndex);
    expandTo(wordIndex);

    words[wordIndex] |= (1L << bitIndex); // Restores invariants

    checkInvariants();
}

private void expandTo(int wordIndex) {
    // 计算 long 型数组需要的长度
    int wordsRequired = wordIndex+1;
    // 如果已经使用过的 long 元素个数小于需要的长度，则需要扩容
    if (wordsInUse < wordsRequired) {
        ensureCapacity(wordsRequired);
        // 设置已经被使用的数组长度
        wordsInUse = wordsRequired;
    }
}

private void ensureCapacity(int wordsRequired) {
    if (words.length < wordsRequired) {
        // 扩容为需要的长度的两倍
        int request = Math.max(2 * words.length, wordsRequired);
        // 数组复制
        words = Arrays.copyOf(words, request);
        sizeIsSticky = false;
    }
}
```

BitSet 通过 set 方法放入元素。首先根据元素值计算该元素应该放在哪个 long 型数组元素上，即计算 wordIndex。然后根据 wordIndex 判断 Bitmap 是否需要扩容，如果需要扩容，则扩容为需要的长度的两倍。接下来的这段代码 `words[wordIndex] |= (1L << bitIndex);` 比较经典，也不好理解，我们可以先传入几个值来观察结果。

```java
System.out.println(1<<0);
System.out.println(1<<1);
System.out.println(1<<2);
System.out.println(1<<3);
System.out.println(1<<4);
System.out.println(1<<5);
System.out.println(1<<6);
System.out.println(1<<7);

1
2
4
8
16
32
64
128
```

左移运算，相当于 `1 * 2^bitIndex`，我们把它们转换成二进制的形式：

```java
00000001
00000010
00000100
00001000
00010000
00100000
01000000
10000000
```

可以看到，每个 bitIndex 的值都被转换成了对应的 bit 位表示。BitSet 正是通过这种方式，将所有的整数值对应的位设置为 1。接下来就是将当前计算得出的值与原先数组元素的值使用按位或来进行合并，即：`words[wordIndex] |= (1L << bitIndex);`。

# get
```java
public boolean get(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    checkInvariants();

    int wordIndex = wordIndex(bitIndex);
    return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);
}
```

get 方法用来判断一个元素是否在 BitSet 中。首先同样根据元素值计算该元素应该放在哪个 long 型数组元素上，即计算 wordIndex。如果已经使用的 long 型数组长度小于或等于该元素需要放置的数组索引值，则表示该元素一定不在 BitSet 中。接下来的 `1L << bitIndex` 可以计算出该元素在哪个 bit 位上，然后同 long 型数组对应的元素进行与运算，如果结果不等于 0，说明该元素存在。举个例子，假设要查找的元素为 5：

```
1L << bitIndex          00100000
// 如果已经存在元素 5
words[wordIndex]        00100000
// 如果并不存在元素 5（这里存在其它元素：0 1 2 3 4）
words[wordIndex]        00011111
```

可以看到，原数组元素上是否存在其它元素并不影响该元素本身的判断，因为 `1L << bitIndex` 只在指定位置上为 1，其它位置均为 0，进行与运算后，其它位置的结果同样是 0。

# 参考
> [Java 的 BitSet 原理及应用](https://www.jianshu.com/p/4fbad3a6d253)

> [漫画：什么是Bitmap算法？](https://juejin.im/post/5c4fd2af51882525da267385)