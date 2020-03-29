---
title: 为何匿名内部类的参数引用要用 final 修饰
date: 2018/5/29 14:12:0
tags: [Java]
categories: [Java]
---

在 JDK 8 中，我们已经可以不用使用 `final` 修饰了，这是因为引入了 `effectively final` 的概念，实际上还是不能在匿名内部类中修改自由变量的值，Lambda 表达式也一样。上一篇文章“Java 闭包”中已经说明了原因，这里针对 Java 中自由变量的位置详细分析一下。

<!--more-->  

匿名内部类（或 Lambda 表达式）中，引用外部环境中的自由变量，这个自由变量可能是：外部类的成员变量、外部类方法参数或方法体中的局部变量。

## 外部类的成员变量

```java
interface Inner {
    int add();
}

public class Outer {
    public int w = 100;

    public Inner getInner() {

        return new Inner() {
            @Override
            public int add() {
                return w;
            }

            // 这里可以修改外部环境中的变量
            public void setW() {
                w = 12;
            }
        };
    }
}
```

刚刚说过匿名内部类中不能修改外部环境中的自由变量，这么快就打脸了？并不是，查看匿名内部类编译后的代码为：  

```java
class Outer$1 implements Inner {
    Outer$1(Outer this$0) {
        this.this$0 = this$0;
    }

    public int add() {
        return this.this$0.w;
    }

    public void setW() {
        this.this$0.w = 12;
    }
}
```

可以看到，匿名内部类中持有一个外部类的引用，而自由变量 w 又是外部类的成员变量，所以完全可以通过外部类的引用来获取这个变量，然后对这个变量进行任何操作。  

## 外部类的方法参数或局部变量

```java
interface Inner {
    int add();
}

public class Outer {

    public Inner getInner(int x) {
        int y = 100;
        return new Inner() {

            int z = 100;

            @Override
            public int add() {
                return x + y + z;
            }

            // 这里会报错
            // Variable 'x' is accessed from within inner class, 
            // needs to be final or effectively final
            public void setX() {
                x = 12;
            }
        };
    }
}
```

同样，反编译查看编译后的匿名内部类，代码如下：  

```java
class Outer$1 implements Inner {
    int z;

    Outer$1(Outer this$0, int var2, int var3) {
        this.this$0 = this$0;
        this.val$x = var2;
        this.val$y = var3;
        this.z = 100;
    }

    public int add() {
        return this.val$x + this.val$y + this.z;
    }
}
```

可以看到，匿名内部类中持有外部类的一个引用 `this$0`，同时，Java 编译器将匿名内部类依赖的外部环境，具体是外部类方法参数和方法体中的局部变量拷贝了一份传入匿名内部类中。  

我们知道，Java 是值传递的。当值是基本类型时，传递的是值的拷贝；当值是引用类型时，传递的是引用的拷贝，无论你怎么改变这个新的引用的指向，原来的引用的指向不变，如：  

```java
public class Test {

    static class Inner {
        private String name;
        private int number;

        public Inner(String name, int number) {
            this.name = name;
            this.number = number;
        }

        @Override
        public String toString() {
            return "Inner{" +
                    "name='" + name + '\'' +
                    ", number=" + number +
                    '}';
        }
    }

    public static void changeRef(Inner inner) {
        // 引用的拷贝指向的对象和原来的引用指向的对象是同一个
        System.out.println(inner.number = 2333);
        // 改变引用的拷贝的指向，原来的引用指向的对象不会改变
        inner = new Inner("saber", 1);
        System.out.println(inner);
    }

    public static void main(String[] args) {
        Inner inner = new Inner("avalon", 2);
        changeRef(inner);
        System.out.println(inner);
    }
}
```

```
2333
Inner{name='saber', number=1}
Inner{name='avalon', number=2333}
```

回归正题，因为 Java 只实现了值捕获，所以匿名内部类中使用的自由变量是原来的自由变量值的一个副本（基本类型是值的副本，引用类型是引用地址值的副本），修改它们的值并不会影响外部环境中的自由变量，为了让使用者使用起来感觉和引用捕获一样，Java 干脆做了限制：在 JDK 8 以前，必须使用 `final` 修饰，在 JDK 8 以后，可以不用 `final` 修饰，但是变量必须是有效只读的，即 `effectively final` 的。这样大家一看是 `final` 的，就不会去修改它了，即便修改也会编译器报错。即使以后 Java 实现了引用捕获，也不会和已有的代码发生不兼容。  

> 注：虽说匿名内部类中不能修改外部环境中的自由变量的值，但如果自由变量是引用类型的，我们可以修改引用指向的对象的属性。  

## 参考

> [java 为什么匿名内部类的参数引用时 final？](https://www.zhihu.com/question/21395848)