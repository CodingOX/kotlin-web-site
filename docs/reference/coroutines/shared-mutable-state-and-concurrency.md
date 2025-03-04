---
type: doc
layout: reference
category: "Coroutine"
title: "共享的可变状态与并发"
---


<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/SharedStateGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class SharedStateGuideTest {
-->
**目录**

<!--- TOC -->

* [共享的可变状态值与并发](#shared-mutable-state-and-concurrency)
  * [问题的产生](#the-problem)
  * [volatile 不能解决这个问题](#volatiles-are-of-no-help)
  * [线程安全的数据结构](#thread-safe-data-structures)
  * [细粒度的线程限定](#thread-confinement-fine-grained)
  * [粗粒度的线程限定](#thread-confinement-coarse-grained)
  * [互斥](#mutual-exclusion)
  * [Actor](#actors)

<!--- END_TOC -->

## 共享的可变状态值与并发

使用多线程的派发器, 比如 [Dispatchers.Default], 协程可以并发执行.
因此协程也面对并发带来的所有问题.
主要问题是访问 **共享的可变状态值** 时的同步问题.
在协程的世界里, 这类问题的有些解决方案与在线程世界中很类似, 但另外一些方案就非常不同.

### 问题的产生

下面我们启动 100 个协程, 每个协程都将同样的操作执行 1000 次.
我们测量一下它们的结束时间, 并做进一步的比较:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

</div>

我们先来执行一个非常简单的操作, 把一个共享的可变变量加 1, 在 [GlobalScope] 中启动协程, 使用多线程的 [Dispatchers.Default].

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*    

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-01.kt)

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码的最终输出结果是什么? 很大概率它不会输出 "Counter = 100000",
因为 100 个协程运行在多线程环境内, 它们同时给 `counter` 加 1, 但没有任何的同步机制.

> 注意: 如果你的系统太旧, 只有 2 个或更少的 CPU, 那么你 _一定会_ 得到确定的结果 100000, 因为这时线程池只有一个线程.
为了重现我们的问题, 你需要对示例程序做如下修改:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val mtContext = newFixedThreadPoolContext(2, "mtPool") // 明确定义上下文, 使用 2 个线程
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    CoroutineScope(mtContext).massiveRun { // 使用自定义的上下文, 而不是 Dispatchers.Default
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-01b.kt)

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

### volatile 不能解决这个问题

有一种常见的错误观念, 认为把变量变为 `volatile` 就可以解决并发访问问题. 我们来试一下:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

@Volatile // 在 Kotlin 中, `volatile` 是注解
var counter = 0

fun main() = runBlocking<Unit> {
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-02.kt)

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

代码运行变慢了, 但最终我们还是得不到 "Counter = 100000", 因为 volatile 变量保证线性的(linearizable) (意思就是 "原子性(atomic)") 读和写操作,
但不能保证更大的操作(在我们的例子中, 就是加 1 操作)的原子性.

### 线程安全的数据结构

一种对于线程和协程都能够适用的解决方案是, 使用线程安全的 (也叫 同步的(synchronized), 线性的(linearizable), 或者 原子化的(atomic)) 数据结构, 这些数据结构会对需要在共享的状态数据上进行的操作提供需要的同步保障.
在我们的简单的计数器示例中, 可以使用 `AtomicInteger` 类, 它有一个原子化的 `incrementAndGet` 操作:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.atomic.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

var counter = AtomicInteger()

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
//sampleEnd    
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-03.kt)

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

对于这个具体的问题, 这是最快的解决方案. 这种方案适用于计数器, 集合, 队列, 以及其他标准数据结构, 以及这些数据结构的基本操作.
但是, 这种方案并不能简单地应用于复杂的状态变量, 或者那些没有现成的线程安全实现的复杂操作.

### 细粒度的线程限定

_线程限定(Thread confinement)_ 是共享的可变状态值问题的一种解决方案, 它把所有对某个共享值的访问操作都限定在唯一的一个线程内.
最典型的应用场景是 UI 应用程序, 所有的 UI 状态都被限定在唯一一个 事件派发(event-dispatch) 线程 或者叫 application 线程内.
通过使用单线程的上下文, 可以很容易地对协程使用这种方案.

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun { // 使用 DefaultDispathcer 运行每个协程
        withContext(counterContext) { // 但把所有的加 1 操作都限定在单一线程的上下文中
            counter++
        }
    }
    println("Counter = $counter")
//sampleEnd
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-04.kt)

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码的运行速度会非常地慢, 因为它进行了 _细粒度(fine-grained)_ 的线程限定.
每一次加 1 操作都必须使用 [withContext], 从多线程的 [Dispatchers.Default] 上下文切换到单一线程上下文.

### 粗粒度的线程限定

在实际应用中, 通常在更大的尺度上进行线程限定, 比如, 将大块的状态更新业务逻辑限定在单个线程中.
下面的示例程序就是这样做的, 它在单一线程的上下文中运行每个协程.
这里我们使用 [CoroutineScope()] 函数来将协程的上下文转换为 [CoroutineScope] 类型:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    CoroutineScope(counterContext).massiveRun { // 在单一线程的上下文中运行每个协程
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-05.kt)

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

现在我们的代码运行的很快, 而且能够得到正确的结果.

### 互斥

对于这个问题的另一个解决方案是互斥(Mutual exclusion), 它使用一个 _临界区(critical section)_ 来保护所有针对共享状态值的修改动作, 临界区内的代码永远不会并发执行.
在阻塞式编程的世界, 你通常会使用 `synchronized` 或 `ReentrantLock` 来实现这个目的.
在线程中的方案叫做 [Mutex]. 它的 [lock][Mutex.lock] 和 [unlock][Mutex.unlock] 函数可以用来界定临界区.
主要的区别在于 `Mutex.lock()` 是一个挂起函数. 它不会阻塞线程.

还有一个扩展函数 [withLock], 它代表 `mutex.lock(); try { ... } finally { mutex.unlock() }`:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val mutex = Mutex()
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        mutex.withLock {
            counter++        
        }
    }
    println("Counter = $counter")
//sampleEnd    
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-06.kt)

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

上面的示例程序中的锁是细粒度的, 因此会产生一些代价. 但是, 对于某些情况下, 你确实需要不时修改某些共享的状态值, 但是这个状态值又没有限定在某个线程之内, 那么使用锁是一种好的选择.

### Actor

[actor](https://en.wikipedia.org/wiki/Actor_model) 是一个实体, 其中包含一个线程, 一个限定并封装在这个线程上的状态值, 以及一个用来与其他协程通信的通道.
一个简单的 actor 可以写成一个函数, 但带有复杂状态的 actor 更适合写成一个类.

有一个 [actor] 协程构建器, 可以将 actor 的信箱通道(mailbox channel) 绑定在它的作用范围上, 用来接受消息, 并将它的送信通道绑定到线程构建器的结果任务对象上,
因此一个指向 actor 的引用就可以作为它的句柄来传递.

使用 actor 的第一步是定义 actor 将要处理的消息类.
Kotlin 的 [封闭类(Sealed Class)](../sealed-classes.html) 非常适合于这个目的.
我们定义一个 `CounterMsg` 封闭类, 其中 `IncCounter` 消息用来对计数器加 1, `GetCounter` 消息用来获取计数器的值.
后一个消息还需要发送一个应答. 我们在这里使用通信原语 [CompletableDeferred] 来实现这个目的, 它表示一个会在未来得到(传来)的单个的值,.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// 供 counterActor 使用的消息类
sealed class CounterMsg
object IncCounter : CounterMsg() // 单向消息, 将计数器加 1
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 一个带应答的请求
```

</div>

然后我们定义一个函数, 它使用 [actor] 协程构建器启动一个 actor:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// 这个函数启动一个新的计数器 actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor 的状态值
    for (msg in channel) { // 遍历所有收到的消息
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

</div>

主代码非常简单:

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程执行操作的重复次数
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 供 counterActor 使用的消息类
sealed class CounterMsg
object IncCounter : CounterMsg() // 单向消息, 将计数器加 1
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 一个带应答的请求

// 这个函数启动一个新的计数器 actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor 的状态值
    for (msg in channel) { // 遍历所有收到的消息
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    val counter = counterActor() // 创建 actor
    GlobalScope.massiveRun {
        counter.send(IncCounter)
    }
    // 发送一个消息, 从 actor 得到计数器值
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // 关闭 actor
//sampleEnd    
}
```

</div>

> 完整的代码请参见 [这里](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-07.kt)

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

(对于结果的正确性来说) actor 本身在哪个上下文中运行是无关紧要的.
actor 是一个协程, 协程是顺序执行的, 因此将状态值限定在特定的协程内, 可以解决共享的可变状态值问题.
actor 确实可以修改它自己的私有状态值, 但它们之间只能通过消息来相互影响 (因此不必使用锁).

在高负载的情况下, actor 比锁的效率更高, 因为这种情况下 actor 总是有工作可做(不必挂起), 而且它完全不必在不同的上下文之间切换.

> 注意, [actor] 协程构建器 与 [produce] 协程构建器刚好相反. actor 绑定到一个通道, 从通道读取消息, 而生产者则绑定到一个通道, 向通道发送数据.

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html
<!--- INDEX kotlinx.coroutines.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/unlock.html
[withLock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/with-lock.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
<!--- END -->
