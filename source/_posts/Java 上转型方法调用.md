---
title: Java 上转型方法调用
date: 2017/7/17 11:10:0
tags: [Java]
categories: [Java]
---
Java 的转型分为上转型和下转型，上转型简单概括即将子类对象转成父类对象。比如：

<!--more-->

```java
Animal animal = new Cat();
```

上转型后，不能使用子类新增的成员变量和成员方法。

如果父类和子类存在同名的成员变量，在转型之前，子类对象调用该变量时，调用的就是子类的成员变量；转型之后，调用该变量时，调用的是父类的成员变量。如果子类覆盖了父类的成员方法，在转型之前，子类调用该方法，调用的就是子类的成员方法；在转型之后，调用该方法，调用是子类的成员方法。

```java
public class Animal {

    public void fun1(Animal animal) {
        System.out.println("Animal , Animal");
    }

    public void fun2(Cat cat) {
        System.out.println("Animal , Cat");
    }

    public void fun3(OrangeCat orangeCat) {
        System.out.println("Animal , OrangeCat");
    }
}
```

```java
public class Cat extends Animal {

    public void fun1(Animal animal) {
        System.out.println("Cat , Animal");
    }

    public void fun2(Cat cat) {
        System.out.println("Cat , Cat");
    }
}
```

```java
public class OrangeCat extends Cat {

}
```

```java
public class Main {

    public static void main(String[] args) {
        Animal animal = new Animal();
        Animal ab = new Cat(); //上转型
        Cat cat = new Cat();
        OrangeCat orangeCat = new OrangeCat();

        animal.fun1(animal); // Animal , Animal
        animal.fun1(ab); // Animal , Animal
        animal.fun2(cat); // Animal , Cat
        animal.fun3(orangeCat); // Animal , OrangeCat

        System.out.println("-------------");

        ab.fun1(animal); // Cat , Animal
        ab.fun1(ab); // Cat , Animal
        ab.fun2(cat); // Cat , Cat
        ab.fun3(orangeCat); // Animal , OrangeCat

        System.out.println("-------------");

        cat.fun1(animal); // Cat , Animal
        cat.fun1(ab); // Cat , Animal
        cat.fun2(cat); // Cat , Cat
        cat.fun3(orangeCat); // Animal , OrangeCat
    }

}
```