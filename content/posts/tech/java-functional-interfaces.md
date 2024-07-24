---
title: "Java 8 Functional 函数式接口"
date: 2020-11-24T20:30:01+08:00
tags: ["tech", "java"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "函数式接口（Functional Interface）就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---

## 01 函数式接口

### 1.1 什么是函数式接口

**函数式接口**（Functional Interface）就是一个有且仅有一个**抽象方法**，但是可以有多个非抽象方法的接口。

函数式接口可以被隐式转换为 Lambda 表达式。

> Lambda 是一个匿名函数，返回一个实现了接口的对象。方法引用实际上也可认为是 Lambda 表达式（简化版）。

### 1.2 自定义函数式接口

如果要实现自己的函数式接口，步骤很简单。

首先，定义一个接口，并声明一个抽象方法。然后给接口添加注解 `@FunctionalInterface`，该接口即为函数式接口：

```java
@FunctionalInterface
interface Converter<T, R> {
    R convert(T from);
}
```

使用同样也很简单，创建一个 Lambda 表达式（要实现的方法）并赋值给该接口的一个变量，然后直接调用定义的那个抽象方法：

```java
// 创建一个函数：字符串转整型
Converter<String, Integer> converter = (from) -> Ingeter.valueOf(from);
converter.convert("123") // 123
```

## 02 Java（8）提供的函数式接口

### 2.1 Converter<T, R>，如上
### 2.2 Predicate<T>

作用：断言，判断是与非。

调用方式：`predicate.test(OBJ)`。

示例：

```java
Predicate<String> predicate = (s) -> s.length() > 0;

predicate.test("foo");              // true
predicate.negate().test("foo");     // false
```

### 2.3 Function<T, R>

作用：接口一个参数，返回一个结果，可以链式调用。

调用方式：`function.apply(OBJ)`。

示例：

```java
Function<String, Integer> toInteger = Integer::valueOf;
// 链式
Function<String, String> backToString = toInteger.andThen(String::valueOf);

backToString.apply("123");     // "123"
```

### 2.4 Supplier<R>

作用：和 **Function** 一样返回一个结果，但不接收参数。

调用方式：`supplier.get()`。

示例：

```java
Supplier<Person> personSupplier = Person::new;
personSupplier.get();   // new Person
```

### 2.5 Consumer<T>

作用：接收一个参数，但不返回（顾名思义），仅执行操作。

调用方式：`consumer.accept(OBJ)`。

示例：

```java
Consumer<Person> greeter = (p) -> System.out.println("Hello, " + p.firstName);
greeter.accept(new Person("Luke", "Skywalker")); // 输出：Hello, Luke
```

### 2.6 Comparator<T>（since 1.2）

作用：接收两个参数，对比它们的大小（0），支持链式。

调用方式：`comparator.compare(o1, o2)`。

示例：

```Java
Comparator<Person> comparator = (p1, p2) -> p1.firstName.compareTo(p2.firstName);

Person p1 = new Person("John", "Doe");
Person p2 = new Person("Alice", "Wonderland");

comparator.compare(p1, p2);             // > 0
comparator.reversed().compare(p1, p2);  // < 0
```

### 2.7 Optional<T>（非函数式接口）

作用：用于防止空指针异常等。

调用方式：`optional.get()` `optional.orElse(如果为空则返回这个值)`。

示例：

```java
// Optional.ofNullable(xx)
Optional<String> optional = Optional.of("bam");

optional.isPresent();           // true
optional.get();                 // "bam"
optional.orElse("fallback");    // "bam"

optional.ifPresent((s) -> System.out.println(s.charAt(0)));     // "b"
```

## 03 示例

### 3.1 实现 `f1.SetVal(f2.getVal())`

实现一个功能如：把 f2 的 val 赋值给 f1 的 val。

首先，用到了两个方法，需要用链式调用，并且

- getVal 需要一个参数一个返回值，用 `Function<V, R>`
- setVal 不需要返回值，用 `Consumer<T>`

可以定义如下，Function 接收一个参数 v，返回一个 Consumer，而 Consumer 接收一个参数 o，执行指定动作。

```JAVA
Function<T, Consumer<T>> valSetter = v -> o -> {
    v.setVal(o.getVal());
};
```

具体使用如下：

```JAVA
// valSetter.apply(f1) == consumer
//   此时 consumer == (o) -> { f1.setVal(o.getVal()); }
// consumer.accept(f2) => 执行上面的 consumer，即f2 代入 o，得到想要的执行结果
valSetter.apply(f1).accept(f2);
```

- apply：先执行 Function 的方法，然后将结果作为参数给下个调用
- accept：由于不需要返回值，就用 Consumer 来接收处理

------------------------------------

## References

[1] winterbe. Java 8 Tutorial[DB/OL]. https://winterbe.com/posts/2014/03/16/java-8-tutorial/.     
