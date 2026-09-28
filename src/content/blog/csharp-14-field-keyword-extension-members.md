---
title: "C# 14 in Production: What the `field` Keyword and Extension Members Actually Change"
description: "C# 14 shipped with .NET 10. The two features that survived contact with my codebase are the `field` keyword and extension blocks — here's how they changed real code, and which of the other features I'm not using."
pubDate: 2026-09-28
tags: [csharp, dotnet, language]
draft: false
---

Every C# version ships a grab-bag: two features you'll use daily, a handful that are quietly transformative for library authors, and a long tail that solves a problem you don't have.

C# 14 shipped with .NET 10. I've been running it in production for a while now, so this is a report from actual use rather than a summary of the release notes.

The short version: **`field` earned its place. Extension members are transformative if you write libraries and irrelevant if you don't. The rest is a very good argument for upgrading and very little reason to rewrite anything.**

## The `field` keyword is the one you'll use daily

Before C# 14, if you wanted a property that normalized its input, you had three options. All of them are noise.

Option one, the full backing field:

```csharp
private string _email;

public string Email
{
    get => _email;
    set => _email = value?.Trim()?.ToLowerInvariant() ?? string.Empty;
}
```

Option two, an expression-bodied property with a null check in the setter — same problem, half the ceremony.

Option three, a method. Which means the property is now `GetEmail()` and you've lost the property, every `nameof` reference, every object initializer, and every binding expression in your frontend.

C# 14 lets you write an accessor body without ever naming the backing field:

```csharp
public string Email
{
    get;
    set => field = value?.Trim()?.ToLowerInvariant() ?? string.Empty;
}
```

`field` is a contextual keyword. The compiler synthesizes a backing field, and you only write the accessor that actually needs a body. The other one is a `get;`-only auto-implemented accessor.

**You can do this for one accessor or both.** The most common real use is guarding or normalizing in the setter while leaving the getter alone.

### The lazy-init case that got a lot shorter

This is where I'd argue the feature really pays:

```csharp
public SqlConnection Connection
{
    get => field ??= new SqlConnection(_connectionString);
}
```

Before this, that was a private field, a `??=` guard, and a lock if you cared about thread safety. Now the shape of the property and the shape of the code match.

### Property change notification without ceremony

If you hand-rolled `INotifyPropertyChanged` (or used a source generator), the equal-check is the part everyone writes:

```csharp
public string DisplayName
{
    get;
    set
    {
        if (field == value) return;
        field = value;
        OnPropertyChanged();
    }
}
```

Note the `string?` question: `field == value` on a nullable-annotated property is a value comparison, not a reference one, so this behaves correctly.

### The gotcha worth knowing about

If you already have a member named `field` — a property, a field, anything — the contextual keyword is ambiguous inside that type. Two escapes:

```csharp
public int Value
{
    get => this.field;   // this.field = the member
    set => field = value; // field = the keyword
}
```

Or `@field`. The compiler will tell you which you need if you hit it. This is a real but small source-compat concern for codebases that use `field` as an identifier — it's the only genuinely breaking edge of the feature, and the fix is a rename.

## Extension members: the biggest change nobody notices yet

C# 14 finally lets extension members be *members*, not just methods. This unlocks three things that were previously impossible in C#:

**Extension properties.** Previously you had to write a static extension *method* and hope people called it like a property:

```csharp
public static bool IsEmpty(this IEnumerable source) => !source.GetEnumerator().MoveNext();
```

which reads as `list.IsEmpty()` — a method call dressed up as a property. Now:

```csharp
public static class EnumerableExtensions
{
    extension<TSource>(IEnumerable<TSource> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

and it reads as `list.IsEmpty`. No parentheses, correct IntelliSense, discoverable as a property.

**Extension blocks.** The `extension(...)` syntax groups everything you extend a type with in one declaration, instead of a scattered `static class` with a naming convention. The receiver is a real named parameter, so members can actually use it — which extension *methods* have always been able to do, but which extension properties cannot, since a property getter takes no parameters.

**Static extension members.** This is the one that surprised me. You can extend the *type* rather than an instance of it:

```csharp
extension<TSource>(IEnumerable<TSource>)
{
    public static IEnumerable<TSource> Identity => Enumerable.Empty<TSource>();

    public static IEnumerable<TSource> Combine(
        IEnumerable<TSource> first,
        IEnumerable<TSource> second)
        => first.Concat(second);
}
```

Called as `IEnumerable<int>.Identity`, and — because the receiver is a type — **user-defined operators can now be written as extension members**:

```csharp
extension<TSource>(IEnumerable<TSource>)
{
    public static IEnumerable<TSource> operator +(
        IEnumerable<TSource> left, IEnumerable<TSource> right)
        => left.Concat(right);
}
```

If you've ever wanted `+` to work on your own type, this is the language-sanctioned way. It used to be impossible without source generators or reflection hacks.

**The honest caveat:** this is a library-author feature. If you write application code and consume libraries, your daily usage is unchanged. If you maintain something like a shared extensions library, this meaningfully improves the API surface you can offer.

## First-class `Span<T>`

C# 14 gives `Span<T>` and `ReadOnlySpan<T>` proper language status, adding implicit conversions between `ReadOnlySpan<T>`, `Span<T>`, and `T[]`, and letting spans be extension method receivers.

The practical effect: extension methods on `ReadOnlySpan<char>` now work, and generic type inference behaves properly when spans are in the mix. This is mostly a benefit to runtime and framework authors, and it's the reason some of the BCL APIs got faster. You won't notice it day to day, which is the highest compliment a compiler feature can earn.

## The small ones that quietly remove annoyance

**`nameof(List<>)` now works.** Previously `nameof` required a closed generic type — you had to write `nameof(List<int>)` and get `"List"`, which is a lie when your type parameter is `T`. Now:

```csharp
nameof(List<>)              // "List"
nameof(Dictionary<,>)       // "Dictionary"
```

**Lambda parameters can carry modifiers without types.** Modifiers like `out`, `ref`, `in`, `scoped`, and `ref readonly` previously forced you to type *every* parameter in the list:

```csharp
// Before: both parameters must be typed
TryParse<int> parse = (string text, out int result) => int.TryParse(text, out result);

// Now: only the parameter you care about
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

(`params` still requires a fully typed list — that's a deliberate carve-out.)

**Partial constructors and partial events.** Partial members now cover constructors and events, not just methods and properties. Partial constructors must have exactly one defining and one implementing declaration, and only the implementing one may carry a `this()`/`base()` initializer. This matters for source generators that need to inject constructor logic — a real niche, but a previously-impossible one.

**Null-conditional assignment** and **user-defined compound assignment** round out the release. Both are ergonomic; neither will change how you write code.

## What I'd actually upgrade for

If you're on C# 12 or older, the honest case is:

1. `field` — you'll use it in the first file you open.
2. Extension members — only if you own a library.
3. Everything else — a reason to upgrade, not a reason to refactor.

If you're already on C# 13, the C# 13 features are the higher-value ones: `params` collections (so `params ReadOnlySpan<T>` and no more array allocations at call sites), the new `Lock` type, `ref struct` implementing interfaces, and overload resolution priority. C# 14's `field` slots in alongside those comfortably.

## The rule I use for language features

A new language feature should be adopted when it's **faster to write than to not write**, not merely when it's available. `field` clears that bar. Extension members clear it for library authors and no one else. I wrote a small number of posts about EF Core and Postgres where the fix was a missing index or a stray `NoTracking` — the language version was never the variable. It's still nice when a language change removes a class of small papercuts, but it isn't where systems get fixed.
