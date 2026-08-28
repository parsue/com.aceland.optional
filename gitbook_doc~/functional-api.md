# Functional API

## Overview
An `Option` is more than a null check — it is a small pipeline. You start with a possibly-missing
value, run it through any number of **transform** and **filter** steps, then **collapse** it back to
a concrete value. Every step quietly does nothing when the value is absent, so you write the "happy
path" once and never repeat the empty-case handling.

This page walks through the three building blocks — `Map`, `Where`, `Reduce` — plus the helpers that
bridge between `Option<T>` and `ValueOption<T>`.

{% hint style="info" %}
The transform and filter methods take a function (`Func<...>`) that describes *what to do with the
value if it is present*. These run on the normal C# runtime, so use any code you like inside them.
{% endhint %}

---

## Map — transform the value
`Map` applies a function to the value **only if it is present**, producing a new Option. If the
source is empty, the result is empty too — your function is never called.

```csharp
// Option<Player> -> Option<string>
Option<string> name = player.ToOption()
    .Map(p => p.Name);
```

There are four `Map` variants, one for each combination of "what I have" and "what I want":

| Method | From | To | Use when |
| --- | --- | --- | --- |
| `Map` | `Option<T>` | `Option<TResult>` | class → class |
| `MapValue` | `Option<T>` | `ValueOption<TResult>` | class → value (e.g. a numeric field) |
| `Map` | `ValueOption<T>` | `Option<TResult>` | value → class |
| `MapValue` | `ValueOption<T>` | `ValueOption<TResult>` | value → value |

```csharp
// class -> value: pull a numeric field out of an object
ValueOption<int> health = player.ToOption()
    .MapValue(p => p.Health);

// value -> value: pure arithmetic
ValueOption<int> doubled = score.ToValueOption()
    .MapValue(v => v * 2);
```

### MapOptional — when the transform itself may be empty
Sometimes your function *also* returns an Option (for example, a lookup that might fail). Using plain
`Map` would give you a nested `Option<Option<T>>`. `MapOptional` flattens it for you.

```csharp
// Each step may fail; the chain stays a single Option.
Option<Weapon> weapon = player.ToOption()
    .MapOptional(p => FindEquipped(p))   // FindEquipped returns Option<Weapon>
    ;

Option<Weapon> FindEquipped(Player p) =>
    p.EquippedWeapon.ToOption();
```

| Method | From | Function returns | Result |
| --- | --- | --- | --- |
| `MapOptional` | `Option<T>` | `Option<TResult>` | `Option<TResult>` |
| `MapOptionalValue` | `Option<T>` | `ValueOption<TResult>` | `ValueOption<TResult>` |
| `MapOptional` | `ValueOption<T>` | `Option<TResult>` | `Option<TResult>` |
| `MapOptionalValue` | `ValueOption<T>` | `ValueOption<TResult>` | `ValueOption<TResult>` |

---

## Where / WhereNot — filter the value
`Where` keeps the value only if it passes a predicate; otherwise the Option becomes empty.
`WhereNot` is the inverse — it keeps the value only if the predicate is **false**.

```csharp
// Keep the player only if alive.
Option<Player> alive = player.ToOption()
    .Where(p => p.IsAlive);

// Keep the player only if NOT banned.
Option<Player> allowed = player.ToOption()
    .WhereNot(p => p.IsBanned);
```

These read like guard clauses, but stay inside the chain instead of forcing a separate `if`.

```csharp
string label = player.ToOption()
    .Where(p => p.IsAlive)
    .WhereNot(p => p.IsBanned)
    .Map(p => p.Name)
    .Reduce("N/A");
```

For `ValueOption<T>`, the same methods exist as `Where` / `WhereNot`:

```csharp
ValueOption<int> positive = score.ToValueOption()
    .Where(v => v > 0);
```

---

## Reduce — collapse back to a real value
`Reduce` is how you leave the Option world. It always returns a concrete `T`: the contained value if
present, or a fallback you supply if empty.

```csharp
// Constant fallback.
Player p = maybePlayer.Reduce(Player.Dummy);

// Lazy fallback — only runs when the Option is empty.
Player q = maybePlayer.Reduce(() => CreateGuestPlayer());
```

{% hint style="warning" %}
Prefer the `Func<T>` overload when the fallback is expensive to build (allocations, lookups): it is
only evaluated if the Option is actually empty.
{% endhint %}

There is also `Get()`, which returns the raw underlying value — `T?` for a reference, or `T?`
(`Nullable`) for a value type. Use `Reduce` in normal code; reach for `Get()` only when you
deliberately want the nullable back.

---

## Bridging reference and value
Real chains often cross between the two worlds. The naming rule is simple:

- a method named `...Value` produces a `ValueOption<T>`;
- a method without `Value` produces an `Option<T>`.

```csharp
// class -> value -> back to a message string (class)
string status = player.ToOption()   // Option<Player>
    .MapValue(p => p.Health)         // ValueOption<int>
    .Where(h => h > 0)               // still ValueOption<int>
    .Map(h => $"HP: {h}")            // Option<string>
    .Reduce("dead");                 // string
```

---

## Extension helpers
`OptionalExtensions` provides convenient entry points so you rarely call `Some` / `None` directly:

| Extension | On | Result | Notes |
| --- | --- | --- | --- |
| `ToOption()` | `T?` (class) | `Option<T>` | `null` → None |
| `Where(pred)` | `T?` (class) | `Option<T>` | Some only if non-null **and** predicate true |
| `WhereNot(pred)` | `T?` (class) | `Option<T>` | Some only if non-null **and** predicate false |
| `ToValueOption()` | `T` (unmanaged) | `ValueOption<T>` | always Some |
| `WhereValue(pred)` | `T` (unmanaged) | `ValueOption<T>` | Some only if predicate true |
| `WhereValueNot(pred)` | `T` (unmanaged) | `ValueOption<T>` | Some only if predicate false |

```csharp
// Start a chain straight from a raw value with a filter built in.
Option<string> nonEmpty = rawName.Where(s => s.Length > 0);

ValueOption<int> validScore = rawScore.WhereValue(v => v >= 0);
```

{% hint style="info" %}
On value types the filtered entry points are named `WhereValue` / `WhereValueNot` to avoid clashing
with the class-based `Where` / `WhereNot` when both could apply.
{% endhint %}

---

## Putting it together
A complete, realistic chain from a raw Inspector field to a safe result:

```csharp
using AceLand.Optional;
using UnityEngine;

public class LabelBinder : MonoBehaviour
{
    [SerializeField] private Player player; // may be null

    public string BuildLabel()
    {
        return player
            .ToOption()                 // Option<Player>
            .Where(p => p.IsAlive)      // drop the dead
            .WhereNot(p => p.IsBanned)  // drop the banned
            .Map(p => p.Name)           // Option<string>
            .Reduce("Spectator");       // guaranteed string
    }
}
```

---

## Best Practices
- Keep `Map` / `Where` functions **pure** — transform or test the value, don't cause side effects.
- Use `MapOptional` (not `Map`) whenever your function returns an Option, to avoid nesting.
- Always finish with `Reduce`; treat `Get()` as an escape hatch, not the default.
- Choose the lazy `Reduce(() => ...)` overload when the fallback is costly to construct.
- Match the method suffix to the world you want next: `...Value` → value type, otherwise reference.
