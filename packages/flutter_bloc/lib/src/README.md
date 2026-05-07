# 🧱 AXKit — Modular Compose UI Framework

> Build apps like assembling a LEGO masterpiece. Each piece is designed for a purpose, and they all fit together seamlessly.

**AXKit** is a modular, predictable UI architecture library for **Jetpack Compose** on Android, powered by the **MVI (Model‑View‑Intent)** pattern. It provides a block‑based component system with built‑in state management, inter‑component communication, and navigation — enabling teams to build complex, scalable features with minimal boilerplate.

---

## Why MVI?

| Benefit | Description |
|---------|-------------|
| **Unidirectional Data Flow** | Data flows in one direction — Intent → Model → State → View — making updates predictable and easy to reason about. |
| **Centralised State** | All UI state lives in a single, observable `StateFlow`, eliminating scattered mutable state and race conditions. |
| **Separation of Concerns** | Business logic (Model), user actions (Intent), and rendering (View) are cleanly separated into distinct layers. |
| **Maintainability & Scalability** | Each feature is a self‑contained block that can be developed, tested, and reused independently. |

---

## Architecture Overview

### Layers of MVI

AXKit implements three core MVI layers:

```
┌─────────────────────────────────────────────────────┐
│                      VIEW                           │
│               AXBlock (Composable)                  │
│     Observes state · Renders UI · Emits intents     │
└────────────────────────┬────────────────────────────┘
                         │ State ↑   │ Intent ↓
┌────────────────────────┴────────────────────────────┐
│                     MODEL                           │
│                  AXBlockModel                       │
│   Processes Intents · Manages state · Emits output  │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────┐
│                     INTENT                          │
│                    AXIntent                         │
│     Button clicks · Swipe gestures · Reloads        │
└─────────────────────────────────────────────────────┘
```

| Layer | Class | Role |
|-------|-------|------|
| **Model** | `AXBlockModel` | Processes Intents and produces new states; contains all business logic. |
| **View** | `AXBlock` | Acts as a passive observer that receives state updates and renders the UI. |
| **Intent** | `AXIntent` | Represents the user's intention or action (e.g. button clicks, swipe gestures). |

### Block Properties

Every Block communicates through four clearly‑defined property types:

```
             ┌──────────────────────────────────┐
             │           AXBlock                │
             │                                  │
  Meta ──────►  AXDomainSpecificMeta (input)    │
             │                                  │
  Intent ────►  AXIntent (user actions)         │
             │                                  │
             │  AXState (internal UI state) ────► View
             │                                  │
             │  AXOutput (outbound events) ─────► Parent / Navigation
             └──────────────────────────────────┘
```

| Property | Purpose |
|----------|---------|
| **AXDomainSpecificMeta** | Immutable input data passed into the block when it is created or updated (e.g. IDs, configuration). | 
| **AXIntent** | User or external actions directed at the block (e.g. clicks, reload, filter changes). | 
| **AXState** | The block's internal UI state, derived from meta, intents, and business logic. Rendered by the UI. |
| **AXOutput** | One‑way events the block emits to the outer world to signal something happened (e.g. navigation, selection, errors). | 

### Contract Layer

AXKit uses a **Contract Layer** to separate **data access** from **UI components and block logic**.

Blocks and BlockModels do **not** own repository, API, or database code directly. Instead, they depend on narrow contract interfaces that describe the data they need. The host application provides concrete implementations of those contracts.

This keeps AXKit components focused on **state management, intent handling, and rendering**, while the app remains responsible for **where data comes from**.

**Why this matters:**
- Blocks stay reusable across screens and products.
- Data sources can change without changing component logic.
- Networking, caching, and persistence remain outside the UI architecture.
- Testing becomes easier because contracts can be mocked or stubbed.

**Conceptual flow:** `Component → Contract Interface → App Implementation → API / DB / Cache`

---

## Core Components

### AXBlock

**The Composable entry point that connects a BlockModel to the UI.**

`AXBlock` is a generic `@Composable` function that:
1. **Collects state** from the model's `StateFlow` via lifecycle‑aware `collectAsStateWithLifecycle()`.
2. **Dispatches meta changes** into the model using `LaunchedEffect(meta)`.
3. **Re‑wires the communicator** when it changes via `LaunchedEffect(communicator)`.
4. **Renders the UI** by invoking the provided composable lambda with the current `AXBlockState`.

```kotlin
@Composable
fun <I : AXIntent, S : AXState, O : AXOutput> AXBlock(
    model: AXBlockModel<I, S, O>,
    communicator: AXCommunicator<I, O>,
    meta: AXDomainSpecificMeta,
    composable: @Composable (state: AXBlockState<S>) -> Unit
)
```

**Key behaviour:** When `meta` changes (e.g. a new filter is selected), the block automatically re‑initialises by dispatching `AXCommonIntent.AXMetaUpdated(meta)`.

---

### AXBlockModel

**The "brain" of every Block — manages all core business logic.**

`AXBlockModel<I, S, O>` extends Android's `ViewModel` and serves as the single source of truth for a block's state.

| Responsibility | API |
|----------------|-----|
| Load data on initialisation | `loadBlock(meta)` — abstract; called when meta is set or updated |
| Handle user/system intents | `handleIntent(intent: I)` — abstract; processes block‑specific intents |
| Publish state to UI | `publishState(newState: AXBlockState<S>)` |
| Update state with a reducer | `updateState(reducer: (AXBlockState<S>) -> AXBlockState<S>)` |
| Dispatch intent to communicator | `dispatchIntent(intent: I)` |
| Emit output to parent | `emitOutput(output: O)` |
| Swap communicator at runtime | `updateCommunicator(new)` — re‑wires intent/output channels |

**Meta‑driven loading:**
- On the first `AXMetaUpdated`, if `initialLoad == true`, calls `loadBlock()`.
- On subsequent changes, only calls `loadBlock()` when the meta value has actually changed.

---

### AXCommunicator

**The transport layer enabling decoupled communication between a Block and its Model, or between two separate BlockModels.**

```
┌────────────────────────── AXCommunicator ──────────────────────────┐
│                                                                    │
│  Intent Channel (UI / parent actions) ─────────────► AXBlockModel  │
│  Output SharedFlow (results / events) ◄──────────── AXBlockModel   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

| Direction | Mechanism | Details |
|-----------|-----------|---------|
| **Intent → Model** | Buffered `Channel<AXIntent>` (capacity 64, `DROP_OLDEST`) | Ensures intents are buffered and processed sequentially. |
| **Model → Parent** | `MutableSharedFlow<O>` | Emits outputs for the parent or host to observe and react to. |

**Key APIs:**

```kotlin
abstract class AXCommunicator<I : AXIntent, O : AXOutput> {
    suspend fun dispatchIntent(intent: I)       // Send intent to model
    suspend fun emitOutput(output: O)           // Send output from model
    val output: SharedFlow<O>                   // Observe outputs
    abstract fun onOutput(output: O)            // Parent callback
}
```

**Specialisations:**
- `AXListCommunicator<T, I, O>` — Extends `AXCommunicator` specifically for list blocks, adding a typed `dispatchListIntent()` method.

> **Design note:** Components interact through communicators without direct dependencies, ensuring a fully modular architecture.

---

### AXWidget

**A pure UI primitive — stateless and logic‑free.**

Unlike a Block, a Widget:
- Contains **no business logic** and has no dedicated BlockModel.
- Is entirely **stateless** — it receives all data and styling externally.
- Relies on a strict separation of **ViewData** (what to display) and **ViewProperties** (how to display it).

Widgets are the visual building blocks that Blocks compose together to render their UI.

---

### AXEventBus

**The global event dispatcher for screen‑to‑screen navigation.**

```kotlin
object AXEventBus {
    fun getNavigationEvents(screenId: String): SharedFlow<AXNavigationEvent>
    suspend fun emitNavigationEvent(event: AXNavigationEvent, screenId: String)
    fun clearNavigationFlow(screenId: String)
}
```

| Feature | Detail |
|---------|--------|
| **Thread‑safe isolation** | Uses a `ConcurrentHashMap` to isolate navigation flows by `screenId`. |
| **Broadcasting** | Provides a `SharedFlow` mechanism for broadcasting `AXNavigationEvent` to multiple observers simultaneously. |
| **Scoped cleanup** | `clearNavigationFlow(screenId)` prevents leaks when a screen leaves composition. |

---

## State & Data Types

### AXDomainSpecificMeta

Immutable, `@Serializable` context data passed to a block when it is created or updated.

```kotlin
@Serializable
abstract class AXDomainSpecificMeta
```

**Example:**
```kotlin
data class MyViewMeta(
    val orgId: String,
    val departmentId: String,
    val viewId: String
) : AXDomainSpecificMeta()
```

---

### AXIntent

The base class for all user or system events directed at a block.

```kotlin
open class AXIntent

sealed class AXCommonIntent : AXIntent() {
    data class AXMetaUpdated<M : AXDomainSpecificMeta>(val meta: M) : AXCommonIntent()
}
```

Blocks define their own sealed intent hierarchies (e.g. `AXListIntent`, `AXPickerOptionsIntent`).

---

### AXState & AXBlockState

```kotlin
open class AXState

sealed class AXBlockState<out S : AXState> {
    data object Invisible : AXBlockState<Nothing>()     // Block not yet loaded
    data class Visible<S : AXState>(val state: S) : AXBlockState<S>()  // Block loaded with state
}
```

- **`Invisible`** — The block has not finished loading; the UI shows a placeholder.
- **`Visible(state)`** — The block is ready; the UI renders the provided state.
- **Helper:** `fun <S : AXState> S.toBlockState(): AXBlockState.Visible<S>`

---

### AXOutput

One‑way events emitted from a Block to its parent, navigation layer, or host app.

```kotlin
open class AXOutput
open class AXNavigationOutput : AXOutput()   // Specifically for navigation events
```

---

### AXResult

A sealed result type for all data operations, ensuring uniform success/failure handling:

```kotlin
sealed class AXResult<out T> {
    class Success<T>(val data: T) : AXResult<T>()
    class Failure(val errorInfo: AXErrorInfo) : AXResult<Nothing>()
}

data class AXErrorInfo(
    val error: AXError = AXError.UnknownError,
    val message: String = "Unknown Error",
    val throwable: Throwable? = null
)
```

**Built‑in error types:** `NoNetwork`, `ServerRequestFailed`, `FailedDbAction`, `PermissionDenied`, `PreconditionFailed`, `NotFound`, `ThrottleLimitExceeded`, `ForbiddenMobileAccess`, `UnknownError`.

---

## Navigation System

AXKit provides a fully decoupled, output‑driven navigation system where **BlockModels never touch `NavController` directly**.

### AXNavTarget

Standardises how navigation intentions are communicated to the handler.

```kotlin
sealed class AXNavTarget(open val navController: NavHostController) {
    data class NavigateTo(
        val route: String,
        override val navController: NavHostController
    ) : AXNavTarget(navController)

    data class NavigateBack(
        override val navController: NavHostController
    ) : AXNavTarget(navController)
}
```

| Target | Behaviour |
|--------|-----------|
| `NavigateTo(route)` | Navigates forward to the specified route string. |
| `NavigateBack` | Triggers a standard pop‑stack operation on the current navigation controller. |

---

### AXNavScreenConfig

Defines the configuration for an individual screen within the navigation graph.

```kotlin
data class AXNavScreenConfig(
    val route: String,
    val content: @Composable (NavBackStackEntry) -> Unit
)
```

Pairs a unique route string with a Composable content block, allowing the passing of `NavBackStackEntry` for argument extraction.

---

### AXNavOrchestrator

**The primary Composable for managing dynamic navigation within the AXKit ecosystem.**

```kotlin
@Composable
fun AXNavOrchestrator(
    screenId: String,
    navController: NavHostController,
    navigationEventHandler: AXNavigationEventHandler,
    startDestination: String,
    screens: List<AXNavScreenConfig>,
    modifier: Modifier = Modifier
)
```

Responsibilities:
- Bridges `AXNavigationEventHandler` with the Jetpack Compose `NavHost` to execute real‑time routing logic.
- Iterates through `AXNavScreenConfig` objects to dynamically build the navigation graph.
- Manages observer lifecycles via `DisposableEffect` — starts the handler on compose, clears on dispose.

---

### AXNavigationEventHandler

**Orchestrates screen transitions by reacting to outputs and event triggers.**

```kotlin
abstract class AXNavigationEventHandler(
    protected open val scope: CoroutineScope
) {
    abstract fun getNavTarget(output: AXOutput): AXNavTarget?
    fun start(screenId: String)
    fun clear(screenId: String)
}
```

| Method | Purpose |
|--------|---------|
| `getNavTarget(output)` | Abstract — map an `AXOutput` to a concrete `AXNavTarget` (or `null` to ignore). |
| `start(screenId)` | Subscribe to `AXEventBus` for the given `screenId` and begin processing navigation events. |
| `clear(screenId)` | Remove the flow from `AXEventBus` to prevent leaks. |

Transforms incoming `AXOutput` events into concrete `AXNavTarget` actions, keeping the `AXBlockModel` fully decoupled from the navigation framework.

---

### Navigation Flow

**End‑to‑end flow:**
1. A `BlockModel` emits a  `AXOutput`.
2. A host component (often through the Communicator's `onOutput()` callback) converts that output into an `AXNavigationEvent` and emits it to the `AXEventBus` scoped by `screenId`.
3. The `AXNavigationEventHandler` collects the event, maps it to an `AXNavTarget` via `getNavTarget()`.
4. The handler executes the navigation action on the `NavController` (`navigate()` or `popBackStack()`).
5. `AXNavOrchestrator` cleans up the event flow when the screen leaves composition.

---

## Dependency Injection

AXKit uses a lightweight singleton‑based injection pattern via `AXKit.init()`.

```kotlin
object AXKit {
    lateinit var dataManager: AXDatamanager
    fun init(dataManager: AXDatamanager)
}

interface AXDatamanager {
    fun providePickerOptionsContract(): AXPickerOptionsContract
    fun provideLabeledDropdownDataContract(): AXLabeledPickerDataContract
}
```

> **Important:** `AXKit.init(dataManager)` must be called **before** composing any block that depends on `AXDatamanager` contracts — typically in `Application.onCreate()`.

The host app provides contract implementations for **AXDatamanager-backed blocks** through `AXKit.init()`:

---

## Tech Stack

| Technology | Details |
|-----------|---------|
| **Language** | Kotlin |
| **UI Framework** | Jetpack Compose (Material 3) |
| **Architecture** | MVI — Unidirectional Data Flow |
| **State Management** | Kotlin `StateFlow` + Compose `collectAsStateWithLifecycle` |
| **Navigation** | Jetpack Navigation Compose + AXKit Event Bus |
| **Min SDK** | 23 |
| **Target/Compile SDK** | 36 |
