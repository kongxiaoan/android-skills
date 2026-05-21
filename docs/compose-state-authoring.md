# Compose State Authoring 深度解析

对应 skill: [`compose-state-authoring`](../skills/compose-state-authoring/SKILL.md)

这个 skill 不是简单讲 `remember { mutableStateOf(...) }` 的用法，而是在讲 Compose 状态编写的两个底层问题：

1. `@Composable` 是会被 runtime 反复执行的 UI 描述函数，不是传统 View 对象实例。
2. Compose runtime 只会追踪它知道的 State 读写，普通 Kotlin 变量和普通对象内部 mutation 不会自动触发重组。

理解这两个点，才能判断一个状态是否有正确的生命周期、是否能触发 UI 更新、是否被放在了正确的 owner 里。

## Composable 不是传统 View 对象

错误示例：

```kotlin
@Composable
fun Counter() {
    var count = 0

    Button(onClick = { count++ }) {
        Text("$count")
    }
}
```

直觉上，这看起来像 `Counter` 创建了一个按钮，`count` 是这个按钮旁边的局部状态。但 Compose 并不是这样工作。

`Counter()` 是一个可能被 runtime 反复调用的函数。每次重组，函数体都会重新执行：

```kotlin
var count = 0
```

也会重新执行。因此这个 `count` 没有 UI 生命周期，它只是当前函数调用栈上的局部变量。点击时即使 `count++` 修改了值，也不会通知 Compose 需要重组；下一次由于任何原因触发重组时，它又会回到 `0`。

这里同时存在两个问题：

- 它不能 survive recomposition。
- 它不能 trigger recomposition。

正确写法：

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("$count")
    }
}
```

这行代码里有两个机制，而且二者都必要：

- `remember { ... }` 让值跟随 composition slot 存活，而不是跟随函数调用栈存活。
- `mutableStateOf(...)` 让值进入 Compose snapshot 系统，写入时可以 invalidate 读过它的 composition scope。

少一个都不完整。

例如：

```kotlin
val items = remember { mutableListOf<String>() }
```

这个 list 可以 survive recomposition，但 `items.add(...)` 不会自动触发重组。

再例如：

```kotlin
var count by mutableStateOf(0)
```

如果它直接写在 composable body 里，没有 `remember`，每次重组都会重新创建新的 state object，也不正确。

专家级判断不是只看“有没有 remember”，而是问：

> 这个状态是否既有正确生命周期，又能被 Compose 正确观察？

## `remember` 是 slot 记忆，不是普通缓存

`remember` 保存的是某个 composition 位置上的值。它不是通用 cache，也不是业务状态容器。

例如：

```kotlin
@Composable
fun Screen(showHeader: Boolean) {
    if (showHeader) {
        Header()
    }

    var text by remember { mutableStateOf("") }

    TextField(value = text, onValueChange = { text = it })
}
```

通常你不需要手动管理 composition slot，但要知道 `remember` 的 identity 和它在 composition tree 中的位置有关。

在动态列表、条件分支、可重排内容里，如果数据会移动，就要给 runtime 稳定 identity。列表中尤其要使用稳定 key：

```kotlin
LazyColumn {
    items(
        items = users,
        key = { it.id },
    ) { user ->
        var expanded by remember { mutableStateOf(false) }

        UserRow(
            user = user,
            expanded = expanded,
            onToggleExpanded = { expanded = !expanded },
        )
    }
}
```

没有稳定 key 时，如果列表重排，remembered state 可能跟着位置走，而不是跟着数据走。结果就是 A item 的展开状态跑到 B item 身上。

核心理解：

> `remember` 记住的是 composition 位置上的值。数据会移动时，要给 runtime 稳定 identity。

## Compose 观察的是 State 读写，不是对象内部 mutation

常见错误：

```kotlin
var users by remember { mutableStateOf(mutableListOf<User>()) }

fun addUser(user: User) {
    users.add(user)
}
```

这段代码经常不会按预期更新 UI。

原因是 Compose 观察到的是 `users` 这个 `MutableState` 的 `.value` 被读取。可是 `users.add(user)` 修改的是 list 内部内容，没有调用 State setter，因此 Compose 不知道状态已经变化。

正确选择通常有两类。

第一类：使用 immutable value replacement。

```kotlin
var users by remember { mutableStateOf<List<User>>(emptyList()) }

fun addUser(user: User) {
    users = users + user
}
```

这种写法简单、可预测、易测试。每次更新都会替换 State value，因此 Compose 能明确观察到变化。

第二类：使用 snapshot-aware collection。

```kotlin
val users = remember { mutableStateListOf<User>() }

fun addUser(user: User) {
    users.add(user)
}
```

`mutableStateListOf` 和 `mutableStateMapOf` 的 mutation 是 snapshot-aware 的，Compose 可以观察到它们的读写。

审查规则：

- 想要简单、可预测、易测试，优先 immutable list + replace。
- 需要频繁局部修改、列表较大、API 更自然时，可以使用 `mutableStateListOf`。
- 避免 `mutableStateOf(mutableListOf())` 后再直接 `.add()` / `.remove()`，这是一种半吊子的状态建模。

## 本地 UI state 和业务 state 要分清

`compose-state-authoring` 处理的是 local UI state。

适合本地 `remember` 的状态通常是：

- 一个 section 是否展开。
- 当前 tab。
- 文本框正在编辑的临时值。
- dialog 是否显示。
- dropdown 是否展开。
- 局部 hover、focus、pressed 交互状态。

例如：

```kotlin
@Composable
fun FilterChipGroup() {
    var selected by remember { mutableStateOf(Filter.All) }

    FilterChip(
        selected = selected == Filter.All,
        onClick = { selected = Filter.All },
    )
}
```

但如果状态代表业务事实，就不应该凭空放在 composable 内部。典型业务状态包括：

- 当前登录用户。
- 购物车内容。
- 远端加载结果。
- 页面数据。
- 权限状态。
- 保存到数据库的数据。

这些状态通常应该由 ViewModel、state holder、repository 或调用方拥有，再传入 UI：

```kotlin
@Composable
fun ProfileScreen(
    state: ProfileUiState,
    onRefresh: () -> Unit,
    onNameChange: (String) -> Unit,
) {
    // Render only.
}
```

核心判断：

> `remember` 管的是这个 composable 实例的 UI 记忆，不是应用状态管理方案。

一旦状态需要被其他地方知道、需要恢复、需要测试业务逻辑、需要跨屏幕、需要持久化，它就不该只是 composable local state。

## `rememberSaveable` 解决保存，不解决所有权

配置变化后需要保留简单 UI 状态时，可以使用：

```kotlin
var query by rememberSaveable { mutableStateOf("") }
```

`rememberSaveable` 会通过 `SaveableStateRegistry` 保存 Bundle 兼容的值。它适合：

- `String`
- `Int`
- `Boolean`
- enum
- `Parcelable`
- `Serializable`
- 自定义 `Saver`

但它只解决 configuration change / process recreation 的保存问题，不改变状态所有权。

`rememberSaveable` 不等于 ViewModel state。判断方式：

- “旋转后不想丢”不自动意味着 ViewModel。
- “业务逻辑依赖它 / 其他模块需要它 / 需要加载保存同步”才更像 ViewModel 或 state holder。

## State 读的位置决定重组范围

Compose snapshot 系统会记录 state 在哪里被读。

```kotlin
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("$count")
        HeavyChild()
    }
}
```

`count` 在 `Parent` 的 composition 中被读。`count` 改变时，读它的 scope 会被 invalidated。

一种更清晰的拆分是：

```kotlin
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }

    Column {
        CountText(count)
        HeavyChild()
    }
}

@Composable
fun CountText(count: Int) {
    Text("$count")
}
```

这并不是要求永远拆小函数，而是要理解：

> State read 越靠上，可能影响的重组范围越大。

普通点击状态、小 UI 状态可以正常本地读写。但高频状态，例如 scroll offset、animation progress、gesture position，应该避免在大型 composable 顶层读取。这是后续 `compose-state-deferred-reads` 和性能诊断的基础。

## `derivedStateOf` 不是普通 computed property

很多计算不需要 `derivedStateOf`：

```kotlin
val fullName = "$firstName $lastName"
```

这种普通派生值通常直接计算即可。

`derivedStateOf` 的价值在于：输入变化很频繁，但输出并不总是变化时，减少下游 invalidation。

典型例子：

```kotlin
val showScrollToTop by remember {
    derivedStateOf {
        listState.firstVisibleItemIndex > 0
    }
}
```

`firstVisibleItemIndex` 可能频繁变化，但 `showScrollToTop` 只有在 `false -> true` 或 `true -> false` 时才值得影响 UI。

不要把所有计算都包进 `derivedStateOf`。它有成本，也会增加代码复杂度。

## `@ReadOnlyComposable` 是 runtime contract

`@ReadOnlyComposable` 适合只读取 composition 并返回值的 accessor。

例如：

```kotlin
val AppColors.primary
    @Composable
    @ReadOnlyComposable
    get() = MaterialTheme.colorScheme.primary
```

或者：

```kotlin
@Composable
@ReadOnlyComposable
fun appSpacing(): Dp {
    return LocalSpacing.current.medium
}
```

它表示这个 composable 只读 composition state，不创建 UI 节点，不调用 `remember`，不注册 side effect，不创建 positional slot。这样 runtime 可以避免为该调用创建普通 composable group。

它不是“这个函数很简单”的注解，而是严格 contract。

错误示例：

```kotlin
@Composable
@ReadOnlyComposable
fun Header() {
    Text("Hello")
}
```

`Text` 会 emit layout node，因此不是 read-only。

错误示例：

```kotlin
@Composable
@ReadOnlyComposable
fun rememberThing(): Thing {
    return remember { Thing() }
}
```

`remember` 需要 positional slot，也不是 read-only。

判断标准：

> 这个函数是否只读取 composition 并返回值？如果它参与构建 UI、记忆状态、注册副作用、调用普通 composable，就不能标 `@ReadOnlyComposable`。

## Review 红旗

看到这些写法时应该警觉：

```kotlin
var x = ...
```

出现在 `@Composable` body 或 `Column {}` / `Row {}` 这类 composable lambda 里。

看到这些写法也应该警觉：

```kotlin
remember { mutableStateOf(mutableListOf()) }
```

后续如果出现 `.add()` / `.remove()`，大概率绕过了 State setter。

看到这些写法要追问状态所有权：

```kotlin
remember { mutableStateOf(repository.load()) }
```

这可能把业务加载或业务事实塞进了 UI local state。

看到 `@ReadOnlyComposable` 时要检查 contract：

- 是否调用了 `Text`、`Box`、`Column`、`LazyColumn` 等 layout/UI composable。
- 是否调用了 `remember`。
- 是否调用了 effect API。
- 是否调用了普通非 read-only composable。
- 是否调用了 composable lambda，例如 `content()`。

## 精髓总结

`compose-state-authoring` 可以压缩成四条专家规则：

1. Composable local variable 没有 UI 生命周期。要 survive recomposition，用 `remember`。
2. 普通 Kotlin mutation 不会自动通知 Compose。要 trigger recomposition，用 `mutableStateOf`、snapshot collection，或者替换 immutable value。
3. 本地 state 只适合本地 UI 记忆。业务状态、共享状态、页面状态应该上提，不要用 `remember` 假装架构。
4. `@ReadOnlyComposable` 是底层 contract。只读 composition 的 accessor 可以标；任何 emit UI、`remember`、effect 的函数都不能标。
