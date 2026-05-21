---
name: android-compose-clean-mvi-architecture
description: Use when designing or reviewing Android Jetpack Compose feature architecture with Clean Architecture, MVI, ViewModel or state-holder boundaries, UiState/Intent/Effect modeling, use cases, repositories, navigation/snackbar effects, and testing strategy.
---

# Android Compose Clean MVI architecture

## Core principle

Build a feature as a one-way pipeline:

> Plain Compose UI emits user intents, a screen state holder turns those intents into immutable UI state through use cases, one-shot effects are explicit outputs, and domain/data layers expose suspending or streaming APIs without owning UI lifecycle.

Clean Architecture is not a folder layout. MVI is not a giant reducer. The useful part is the boundary discipline:

- UI renders state and emits user intent.
- The screen state holder owns business-facing UI state production.
- Use cases own business operations.
- Repositories/data sources expose `suspend` and `Flow` APIs.
- One-shot effects are not render state.
- Compose runtime objects stay in composition, not in ViewModels.

## When to use this skill

Use this when designing or reviewing an Android feature that:

- Uses Jetpack Compose with a ViewModel, Decompose Component, Molecule presenter, Orbit/Mavericks-style store, or custom MVI state holder.
- Needs a screen contract with `UiState`, user intents/actions, and one-shot effects.
- Mixes navigation, snackbars, analytics, focus commands, repository calls, and layout in the same composable.
- Passes ViewModels/components through child composables.
- Puts repository calls in composable bodies or UI composables.
- Stores `LazyListState`, `FocusRequester`, `SnackbarHostState`, `PagerState`, `DrawerState`, or `CoroutineScope` in a ViewModel.
- Uses `MutableSharedFlow()` for navigation/snackbar events without deliberate buffering semantics.
- Needs a testing strategy across UI, ViewModel/state holder, use cases, repositories, and integration boundaries.

## Architecture model

```mermaid
flowchart TB
    UI["Plain Compose UI\nUiState + callbacks"]
    Route["Route composable\ncollect state/effects\nwire callbacks"]
    Store["ViewModel / MVI Store\naccept intents\nreduce state\nemit effects"]
    UseCase["Use cases\nbusiness operations"]
    Repo["Repository interfaces"]
    Data["Data sources\nnetwork/db/platform"]
    EffectTargets["Imperative targets\nnavigator, snackbar,\npermission launcher, analytics"]

    UI -->|callbacks / intents| Route
    Route -->|accept(intent)| Store
    Store -->|call| UseCase
    UseCase --> Repo
    Repo --> Data
    Store -->|StateFlow UiState| Route
    Store -->|Flow Effect| Route
    Route --> UI
    Route --> EffectTargets
```

Dependency direction:

- UI layer depends on presentation contracts.
- Presentation depends on use cases and UI models.
- Domain use cases depend on repository interfaces.
- Data implements repository interfaces.
- Domain does not depend on Android UI, Compose, navigation, or DI framework details.

## Layer responsibilities

| Layer | Owns | Must not own |
|---|---|---|
| Plain Compose UI | Layout, modifiers, semantics, UI-local state, callbacks | ViewModel, repository, navigator, business Flow collection |
| Route/state-holder composable | Lifecycle-aware collection, effect collection, callback wiring, imperative targets | Large layout trees, business rules |
| ViewModel/MVI store | `UiState`, intent handling, use case calls, effect emission | Compose runtime objects, layout mechanics, repository implementation details |
| Use case/domain | Business rules, suspending operations, domain models | UI state, navigation, snackbar, Android view objects |
| Repository/data | Persistence, network, DTO/entity mapping | UI lifecycle, fire-and-forget launches, Compose state |

## Feature contract

A feature should normally expose three presentation concepts:

```kotlin
data class ProductUiState(
    val isLoading: Boolean,
    val products: ImmutableList<ProductUi>,
    val errorMessage: String?,
    val canRetry: Boolean,
)

sealed interface ProductIntent {
    data object RetryClicked : ProductIntent
    data object BackClicked : ProductIntent
    data class ProductClicked(val id: ProductId) : ProductIntent
}

sealed interface ProductEffect {
    data object Back : ProductEffect
    data class OpenProduct(val id: ProductId) : ProductEffect
    data class ShowMessage(val message: String) : ProductEffect
}
```

Names vary by project (`Action`, `Event`, `Message`, `Command`), but the semantics should stay clear:

- `UiState` is durable render state.
- `Intent` is a user or lifecycle input to the state holder.
- `Effect` is a one-shot imperative output.

## Compose boundary

Use a state-holder composable or `Route` composable for wiring:

```kotlin
@Composable
fun ProductRoute(
    viewModel: ProductViewModel,
    navigator: ProductNavigator,
    snackbarHostState: SnackbarHostState,
    modifier: Modifier = Modifier,
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(viewModel.effects, snackbarHostState, navigator) {
        viewModel.effects.collect { effect ->
            when (effect) {
                ProductEffect.Back -> navigator.back()
                is ProductEffect.OpenProduct -> navigator.openProduct(effect.id)
                is ProductEffect.ShowMessage ->
                    snackbarHostState.showSnackbar(effect.message)
            }
        }
    }

    ProductScreen(
        state = state,
        snackbarHostState = snackbarHostState,
        onIntent = viewModel::accept,
        modifier = modifier,
    )
}
```

Keep the UI composable plain:

```kotlin
@Composable
fun ProductScreen(
    state: ProductUiState,
    snackbarHostState: SnackbarHostState,
    onIntent: (ProductIntent) -> Unit,
    modifier: Modifier = Modifier,
) {
    Scaffold(
        modifier = modifier,
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { padding ->
        ProductContent(
            state = state,
            onRetryClick = { onIntent(ProductIntent.RetryClicked) },
            onBackClick = { onIntent(ProductIntent.BackClicked) },
            onProductClick = { id -> onIntent(ProductIntent.ProductClicked(id)) },
            modifier = Modifier.padding(padding),
        )
    }
}
```

Child composables should take the smallest useful state and callbacks:

```kotlin
@Composable
private fun ProductContent(
    state: ProductUiState,
    onRetryClick: () -> Unit,
    onBackClick: () -> Unit,
    onProductClick: (ProductId) -> Unit,
    modifier: Modifier = Modifier,
) {
    // Layout only.
}
```

Do not pass `ProductViewModel`, `ProductComponent`, or `ProductNavigator` into child composables.

## MVI state holder

A simple ViewModel implementation:

```kotlin
class ProductViewModel(
    private val loadProducts: LoadProducts,
) : ViewModel() {
    private val _state = MutableStateFlow(ProductUiState.initial())
    val state: StateFlow<ProductUiState> = _state.asStateFlow()

    private val _effects = Channel<ProductEffect>(Channel.BUFFERED)
    val effects: Flow<ProductEffect> = _effects.receiveAsFlow()

    fun accept(intent: ProductIntent) {
        when (intent) {
            ProductIntent.RetryClicked -> load()
            ProductIntent.BackClicked -> emitEffect(ProductEffect.Back)
            is ProductIntent.ProductClicked ->
                emitEffect(ProductEffect.OpenProduct(intent.id))
        }
    }

    private fun load() {
        viewModelScope.launch {
            _state.update { current ->
                current.copy(isLoading = true, errorMessage = null)
            }

            try {
                val products = loadProducts()
                _state.update { current ->
                    current.copy(
                        isLoading = false,
                        products = products.map(Product::toUi).toImmutableList(),
                        canRetry = false,
                    )
                }
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                _state.update { current ->
                    current.copy(
                        isLoading = false,
                        errorMessage = e.toUserMessage(),
                        canRetry = true,
                    )
                }
                _effects.send(ProductEffect.ShowMessage("Could not load products"))
            }
        }
    }

    private fun emitEffect(effect: ProductEffect) {
        viewModelScope.launch {
            _effects.send(effect)
        }
    }
}
```

This is the UI-event boundary where launching from a non-suspending method is acceptable:

- The caller is UI.
- The owner is a screen state holder.
- The scope is lifecycle-bound (`viewModelScope`).

Repository, use case, manager, and data source layers underneath still expose `suspend` APIs instead of storing scopes.

## Reducers and side effects

Keep state transformations pure and fast:

```kotlin
_state.update { current ->
    current.copy(selectedId = id)
}
```

Do not put network calls, database writes, logging, random IDs, or time reads inside `update { ... }`. `MutableStateFlow.update` can retry the lambda.

A strict reducer style can be useful:

```kotlin
private fun reduce(reducer: ProductUiState.() -> ProductUiState) {
    _state.update { current -> current.reducer() }
}
```

But do not force all work into the reducer. Suspend work belongs in the state holder's coroutine, and business logic belongs in use cases.

## Effect modeling

One-shot effects are not render state.

Avoid:

```kotlin
data class ProductUiState(
    val shouldNavigateBack: Boolean = false,
    val snackbarMessage: String? = null,
)
```

This creates reset protocols, duplicate-consumption bugs, and configuration-change ambiguity.

Prefer:

```kotlin
sealed interface ProductEffect {
    data object Back : ProductEffect
    data class ShowMessage(val message: String) : ProductEffect
}
```

For a single UI consumer, a buffered `Channel` exposed as `Flow` often matches navigation/snackbar semantics:

```kotlin
private val _effects = Channel<ProductEffect>(Channel.BUFFERED)
val effects: Flow<ProductEffect> = _effects.receiveAsFlow()
```

Remember: `receiveAsFlow()` is fan-out, not broadcast. If multiple collectors must all see every effect, model durable state or deliberately configure a broadcast stream.

## UiState modeling

`UiState` should be:

- Immutable.
- Stable-friendly for Compose.
- Free of callbacks and services.
- Free of Compose runtime objects.
- Shaped for rendering, not raw persistence.

Prefer immutable collections at UI-state boundaries:

```kotlin
data class SearchUiState(
    val results: ImmutableList<SearchResultUi>,
    val selectedFilters: ImmutableSet<FilterId>,
)
```

Do not put these in `UiState`:

- `LazyListState`
- `FocusRequester`
- `SnackbarHostState`
- `PagerState`
- `DrawerState`
- `CoroutineScope`
- repository/use case/service instances
- mutable collections such as `MutableList`

UI-local state stays in composition or a plain state holder:

```kotlin
@Composable
fun ProductScreen(...) {
    val listState = rememberLazyListState()
    ProductList(state = state, listState = listState)
}
```

If UI-local behavior becomes coordinated and named, extract a plain state holder remembered in composition.

## Intent modeling

Intents should express user or lifecycle intent, not arbitrary setter operations.

Prefer:

```kotlin
sealed interface CheckoutIntent {
    data object PayClicked : CheckoutIntent
    data object RetryClicked : CheckoutIntent
    data class AddressChanged(val value: String) : CheckoutIntent
}
```

Avoid:

```kotlin
sealed interface CheckoutIntent {
    data class SetLoading(val value: Boolean) : CheckoutIntent
    data class NavigateTo(val route: String) : CheckoutIntent
}
```

The state holder decides loading and navigation effects. UI reports what the user did.

For simple screens, explicit callbacks may be clearer than a single `onIntent`:

```kotlin
ProductScreen(
    state = state,
    onRetryClick = viewModel::retry,
    onBackClick = viewModel::back,
)
```

Do not force `Intent` ceremony when there are only one or two callbacks and no store abstraction benefits.

## Domain layer

Use cases expose suspending operations or cold flows:

```kotlin
class LoadProducts(
    private val repository: ProductRepository,
) {
    suspend operator fun invoke(): List<Product> {
        return repository.loadProducts()
    }
}
```

Domain rules belong here:

```kotlin
class ValidateCheckout {
    operator fun invoke(input: CheckoutInput): CheckoutValidation {
        // Business rules.
    }
}
```

Use domain value classes for identity and constrained values:

```kotlin
@JvmInline value class ProductId(val value: String)
@JvmInline value class UserId(val value: String)
```

Domain must not depend on:

- Compose
- Android `View`
- Navigator
- Snackbar
- Activity/Fragment
- UI state classes

## Repository and data layer

Repository interfaces belong at the boundary the domain/presentation depends on:

```kotlin
interface ProductRepository {
    suspend fun loadProducts(): List<Product>
    fun observeProducts(): Flow<List<Product>>
}
```

Repository implementations may use network, database, DTOs, entities, caches, and platform APIs.

Rules:

- Do not store a `CoroutineScope` just to fire-and-forget.
- Do not launch in `init`.
- Expose `suspend` and `Flow`.
- Map DTO/entity models before they reach UI.
- Do not invent fake domain sentinels to satisfy `StateFlow` initial values.

## Coroutine and Flow rules

Use these defaults:

- ViewModel/state holder: may launch from UI event methods on `viewModelScope`.
- Use case/repository/data source: expose `suspend` or `Flow`; do not own UI event scopes.
- Render state: expose `StateFlow<UiState>`.
- One-shot single-consumer effects: consider `Channel(BUFFERED).receiveAsFlow()`.
- Derived `StateFlow`: use `stateIn` as a property, not inside a getter function.
- `MutableStateFlow`: mutate with `update { ... }`.
- Broad `catch` around suspend calls must rethrow `CancellationException`.

## Testing strategy

Test each contract at the smallest useful level:

| Contract | Test shape |
|---|---|
| Plain UI renders state and calls callbacks | Compose UI test with fake state |
| ViewModel handles intents and reduces state | ViewModel/unit test with fake use cases |
| Effect emission | ViewModel/unit test collecting `effects` |
| Use case business rules | Plain unit test |
| Repository mapping/caching | Repository test with fake data sources |
| Navigation/lifecycle/DI integration | Integration test or smoke test |
| Visual layout | Screenshot test |

Do not use a full app graph to test a simple UI branch:

```kotlin
composeTestRule.setContent {
    ProductScreen(
        state = ProductUiState(
            isLoading = false,
            products = persistentListOf(),
            errorMessage = "Network error",
            canRetry = true,
        ),
        snackbarHostState = remember { SnackbarHostState() },
        onIntent = { capturedIntent = it },
    )
}
```

For ViewModel tests, verify state and effects, not private reducer calls.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| `Screen(viewModel)` contains all layout | Hard to preview/test; dependencies leak | Add plain UI composable with state + callbacks |
| Child composables take `ViewModel` or `Component` | Whole state holder leaks through tree | Pass explicit values and callbacks |
| UI composable calls repository/use case | UI owns business and coroutine lifecycle | Route through ViewModel/state holder |
| One-shot effect stored in `UiState` | Reset/duplicate consumption bugs | Model `Effect` stream |
| `MutableSharedFlow()` for nav/snackbar by default | Emissions can be dropped | Use `Channel(BUFFERED)` for single-consumer handoff or configure deliberately |
| ViewModel stores `LazyListState` or `FocusRequester` | Compose runtime object escapes composition | Keep in UI/plain state holder |
| Repository stores `CoroutineScope` | Silent cancellation and hidden lifecycle | Expose `suspend` APIs |
| Reducer does network/logging/random/time | `update` can retry; reducer becomes impure | Do side effects outside reducer; pass captured values in |
| `UiState` uses mutable collections | Compose stability and mutation visibility problems | Use immutable values/collections |
| DTOs reach UI directly | UI absorbs data-layer shape and business rules | Map to domain/UI models |

## When NOT to apply rigid MVI ceremony

Do not force this full shape when:

- The composable is a tiny stateless reusable component.
- A screen has one or two callbacks and no meaningful state machine.
- A local UI element state is private to one composable.
- The state holder/UI split already isolates dependencies and a sealed `Intent` would add noise.
- A framework or design-system primitive should expose values, slots, and modifiers instead of feature intents.

Use the architecture where it reduces lifecycle, state, and dependency ambiguity. Do not turn it into boilerplate.

## Review order

1. Identify the feature boundary.
2. Locate the plain UI composable and the state-holder composable.
3. Check `UiState`, `Intent`, and `Effect` semantics.
4. Check where Flow state and effect streams are collected.
5. Check whether one-shot effects are separated from render state.
6. Check whether use cases/repositories expose `suspend` and `Flow`.
7. Check whether Compose runtime objects stay in composition.
8. Check whether tests target the smallest contract.

## Related skills

- [`compose-state-holder-ui-split`](../compose-state-holder-ui-split/SKILL.md) — split state-holder wiring from plain UI rendering.
- [`compose-state-hoisting`](../compose-state-hoisting/SKILL.md) — decide local UI state, plain state holder, or screen state holder ownership.
- [`compose-side-effects`](../compose-side-effects/SKILL.md) — choose effect APIs and handle event Flow collection.
- [`kotlin-flow-state-event-modeling`](../kotlin-flow-state-event-modeling/SKILL.md) — choose `StateFlow`, `SharedFlow`, `Channel`, `stateIn`, and event semantics.
- [`kotlin-coroutines-structured-concurrency`](../kotlin-coroutines-structured-concurrency/SKILL.md) — scope ownership, fire-and-forget boundaries, cancellation, and `runBlocking`.
- [`compose-ui-testing-patterns`](../compose-ui-testing-patterns/SKILL.md) — test plain UI, callbacks, focus, screenshots, and integration boundaries.
- [`compose-stability-diagnostics`](../compose-stability-diagnostics/SKILL.md) — keep UI state stable and skippable.
- [`kotlin-types-value-class`](../kotlin-types-value-class/SKILL.md) — model domain identifiers and single-field types.

## Red flags during review

- "The UI just needs the ViewModel, passing it down is simpler."
- "Navigation is a boolean in UiState."
- "This repository launches internally because callers do not care."
- "The reducer calls the API and then returns new state."
- "We use `MutableSharedFlow()` for all events."
- "The ViewModel keeps `LazyListState` so it survives rotation."
- "DTOs are fine in UI because they already have the fields."
- "We need a full DI graph to test this button."
- "Clean Architecture means every feature must have use case classes even when there is no business operation."

## Quick reference

| Question | Default answer |
|---|---|
| Where is render state collected? | Route/state-holder composable |
| What does plain UI receive? | Immutable `UiState` + callbacks |
| Where are user intents handled? | ViewModel/MVI store |
| Where are repository calls made? | ViewModel calls use cases; use cases call repositories |
| How are one-shot events modeled? | `Effect` stream, often `Channel(BUFFERED).receiveAsFlow()` for one UI consumer |
| Where are navigation/snackbar executed? | Route/state-holder composable effect collector |
| Where does scroll/focus/pager state live? | Composition or plain UI state holder, not ViewModel |
| What do repositories expose? | `suspend` and `Flow`, no hidden scopes |
| How is UI tested? | Plain state-driven Compose tests first |
