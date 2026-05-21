# Android Skills

这是一个面向 Android 开发的 Codex/Claude Skills 知识体系，覆盖 Kotlin、Jetpack Compose、协程、Flow、KMP 边界、UI 测试、性能诊断，以及基于 Clean Architecture + MVI 的 Compose Feature 架构设计。

本项目基于来源项目 [`chrisbanes/skills`](https://github.com/chrisbanes/skills) 整理和扩展：

- 保留来源项目中高质量的 Kotlin 与 Jetpack Compose 专项 skills。
- 新增 [`android-compose-clean-mvi-architecture`](skills/android-compose-clean-mvi-architecture/SKILL.md)，用于把 Compose 状态、Side Effect、Flow、协程、测试和稳定性规则串成完整的 Android Feature 架构审查路径。
- `docs/` 目录包含中文解析文档，用来解释各个 skill 背后的设计原则、适用边界、常见错误和工程判断。

## 安装

使用 [skills CLI](https://skills.sh)：

```bash
npx skills add kongxiaoan/android-skills
```

或作为 Claude Code plugin 安装：

```text
/plugin marketplace add kongxiaoan/android-skills
/plugin install chrisbanes-skills@chrisbanes-skills
```

## 如何使用

### 推荐入口

- 设计完整 Android Compose Feature 架构时，先看 [`android-compose-clean-mvi-architecture`](skills/android-compose-clean-mvi-architecture/SKILL.md)，再按问题深入到 Compose、Flow、协程和测试相关 skill。
- 处理 Compose 状态或副作用时，从 [`compose-state-authoring`](skills/compose-state-authoring/SKILL.md)、[`compose-state-hoisting`](skills/compose-state-hoisting/SKILL.md)、[`compose-state-holder-ui-split`](skills/compose-state-holder-ui-split/SKILL.md)、[`compose-side-effects`](skills/compose-side-effects/SKILL.md) 开始。
- 排查重组、稳定性或卡顿时，从 [`compose-recomposition-performance`](skills/compose-recomposition-performance/SKILL.md) 开始。
- 审查 Flow 或协程架构时，从 [`kotlin-flow-state-event-modeling`](skills/kotlin-flow-state-event-modeling/SKILL.md) 或 [`kotlin-coroutines-structured-concurrency`](skills/kotlin-coroutines-structured-concurrency/SKILL.md) 开始。

## Skills

### Android 架构

- [`android-compose-clean-mvi-architecture`](skills/android-compose-clean-mvi-architecture/SKILL.md) — 使用 Clean Architecture + MVI 设计和审查 Android Compose Feature 架构，包括 ViewModel/state-holder 边界、不可变 UI State、Intent/Effect 建模、UseCase、Repository、导航/snackbar 副作用和测试策略。

### Jetpack Compose

#### 状态与副作用

- [`compose-state-authoring`](skills/compose-state-authoring/SKILL.md) — 正确编写 Compose 本地 mutable state 与 read-only composable accessor。
- [`compose-state-hoisting`](skills/compose-state-hoisting/SKILL.md) — 判断 Compose UI element state 应留在本地、上提到父级、抽 plain state holder，还是交给 screen-level state holder。
- [`compose-state-holder-ui-split`](skills/compose-state-holder-ui-split/SKILL.md) — 拆分 state-holder wiring 与 plain UI rendering，让 screen 更容易 preview、测试和复用。
- [`compose-side-effects`](skills/compose-side-effects/SKILL.md) — 选择并正确 key Compose effect API，用于 event Flow 收集、callback、cleanup、navigation、snackbar、analytics 等副作用。

#### 性能

- [`compose-recomposition-performance`](skills/compose-recomposition-performance/SKILL.md) — 在稳定性/equality 问题和 deferred state read 问题之间做重组性能分诊。
- [`compose-stability-diagnostics`](skills/compose-stability-diagnostics/SKILL.md) — 诊断 Compose compiler reports、strong skipping、unstable parameters 和稳定性修复。
- [`compose-state-deferred-reads`](skills/compose-state-deferred-reads/SKILL.md) — 将滚动、动画、手势等 frame-rate State 读取从 composition 移到 layout 或 draw。

#### UI API、布局与交互

- [`compose-modifier-and-layout-style`](skills/compose-modifier-and-layout-style/SKILL.md) — 让 Compose layout API 可由 caller 放置，并保持 modifier chain 清晰。
- [`compose-slot-api-pattern`](skills/compose-slot-api-pattern/SKILL.md) — 设计可复用 Compose 组件时，用 caller-provided slots 表达可变视觉区域。
- [`compose-animations`](skills/compose-animations/SKILL.md) — 选择 Compose 动画 API，包括 visibility、target value、coordinated transition、content swap 等。
- [`compose-focus-navigation`](skills/compose-focus-navigation/SKILL.md) — 设计和测试 keyboard、TV、D-pad、focus-first Compose navigation 行为。

#### 测试

- [`compose-ui-testing-patterns`](skills/compose-ui-testing-patterns/SKILL.md) — 在 plain UI tests、semantics assertions、key/focus tests、MutableInteractionSource interaction tests、screenshot tests 和 integration tests 之间选择合适测试形态。

### Kotlin

- [`kotlin-coroutines-structured-concurrency`](skills/kotlin-coroutines-structured-concurrency/SKILL.md) — 审查协程 scope ownership、init/fire-and-forget 边界、取消处理和 blocking 边界。
- [`kotlin-flow-state-event-modeling`](skills/kotlin-flow-state-event-modeling/SKILL.md) — 建模 `StateFlow`、`SharedFlow`、`Channel`、`stateIn`、sharing policy 和 one-shot events，避免 lossy defaults。
- [`kotlin-multiplatform-expect-actual`](skills/kotlin-multiplatform-expect-actual/SKILL.md) — 为 Kotlin Multiplatform 设计语义化 expect/actual 与 interface 平台边界。
- [`kotlin-types-value-class`](skills/kotlin-types-value-class/SKILL.md) — 为单字段 domain type 选择 `@JvmInline value class` 而不是 data class，并理解 Compose stability 影响。

## 中文解析文档

`docs/` 目录包含按知识顺序整理的中文深度解析。文档用于学习和体系化理解；真正触发 agent 行为的入口仍然是 `skills/<skill-name>/SKILL.md`。

这些文档可以作为 Android/Compose 架构学习路线阅读：先看架构总览和 Compose runtime，再进入状态、副作用、性能、UI API、测试和 Kotlin 基础设施。每篇文档都围绕一个 skill 展开，补充背景模型、决策标准、典型反例和审查清单，帮助把零散规则整理成可执行的工程判断。

### 架构总览

- [Android Compose Clean MVI Architecture](docs/android-compose-clean-mvi-architecture.md) — 说明如何用 Clean Architecture + MVI 组织 Compose Feature，包括 UI、Route、ViewModel/Store、UseCase、Repository、Effect 和测试边界。
- [Compose Runtime、SlotTable 与 Snapshot 系统](docs/compose-runtime-slot-snapshot-deep-dive.md) — 解释 Compose runtime、SlotTable、positional memoization、Snapshot 读写和 recomposition 的底层关系。

### Compose 状态与副作用

- [Compose State Authoring](docs/compose-state-authoring.md) — 深入解释 `remember`、`mutableStateOf`、snapshot collection、`rememberSaveable`、State read 范围和 `@ReadOnlyComposable`。
- [Compose State Hoisting](docs/compose-state-hoisting.md) — 说明 local state、最低共同 owner、plain state holder、ViewModel/screen state holder 的状态归属判断。
- [Compose State Holder / UI Split](docs/compose-state-holder-ui-split.md) — 说明 Route/state-holder composable 与 plain UI composable 的职责拆分。
- [Compose Side Effects](docs/compose-side-effects.md) — 说明 `SideEffect`、`LaunchedEffect`、`DisposableEffect`、`rememberCoroutineScope`、`rememberUpdatedState`、`snapshotFlow` 的准确边界。

### Compose 性能

- [Compose Recomposition Performance](docs/compose-recomposition-performance.md) — 作为重组性能分诊入口，区分参数稳定性问题和高频 State 读取阶段问题。
- [Compose Stability Diagnostics](docs/compose-stability-diagnostics.md) — 说明 strong skipping、compiler reports、stable/unstable 参数、immutable collection 和 `@Stable`/`@Immutable`。
- [Compose State Deferred Reads](docs/compose-state-deferred-reads.md) — 说明如何把滚动、动画、手势等 frame-rate State 读取推迟到 layout/draw 阶段。

### Compose UI API、交互与测试

- [Compose Modifier and Layout Style](docs/compose-modifier-and-layout-style.md) — 说明 `modifier` 参数、root modifier、modifier chain、布局职责和条件布局风格。
- [Compose Slot API Pattern](docs/compose-slot-api-pattern.md) — 说明如何用 `@Composable` slots 设计可复用组件的可变视觉区域。
- [Compose Animations](docs/compose-animations.md) — 说明 `AnimatedVisibility`、`animate*AsState`、`rememberTransition`、`AnimatedContent`、`Crossfade`、`Animatable` 的选择。
- [Compose Focus Navigation](docs/compose-focus-navigation.md) — 说明 keyboard、TV、D-pad、FocusRequester、focusProperties、key event 和 focus 测试。
- [Compose UI Testing Patterns](docs/compose-ui-testing-patterns.md) — 说明 plain UI test、semantics、callback、interaction source、focus/key、screenshot 和 integration test 的选择。

### Kotlin

- [Kotlin Coroutines Structured Concurrency](docs/kotlin-coroutines-structured-concurrency.md) — 说明协程 scope ownership、fire-and-forget、init launch、CancellationException 和 `runBlocking` 边界。
- [Kotlin Flow State and Event Modeling](docs/kotlin-flow-state-event-modeling.md) — 说明 `StateFlow`、`SharedFlow`、`Channel.receiveAsFlow()`、`stateIn`、`WhileSubscribed`、`.value` 和 one-shot event 建模。
- [Kotlin Multiplatform Expect/Actual Boundaries](docs/kotlin-multiplatform-expect-actual.md) — 说明 KMP common API、expect/actual、interface、平台服务和 Compose Multiplatform leaf 边界。
- [Kotlin Value Class vs Data Class](docs/kotlin-types-value-class.md) — 说明单字段 domain type、`@JvmInline value class`、Compose stability、boxing、serialization 和 Java interop。

## 贡献约定

Skills 位于 `skills/<skill-name>/SKILL.md`，保持扁平结构，不按语言或主题嵌套目录。`SKILL.md` frontmatter 中的 `name:` 必须与目录名完全一致。

添加、重命名或删除 skill 时，也要同步更新本 README 的 skill 列表。

## License

[Apache 2.0](LICENSE)

[plugins]: https://docs.claude.com/en/docs/claude-code/plugins
