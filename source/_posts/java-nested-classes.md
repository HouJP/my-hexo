---
title: Java Nested Classes
date: 2018-02-12 09:20:29
tags: [java]

---

Java允许我们在类中定义新的类，像这样：

```Java
class OuterClass {
  ...
  class NestedClass {
    ...
  }
}
```

<!-- more -->

嵌套在类中的类，被称为*Nested Classes*。*Nested Classes*被分为两类：

1. static: 声明为`static`的nested classes。
2. non-static: 没有声明为`static`的nested classes，也被称为inner classes。

```Java
class OuterClass {
    ...
    static class StaticNestedClass {
        ...
    }
    class InnerClass {
        ...
    }
}
```

Nested class是包含它的类 (enclosing class) 的成员。Non-static nested classes (inner class) 有权限访问enclosing class的包括私有成员在内的其它成员。而static nested class没有权限访问enclosing class的其他成员。

作为OuterClass的成员，nested class可以被声明为private, public, protected或package private (Outer classes只能声明为public或package private)。

> <font color='red'>TODO</font> 区分protected和package private。 

### Why Use Nested Classes?

使用nested classes的原因如下：

* 类只在一个地方被用到，那么这种方式可以将他们组织在一起。
* 增强了密封性。
* 增强可读性，并且利于维护。

### Static Nested Classes

与类的方法以及变量类似，static nested classes与它的outer class相关联。一个static nested class不能直接获取定义在enclosing class中的实例的变量或者方法，而是只能通过对象引用，这一点与静态类方法相同。

通过enclosing class的名字来获取static nested classes:

```Java
OuterClass.StaticNestedClass
```

创建对象：

```Java
OuterClass.StaticNestedClass nestedObject =
     new OuterClass.StaticNestedClass();
```

### Inner Classes

Inner class与它的enclosing class的实例相关联，并且可以访问对象的方法及字段。也正是因为它与实例相关联，所以不能在其中定义任何的静态成员。

Inner class的实例依托于outer class的实例而存在。例如，对于以下的情况：

```Java
class OuterClass {
    ...
    class InnerClass {
        ...
    }
}
```

`InnerClass`的实例必须依托于`OuterClass`的实例而存在，并且可以直接访问其enclosing instance的方法和字段。。

为了实例化inner class，必须先实例化outer class。然后，通过以下方式来创建inner class对象：

```Java
OuterClass.InnerClass innerObject = outerObject.new InnerClass();
```

有两种特殊的inner classes: 

* local classes
* anonymous classes

> <font color='red'>TODO</font> local classes和anonymous classes。

### Shadowing

如果某个特殊范围(如inner class或者某个方法定义)内的一个类型的声明(如成员变量或者某个参数名)与enclosing scope中的另一个声明有相同的名字，那么enclosing scope中的声明就会被遮盖(shadow)。举例：

```Java
public class ShadowTest {

    public int x = 0;

    class FirstLevel {

        public int x = 1;

        void methodInFirstLevel(int x) {
            System.out.println("x = " + x);
            System.out.println("this.x = " + this.x);
            System.out.println("ShadowTest.this.x = " + ShadowTest.this.x);
        }
    }

    public static void main(String... args) {
        ShadowTest st = new ShadowTest();
        ShadowTest.FirstLevel fl = st.new FirstLevel();
        fl.methodInFirstLevel(23);
    }
}

```

输出：

```Java
x = 23
this.x = 1
ShadowTest.this.x = 0
```

### Serialization

强烈反对序列化inner classes。

### Reference

* [The Java Tutorials: Nested Classes](https://docs.oracle.com/javase/tutorial/java/javaOO/nested.html)