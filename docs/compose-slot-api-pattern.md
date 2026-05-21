# Compose Slot API Pattern 深度解析

对应 skill: [`compose-slot-api-pattern`](../skills/compose-slot-api-pattern/SKILL.md)

这一篇讲可复用 Compose 组件的内容扩展方式：

> Reusable component 的职责是描述布局结构，不是枚举所有可能内容。

当一个组件参数开始出现 `title: String`、`subtitle: String?`、`leadingIcon: ImageVector?`、`showChevron: Boolean`、`showSwitch: Boolean`、`trailingText: String?` 时，组件已经在枚举 call site，而不是提供可扩展结构。

## 核心原则

> Delegate variable visual regions to the caller through `@Composable` slots.

组件贡献：

- 哪些区域存在；
- 区域之间如何排列；
- spacing、alignment、semantics、interaction surface；
- 默认样式和约束。

调用方贡献：

- headline 里具体是什么；
- leading 是 icon、avatar、badge 还是自定义图形；
- trailing 是 chevron、switch、text、loading indicator 还是按钮组。

Material 3 `ListItem` 是典型例子：`headlineContent`、`supportingContent`、`leadingContent`、`trailingContent`、`overlineContent` 都是 slots。

## 反例：primitive content parameters

```kotlin
@Composable
fun SettingsRow(
    title: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    subtitle: String? = null,
    leadingIcon: ImageVector? = null,
    showChevron: Boolean = false,
    trailingText: String? = null,
) {
    // layout
}
```

短期看方便，长期问题明显：

- title 想加 badge，要改组件。
- leading 想用 avatar，不是 `ImageVector`，要改组件。
- trailing 想从 chevron 变 switch，要加 flag。
- subtitle 想变成 chips row，要改组件。
- 参数越来越多，组合越来越多，测试分支越来越多。

## 正确：把视觉区域变成 slots

```kotlin
@Composable
fun SettingsRow(
    headlineContent: @Composable () -> Unit,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    supportingContent: (@Composable () -> Unit)? = null,
    leadingContent: (@Composable () -> Unit)? = null,
    trailingContent: (@Composable () -> Unit)? = null,
) {
    Row(
        modifier = modifier.clickable(onClick = onClick),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        if (leadingContent != null) {
            Box(Modifier.padding(end = 16.dp)) {
                leadingContent()
            }
        }

        Column(Modifier.weight(1f)) {
            headlineContent()
            if (supportingContent != null) {
                supportingContent()
            }
        }

        if (trailingContent != null) {
            Box(Modifier.padding(start = 16.dp)) {
                trailingContent()
            }
        }
    }
}
```

Call site：

```kotlin
SettingsRow(
    headlineContent = { Text("Account") },
    supportingContent = { Text("Signed in") },
    leadingContent = {
        Icon(Icons.Default.Person, contentDescription = null)
    },
    trailingContent = {
        SettingsRowDefaults.Chevron()
    },
    onClick = onAccountClick,
)
```

特殊 call site 不需要改组件：

```kotlin
SettingsRow(
    headlineContent = {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("Inbox")
            Spacer(Modifier.width(8.dp))
            Badge { Text("3") }
        }
    },
    leadingContent = {
        Avatar(user.avatarUrl)
    },
    trailingContent = {
        Switch(checked = enabled, onCheckedChange = onEnabledChange)
    },
    onClick = onClick,
)
```

## Slot 命名

推荐：

- `headlineContent`
- `supportingContent`
- `leadingContent`
- `trailingContent`
- `overlineContent`

这是 Material 3 风格，适合自由视觉区域。

也可以用语义名：

```kotlin
Scaffold(
    topBar = { ... },
    bottomBar = { ... },
    floatingActionButton = { ... },
)
```

注意：

- 同一个组件里不要混用 `content` 和多个 `xxxContent` 造成语义混乱。
- Required slot 不给默认值。
- Optional slot 用 nullable，通常不要用空 lambda。

## Optional slot 用 nullable

不推荐：

```kotlin
leadingContent: @Composable () -> Unit = {}
```

推荐：

```kotlin
leadingContent: (@Composable () -> Unit)? = null
```

原因：空 lambda 仍然可能让组件保留 leading 区域、padding、spacer。`null` 明确表示这个区域不存在。

```kotlin
if (leadingContent != null) {
    LeadingSlot {
        leadingContent()
    }
}
```

## Scope receiver slot

如果 slot 被放进 `Row`，且 caller 需要 `RowScope` 能力，例如 `Modifier.weight`，应该声明：

```kotlin
actions: @Composable RowScope.() -> Unit
```

而不是：

```kotlin
actions: @Composable () -> Unit
```

示例：

```kotlin
@Composable
fun MyTopBar(
    title: @Composable () -> Unit,
    actions: @Composable RowScope.() -> Unit = {},
) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Box(Modifier.weight(1f)) {
            title()
        }
        actions()
    }
}
```

不要给所有 slot 都加 receiver。receiver 应该匹配真实 parent layout：

- inside `Row` -> `RowScope`
- inside `Column` -> `ColumnScope`
- inside `Box` -> `BoxScope`
- 不需要 scope 能力 -> 不加 receiver

## Defaults 放到 `XxxDefaults`

如果常见 slot 内容反复出现，不要把它做成 boolean flag：

```kotlin
showChevron: Boolean = true
```

更好：

```kotlin
object SettingsRowDefaults {
    @Composable
    fun Chevron() {
        Icon(
            imageVector = Icons.AutoMirrored.Filled.KeyboardArrowRight,
            contentDescription = null,
        )
    }
}
```

Call site：

```kotlin
SettingsRow(
    headlineContent = { Text("Notifications") },
    trailingContent = { SettingsRowDefaults.Chevron() },
    onClick = onClick,
)
```

这样默认内容可复用，但 slot 仍然开放。

## 什么时候不要 slot

不要机械 slot 化。

不适合：

- 单 use site 的 private composable。
- design-system primitive 故意统一外观，例如 `Heading2(text: String)`。
- `Switch(checked, onCheckedChange)` 这种语义参数，不是视觉内容枚举。
- 性能极端敏感的 framework-level hot path。
- 组件拥有文本、图标、语义以保证产品一致性。

判断：

> 这个区域是否由 caller 控制视觉内容？如果是，slot。  
> 这个参数是否是组件自身语义模型？如果是，保留 primitive。

## 与 modifier 规则的关系

可复用组件通常同时需要：

```kotlin
modifier: Modifier = Modifier
```

和 slots。

caller 既要能决定组件怎么放置，也要能决定可变视觉区域放什么。

详见：

[compose-modifier-and-layout-style.md](./compose-modifier-and-layout-style.md)

## 专家级审查清单

1. 组件是否计划复用，或已经有多个 call site？
2. 参数里是否有 `title: String`、`subtitle: String?`、`icon: ImageVector?` 这类 caller-controlled content？
3. 是否有 `showChevron`、`showSwitch`、`mode` 这类 shape flag？
4. 是否某个 slot 已经存在，但其他区域仍然 primitive？
5. Optional slot 是否用 nullable 表达缺席？
6. Slot 是否需要 `RowScope` / `ColumnScope` / `BoxScope` receiver？
7. 常见 slot 内容是否应该放到 `XxxDefaults`？
8. 是否把本应统一的 design-system primitive 过度 slot 化？

## 精髓总结

1. Reusable component 描述结构，不枚举内容。
2. Caller-controlled visual region 应该是 `@Composable` slot。
3. Optional slot 用 nullable，缺席就是结构上不存在。
4. Slot receiver 要匹配真实 layout scope。
5. 常见默认内容放 `XxxDefaults`，不要变成 flag soup。
