# Udon Observable

Reactive observable streams for VRChat worlds — built for UdonSharp.

Lets you push values through typed streams and react to changes via subscriptions, instead of polling state every frame or writing one-off event wiring by hand.

If you find this useful, consider supporting development: [![Ko-fi](https://img.shields.io/badge/Ko--fi-Support%20me-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/theexilednivera)

---

## Installation

Install via the **VRChat Creator Companion (VCC)**: <https://exilednivera.github.io/vpm/>

**Requirement:** VRChat Worlds SDK `>= 3.10.0`

---

## Quick Start

### 1. Place the factory in your scene

In the Unity menu bar go to **Tools → ExiledNivera → Create Udon Database**.

This spawns the pre-wired `ObservableFactory` prefab into the open scene and selects it. The menu item is greyed out if a factory is already present. You only need to assign a **Spawn Parent** transform on the component — a plain empty GameObject works fine.

### 2. Reference the factory in your script

```csharp
[SerializeField] private ObservableFactory _factory;
```

Wire it in the Inspector. That is the only Inspector step required — everything else happens in code.

### 3. Create an observable

```csharp
private ObservableInt _score;

private void Start() {
    _score = _factory.CreateInt("PlayerScore");
}
```

Available types: `CreateInt`, `CreateFloat`, `CreateBool`, `CreateString`, `CreateObject`, `CreateToken` (untyped).

### 4. Subscribe

Your subscriber must be an `UdonSharpBehaviour` with a matching `public void` method (no parameters). Read the current value from your stored reference inside the callback.

```csharp
_score.token.Subscribe(this, nameof(OnScoreChanged));

public void OnScoreChanged() {
    Debug.Log("Score is now: " + _score.IntValue);
}
```

### 5. Emit a value

```csharp
_score.Next(100);   // emits 100 to all subscribers
```

### 6. Add operators

Operators are chained directly on the observable. Each returns its own chainable type, so you can keep building the pipeline.

```csharp
// Only react when score drops below 10, and only if the value actually changed
_score
    .Where(WhereIntMode.LessThan, 10)
    .Distinct()
    .Subscribe(this, nameof(OnLowScore));

public void OnLowScore() {
    Debug.Log("Low score warning: " + _score.IntValue);
}
```

---

## Operators

### Where — filter values

| Method | Description |
|---|---|
| `.Where(WhereIntMode, int)` | Integer comparisons: `GreaterThan`, `LessThan`, `Between`, `Equals`, `NotEquals` |
| `.Where(WhereFloatMode, float)` | Float comparisons: same modes plus `NotEquals` |
| `.Where(WhereStringMode, string)` | `Equals`, `NotEquals`, `StartsWith`, `Contains`, `NonEmpty` |
| `.WhereBetween(min, max)` | Shorthand for between — available on int and float |
| `.WhereNonEmpty()` | String-specific: passes only non-empty values |
| `.WhereBoolIs(bool)` | Bool-specific: passes only `true` or only `false` |

### Distinct / First / Take — deduplicate or limit

| Method | Description |
|---|---|
| `.Distinct()` | Only emits when the value is different from the last emit |
| `.First()` | Emits once, then stops |
| `.Take(n)` | Emits up to `n` times, then stops |

### Time-based — control emission frequency

| Method | Description |
|---|---|
| `.Debounce(seconds)` | Waits for silence — emits only after no new value arrives for `seconds` |
| `.Throttle(seconds)` | Emits at most once per `seconds` interval |
| `.Sample(seconds)` | Emits the latest value on a fixed timer, every `seconds` |

### Chaining operators

Every operator returns a chainable type. You can keep stacking:

```csharp
_health
    .Where(WhereFloatMode.LessThan, 30f)   // only low health
    .Distinct()                             // skip if already low
    .Debounce(0.5f)                         // wait for stability
    .Subscribe(this, nameof(OnCriticalHealth));
```

---

## Reading the current value

Typed observables expose a convenience property:

```csharp
int  current = _score.IntValue;
float hp     = _health.FloatValue;
bool isOn    = _flag.BoolValue;
string name  = _label.StringValue;
```

For operators (WhereToken, AnyToken, DebounceToken), read from the raw output token:

```csharp
int filtered = _myWhereOp.output.Value.Int;
```

---

## Re-emitting the current value

Call `Replay()` on the underlying token to re-broadcast the current value to all subscribers — useful for late-joining listeners that need the initial state: 

```csharp
_score.token.Replay();
```

---

## Unsubscribing

```csharp
_score.token.Unsubscribe(this, nameof(OnScoreChanged));
```

---

## Limitations

- Callback methods must be `public void` with no parameters.
- Synced state must be managed separately with `[UdonSynced]` fields.
- Operators are one-way pipelines — no built-in combine/merge of multiple sources.
