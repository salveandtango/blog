---
title: Java 的静态工厂
keywords: Effective Java, 创建和销毁对象, 用静态工厂方法代替构造器, 构建器
categories: ["technical"]
---

## 前言

手上这本《Effective Java》第三版是 21 年 6 月中旬购买的，到这篇文章开始写时，大致只看了五分之一。有很多因素影响了读书的进度，长久下去即使读完全书也多多少少会忘记前几章的内容。于是决定每读完一个章节就做一篇文章总结，复习时用。

_（注：以下内容多为摘抄、总结）_

## 用静态工厂方法代替构造器

### 优势

- 方法有名称
- 不必在每次调用他们的时候都创建一个新对象
- [可以返回原返回类型的任何子类型的对象](#good_3)
- [所返回的对象的类可以随着每次调用而发生变化，这取决于静态工厂方法的参数值](#good_4)
- [方法返回的对象所属的类，在编写包含该静态工厂方法的类时可以不存在](#good_5)

前两点不再赘述，我们来讨论其他优点。

#### <span id="good_3">可以返回原返回类型的任何子类型的对象</span>

使用静态工厂方法，可以在选择返回对象的类时有更大的灵活性。例如它可以使 API 返回对象，但是又不会使对象的类变为公有的。以该形式隐藏实现类会使 API 变得非常简洁。

类似的还有 Java Collection Framework _（一个基于接口的框架）_ 将所有实现都通过静态工厂方法在同一个不可实例化的类中导出，所有返回对象的类都是非公有的。

使用静态工厂不仅仅可以让 API 数量上减少，也是概念意义上的减少：为了使用这个 API，用户必须掌握的概念在数量上和难度上也减少了。甚至可以要求客户端通过接口而不是实现类来引用被返回的对象，这是一种良好的习惯。

#### <span id="good_4">所返回的对象的类可以随着每次调用而发生变化</span>

只要是已声明的返回类型的子类型，都是允许的。返回对象的类也可能随着发行版本的不同而不同。例如 EnumSet 中的 noneOf()方法，根据输入的参数返回不同的子类：

```java
/**
* Creates an empty enum set with the specified element type.
*
* @param <E> The class of the elements in the set
* @param elementType the class object of the element type for this enum
*     set
* @return An empty enum set of the specified type.
* @throws NullPointerException if {@code elementType} is null
*/
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
  Enum<?>[] universe = getUniverse(elementType);
  if (universe == null)
      throw new ClassCastException(elementType + " not an enum");

  if (universe.length <= 64)
      return new RegularEnumSet<>(elementType, universe);
  else
      return new JumboEnumSet<>(elementType, universe);
}
```

#### <span id="good_5">方法返回的对象所属的类，在编写包含该静态工厂方法的类时可以不存在</span>

这种灵活的静态工厂方法构成了服务提供者框架 （Service Provider Framework），它包括了以下 4 个组件：

- 服务接口 _（由提供者实现）_
- 提供者注册 API _（提供者用于注册实现）_
- 服务访问 API / Service Access API _（客户端用于获取服务实例）_
- 服务提供者接口 / Service Provider Interface _（表示产生服务接口之实例的工厂对象）_

其中，服务提供者接口是可选的，如果没有服务提供者接口，实现就通过反射方式进行实例化。对于 JDBC 来说，Connection 是其服务接口的一部分，DriverManager.registerDriver 是提供者注册 API，DriverManager.getConnection 是服务访问 API，Driver 是服务提供者接口。

服务提供者框架的具体描述和示范实例可以在 [这里](https://cloud.tencent.com/developer/article/1143378) 了解。

### 缺点

- 类如果不含公有的或受保护的构造器，就不能被子类化
- 程序员很难发现他们

### 总结

总而言之，静态工厂方法和共有构造器都各有用处，而静态常常更加合适。

## 有多个构造器参数时需要考虑使用构建器（Builder）

静态工厂和构造器有个共同的局限性：<font color="skyblue">它们都不能很好地扩展到大量的可选参数。</font>或许重叠构构造器可行，但是有许多参数的时候，客户端代码会很难编写，而 Java Bean 在构造过程中可能会出现不一致状态，这就需要程序员付出额外努力来确保它的线程安全。

<font color="skyblue">不过 Builder 解决了问题，它既能保证像重叠构造器模式那样的安全性，也有 JavaBean 模式那么好的可读性。</font>它不直接生成想要的对象，而是让客户端利用所有必要的参数调用构造器或者静态工厂，得到一个 builder 对象。然后客户端在 builder 上调用类似于 setter 的方法，来设置每个相关的可选参数。最后客户端调用无参的 build 方法来生成通常是不可变的对象。示例：

```java
pbulic class NutritionFacts {
  private final int servingSize;
  private final int servings;
  private final int calories;
  private final int fat;
  private final int sodium;
  private final int carbohydrate;

  public static class Builder {
    // Required parameters
    private final int servingSize;
    private final int servings;

    // Optional parameters - initialized to defalut values
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;

    public Builder(int servingSize, int servings) {
      this.servingSize = servingSize;
      this.servings = servings;
    }

    public Builder calories(int val) {
      calories = val;
      return this;
    }

    public Builder fat(int val) {
      fat = val;
      return this;
    }

    public Builder sodium(int val) {
      ...
    }

    public Builder carbohydrate(int val) {
      ...
    }

    public NutritionFacts build() {
      return new NutritionFacts(this);
    }
  }

  private NutritionFacts(Builder builder) {
    servingSize = builder.servingSize;
    servings = builder.servings;
    calories = builder.calories;
    ...
  }
}
```

客户端代码：

```java
NutritionFacts cocaCloa = new NutritionFacts.Builder(240, 8)
                              .calories(100).sodium(35).carbohydrate(27).build();
```

当然也可以使用 [Lombok](https://projectlombok.org/) 的 *@Builder* 注解，示例如下：

```java
package effictiveJava.builder;

import lombok.Builder;
import lombok.ToString;

/**
 * @author Fixed
 * @Description 实体类
 * @Date 2021/6/30 11:57
 */
@Builder
@ToString
public class Entity {
    private String name;
    private String lastName;
    private Integer age;
    private Integer gender;
    private Integer grade;
    private Integer score;
}
```

```java
package effictiveJava.builder;

/**
 * @author Fixed
 * @Description 实体类测试
 * @Date 2021/6/30 12:00
 */
public class Domain {
    public static void main(String[] args) {
        Entity entity = Entity.builder().age(23).gender(1).lastName("Fixed").build();
        System.out.println(entity.toString());
        // Entity(name=null, lastName=Fixed, age=23, gender=1, grade=null, score=null)
    }
}
```

与构造器相比，builder 的微略优势在于，它可以有多个可变（varargs）参数。因为 builder 是利用单独的方法来设置每一个参数。此外，构建器还可以将多次调用某一个方法而传入的参数集中到一个域中。与 Java Bean 模式相比，也更加安全。

但是它的缺点在于：为了创建对象，它必须先创建它的构建器，可能会略微损失一些开销。并且 Builder 模式比重叠构造器还冗长，因此它只有在很多参数的时候才使用，比如 4 个以上的参数。但是如果这个类将来需要添加参数，那么在一开始就使用 Builder 模式会更好。

简而言之，如果类的构造器或者静态工厂中具有多个参数，设计这种类时，Builder 模式就是一种不错的选择，特别是当大多数参数都是可选或者类型相同的时候。

## 避免创建不必要的对象

### 重用对象

一般来说，最好能重用单个对象，而不是在每次需要的时候就创建一个相同功能的新对象。重用方式既快速，又流行。如果对象是不可变的（immutable），他就始终可以被重用。举例：

```java
// 该语句每次被执行的时候都创建一个新的String实例
String s = new String("s");

// 正确例子
String s = "s";
```

遗憾的是，在创建重复使用的对象时，并非总是那么显而易见。下列代码用于判断一个字符串是否为有效的罗马数字：

```java
static boolean isRomanNumeral(String s) {
  return s.matches("....");//正则表达式
}
```

问题在于它依赖了 String.matches 方法，它不适合在注重性能的情形中重复使用。问题在于：它创建了一个 Pattern 实例，却只用了一次，之后就可以进入垃圾回收了。创建 Pattern 实例的成本很高，为了提升性能，应该显示的将正则表达式编译成一个 Pattern 示例，让它成为类初始化的一部分，并让它缓存起来，每当调用 isRomanNumeral 方法时就重用同一个实例：

```java
public class RomanNumerals {
  private static final Pattern ROMAN = Pattern.compile("...");//正则表达式

  static boolean isRomanNumeral(String s) {
    return ROMAN.matcher(s).matches();
  }
}
```

### 当心无意识的装箱

另一种创建多余对象的方法，称作自动装箱（autoboxing），它允许程序员将基本类型和装箱基本类型（Boxed Primitive Type）混用，按需要自动装箱和拆箱。自动装箱使得基本类型和装箱基本类型之间的差别变得模糊起来，但是并没有完全消除。它们在语义上还有着微妙的差别，在性能上也有着比较明显的差别。

```java
private static long sum(){
  Long sum = 0L;
  for (long i = 0; i<= Integer.Max_VALUE; i++){
    sum += i;
  }
  return sum;
}
```

上述程序计算出的答案是正确的，但是因为将声明 sum 的 long 写成了 Long，程序构造了大约 2 的 31 次方个多余的 Long 实例。

## 消除过期的对象引用

Java 中虽然有垃圾回收机制，但是在部分情况下，还是有可能会发生内存泄漏。例如一个栈先增长，再收缩，那么从栈中弹出来的对象将不会被当作垃圾回收，即使使用栈的程序不再引用这些对象，他们也不会被回收。这是因为栈内部维护者队这些对象的过期引用（obsolete refreence）。所谓过期引用，是指永远也不会在被接触的引用。解决的办法很简单：一旦对象引用已经过期，只需清空这些引用即可。

那么何时应该清空引用呢？<font color="skyblue">一般来说，只要类是自己管理内存，程序员就应该警惕内存泄露问题。</font>一旦元素被释放掉，则该元素中包含的任何对象引用都应该被清空。内存泄漏的另外两个常见来源是缓存和 API 回调，合理使用 WeakHashMap 可以有效解决这些问题。

## try-with-resources 优先于 try-finally

Java 类库中包括许多必须通过调用 close 方法来手工关闭的资源。例如 InputStream、OutputStream 和 java.sql.Connection。try-finally 语句是确保资源会被适时关闭的有效方法，但是存在着些许不足。

```java
static String firstLineOfFile(String path) thorws IOException {
  BufferedReader br = new BufferedReader(new FileReader(path));
  try {
    return br.readLine();
  } finally {
    br.close();
  }
}
```

上段方法中，如果底层物理设备异常，那么调用 readLine 就会抛出异常，基于同样的原因，调用 close 也会出现异常。在这种情况下，第二个异常完全抹除了第一个异常。

在 Java 7 引入 try-with-resources 后，这个问题迎刃而解。要使用这个构造的资源，必须先实现 AutoCloseable 接口，其中包含了单个返回 void 的 close 方法。Java 类库与第三方类库中的类和接口都实现或拓展了 AutoCloseable 接口。如果编写了一个类，它代表的是必须被关闭的资源，那么这个类也应该实现 AutoCloseable。以 firstLineOfFile 方法举例，它的 try-with-resources 结构是：

```java
static String firstLineOfFile(String path) thorws IOException {
  try (BufferedReader br = new BufferedReader(new FileReader(path))){
    return br.readLine();
  }
}
```

结论很明显：<font color="skyblue">在处理必须关闭的资源时，始终要优先考虑使用 try-with-resources，而不是使用 try-finally。</font>这样得到的代码将更加清洁、清晰，产生的异常也更有价值。有了 try-with-resources 语句，在使用必须关闭的资源时，就能更轻松地正确编写代码了。实践证明，这个用 try-finally 是不可能做到的。

## 结尾

以上便是《Effective Java》第二章的大部分内容，其中第 4、第 5、第 8 条因未进行实践测试，没有完全理解文章中的内容，暂不进行总结记录。感兴趣的话可以自行购买阅读原文。