# Kotlin Multiplatform Expect/Actual Boundaries 深度解析

对应 skill: [`kotlin-multiplatform-expect-actual`](../skills/kotlin-multiplatform-expect-actual/SKILL.md)

这一篇讲 Kotlin Multiplatform 中 common code 与平台代码的边界设计：

> Keep common APIs semantic and stable. Hide platform mechanics behind small expect/actual declarations or interfaces.

## 核心原则

`commonMain` 应该表达产品语义，不应该泄漏 Android / iOS / Desktop 的平台机制。

错误：

```kotlin
expect fun currentRegionFromAndroidLocale(context: Context): Region
```

正确：

```kotlin
expect fun currentRegion(): Region
```

Android actual 可以用 `Locale` / `Context`，iOS actual 可以用 Foundation。common caller 不应该知道。

## 边界选择

| Situation | Prefer |
|---|---|
| 简单 compile-time 平台差异 | `expect` / `actual` function/value/typealias/leaf composable |
| 需要 DI、fake、runtime selection、lifecycle owner | common interface + platform binding |
| UI 大部分共享，只一个 leaf 不同 | common composable 调用 `expect` leaf |
| 整屏平台差异很大 | common navigation contract + separate platform screens |
| 只是常量/resource 不同 | common semantic API，actual 提供平台值 |

## `expect/actual` 适合小而纯的平台翻译

适合：

```kotlin
// commonMain
expect fun currentTimeMillis(): Long
```

```kotlin
// androidMain / iosMain
actual fun currentTimeMillis(): Long = ...
```

适合特征：

- 无 lifecycle ownership。
- 无 DI/fake 需求。
- 无 runtime implementation 切换。
- 只是平台 API translation。

不适合把一个巨大 `Platform` 对象塞进 common：

```kotlin
expect object Platform {
    fun share(...)
    fun openSettings(...)
    fun requestPermission(...)
    fun haptic(...)
    fun currentRegion(...)
}
```

应该按 capability 拆：

- `Clipboard`
- `ShareSheet`
- `Haptics`
- `PermissionGateway`
- `SettingsOpener`

## Interface 适合 lifecycle / DI / testing

如果操作需要 Activity、UIViewController、lifecycle owner、fake、runtime choice，优先 interface。

```kotlin
// commonMain
interface ShareSheet {
    suspend fun shareText(text: String)
}
```

```kotlin
// androidMain
class AndroidShareSheet(
    private val activity: Activity,
) : ShareSheet {
    override suspend fun shareText(text: String) {
        val intent = Intent(Intent.ACTION_SEND)
            .setType("text/plain")
            .putExtra(Intent.EXTRA_TEXT, text)

        activity.startActivity(Intent.createChooser(intent, null))
    }
}
```

这里用 `Activity` 而不是 generic `Context`，因为分享 UI 的 lifecycle owner 是 Activity。隐藏这个事实会制造错误调用。

注意定义 suspend 语义：很多平台 UI API 的 `suspend` 只表示“已启动 platform UI”，不表示“用户完成操作”。

## Actual 要薄

Actual 里只做平台翻译：

- Android API 调用。
- iOS Foundation / UIKit 调用。
- Desktop API 调用。
- 类型转换。
- permission/platform error mapping。

不要在 actual 里写业务规则：

```kotlin
actual fun canBuyProduct(user: User, product: Product): Boolean {
    // duplicated business rules
}
```

业务规则应该在 common：

```kotlin
fun canBuyProduct(user: User, product: Product): Boolean {
    // common business rule
}
```

Actual 只提供必要平台信息。

## Compose Multiplatform 指导

共享 UI 应尽量保持 plain：

- common composable 接收 state + callbacks。
- platform-specific UI 推到 leaf。
- expected composable 如果 emit UI，也要传 `modifier`。
- 不在 common signature 暴露 `Context`、`Activity`、`Uri`、`Bundle`、`UIViewController`、`NSBundle`。

示例：

```kotlin
// commonMain
@Composable
expect fun PlatformMapView(
    location: Location,
    modifier: Modifier = Modifier,
)
```

Android actual 可以内部用 `AndroidView`，iOS actual 可以用 `UIKitView`。common UI 不知道。

Actual composable 也遵守 Compose 规则：

- 不在 body 直接做 side effect。
- 用 `remember` 管 composition-owned 对象。
- 用 `LaunchedEffect` / `DisposableEffect` 处理 imperative lifecycle。
- key 要稳定且语义正确。

## 平台类型不要泄漏进 common

Red flags：

```kotlin
import android.content.Context
import android.net.Uri
import android.os.Bundle
```

出现在 `commonMain`。

应该替换为 semantic common type：

```kotlin
@JvmInline value class DeepLink(val value: String)
@JvmInline value class FilePath(val value: String)
data class SharePayload(val text: String)
```

然后平台 actual / implementation 再转换。

## 测试与 fake

如果 common business logic 需要测试，边界必须 fakeable。

```kotlin
class FakeClipboard : Clipboard {
    var text: String? = null

    override suspend fun setText(text: String) {
        this.text = text
    }
}
```

如果用 direct `expect fun setClipboardText(text: String)`，common test 想替换行为会更困难。需要 fake / DI 时，interface 更合适。

## 常见错误

1. common API 名字带平台实现细节。
2. common signature 暴露平台类型。
3. expect function 参数只对某个平台有意义。
4. actual 中复制业务规则。
5. 一个巨大 `Platform` expect object 聚合所有能力。
6. 平台 UI 泄漏到高层 screen。
7. common tests 必须跑 Android/iOS runtime 才能验证业务逻辑。

## 专家级审查清单

1. common API 是否表达产品语义，而非平台机制？
2. commonMain 是否 import 平台包？
3. actual 是否薄，还是包含业务规则？
4. 这个边界是否需要 fake / DI / lifecycle？如果需要，是否该用 interface？
5. 第三个平台加入时，common caller 是否需要改？
6. Compose platform-specific UI 是否被推到 leaf？
7. expected composable 是否保留 `modifier`？
8. platform side effects 是否用 Compose effect API 管理？

## 精髓总结

1. common API 讲产品语义，不讲平台机制。
2. `expect/actual` 适合简单 compile-time 平台翻译。
3. 需要 lifecycle、DI、fake、runtime selection 时用 interface。
4. Actual 要薄，业务规则留在 common。
5. Compose platform interop 推到 leaf，并遵守普通 Compose runtime 规则。
