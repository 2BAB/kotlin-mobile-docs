[//]: # (title: Concurrency and coroutines)
[//]: # (auxiliary-id: Concurrency_and_Coroutines)

开发移动平台应用时，可能需要编写多线程代码并行执行。你可以使用[标准库](#coroutines) 提供的 `kotlinx.coroutines` 库，或者这个库的[多线程版本](#multithreaded-coroutines)，再或者使用[其他方案](#alternatives-to-kotlinx-coroutines)。

权衡每个方案的利弊，选择一个最适合你场景的。

了解更多关于 [KMM 并发的现状，和将要进行的改进](concurrency-overview.md)。

<a id="coroutines"></a>
## 协程

协程是轻量化的线程，你可以使用它编写异步的非阻塞代码。Kotlin 提供的 [`kotlinx.coroutines`](https://github.com/Kotlin/kotlinx.coroutines) 
库有许多支持协程的高级原语。

当前可以用于 iOS 版本的 `kotlinx.coroutines` 仅支持在单线程内的操作。你不能通过改变 [dispatcher](#dispatcher-for-changing-threads) 把任务调度到其他线程。 

Kotlin %kotlinVersion%， 推荐使用协程版本 `%coroutinesVersion%`。

这个版本的 `kotlinx.coroutines` 还不能自己切换线程。你可以挂起正在执行的任务，使用另外的机制，在其他线程上调度并管理任务。

其实还有[一个支持多线程版本](#多线程版本协程)的 `kotlinx.coroutines`。

了解一下使用协程相关的几个主要概念：

* [异步 vs. 并行处理](#asynchronous-vs-parallel-processing)
* [使用 Dispatcher 切换线程](#dispatcher-for-changing-threads)
* [冻结捕获的数据](#frozen-captured-data)
* [冻结返回的数据](#frozen-returned-data)

<a id="asynchronous-vs-parallel-processing"></a>
### 异步 vs 并行处理

异步和并行处理不同。

在协程中，任务序列可能会被挂起，稍后恢复执行。利用这点，我们不需要 callback 或者 promise 就可以编写异步，非阻塞代码。这就是异步处理，所有与这个协程相关联的可以发生在同一个线程上。

下面这个例子使用 [Ktor](https://ktor.io/) 执行了一个网络请求。在主线程，请求被初始化并挂起，再由另一个底层程序执行实际网络请求。请求完成后，主线程代码恢复执行。

```kotlin
val client = HttpClient()
// 运行在主线程，开始一个 `get` 调用
client.get<String>("https://example.com/some/rest/call")
// get 调用会被挂起，让其他任务在主线程执行，等到 get 调用完成后，再恢复执行
```

并行代码和这个不一样，它需要在其他线程上执行。取决于你的目的和使用的库，你可能永远都不需要多线程。

<a id="dispatcher-for-changing-threads"></a>
### 使用 Dispatcher 切换线程

协程由 dispatcher 执行，dispatcher 可以控制协程运行在哪个线程上。
有许多方法指定或者改变协程的 dispatcher。比如：

```kotlin
suspend fun differentThread() = withContext(Dispatchers.Default){
    println("Different thread")
}
```

`withContext` 接收两个参数，dispatcher 和一个代码块，代码块的内容会由 dispatcher 指定的线程执行。了解更多关于[协程的上下文和 dispatcher](https://kotlinlang.org/docs/reference/coroutines/coroutine-context-and-dispatchers.html)。

要在不同的线程上执行任务，需要指定不同的 dispatcher 和代码块来执行。一般来说，切换 dispatcher 和线程的工作方式与 JVM 类似，但在处理冻结捕获和返回的数据时有所不同。

<a id="frozen-captured-data"></a>
### 冻结捕获的数据

我们传递一个 `functionBlock` 参数，希望在其他线程上执行这段代码，这个 block 先会被冻结，之后才能在其他线程上执行。

```kotlin
fun <R> runOnDifferentThread(functionBlock: () -> R)
```

需要像下面这样调用：

```kotlin
runOnDifferentThread {
    // 代码运行在其他线程之上
}
```

[Concurrency 概览](concurrency-overview.md)中提到过，在 Kotlin/Native 中，线程间共享的状态一定会被冻结。一个函数参数本身就是一个状态，会被冻结，并且任何被它捕获的状态也会被冻结。

跨线程的协程函数遵循同样的模式。函数块会被冻结，以便在其他线程上执行。

下面的示例中，数据类实例 `dc` 会被函数块捕获，并在跨线程时冻结。`println` 语句会输出 `true`。

```kotlin
val dc = DataClass("Hello")
withContext(Dispatchers.Default) {
    println("${dc.isFrozen}")
}
```

当执行并行代码，需要小心对待被捕获的状态。
有时可以很轻易的看出状态被捕获，但并不总是如此。比如：

```kotlin
class SomeModel(val id:IdRec){
    suspend fun saveData() = withContext(Dispatchers.Default){
        saveToDb(id)
    }
}
```

`saveData` 内的代码运行在其他线程之上。 `id` 会被冻结，同时由于 `id` 是其父类的一个属性，所以它的父类也会被冻结。

<a id="frozen-returned-data"></a>
### 冻结返回的数据

从不同线程返回的数据也是被冻结的。虽然我们推荐返回不可变数据，但是你仍然可以选择返回可变状态，只不过这个返回值就不允许修改了。

```kotlin
val dc = withContext(Dispatchers.Default) {
    DataClass("Hello Again")
}

println("${dc.isFrozen}")
```

假设一个可变状态独立存在于某个线程之中，并且使用了协程的线程操作进行通信。如果你返回的数据持有了这个可变状态的引用，这个可变状态也会因为被关联而被冻结，这会是一个问题。

了解更多关于[线程隔离的状态](concurrent-mutability.md#thread-isolated-state)。


<a id="multithreaded-coroutines"></a>
## 多线程版本协程

`kotlinx.coroutines` 有一个[特殊分支](https://github.com/Kotlin/kotlinx.coroutines/tree/native-mt)，提供了对多线程的支持。[这篇文章](https://blog.jetbrains.com/kotlin/2020/07/kotlin-native-memory-management-roadmap/)介绍了它被单独放到一个分支里的原因。

不过你仍然可以在生产环境中使用支持多线程的 `kotlinx.coroutines`，考虑到它的具体情况。

适配 Kotlin %kotlinVersion% 的版本号是 `%coroutinesVersion%-native-mt`

使用多线程版本需要在 `commonMain` 目录下的 `build.gradle.kts` 文件里声明依赖：

```kotlin
commonMain {
    dependencies {
        implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%-native-mt"
    }
}
```

如果你使用了其他依赖了 `kotlinx.coroutines` 的三方库，比如 Ktor，请确保指定了多线程版本 `kotlinx-coroutines`。你可以使用 `strictly` 进行限制：

```kotlin
implementation ("org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%-native-mt"){
    version {
        strictly("%coroutinesVersion%-native-mt")
    }
}
```

因为单线程的 `kotlinx.coroutines` 是主版本，大部分三方库都会依赖这个版本。如果你在进行协程相关操作时，遇到了 `InvalidMutabilityException` 异常，很可能是因为你使用了版本不匹配的协程库。

> 使用多线程的协程可能会导致 _内存泄漏_ 。这在一定负载下的复杂协程场景可能是个问题。
> 我们正在解决这个问题。
>
{type="note"}

查看[在 KMM 应用中使用多线程版本协程的完整示例](https://github.com/touchlab/KaMPKit)。

<a id="alternatives-to-kotlinx-coroutines"></a>
## `kotlinx-coroutines` 之外的备选方案

除了协程，还有几种备选方案可以并行执行代码。

### CoroutineWorker

[`CoroutinesWorker`](https://github.com/Autodesk/coroutineworker) 是 AutoDesk 发布的一个库，它使用单线程版本的 `kotlinx.coroutines` 实现了一些协程跨线程的特性。

对于简单的 suspend 函数，这是一个很好的选择，但它不支持 Flow 和其他结构。

### Reaktive

[Reaktive](https://github.com/badoo/Reaktive) 是一个类似 Rx 的库，为 Kotlin Multiplatform 实现了 Reactive 扩展。 
它提供一些协程扩展，但主要还是围绕 RX 和线程进行设计的。

### 自定义 processor

对于简单的后台任务，你可以基于具体平台 API 进行一层封装，创建自己的 processor。
[一个简单的示例](https://github.com/touchlab/KMMWorker)。

### Platform concurrency

在生产中，你也可以依靠平台来处理并发性。
如果共享的 Kotlin 代码将被用于业务逻辑或数据操作，而不是用于架构，这可能会有帮助。

要在 iOS 中跨线程共享一个状态，该状态需要被[冻结](concurrency-overview.md#immutable-and-frozen-state)。这里提到的并发库 
会自动冻结你的数据。你很少需要明确地这样做。

如果你返回给 iOS 平台的数据会在线程间共享，要确保在离开 iOS 边界前，数据是被冻结的。

Kotlin 只在 Kotlin/Native 相关的平台上（比如 iOS）有冻结这个概念。为了能在 common 代码中使用 `freeze`，可以使用 expect 和 actual 实现 `freeze`，
或者使用 [`stately-common`](https://github.com/touchlab/Stately#stately-common)，它提供了这个功能。
在 Kotlin/Native 中，`freeze` 会冻结你的状态，在 JVM 上什么都不做。

使用 `stately-common` 需要在 `commonMain` 目录下的 `build.gradle.kts` 文件里声明依赖：

```kotlin
commonMain {
    dependencies {
        implementation "co.touchlab:stately-common:1.0.x"
    }
}
```

_This material was prepared by [Touchlab](https://touchlab.co/) for publication by JetBrains._
