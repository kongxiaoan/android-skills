# Kotlin Value Class vs Data Class 深度解析

对应 skill: [`kotlin-types-value-class`](../skills/kotlin-types-value-class/SKILL.md)

这一篇讲 Kotlin 单字段 domain type 的建模：

> Prefer `@JvmInline value class` for single-field types that carry domain meaning.

Data class 用来聚合多个字段；value class 用来给一个 underlying value 增加类型语义，同时尽量避免 wrapper allocation。

## 核心原则

如果一个类型：

- 只有一个字段；
- 这个字段有明确 domain meaning；
- 不需要自定义 equality / hash / toString；

优先：

```kotlin
@JvmInline
value class UserId(val value: String)
```

而不是：

```kotlin
data class UserId(val value: String)
```

## 为什么不用 primitive

错误倾向：

```kotlin
fun transfer(
    from: String,
    to: String,
    amount: Long,
)
```

`from`、`to`、`amount` 都是 primitive，调用时很容易传错。

更好：

```kotlin
@JvmInline value class AccountId(val value: String)
@JvmInline value class MoneyCents(val value: Long)

fun transfer(
    from: AccountId,
    to: AccountId,
    amount: MoneyCents,
)
```

Value class 给 primitive 增加 type safety。

## 决策表

| Situation | Prefer |
|---|---|
| 单字段 + domain meaning | `@JvmInline value class` |
| 单字段 + 只是别名、无强类型需求 | typealias 或 primitive |
| 多字段 | data class |
| 需要自定义 equality/hash/toString | data class |
| nullable/generic hot path | measure 后决定，可能 primitive 更合适 |

## 好例子

```kotlin
@JvmInline value class UserId(val value: String)
@JvmInline value class EmailAddress(val value: String)
@JvmInline value class Percentage(val value: Float)
@JvmInline value class ProductId(val value: Long)
```

坏例子：

```kotlin
@JvmInline value class Wrapper(val value: String)
```

如果没有 domain meaning，只是包一下，不如 primitive 或 typealias。

## Compose stability

当 underlying type stable 时，`@JvmInline value class` 在 Compose 边界通常是 stable。

```kotlin
@JvmInline value class UserId(val value: String)

data class ProfileUiState(
    val userId: UserId,
)
```

好处：

- 避免 primitive 混用。
- 避免单字段 data class wrapper allocation。
- 不需要为了这个 wrapper 加 `@Immutable`。
- Compose compiler 更容易理解参数稳定性。

这和 [`compose-stability-diagnostics`](./compose-stability-diagnostics.md) 相关。

## Value class 不是 data class

Value class 没有：

- `copy()`
- `component1()`
- 自定义 `equals`
- 自定义 `hashCode`
- 自定义 `toString`
- backing fields
- `init` 中复杂状态

所以如果你需要：

```kotlin
data class CaseInsensitiveString(val value: String) {
    override fun equals(other: Any?): Boolean = ...
}
```

那就不要用 value class。

Value class equality 委托给 underlying value。

## Boxing gotchas

Value class 通常可以被编译器 unbox，但以下场景可能 boxing：

- nullable：`UserId?`
- generic：`List<UserId>`
- `Any`
- vararg
- interface boundary
- reflection / framework interop

大多数 app code 不必过早担心。但在 hot path 中要 measure。

## Serialization 语义

`kotlinx.serialization` 中：

```kotlin
@Serializable
data class UserId(val value: String)
```

通常序列化为：

```json
{"value":"123"}
```

而：

```kotlin
@Serializable
@JvmInline
value class UserId(val value: String)
```

通常序列化为 underlying value：

```json
"123"
```

因此把单字段 data class 改成 value class 可能是 API / JSON contract breaking change。

Jackson、ORM、DI、reflection-heavy framework 也可能需要额外配置或有 runtime type 观察差异。

## Java interop

从 Java 看，value class 经常表现为 underlying type 或 mangled method signature。Java caller 可能绕过 Kotlin type safety。

如果 Java interop 是核心 API，需要额外检查 generated signature 和调用体验。

## Packing multiple values

Value class 只能有一个字段。Compose 在 `androidx.compose.ui.util` 提供 `packFloats` / `packInts` 等工具，可以把两个 primitive pack 到一个 `Long`：

```kotlin
@JvmInline
value class PackedPoint(val packedValue: Long)

fun PackedPoint(x: Float, y: Float): PackedPoint =
    PackedPoint(packFloats(x, y))

val PackedPoint.x: Float get() = unpackFloat1(packedValue)
val PackedPoint.y: Float get() = unpackFloat2(packedValue)
```

只在性能关键路径使用。普通业务/UI 类型用 data class 更可读。

## 常见错误

1. 单字段 domain wrapper 用 data class。
2. 无 domain meaning 的 wrapper 用 value class。
3. 需要 custom equality 却用 value class。
4. 忽略 serialization contract 变化。
5. 在 hot generic path 大量使用 value class 却不 measure boxing。
6. 用 typealias 以为有强类型保护。

`typealias UserId = String` 不提供类型安全：

```kotlin
typealias UserId = String
typealias ProductId = String

fun openUser(id: UserId) {}

val productId: ProductId = "p1"
openUser(productId) // compiles
```

Value class 可以阻止这种混用。

## 专家级审查清单

1. 这个类型是否只有一个字段？
2. 这个字段是否有 domain meaning？
3. 是否需要 custom equality/hash/toString？
4. 是否会改变 serialization/API contract？
5. 是否处于 nullable/generic hot path？
6. 是否有 Java/framework reflection interop？
7. Compose compiler report 中是否有单字段 wrapper stability 问题？
8. typealias 是否不足以表达强类型语义？

## 精髓总结

1. 单字段 domain type 优先 `@JvmInline value class`。
2. 多字段聚合用 data class。
3. 无 domain meaning 不要为了包装而包装。
4. 需要 custom equality 就不是 value class。
5. 注意 serialization、boxing、Java interop 这些边界成本。
