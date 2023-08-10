---
title: Lambda 表达式
date: 2021-07-06 10:47:57
categories: ["technical"]
---

## 前言
一篇关于 Lambda 表达式的阅读笔记。

<!--more-->
> 在 Java 8 中，增加了函数接口、Lambda 和方法引用，使得创建函数对象变得很容易。与此同时，还增加了 Stream API，为处理数据元素的序列提供了类库级支持。在本章中，将讨论如何最佳的利用这些机制。

以上是《Effective Java》第三版第七章的开头。我第一次接触到 Lambda 表达式和方法引用是在我的第一份编程工作，当时这种与普通的 Java 代码格格不入的语法让我一时间难以理解。这篇文章详细介绍了 Lambda 语法、使用情景以及相关延申问题。

## 匿名类与函数式接口

要了解 Lambda，首先我们需要了解一下匿名类。<font color="skyblue">匿名类是指没有类名的内部类</font>，必须在创建时使用 new 语句来声明类。其语法形式如下：

```java
new <类或接口>(){
    //类的主体
};
```

这种形式的 new 语句声明一个新的匿名类，它对一个给定的类进行扩展，<font color="skyblue">通常它用来继承并重写一个类的方法，或者实现接口的方法</font>。在一些情况下，使用匿名类可使代码更加简洁、紧凑，模块化程度更高，因为它不用让你繁琐的去创建抽象方法、接口的实现类。例如：

```java
// 接口
public interface ShowInterface {
    void show();
}

// 测试类
public class Domain {
    public static void main(String[] args) {
        new ShowInterface(){
            @Override
            public void show() {
                System.out.println("我实现了接口的show()方法");
            }
        }.show();
    }
}
```

值得指出的是，上述代码中的`ShowInterface()`接口只含有<font color="skyblue">一个</font>抽象方法，而这种方法从 Java 8 开始，被称作“函数式接口”，并允许使用 Lambda 表达式创建这些接口的实例。

## Lambda 表达式

首先我们来看 RUNOOB.COM 对 Lambda 的介绍：

> Lambda 表达式，也可称为闭包，它是推动 Java 8 发布的最重要新特性。Lambda 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。使用 Lambda 表达式可以使代码变的更加简洁紧凑。

```java
// Lamdba的语法如下：（如果只有单个参数，小括号可以去掉，如果只有单行实现代码，大括号可以去掉）
(parameters) -> experssion;
// 或者
(prarmeters) -> {experssion};
```

按照语法要求，我们上节里的 Domain 测试类则可以转换为以下代码：

```java
// 测试类
public class Domain {
    public static void main(String[] args) {
        // 返回值为void，则小括号内没有值
        ShowInterface showInterface = () -> System.out.println("Lambda表达式实现了接口的Show()方法");
        showInterface.show();
    }
}
```

显而易见的，使用 Lambda 代替匿名类后代码量减少了很多，而一般情况下，Lambda 的类型、参数类型、和返回值类型都是不需要声明的，我们来看这个例子：

```java
// 函数式接口可以通过这个注解来进行声明
@FunctionalInterface
public interface MyIntegerCalculator {
    Integer calcIt(Integer s1);
}

// 测试类
public class Domain {
    public static void main(String[] args) {
        MyIntegerCalculator calculator = s1 -> s1 * 2;
        // 输出18
        System.out.println(calculator.calcIt(9));
    }
}
```

上述例子中的 Lambda 表达式并不用声明，也输出了正确的值，因为编译器利用一个称作类型推导的过程，根据上下文推断出这些类型。在某些情况下，编译器无法确定类型，你就必须指定。《Effective Java》的作者建议：<font color="skyblue">删除所有 Lambda 类型吧，除非它们的存在能够使程序变得更加清晰</font>。如果编译器产生一条错误消息，告诉你无法推导出 Lambda 参数的类型，那么你就指定类型。当然使用 Lambda 也有基本的规则限制：

> <font color="skyblue">Lambda 没有名称和文档，如果一个计算本身不是自描述的，或者超出了几行，那就不要把它放在一个 Lambda 中</font>。对于 Lambda 而言，一行是最理想的，三行是合理的最大极限。如果违背了这个规则，可能对程序的可读性造成严重的危害。如果 Lambda 很长或者难以阅读，要么找一种方法将它简化，要么重构程序来消除它。

同样的，Lambda 也不能完全淘汰匿名类。Lambda 限于函数式接口，在创建抽象类实例，或者拥有多个抽象方法的接口实例时，应该使用匿名类。而且 Lambda 无法获得对自身的引用，在 Lambda 中，关键字 this 是指外围实例，而匿名类则是指向自身。

## 方法引用

> 与匿名类相比，Lambda 的主要优势在于更加简洁。Java 提供了生成比 Lambda 更简洁函数对象的方法：方法引用。

方法引用是通过方法的名字来指向一个方法，语法如下：

```java
// 语法
Class::method

// 举例，这里我们使用《Effective Java》中的map.merge例子
public class Domain {
    public static void main(String[] args) {
        Map<Integer, Integer> map = new HashMap<>();
        map.put(0, 0);
        map.put(1, 1);
        map.put(2, 2);
        map.put(3, 3);
        map.put(4, 4);
        map.put(5, 5);
        map.put(6, 6);
        map.put(7, 7);
        map.put(8, 8);
        map.put(9, 9);
        // Lambda
        map.merge(1, 12, (oldValue, newValue) -> oldValue + newValue);
        // 方法引用
        map.merge(1, 12, Integer::sum);
    }

}
```

_（注：关于 Map.merge 方法的介绍，可以阅读[这篇博客](https://www.cnblogs.com/wy697495/p/10952380.html)）_

对于上述例子中的情况来说，使用方法引用更加合适，也就是 Lambda 表达式下一行的代码。《Effective Java》作者这样说道：

> 这样的代码读起来清晰明了，但仍有些样板代码。参数 oldValue 和 newValue 没有添加太多价值，但是却占用不少空间。实际上，Lambda 要告诉你的就是，该函数返回的是它两个参数的和。——（稍有修改）

我门也可以这样引用静态方法：

```java
List<String> names = new ArrayList();
names.add("Google");
names.add("Runoob");
names.add("Taobao");
names.add("Baidu");
names.add("Sina");
names.forEach(System.out::println);
/*
输出结果：
  Google
  Runoob
  ....
*/
```

不过方法引用并不是在所有情况下都优于 Lambda，例如类名或者方法名太长时，方法引用不一定比 Lambda 表达式更简短；而引用的方法实现非常简单时，方法引用也不一定比 Lambda 表达式更加清晰。

> 总而言之，方法引用常常比 Lambda 表达式更加简洁明了。<font color="skyblue">只要方法引用更加简洁、清晰，就用方法引用；如果方法引用并不简洁，就坚持使用 Lambda</font>。
