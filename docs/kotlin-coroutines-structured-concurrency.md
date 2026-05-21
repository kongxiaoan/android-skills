# Kotlin Coroutines Structured Concurrency 深度解析

对应 skill: [`kotlin-coroutines-structured-concurrency`](../skills/kotlin-coroutines-structured-concurrency/SKILL.md)

这一篇讲 Kotlin 协程架构中最重要的审查原则：

> 异步工作的生命周期应该由调用方或明确 lifecycle owner 决定，而不是由 callee 偷偷持有一个 scope 并 fire-and-forget。

## 核心原则

> A well-structured coroutine is a self-contained unit of asynchronous work: single entry, single exit, scoped to a lifecycle known at the call site.

通常，repository、manager、use case、data source 不应该存 `CoroutineScope`。

它们应该暴露：

```kotlin
suspend fun refresh(): User
```

而不是：

```kotlin
fun refresh() {
    scope.launch { ... }
}
```

因为后者让 callee 替 caller 做了生命周期、错误处理、取消语义的决定。

## Stored scope 的 silent-cancellation bug

危险代码：

```kotlin
class UserRepository(
    private val scope: CoroutineScope,
    private val api: UserApi,
) {
    fun refresh() {
        scope.launch {
            api.fetchUser()
        }
    }
}
```

问题不只是“scope 生命周期不清楚”，还有一个隐蔽 bug：

> 一旦这个 scope 被 cancel，之后所有 `launch` 都会立即以 cancelled 状态完成，没有异常、没有日志、没有工作执行。

调用方以为 `refresh()` 发起了工作，实际什么都没有发生。

如果 API 是 suspend：

```kotlin
class UserRepository(
    private val api: UserApi,
) {
    suspend fun refresh(): User = api.fetchUser()
}
```

caller 的 scope 是否 alive 由 caller 自己知道。取消也会沿调用链传播。

## 反模式一：repository / manager 存 scope

错误：

```kotlin
class AnalyticsClient(
    private val scope: CoroutineScope,
    private val api: AnalyticsApi,
) {
    fun track(event: Event) {
        scope.launch {
            api.send(event)
        }
    }
}
```

问题：

- caller 看不到成功/失败。
- caller 不能取消。
- caller 不知道工作是否启动。
- scope cancel 后 silent failure。

正确：

```kotlin
class AnalyticsClient(
    private val api: AnalyticsApi,
) {
    suspend fun track(event: Event) {
        api.send(event)
    }
}
```

UI state holder 可以决定：

```kotlin
fun onButtonClick() {
    viewModelScope.launch {
        analytics.track(Event.Clicked)
    }
}
```

## UI state holder carve-out

UI callback 通常不能 suspend：

```kotlin
Button(onClick = { viewModel.onSaveClick() })
```

所以 ViewModel / Component / feature state holder 可以把非 suspending UI event 转成 lifecycle-bound coroutine：

```kotlin
class ProfileViewModel(
    private val repository: ProfileRepository,
) : ViewModel() {
    fun onSaveClick() {
        viewModelScope.launch {
            repository.save()
        }
    }
}
```

这是合理边界，条件是：

1. 它真的是 UI surface 的 state holder。
2. scope 是 lifecycle-bound，例如 `viewModelScope`。
3. caller 真的是 UI event。

这不是给 repository / use case fire-and-forget 的许可。

## 反模式二：`init { scope.launch { ... } }`

错误：

```kotlin
class UserSession(
    private val scope: CoroutineScope,
    private val api: Api,
) {
    init {
        scope.launch {
            user = api.load()
        }
    }
}
```

构造函数返回时：

- load 不一定完成；
- error 不可见；
- caller 无法 await；
- lifecycle 不透明；
- 初始化顺序依赖 DI realization。

更清楚：

```kotlin
class UserSession(
    private val api: Api,
) {
    private var _user: User? = null
    val user: User get() = checkNotNull(_user) { "Call init() first" }

    suspend fun init() {
        _user = api.load()
    }
}
```

或者把加载建模为显式 `StateFlow`，由明确 owner 启动。

## 反模式三：内部创建 scope

错误：

```kotlin
class FooManager {
    private val scope = MainScope()
}
```

或：

```kotlin
fun foo() {
    CoroutineScope(Dispatchers.Default).launch { ... }
}
```

这比 injected scope 更糟：生命周期完全不可见。通常应改成 suspend API。

## DI singleton / Initializer 不能偷偷 launch

错误：

```kotlin
@Singleton
class TokenRefresher(
    private val appScope: CoroutineScope,
    private val auth: AuthService,
) {
    init {
        appScope.launch {
            while (isActive) {
                delay(5.minutes)
                auth.refreshIfNeeded()
            }
        }
    }
}
```

问题：

- start time 是 DI 何时创建它，不是业务定义。
- 谁在运行不可见。
- 错误不可见。
- stop/restart 不明确。
- grep 不到 launch site。

更好的模式：

```kotlin
class TokenRefresher(
    private val auth: AuthService,
) {
    suspend fun run() {
        while (currentCoroutineContext().isActive) {
            delay(5.minutes)
            auth.refreshIfNeeded()
        }
    }
}

applicationScope.launch {
    tokenRefresher.run()
}
```

launch site 显式出现在 startup orchestrator。

Initializer 可以注册东西，但不应该 launch：

```kotlin
class ContributorInitializer(
    private val registry: Registry,
    private val contributor: Contributor,
) : Initializer {
    override fun initialize() {
        registry.register(contributor)
    }
}
```

## CancellationException 必须传播

错误：

```kotlin
suspend fun fetch() {
    try {
        api.load()
    } catch (e: Exception) {
        logger.warn("load failed", e)
    }
}
```

`CancellationException` 是 `Exception` 子类。这样会吞掉取消。

正确：

```kotlin
try {
    api.load()
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    logger.warn("load failed", e)
}
```

或：

```kotlin
try {
    api.load()
} catch (e: Exception) {
    currentCoroutineContext().ensureActive()
    logger.warn("load failed", e)
}
```

`runCatching` 也有同样问题：

```kotlin
runCatching { api.load() }
    .onFailure {
        if (it is CancellationException) throw it
        logger.warn("load failed", it)
    }
```

窄 catch 非取消异常是可以的：

```kotlin
catch (e: IOException) { ... }
```

## `runBlocking`

`runBlocking` 会阻塞当前线程直到 coroutine 完成。它是同步边界工具，不是 application code 中桥接 suspend 的常规方式。

错误：

```kotlin
fun saveUser(user: User) {
    runBlocking {
        repository.save(user)
    }
}
```

正确：

```kotlin
suspend fun saveUser(user: User) {
    repository.save(user)
}
```

测试中用 `runTest`：

```kotlin
@Test
fun loadsUser() = runTest {
    assertThat(repository.load().name).isEqualTo("Ada")
}
```

`runBlocking` 合理边界：

- CLI `main`。
- 必须同步返回的 Java interop 边界。
- 少数 framework 同步 callback。
- Android `ContentProvider` 成员函数如 `query` / `insert` / `update` / `delete`。

即便合理，也要 body 最小化，立即调用 suspend API。

## 专家级审查清单

1. 非 UI class 是否存了 `CoroutineScope`？
2. public non-suspend function 是否内部 `scope.launch`？
3. `init` / constructor / initializer 是否 launch？
4. scope 是否 ad-hoc 创建？
5. launch site 是否可 grep、可观察、可停止？
6. UI state holder carve-out 是否真的满足三条件？
7. broad catch 是否吞了 `CancellationException`？
8. `runCatching` 是否有 cancellation guard？
9. `runBlocking` 是否出现在 suspend-capable app path 或测试中？

## 精髓总结

1. Repository / use case / data source 暴露 suspend API，让 caller 选择 scope。
2. UI state holder 是非 suspend UI event 到 coroutine 的合法边界。
3. 不要从 `init`、DI singleton、Initializer 偷偷 launch。
4. broad catch 必须传播 `CancellationException`。
5. `runBlocking` 只属于真实同步边界，测试用 `runTest`。
