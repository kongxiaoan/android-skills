# Kotlin Flow State and Event Modeling 深度解析

对应 skill: [`kotlin-flow-state-event-modeling`](../skills/kotlin-flow-state-event-modeling/SKILL.md)

这一篇讲 Kotlin Flow 里状态和事件的建模：

> Pick the primitive that matches replay, fan-out, and synchronous-read requirements.

`StateFlow`、`SharedFlow`、`Channel.receiveAsFlow()` 和 cold `Flow` 的差异不是 API 风格，而是语义差异。

## 决策维度

选择前先问：

1. 是否始终有当前值？
2. 是否需要同步 `.value`？
3. 是否所有 collector 都要看到每次 emission？
4. emission 丢失是否是 bug？
5. 是 durable state，还是 one-shot event？
6. upstream sharing coroutine 应该何时启动/停止？

## Flow primitive 决策表

| Need | Primitive |
|---|---|
| 始终有当前值，异步 collect + 同步 `.value` | `StateFlow` |
| hot stream，多 subscriber，无同步 `.value` | `SharedFlow` |
| 单 consumer exactly-once handoff event | `Channel(BUFFERED).receiveAsFlow()` |
| 每次 collect 独立执行 | cold `Flow` |

## `StateFlow`

适合 state：

- 当前 UI state。
- 当前 session。
- 当前 preferences。
- 当前 selection。

特点：

- 必须有 initial value。
- 有 `.value`。
- conflated，保留最新值。
- collector 立即收到当前值。

### 不要用 fake sentinel 污染 domain

错误：

```kotlin
private val _user = MutableStateFlow<User>(NoUser)
val user: StateFlow<User> = _user.asStateFlow()
```

`NoUser` 如果不是真实 domain value，会污染所有 consumer。

修复一：显式建模 absence/loading。

```kotlin
sealed interface UserState {
    data object Loading : UserState
    data object SignedOut : UserState
    data class SignedIn(val user: User) : UserState
}
```

修复二：phase initialization。

```kotlin
class UserSession(private val db: Db) {
    private var _user: MutableStateFlow<User>? = null

    val user: StateFlow<User>
        get() = checkNotNull(_user) { "Call login() first" }

    suspend fun login() {
        _user = MutableStateFlow(db.load())
    }
}
```

如果 loading/absence/error 是真实状态，就显式建模。问题不是 initial value，而是假 domain value。

## `MutableStateFlow.update`

错误：

```kotlin
_state.value = _state.value.copy(selectedId = id)
```

这是 read-modify-write，多个 coroutine 并发更新时可能丢失变化。

正确：

```kotlin
_state.update { current ->
    current.copy(selectedId = id)
}
```

`update` lambda 可能重试，因此：

- 保持纯函数。
- 保持快速。
- 不做网络、数据库、日志、random、time reads。
- 不在里面构造不依赖 current 的昂贵对象。

```kotlin
val details = Details.from(response)

_state.update { current ->
    current.copy(details = details)
}
```

## `SharedFlow`

适合 hot broadcast stream，但默认 `MutableSharedFlow()`：

- replay = 0；
- no extra buffer；
- 没有 collector 时 emission 会丢。

所以不要无脑用于 navigation/snackbar：

```kotlin
private val _events = MutableSharedFlow<NavigationEvent>()
```

如果没有 collector 的瞬间发出 event，event 就没了。

如果丢 event 是 bug，并且只有一个 UI consumer，考虑 Channel。

## Channel-backed event Flow

单 consumer one-shot event：

```kotlin
private val _navEvents = Channel<NavigationEvent>(Channel.BUFFERED)
val navEvents: Flow<NavigationEvent> = _navEvents.receiveAsFlow()
```

语义：

- buffered handoff；
- collector 稍晚启动时仍可能收到 buffer 中 event；
- with multiple collectors 是 fan-out，不是 broadcast；
- 每个 event 只交给一个 collector；
- buffer bounded，send 可能 suspend，`trySend` 可能 fail。

如果多个 observer 必须都看到 event：

- 用 durable state；
- 或 deliberate `SharedFlow` 配置；
- 或持久化事件队列。

## `stateIn`

错误：

```kotlin
fun getPreferences(): StateFlow<Prefs> =
    repo.prefsFlow.stateIn(
        scope,
        SharingStarted.Eagerly,
        Prefs.Default,
    )
```

每次调用都会创建新的 sharing coroutine。

正确：

```kotlin
val preferences: StateFlow<Prefs> =
    repo.prefsFlow.stateIn(
        viewModelScope,
        SharingStarted.Eagerly,
        Prefs.Default,
    )
```

`stateIn` 通常应该是 property / cached shared instance，而不是普通 getter function 里每次创建。

## `SharingStarted.WhileSubscribed` 与 `.value`

`WhileSubscribed(timeout)` 没有 active collector 时会停止 upstream。此时 `.value` 只是最后缓存值，可能 stale，也可能仍是 initial value。

规则：

> 如果同步 `.value` 必须 fresh / initialized，不要依赖 `WhileSubscribed`。

选择：

- `SharingStarted.Eagerly`
- 显式初始化
- 避免同步 `.value`
- 接受 cached/stale 语义并明确文档化

## `.map` on StateFlow loses `.value`

```kotlin
val name: Flow<String> = userState.map { it.name }
```

结果是普通 `Flow`，没有 `.value`。

如果 consumer 需要同步读取：

```kotlin
val name: StateFlow<String> = userState
    .map { it.name }
    .stateIn(
        viewModelScope,
        SharingStarted.Eagerly,
        userState.value.name,
    )
```

不要以为 `StateFlow.map` 仍然是 state。

## 专家级审查清单

1. 这是 durable state 还是 one-shot event？
2. 是否需要同步 `.value`？
3. 是否允许 emission 在无 collector 时丢失？
4. 多 collector 时是 broadcast 还是 fan-out？
5. `MutableSharedFlow()` 默认配置是否符合事件语义？
6. `Channel.receiveAsFlow()` 是否只有一个 consumer？
7. `StateFlow` initial value 是否是假 domain sentinel？
8. `stateIn` 是否在 function 中每次调用创建？
9. `WhileSubscribed` 和 `.value` fresh requirement 是否冲突？
10. `_state.value = _state.value.copy()` 是否应该改 `update`？

## 精髓总结

1. `StateFlow` 是当前值，不是事件流。
2. `SharedFlow` 默认会丢无 collector 时的 emission。
3. `Channel.receiveAsFlow()` 是单 consumer handoff，不是 broadcast。
4. `stateIn` 应共享一次，不要每次调用创建。
5. `.value` fresh requirement 会影响 `SharingStarted` 选择。
