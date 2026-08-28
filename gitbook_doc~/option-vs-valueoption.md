# Option vs ValueOption

## Overview
`AceLand.Optional` deliberately ships **two** Option types instead of one. C# treats reference types
(classes) and value types (structs) very differently when it comes to "no value", so each gets its
own purpose-built wrapper:

| Type | For | Constraint | Empty means |
| --- | --- | --- | --- |
| `Option<T>` | reference types (classes) | `where T : class` | the reference is `null` |
| `ValueOption<T>` | value types (unmanaged structs) | `where T : unmanaged` | the underlying `T?` has no value |

Picking the right one is usually automatic — the compiler will simply refuse the wrong constraint —
but understanding *why* they differ helps you use them well.

---

## Option&lt;T&gt; — for classes
Use `Option<T>` when `T` is a class: a `Transform`, a `GameObject`, a `Player`, a `string`, or any of
your own reference types. Internally it stores a single `T?` reference; "None" is simply a `null`
reference.

```csharp
using AceLand.Optional;

// A repository lookup that may not find anything.
public Option<Player> FindPlayer(string id)
{
    Player found = _players.GetValueOrDefault(id); // may be null
    return found.ToOption();                       // null -> None, otherwise Some
}

// Caller never sees a raw null.
string name = FindPlayer("p_01")
    .Map(p => p.Name)
    .Reduce("<unknown>");
```

{% hint style="info" %}
`string` is a reference type, so it uses `Option<string>`, **not** `ValueOption<string>`.
{% endhint %}

---

## ValueOption&lt;T&gt; — for unmanaged structs
Use `ValueOption<T>` when `T` is an unmanaged value type: `int`, `float`, `bool`, `Vector3`, an
`enum`, or your own struct made of such fields. Internally it stores a `T?` (`Nullable<T>`); "None"
is a `Nullable` with no value.

```csharp
using AceLand.Optional;

// A parse that may fail.
public ValueOption<int> TryParseScore(string raw) =>
    int.TryParse(raw, out var value)
        ? ValueOption<int>.Some(value)
        : ValueOption<int>.None();

int score = TryParseScore(userInput)
    .WhereValue(v => v >= 0)   // reject negatives (see note below on extension names)
    .Reduce(0);
```

{% hint style="warning" %}
The `unmanaged` constraint means `ValueOption<T>` cannot hold a class, a `string`, or a struct that
contains references. Those belong in `Option<T>`.
{% endhint %}

---

## Why the split matters
If there were only a single `Option<T>` with a `struct` constraint, then:

- reference types could not use it (a class is not a `struct`), and
- for value types, representing "empty" would need an extra flag, doubling the checks.

By splitting them, each type uses the most natural, allocation-free representation of "empty":
`null` for references and `Nullable<T>` for values. Both remain `struct`s themselves, so wrapping a
value never allocates on the heap.

{% hint style="info" %}
Version `3.0.0` tightened the `ValueOption<T>` constraint from `struct` to `unmanaged`. This makes
the empty representation exact and keeps the value blittable, at the cost of no longer allowing
structs that contain managed references (which were never a good fit anyway).
{% endhint %}

---

## Creating each type
| Goal | Reference (`Option<T>`) | Value (`ValueOption<T>`) |
| --- | --- | --- |
| Wrap existing data | `obj.ToOption()` | `value.ToValueOption()` |
| Explicit "has value" | `Option<T>.Some(obj)` | `ValueOption<T>.Some(value)` |
| Explicit "empty" | `Option<T>.None()` | `ValueOption<T>.None()` |
| Implicit from raw | `Option<T> o = obj;` | `ValueOption<T> o = value;` |

Both types also convert implicitly to `bool`, so either can be tested directly in an `if`:

```csharp
Option<Player> maybePlayer = FindPlayer("p_01");
if (maybePlayer) { /* present */ }

ValueOption<int> maybeScore = TryParseScore(raw);
if (maybeScore) { /* present */ }
```

---

## Crossing between the two
Real code often needs to move from a reference to one of its value fields, or from a value into a
lookup that returns a reference. The `MapValue` / `Map` / `MapOptional` families handle both
directions — see [Functional API](functional-api.md#bridging-reference-and-value).

```csharp
// Player (class) -> Health (int)
int health = player.ToOption()
    .MapValue(p => p.Health)   // Option<Player> -> ValueOption<int>
    .Reduce(0);
```

---

## Best Practices
- Let the compiler guide you: if a constraint error appears, you are simply using the wrong one of
  the two types.
- Keep `string` and Unity objects (`Transform`, `GameObject`, …) in `Option<T>`.
- Keep numbers, enums and small blittable structs in `ValueOption<T>`.
- Use `MapValue` when a class exposes a numeric/struct field you want to continue the chain on.
