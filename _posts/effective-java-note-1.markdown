---
layout: post
title:  Effective Java学习笔记（一）考虑使用静态工厂方法替代构造方法
date:   2020-03-29 18:33:36 +0800
categories: Java
---
# 写在前面

本文是我的《Effective Java》学习笔记开篇，主要参考的是第3版，有些地方也参考了第2版。我想通过自己的语言转述或概括书中的内容以表达我在阅读与学习过程中得到的理解和产生的困惑，会尽可能全面地覆盖书中提到的内容。如果有写得不对或不全的地方，欢迎指正和补充，谢谢~

# 什么是静态工厂方法？

我们都知道，想要创建对象的时候可以使用类的公共构造方法，然而还有另一种方式——_静态工厂方法_（static factory method）。静态工厂方法是一个返回类实例的静态方法，如下所示是`Boolean`包装类的一个静态工厂方法。

```java
    public static Boolean valueOf(boolean b) {
        return b ? Boolean.TRUE : Boolean.FALSE;
    }

```

不过正如书中所说，静态工厂方法只是一种构造类实例的方式，和设计模式中的工厂方法模式还是不一样的。

那么使用静态工厂方法有什么好处呢？

# 静态工厂方法的优势

## 1. 静态工厂方法有自己的名字
Java语言中类的构造方法必须与类同名，这使得我们较难分清多个构造方法的不同用途。但是静态工厂方法是一个方法，这就意味着可以随便起名字，可以更清晰地标识出每个方法的用途。

例如，我们在提供RPC服务时，通常会封装一个Result对象包装返回的结果，这个时候用不同的静态工厂方法就可以表明要返回的不同结果，如下所示。

```java
public class Result<T> {

    /**
     * 错误码  
     */
    private String code;

    /**
     * 错误信息
     */
    private String msg;

    /**
     * 返回业务对象
     */
    private T data;

    /*---------下面是一些构造方法-----------*/

    public Result() {}

    public Result(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public Result(String code, String msg, T data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }

    /*---------下面是一些静态工厂方法----------*/

    public static <T> Result<T> success(T data) {
        return new Result<T>(ResultEnum.SUCCESS.getCode(), ResultEnum.SUCCESS.getMsg(), data);
    }

    public static <T> Result<T> error(ResultEnum resultEnum) {
        return new Result<T>(resultEnum.getCode(), resultEnum.getMsg());
    }

}
```

上面这个例子提供了两个静态工厂方法，分别用于业务逻辑正常和出错时返回对应Result对象的。有了语义上的区分，服务提供者在编写不同业务逻辑代码的时候就知道该调用哪个方法返回结果了，写出来的代码逻辑性和可读性都会比构造方法强。

## 2. 不需要每次调用都新建对象

我们知道，每次调用构造方法都必定会new一个对象返回，但是静态工厂方法允许我们返回预先构建好的实例，这样我们可以对实例保持严格的控制。对实例进行控制可以保证类是单例的或是非实例化的，也可以返回预先构建好的不可变对象。比如文首的`Boolean.valueOf`方法返回的其实就是`Boolean`的两个不可变对象

```java
    /**
     * The {@code Boolean} object corresponding to the primitive
     * value {@code true}.
     */
    public static final Boolean TRUE = new Boolean(true);

    /**
     * The {@code Boolean} object corresponding to the primitive
     * value {@code false}.
     */
    public static final Boolean FALSE = new Boolean(false);
```

实例控制保证多次请求返回等价对象，如果两次返回的对象是a和b，那么当且仅当`a == b` 时 才有`a.equals(b)`，这样能够减少new对象的开销，尤其适用于创建大对象的时候。

## 3. 可以返回子类

使用静态工厂方法意味着不必像构造方法那样，只能死板地返回这个类，而可以返回它的任意子类，且这个子类不必是`public`的。例如，`java.util.Collections`类中提供了下面这样一个静态工厂方法：

```java
    public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c) {
        return new UnmodifiableCollection<>(c);
    }
```

该方法用于返回一个不可变集合，这个不可变集合是`Collection`的一个实现，且就写在了`Collections`中，作为其内部的一个非公有类。

## 4. 返回对象的类可以根据输入参数的不同而不同

这点还是强调灵活性，因为静态工厂方法是一个自由的方法，所以你完全可以自己在里面制定策略。比如`java.util.EnumSet`中有一个静态构造方法`noneOf`，用于返回一个指定枚举类型的空EnumSet。它会根据传入的枚举类的元素个数决定返回的对象。当枚举类元素小于64时返回`RegularEnumSet`，超过时返回`JumboEnumSet`，代码如下：

```java
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

这种实现方式很具有启发性。例子中的这两个实现类对调用方而言是完全透明的，今后随着迭代的演进，提供方完全可以制定其他的策略，更改返回的子类对象（比如直接弃用`RegularEnumSet`），这些调用方都无需关心。

## 5. 静态工厂方法返回对象的类可以不存在于静态工厂方法所处的类中
这一点比较难理解，我认为作者的意思是如果静态工厂方法在类A里，它可以返回类B的对象。前面讨论的几点中，静态工厂方法返回的对象和方法本身是在同一个类里的，而这一点又扩展了静态工厂方法的能力范围。它构成了 _服务提供者框架_（Service Provider Framework）的基础。引用书中的定义，“服务提供者框架是指这样一个系统：多个服务提供者实现一个服务，系统为服务提供者的客户端提供多个实现，并把他们从多个实现中解耦出来”。

服务提供者框架有三个重要组件：

* 服务接口（Service Interface）
* 提供者注册API（Provider Registration API）
* 服务访问API（Service Access API）

和一个可选组件：

* 服务提供者接口（Service Provider Interface）

和书中一样，我们以JDBC API为例。JDBC中的**服务接口**是`Connection`，它维护了一个与数据库连接的上下文，调用者所有与数据库相关的操作都在这个上下文里执行；`DriverManager.registerDriver`是**提供者注册API**，系统会调用这个方法，将所有Driver注册进DriverManager里；
`DriverManager.getConnection`是**服务访问API**，调用者可以通过这个方法拿到自己所需要的Connection；`Driver`是服务提供者接口，负责创建其服务实现的实例（提供Connection）。如果服务没有一个提供者接口用于继承，那么提供方提供的服务就要以类名注册，并以反射的方式进行实例化。

# 静态工厂方法的缺点

所有的架构和设计模式都是有利也有弊的，静态工厂方法也不例外。

## 1. 如果只提供静态工厂方法，那么该类不能被继承

Java语言规定了一个没有`public`或者`protected`构造方法的类不能被继承（如果一个类的构造方法是默认可见性的，那么能在同包内被继承，但这点意义不大）。就像前面提到的`UnmodifiableCollection`由于是在`Collections`中的内部静态类，且只能通过静态构造方法来获取，那么它就不能被继承了。

## 2. 静态工厂方法不容易被注意到

静态工厂方法由于是普通方法，所以不会在API文档中被明确标注出来。我们在构造某个类的时候，如果没有阅读它的源码，也可能会忽略里面的静态工厂方法。

# 附录 静态工厂方法的常用命名

附录部分直接引自原书第3版

* `from` —— 类型转换方法，它接受单个参数并返回此类型的相应实例，例如：Date d = Date.from(instant);
* `of` —— 聚合方法，接受多个参数并返回该类型的实例，并把他们合并在一起，例如：Set faceCards = EnumSet.of(JACK, QUEEN, KING);
* `valueOf` —— from 和 to 更为详细的替代 方式，例如：BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
* `instance` 或 getinstance —— 返回一个由其参数 (如果有的话) 描述的实例，但不能说它具有相同的值，例如：StackWalker luke = StackWalker.getInstance(options);
* `create` 或 `newInstance` —— 与 instance 或 getInstance 类似，除此之外该方法保证每次调用返回一个新的实例，例如：Object newArray = Array.newInstance(classObject, arrayLen);
* `getType` —— 与 `getInstance` 类似，但是在工厂方法处于不同的类中的时候使用。getType 中的 Type 是工厂方法返回的对象类型，例如：FileStore fs = Files.getFileStore(path);
* `newType` —— 与 `newInstance` 类似，但是在工厂方法处于不同的类中的时候使用。newType中的 Type 是工厂方法返回的对象类型，例如：BufferedReader br = Files.newBufferedReader(path);
* `type` —— `getType` 和 `newType` 简洁的替代方式，例如：List litany = Collections.list(legacyLitany);

# 总结

总而言之，个人认为静态工厂方法还是利大于弊的，在实际编程开发中应该得到实践。

转载请注明出处，谢谢。

