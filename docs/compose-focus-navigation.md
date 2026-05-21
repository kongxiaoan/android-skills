# Compose Focus Navigation 深度解析

对应 skill: [`compose-focus-navigation`](../skills/compose-focus-navigation/SKILL.md)

这一篇讲 Compose 中键盘、TV、D-pad、桌面、ChromeOS 和 accessibility 场景下的 focus 行为。

核心问题不是“怎么让某个节点变大/变亮”，而是：

> Focus 是一种 stateful UI behavior，必须有明确 target、明确请求时机、明确导航规则，并用真实输入模型测试。

## 核心原则

> Make focus targets explicit, request focus after composition succeeds, and test navigation with the same input model users use.

Focus 相关代码通常涉及：

- `FocusRequester`
- `Modifier.focusRequester(...)`
- `Modifier.focusProperties { ... }`
- `Modifier.onFocusChanged { ... }`
- `Modifier.focusable()`
- key events
- lazy list/grid focus restoration
- TV / D-pad navigation

## Focus target 不是所有节点都要加

优先使用已经 focusable 的组件：

- `Button`
- `TextField`
- clickable/selectable surface
- Material interactive components

不要给所有 `Box`、`Row`、`Column` 都加 `focusable()`。

规则：

| Need | Add |
|---|---|
| 普通 button/text field/clickable focus | 通常不需要额外加 |
| 程序化请求初始/恢复 focus | `FocusRequester` + `focusRequester` |
| 根据 focus 改变视觉状态 | `onFocusChanged` |
| 自定义交互 surface 本身不 focusable | `focusable()` + role / semantics |

示例：

```kotlin
val requester = remember { FocusRequester() }
var isFocused by remember { mutableStateOf(false) }

Button(
    onClick = onClick,
    modifier = Modifier
        .focusRequester(requester)
        .onFocusChanged { state -> isFocused = state.isFocused },
) {
    Text("Play")
}
```

只在需要 request 或 observe 时加对应 hook。

## 请求 focus 必须在 effect 中

错误：

```kotlin
@Composable
fun Screen() {
    val requester = remember { FocusRequester() }
    requester.requestFocus()
}
```

Composable body 可以被多次执行、跳过、放弃。focus request 是 imperative side effect，必须放到 effect。

正确：

```kotlin
val requester = remember { FocusRequester() }

LaunchedEffect(requester) {
    requester.requestFocus()
}
```

如果目标是加载后才出现：

```kotlin
LaunchedEffect(items.isNotEmpty()) {
    if (items.isNotEmpty()) {
        firstItemRequester.requestFocus()
    }
}
```

key 不能无脑用 `Unit`。如果目标出现条件变化，effect lifecycle 应该跟随这个条件。

## Lazy list/grid 中的 focus identity

错误倾向：

```kotlin
val requesters = remember { mutableMapOf<Int, FocusRequester>() }
```

用 index 作为 identity 在列表插入、删除、排序时会出错。

更稳：

```kotlin
val requesters = remember { mutableMapOf<ItemId, FocusRequester>() }

items(
    items = items,
    key = { it.id },
) { item ->
    val requester = requesters.getOrPut(item.id) { FocusRequester() }
    ItemCard(
        item = item,
        modifier = Modifier.focusRequester(requester),
    )
}
```

Focus restoration 应该基于 semantic identity：

- 记录 focused item id，而不是 index。
- lazy list 使用 stable `key`。
- refresh 后如果同 id 还存在，恢复它。
- 如果不存在，选择 deterministic fallback：nearest neighbor、first item、parent container。

## Directional navigation

默认 spatial search 通常够用。只有当设计需要特殊跳转或 trap 时，才用 `focusProperties`。

```kotlin
Modifier.focusProperties {
    up = headerRequester
    down = firstRowRequester
    left = FocusRequester.Cancel
}
```

使用原则：

- 少用 hard-coded focus links。
- focus graph 应随 layout 变化维护。
- modal / dialog / carousel / grid 可以有明确 focus trap。
- 过多 `focusProperties` 会让布局变化后 focus graph 过期。

## Key event

只为非默认行为处理 key。

```kotlin
Modifier.onPreviewKeyEvent { event ->
    if (event.type == KeyEventType.KeyUp && event.key == Key.Back) {
        onBack()
        true
    } else {
        false
    }
}
```

返回值很重要：

- `true` 表示已消费，parent 和默认行为不会再处理。
- `false` 表示未消费，事件继续传播。

不要：

```kotlin
Modifier.onPreviewKeyEvent { true }
```

这会破坏 text input、accessibility shortcuts、parent navigation。

## Focus 与 selection 不是一回事

TV UI 里常见混淆：

- focus：当前输入焦点在哪里。
- selection：当前业务选中项是什么。

一个列表项可能 focused 但未 selected，也可能 selected 但当前 focus 在按钮上。

不要用 selected state 推断 focus。需要 focus 就用 focus semantics / `onFocusChanged`。

## 测试 focus

测试应该使用真实输入模型：

```kotlin
composeTestRule.onNodeWithTag("screen").performKeyInput {
    pressKey(Key.DirectionDown)
}

composeTestRule.onNodeWithTag("play-button").assertIsFocused()
```

不要只用：

```kotlin
performClick()
```

来证明 TV / keyboard / D-pad 导航正确。点击能工作不代表焦点导航能工作。

断言 focus ownership 用 semantics：

```kotlin
assertIsFocused()
```

截图测试只适合验证 focus ring / scale / color 等视觉表现，不适合作为 focus ownership 的唯一证据。

## 常见错误

1. 给每个按钮都加 `focusRequester` 和 `onFocusChanged`。
2. 在 composable body 中调用 `requestFocus()`。
3. 初始 focus 用 `LaunchedEffect(Unit)`，但目标晚于数据加载才出现。
4. lazy list focus requester 用 index 存储。
5. 所有方向都手写 `focusProperties`，导致布局变化后 stale graph。
6. key handler 对所有 key 返回 `true`。
7. TV UI 测试只用 click，不发 key input。

## 专家级审查清单

1. 这个 UI 是否 keyboard / TV / desktop / D-pad first？
2. focus target 是否是实际 interactive element？
3. 是否过度添加 `focusable()`？
4. `requestFocus()` 是否在 effect 中？
5. effect key 是否跟随目标出现条件？
6. lazy list/grid 是否用 stable item id 管 focus？
7. `focusProperties` 是否只覆盖必要 edge？
8. key handler 是否只 consume 自己处理的 key？
9. 测试是否用 key input 驱动，并 assert focus semantics？

## 精髓总结

1. Focus 是 UI 行为状态，不是纯视觉状态。
2. 只给真正需要的节点加 focus hooks。
3. Focus request 是 side effect，放在 `LaunchedEffect`。
4. Lazy content 用 stable id 恢复 focus，不用 index。
5. Keyboard / TV UI 必须用 key input 测试。
