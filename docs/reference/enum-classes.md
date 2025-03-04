---
type: doc
layout: reference
category: "Syntax"
title: "枚举类"
---

# 枚举类(Enum Class)

枚举类最基本的用法, 就是实现类型安全的枚举值:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
enum class Direction {
    NORTH, SOUTH, WEST, EAST
}
```
</div>

每个枚举常数都是一个对象. 枚举常数之间用逗号分隔.

## 初始化

由于每个枚举值都是枚举类的一个实例, 因此枚举值可以这样初始化:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
enum class Color(val rgb: Int) {
        RED(0xFF0000),
        GREEN(0x00FF00),
        BLUE(0x0000FF)
}
```
</div>

## 匿名类

枚举常数也可以定义它自己的匿名类:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
enum class ProtocolState {
    WAITING {
        override fun signal() = TALKING
    },

    TALKING {
        override fun signal() = WAITING
    };

    abstract fun signal(): ProtocolState
}
```
</div>

枚举常数的匿名类可以有各自的方法, 还可以覆盖基类的方法. 注意, 与 Java 中一样, 如果枚举类中定义了任何成员, 你需要用分号将枚举常数的定义与枚举类的成员定义分隔开.

除内部类(Inner class)之外, 枚举常数中不能包含嵌套类型 (这个限制在 Kotlin 1.2 版中已被废除).

## 在枚举类中实现接口

枚举类也可以实现接口 (但不能继承其他类), 可以对所有的枚举常数提供唯一的接口实现, 也可以在不同的枚举常数的匿名类中提供不同的实现.
枚举类实现接口时, 只需要在枚举类的声明中加入接口名, 示例如下:

<div class="sample" markdown="1" theme="idea">

```kotlin
import java.util.function.BinaryOperator
import java.util.function.IntBinaryOperator

//sampleStart
enum class IntArithmetics : BinaryOperator<Int>, IntBinaryOperator {
    PLUS {
        override fun apply(t: Int, u: Int): Int = t + u
    },
    TIMES {
        override fun apply(t: Int, u: Int): Int = t * u
    };

    override fun applyAsInt(t: Int, u: Int) = apply(t, u)
}
//sampleEnd

fun main() {
    val a = 13
    val b = 31
    for (f in IntArithmetics.values()) {
        println("$f($a, $b) = ${f.apply(a, b)}")
    }
}
```
</div>

## 使用枚举常数

与 Java 中一样, Kotlin 中的枚举类拥有编译器添加的方法, 可以列出枚举类中定义的所有枚举常数值, 可以通过枚举常数值的名称字符串得到对应的枚举常数值. 这些方法的签名如下(这里假设枚举类名称为 `EnumClass`):

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
EnumClass.valueOf(value: String): EnumClass
EnumClass.values(): Array<EnumClass>
```
</div>

如果给定的名称不能匹配枚举类中定义的任何一个枚举常数值, `valueOf()` 方法会抛出 `IllegalArgumentException` 异常.

从 Kotlin 1.1 开始, 可以通过 `enumValues<T>()` 和 `enumValueOf<T>()` 函数, 以泛型方式取得枚举类中的常数:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
enum class RGB { RED, GREEN, BLUE }

inline fun <reified T : Enum<T>> printAllValues() {
    print(enumValues<T>().joinToString { it.name })
}

printAllValues<RGB>() // 打印结果为 RED, GREEN, BLUE
```
</div>

每个枚举常数值都拥有属性, 可以取得它的名称, 以及它在枚举类中声明的顺序:

<div class="sample" markdown="1" theme="idea" data-highlight-only>
```kotlin
val name: String
val ordinal: Int
```
</div>

枚举常数值还实现了 [Comparable](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-comparable/index.html) 接口, 枚举常数值之间比较时, 会使用枚举常数值在枚举类中声明的顺序作为自己的大小顺序.
