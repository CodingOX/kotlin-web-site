---
type: doc
layout: reference
category: "Syntax"
title: "高阶函数与 Lambda 表达式"
---

# 高阶函数与 Lambda 表达式

在 Kotlin 中函数是 [*一级公民*](https://en.wikipedia.org/wiki/First-class_function),
也就是说, 函数可以保存在变量和数据结构中, 可以作为参数来传递给 [高阶函数](#higher-order-functions), 也可以作为 [高阶函数](#higher-order-functions) 的返回值.
你可以就象对函数之外的其他数据类型值一样, 任意操作函数.

为了实现这些功能, Kotlin 作为一种静态类型语言, 使用了一组 [函数类型](#function-types) 来表达函数, 并提供了一组专门的语言结构, 比如 [lambda 表达式](#lambda-expressions-and-anonymous-functions).

## 高阶函数(Higher-Order Function)

高阶函数(higher-order function)是一种特殊的函数, 它接受函数作为参数, 或者返回一个函数.

高阶函数的一个很好的例子就是 [函数式编程(functional programming) 中对集合的 `折叠(fold)`](https://en.wikipedia.org/wiki/Fold_(higher-order_function)),
这个折叠函数的参数是一个初始的累计值, 以及一个结合函数, 然后将累计值与集合中的各个元素逐个结合, 最终得到结果值:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun <T, R> Collection<T>.fold(
    initial: R,
    combine: (acc: R, nextElement: T) -> R
): R {
    var accumulator: R = initial
    for (element: T in this) {
        accumulator = combine(accumulator, element)
    }
    return accumulator
}
```

</div>

上面的示例代码中, `combine` 参数是 [函数类型](#function-types) `(R, T) -> R`, 所以这个参数接受一个函数, 函数又接受两个参数, 类型为 `R` 和 `T`, 返回值类型为 `R`.
这个函数在 *for*{: .keyword } 循环内被 [调用](#invoking-a-function-type-instance), 函数的返回值被赋值给 `accumulator`.

要调用上面的 `fold` 函数, 我们需要向它传递一个 [函数类型的实例](#instantiating-a-function-type) 作为参数,
在调用高阶函数时, 我们经常使用 Lambda 表达式作为这种参数(详细介绍请参见 [后面的章节](#lambda-expressions-and-anonymous-functions)):

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
    //sampleStart
    val items = listOf(1, 2, 3, 4, 5)

    // Lambda 表达式是大括号括起的那部分代码.
    items.fold(0, {
        // 如果 Lambda 表达式有参数, 首先声明这些参数, 后面是 '->' 符
        acc: Int, i: Int ->
        print("acc = $acc, i = $i, ")
        val result = acc + i
        println("result = $result")
        // Lambda 表达式内的最后一个表达式会被看作返回值:
        result
    })

    // Lambda 表达式的参数类型如果可以推断得到, 那么参数类型的声明可以省略:
    val joinedToString = items.fold("Elements:", { acc, i -> acc + " " + i })

    // 在高阶函数调用中也可以使用函数引用:
    val product = items.fold(1, Int::times)
    //sampleEnd
    println("joinedToString = $joinedToString")
    println("product = $product")
}
```

</div>

在这里我们提到了一些新的概念, 我们会在后面的章节中进行详细解释.

## 函数类型(Function Type)

为了在类型和参数声明中处理函数, 比如: `val onClick: () -> Unit = ...` , Kotlin 使用了一系列的函数类型(Function Type), 比如 `(Int) -> String` .

这种函数类型使用一种特殊的表示方法, 用于表示函数的签名部分, 也就是说, 用于表示函数的参数和返回值:

* 所有的函数类型都带有参数类型列表, 用括号括起, 以及返回值类型: `(A, B) -> C` 表示一个函数类型, 它接受两个参数, 类型为 `A` 和 `B`, 返回值类型为 `C`.
 参数类型列表可以为空, 比如 `() -> A`. [`Unit` 类型的返回值](functions.html#unit-returning-functions) 不能省略.

* 函数类型也可以带一个额外的 *接受者* 类型, 以点号标记, 放在函数类型声明的前部:
 `A.(B) -> C` 表示一个可以对类型为 `A` 的接受者调用的函数, 参数类型为`B`, 返回值类型为 `C`.
 对这种函数类型, 我们经常使用 [带接受者的函数字面值](#function-literals-with-receiver).

* [挂起函数(Suspending function)](coroutines.html#suspending-functions) 是一种特殊类型的函数, 它的声明带有一个特殊的 *suspend*{: .keyword} 修饰符, 比如: `suspend () -> Unit`, 或者: `suspend A.(B) -> C`.

函数类型的声明也可以指定函数参数的名称: `(x: Int, y: Int) -> Point`.
参数名称可以用来更好地说明参数含义.

> 为了表示函数类型是 [可以为 null 的](null-safety.html#nullable-types-and-non-null-types), 可以使用括号: `((Int, Int) -> Int)?`.
>
> 函数类型也可以使用括号组合在一起: `(Int) -> ((Int) -> Unit)`
>
> 箭头符号的结合顺序是右侧优先, `(Int) -> (Int) -> Unit` 的含义与上面的例子一样, 而不同于: `((Int) -> (Int)) -> Unit`.

你也可以使用 [类型别名](type-aliases.html) 来给函数类型指定一个名称:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
typealias ClickHandler = (Button, ClickEvent) -> Unit
```

</div>

### 创建函数类型的实例

有几种不同的方法可以创建函数类型的实例:

* 使用函数字面值, 采用以下形式之一:
    * [Lambda 表达式](#lambda-expressions-and-anonymous-functions): `{ a, b -> a + b }`,
    * [匿名函数(Anonymous Function)](#anonymous-functions): `fun(s: String): Int { return s.toIntOrNull() ?: 0 }`

   [带接受者的函数字面值](#function-literals-with-receiver) 可以用作带接受者的函数类型的实例.

* 使用已声明的元素的可调用的引用:
    * 顶级[函数](reflection.html#function-references), 局部[函数](reflection.html#function-references), 成员[函数](reflection.html#function-references), 或扩展[函数](reflection.html#function-references), 比如: `::isOdd`, `String::toInt`,
    * 顶级[属性](reflection.html#property-references), 成员[属性](reflection.html#property-references), 或扩展[属性](reflection.html#property-references), 比如: `List<Int>::size`,
    * [构造器](reflection.html#constructor-references), 比如: `::Regex`

   以上几种形式都包括 [绑定到实例的可调用的引用](reflection.html#bound-function-and-property-references-since-11), 也就是指向具体实例的成员的引用: `foo::toString`.

* 使用自定义类, 以接口的方式实现函数类型:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
class IntTransformer: (Int) -> Int {
    override operator fun invoke(x: Int): Int = TODO()
}

val intFunction: (Int) -> Int = IntTransformer()
```

</div>

如果有足够的信息, 编译器可以推断出变量的函数类型:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val a = { i: Int -> i + 1 } // 编译器自动推断得到的类型为 (Int) -> Int
```

</div>

带接受者和不带接受者的函数类型的 *非字面* 值是可以互换的, 也就是说, 接受者可以代替第一个参数, 反过来第一个参数也可以代替接受者.
比如, 如果参数类型或变量类型为 `A.(B) -> C`, 那么可以使用 `(A, B) -> C` 函数类型的值, 反过来也是如此:

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
    //sampleStart
    val repeatFun: String.(Int) -> String = { times -> this.repeat(times) }
    val twoParameters: (String, Int) -> String = repeatFun // OK

    fun runTransformation(f: (String, Int) -> String): String {
        return f("hello", 3)
    }
    val result = runTransformation(repeatFun) // OK
    //sampleEnd
    println("result = $result")
}
```

</div>

> 注意, 自动推断的结果默认是不带接受者的函数类型, 即使给变量初始化赋值为一个扩展函数的引用, 也是如此.
> 要改变这种结果, 你需要明确指定变量类型.

### 调用一个函数类型的实例

要调用一个函数类型的值, 可以使用它的 [`invoke(...)` 操作符](operator-overloading.html#invoke): `f.invoke(x)`, 或者直接写 `f(x)`.

如果函数类型值有接受者, 那么接受者对象实例应该作为第一个参数传递进去. 调用有接受者的函数类型值的另一种方式是, 将接受者写作函数调用的前缀, 就像调用 [扩展函数](extensions.html) 一样: `1.foo(2)`,

示例:

<div class="sample" markdown="1" theme="idea">

```kotlin
fun main() {
    //sampleStart
    val stringPlus: (String, String) -> String = String::plus
    val intPlus: Int.(Int) -> Int = Int::plus

    println(stringPlus.invoke("<-", "->"))
    println(stringPlus("Hello, ", "world!"))

    println(intPlus.invoke(1, 1))
    println(intPlus(1, 2))
    println(2.intPlus(3)) // 与扩展函数类似的调用方式
    //sampleEnd
}
```

</div>

### 内联函数(Inline Function)

有些时候, 使用 [内联函数](inline-functions.html) 可以为高阶函数实现更加灵活的控制流程.

## Lambda 表达式与匿名函数(Anonymous Function)

Lambda 表达式和匿名函数, 都是"函数字面值(function literal)", 也就是, 这个函数本身没有类型声明, 而是立即作为表达式传递出去.
我们来看看下面的示例:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
max(strings, { a, b -> a.length < b.length })
```

</div>

函数 `max` 是一个高阶函数, 它接受一个函数值作为第二个参数. 第二个参数是一个表达式, 本身又是另一个函数, 也就是说, 它是一个函数字面量.
这个函数字面量, 等价于下面的这个有名称的函数:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun compare(a: String, b: String): Boolean = a.length < b.length
```

</div>

### Lambda 表达式的语法

Lambda 表达式的完整语法形式如下:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val sum = { x: Int, y: Int -> x + y }
```

</div>

Lambda 表达式包含在大括号之内, 在完整语法形式中, 参数声明在大括号之内, 参数类型的声明是可选的, 函数体在 `->` 符号之后.
如果 Lambda 表达式自动推断的返回值类型不是 `Unit`, 那么 Lambda 表达式函数体中, 最后一条(或者就是唯一一条)表达式的值, 会被当作整个 Lambda 表达式的返回值.

如果我们把所有可选的内容都去掉, 那么剩余的部分如下:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val sum: (Int, Int) -> Int = { x, y -> x + y }
```

</div>

### 将 Lambda 表达式用作函数调用的最后一个参数

在 Kotlin 中有一种规约, 如果函数的最后一个参数接受一个函数类型的值, 那么如果使用 Lambda 表达式作为这个参数的值,
可以将 Lambda 表达式写在函数调用的括号之外:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val product = items.fold(1) { acc, e -> acc * e }
```

</div>

如果 Lambda 表达式是函数调用时的唯一一个参数, 括号可以完全省略:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
run { println("...") }
```

</div>

### `it`: 单一参数的隐含名称

很多情况下 Lambda 表达式只有唯一一个参数.

如果编译器能够自行识别出 Lambda 表达式的参数定义, 那么我们可以不必声明这个唯一的参数, 并省略 `->` 符号, 这个参数会隐含地声明为 `it`:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
ints.filter { it > 0 } // 这个函数字面值的类型是 '(it: Int) -> Boolean'
```

</div>

### 从 Lambda 表达式中返回结果值

如果使用 [带标签限定的 return](returns.html#return-at-labels) 语法, 我们可以在 Lambda 表达式内明确地返回一个结果值.
否则, 会隐含地返回 Lambda 表达式内最后一条表达式的值.

因此, 下面两段代码是等价的:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
ints.filter {
    val shouldFilter = it > 0
    shouldFilter
}

ints.filter {
    val shouldFilter = it > 0
    return@filter shouldFilter
}
```

</div>

使用这个规约, 再加上 [在括号之外传递 Lambda 表达式作为函数调用的参数](#passing-a-lambda-to-the-last-parameter),
我们可以编写 [LINQ 风格](http://msdn.microsoft.com/en-us/library/bb308959.aspx) 的程序:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
strings.filter { it.length == 5 }.sortedBy { it }.map { it.toUpperCase() }
```

</div>

### 使用下划线代替未使用的参数 (从 Kotlin 1.1 开始支持)

如果 Lambda 表达式的某个参数未被使用, 你可以用下划线来代替参数名:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
map.forEach { _, value -> println("$value!") }
```

</div>

### 在 Lambda 表达式中使用解构声明 (从 Kotlin 1.1 开始支持)

关于在 Lambda 表达式中使用解构声明, 请参见 [解构声明(destructuring declaration)](multi-declarations.html#destructuring-in-lambdas-since-11).

### 匿名函数(Anonymous Function)

上面讲到的 Lambda 表达式语法, 还缺少了一种功能, 就是如何指定函数的返回值类型. 大多数情况下, 不需要指定返回值类型, 因为可以自动推断得到. 但是, 如果的确需要明确指定返回值类型, 你可以可以选择另一种语法: _匿名函数(anonymous function)_.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun(x: Int, y: Int): Int = x + y
```

</div>

匿名函数看起来与通常的函数声明很类似, 区别在于省略了函数名. 函数体可以是一个表达式(如上例), 也可以是多条语句组成的代码段:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun(x: Int, y: Int): Int {
    return x + y
}
```

</div>

参数和返回值类型的声明与通常的函数一样, 但如果参数类型可以通过上下文推断得到, 那么类型声明可以省略:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
ints.filter(fun(item) = item > 0)
```

</div>

对于匿名函数, 返回值类型的自动推断方式与通常的函数一样: 如果函数体是一个表达式, 那么返回值类型可以自动推断得到, 如果函数体是多条语句组成的代码段, 则返回值类型必须明确指定(否则被认为是 `Unit`).

注意, 匿名函数参数一定要在圆括号内传递. 允许将函数类型参数写在圆括号之外语法, 仅对 Lambda 表达式有效.

Lambda 表达式与匿名函数之间的另一个区别是, 它们的 [非局部返回(non-local return)](inline-functions.html#non-local-returns) 的行为不同. 不使用标签的 *return*{: .keyword } 语句总是从 *fun*{: .keyword } 关键字定义的函数中返回. 也就是说, Lambda 表达式内的 *return*{: .keyword } 将会从包含这个 Lambda 表达式的函数中返回, 而匿名函数内的 *return*{: .keyword } 只会从匿名函数本身返回.

### 闭包(Closure)

Lambda 表达式, 匿名函数 (此外还有 [局部函数](functions.html#local-functions), [对象表达式](object-declarations.html#object-expressions)) 可以访问它的 _闭包_, 也就是, 定义在外层范围中的变量. 与 Java 不同, 闭包中捕获的变量是可以修改的 (译注: Java 中必须为 final 变量):

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
var sum = 0
ints.filter { it > 0 }.forEach {
    sum += it
}
print(sum)
```

</div>

### 带有接受者的函数字面值

带接受者的 [函数类型](#function-types), 比如 `A.(B) -> C`, 可以通过一种特殊形式的函数字面值来创建它的实例, 也就是带接受者的函数字面值.

上文中我们讲到, Kotlin 提供了一种能力, 在 [调用带接受者的函数类型的实例](#invoking-a-function-type-instance) 时, 可以指定一个 _接收者对象(receiver object)_.

在这个函数字面值的函数体内部, 传递给这个函数调用的接受者对象会成为一个 *隐含的* *this*{: .keyword},
因此你可以访问接收者对象的成员, 而不必指定任何限定符, 也可以使用 [`this` 表达式](this-expressions.html) 来访问接受者对象.

这种行为与 [扩展函数](extensions.html) 很类似, 在扩展函数的函数体中, 你也可以访问接收者对象的成员.

下面的例子演示一个带接受者的函数字面值, 以及这个函数字面值的类型, 在函数体内部, 调用了接受者对象的 `plus` 方法:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val sum: Int.(Int) -> Int = { other -> plus(other) }
```

</div>

匿名函数语法允许你直接指定函数字面值的接受者类型.
如果你需要声明一个带接受者的函数类型变量, 然后在将来的某个地方使用它, 那么这种功能就很有用.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
val sum = fun Int.(other: Int): Int = this + other
```

</div>

如果接受者类型可以通过上下文自动推断得到, 那么 Lambda 表达式也可以用做带接受者的函数字面值.
这种用法的一个重要例子就是 [类型安全的构建器(Type-Safe Builder)](type-safe-builders.html):

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
class HTML {
    fun body() { ... }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()  // 创建接受者对象
    html.init()        // 将接受者对象传递给 Lambda 表达式
    return html
}

html {       // 带接受者的 Lambda 表达式从这里开始
    body()   // 调用接受者对象上的一个方法
}
```

</div>
