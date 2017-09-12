---
type: doc
layout: reference
category: Basics
title: 编码规约
---

# 编码规约

本章介绍 Kotlin 语言目前的编码风格.

## 命名风格
如果对某种规约存在疑问, 请默认使用 Java 编码规约, 比如:

* 在名称中使用驼峰式大小写(并且不要在名称中使用下划线)
* 类型首字母大写
* 方法和属性名称首字母小写
* 语句缩进使用 4 个空格
* public 函数应该编写文档, 这些文档将出现在 Kotlin Doc 中

## 冒号

当冒号用来分隔类型和父类型时, 冒号之前要有空格, 当冒号用来分隔类型和实例时, 冒号之前不加空格:

``` kotlin
interface Foo<out T : Any> : Bar {
    fun foo(a: Int): T
}
```

## Lambda 表达式

在 Lambda 表达式中, 大括号前后应该加空格, 分隔参数与函数体的箭头符号前后也应该加空格. 如果有可能, 将 Lambda 表达式作为参数传递时, 应该尽量放在圆括号之外.

``` kotlin
list.filter { it > 10 }.map { element -> element * 2 }
```

在简短, 并且无嵌套的 Lambda 表达式中, 推荐使用惯用约定的 `it` 作为参数名, 而不要明确地声明参数. 在有参数, 并且嵌套的 Lambda 表达式中, 应该明确地声明参数.

## Class 的头部格式

如果类只有少量参数, 可以写成单独的一行:

```kotlin
class Person(id: Int, name: String)
```

如果类的头部很长, 应该调整代码格式, 将主构造器(primary constructor)的每一个参数放在单独的行中, 并对其缩进. 同时, 闭括号也应放在新的一行.
如果我们用到了类的继承, 那么对超类构造器的调用, 以及实现的接口的列表, 应该与闭括号放在同一行内:

```kotlin
class Person(
    id: Int,
    name: String,
    surname: String
) : Human(id, name) {
    // ...
}
```

对于多个接口的情况, 对超类构造器的调用应该放在最前, 然后将每个接口放在单独的行中:

```kotlin
class Person(
    id: Int,
    name: String,
    surname: String
) : Human(id, name),
    KotlinMaker {
    // ...
}
```

构造器参数应该使用通常缩进(regular indent), 或者使用连续缩进(continuation indent) (比通常缩进多缩进一倍).

## Unit

如果函数的返回值为 Unit 类型, 那么返回值的类型声明应当省略:

``` kotlin
fun foo() { // 此处省略了 ": Unit"

}
```

## 函数 vs 属性

有些情况下, 无参数的函数可以与只读属性相互替代.
虽然它们在语义上是相似的, 但从编程风格上的角度看, 存在一些规约来决定在什么时候应该使用函数, 什么时候应该使用属性.

当以下条件成立时, 应该选择使用只读属性, 而不是使用函数:

* 不会抛出异常
* 复杂度为 `O(1)`
* 计算过程消费的资源不多(或者在初次运行时缓存了计算结果)
* 多次调用时会返回相同的结果
