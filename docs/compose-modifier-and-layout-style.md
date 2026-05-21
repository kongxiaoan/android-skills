# Compose Modifier and Layout Style 深度解析

对应 skill: [`compose-modifier-and-layout-style`](../skills/compose-modifier-and-layout-style/SKILL.md)

这一篇讲 Compose 可复用 UI API 的基础规范：

> 一个会 emit layout 的 composable，应该让 parent 决定它如何被放置。

这听起来像风格问题，实际上是 API ownership 问题。`modifier` 参数、modifier 顺序、root layout 是否硬编码尺寸、条件布局放在哪里，都会影响组件能否复用、能否参与 deferred reads、能否被父级组合。

## 核心原则

> A composable that emits layout is a leaf the parent places.

也就是：

- composable 自己负责内部结构；
- parent 负责位置、尺寸、padding、alignment、weight；
- composable 必须暴露 `modifier: Modifier = Modifier`；
- caller-provided modifier 应该应用到 root；
- 不要在 root 上硬编码父级布局决策。

## 为什么 `modifier` 是 API contract

错误：

```kotlin
@Composable
fun HomeScreenHeader(title: String, subtitle: String) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp),
    ) {
        Text(title)
        Text(subtitle)
    }
}
```

问题：

- caller 不能决定宽度。
- caller 不能决定 padding。
- caller 不能加 test tag、semantics、offset、graphicsLayer。
- caller 无法把 deferred read 放到 modifier 里。
- 第二个 use site 出现时只能改组件源码。

正确：

```kotlin
@Composable
fun HomeScreenHeader(
    title: String,
    subtitle: String,
    modifier: Modifier = Modifier,
) {
    Column(modifier = modifier) {
        Text(title)
        Text(subtitle)
    }
}
```

调用方：

```kotlin
HomeScreenHeader(
    title = title,
    subtitle = subtitle,
    modifier = Modifier
        .fillMaxWidth()
        .padding(horizontal = 16.dp),
)
```

## `modifier` 参数位置与命名

规范：

```kotlin
@Composable
fun Foo(
    required: Required,
    modifier: Modifier = Modifier,
    optional: Optional = Optional.Default,
    content: @Composable () -> Unit,
)
```

常见规则：

- 名字就是 `modifier`，不是 `mod`、`m`、`wrapperModifier`。
- 默认值是 `Modifier`。
- 通常放在 required params 后、optional params / content lambdas 前。
- 只要 composable emit layout，就应该有这个参数。

不需要 `modifier` 的情况：

- `@ReadOnlyComposable` token accessor。
- 只返回 `Color` / `Dp` / typography token 的 composable。
- `@Preview` entry point。
- test-only throwaway composable。

## 应用到 root，并且放在链首

错误一：声明了但没用。

```kotlin
@Composable
fun Avatar(url: String, modifier: Modifier = Modifier) {
    Image(
        painter = rememberAsyncImagePainter(url),
        contentDescription = null,
    )
}
```

错误二：应用到 child，不是 root。

```kotlin
@Composable
fun Avatar(url: String, modifier: Modifier = Modifier) {
    Box {
        Image(
            painter = rememberAsyncImagePainter(url),
            contentDescription = null,
            modifier = modifier,
        )
    }
}
```

错误三：caller modifier 放在最后。

```kotlin
Image(
    painter = painter,
    contentDescription = null,
    modifier = Modifier
        .clip(CircleShape)
        .size(48.dp)
        .then(modifier),
)
```

正确：

```kotlin
Image(
    painter = painter,
    contentDescription = null,
    modifier = modifier
        .clip(CircleShape)
        .size(48.dp),
)
```

Modifier 链中更早的 segment 是更外层 wrapper。caller modifier 放在前面，才能让 parent 先决定这个组件如何被约束和放置。

## 哪些 modifier 属于 parent，哪些属于组件 identity

Parent 应该决定：

- `fillMaxWidth`
- `fillMaxSize`
- `padding`，尤其是 screen spacing
- `weight`
- `align`
- `offset`
- `testTag`
- 外部 semantics
- scroll / animation deferred read modifier

组件自己可以保留属于 identity 的 modifier：

- avatar 的 `clip(CircleShape)`
- icon button 的最小 touch target
- divider 的 intrinsic thickness
- component 内部固定 shape / border

判断：

> 能不能想象一个 caller 合理地想去掉这个 modifier？

如果能，放到 caller。

## Modifier chain 构造

避免：

```kotlin
var m = Modifier
m = m.padding(16.dp)
m = m.fillMaxSize()
Box(m)
```

正确：

```kotlin
val m = Modifier
    .padding(16.dp)
    .fillMaxSize()

Box(m)
```

短链可以 inline：

```kotlin
Box(modifier = Modifier.fillMaxWidth())
```

三段及以上建议 multiline：

```kotlin
Box(
    modifier = modifier
        .fillMaxSize()
        .padding(16.dp)
        .weight(1f),
)
```

条件段不要用 `var`，用 `.then(...)`：

```kotlin
Box(
    modifier = modifier
        .fillMaxWidth()
        .then(if (selected) Modifier.background(Color.Red) else Modifier),
)
```

## 单条件 layout 的位置

错误：

```kotlin
Column {
    if (showHeader) {
        Text("Title")
        Text("Subtitle")
    }
}
```

如果 `Column` 没有其他内容、没有 modifier、没有 alignment/arrangement 等容器语义，它只是为了包住 `if`。更清楚：

```kotlin
if (showHeader) {
    Column {
        Text("Title")
        Text("Subtitle")
    }
}
```

不要机械套用。以下情况保留内部 `if`：

- layout 自己有 `modifier` 或 alignment 语义；
- layout 还有其他 siblings；
- `if/else` 两边都贡献内容；
- container 必须一直存在，例如占位、语义、动画边界。

## 与其他 skill 的关系

- Slot API：可复用组件通常同时需要 `modifier` 和 content slots。
- Deferred reads：caller 需要通过 `modifier` 把高频 reads 推迟到 layout/draw。
- UI testing：caller 经常通过 `modifier.testTag(...)` 或 semantics 定位节点。

## 专家级审查清单

1. 这个 composable 是否 emit layout？
2. 是否有 `modifier: Modifier = Modifier`？
3. `modifier` 是否应用到 root？
4. caller modifier 是否在 chain 的开头？
5. root 是否硬编码了 parent 应该决定的 `fillMaxWidth` / padding / weight？
6. modifier chain 是否用 `var` 分步拼？
7. 三段以上 modifier 是否格式化为多行？
8. layout 是否只是包了一个单独 `if`？

## 精髓总结

1. 会 emit layout 的 composable 必须让 parent 放置它。
2. `modifier` 是 API contract，不是可选装饰。
3. caller modifier 应用到 root，并放在 chain 前面。
4. root 不要硬编码 parent layout 决策。
5. 条件和 modifier chain 的写法要表达真实结构，而不是制造无意义容器。
