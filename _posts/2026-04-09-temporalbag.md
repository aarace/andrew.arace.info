---
layout: post
title: "TemporalBag"
description: "A thread-safe .NET collection whose items expire automatically, published on NuGet."
tags: [dotnet, open-source, nuget]
---
On a side project I needed a .NET collection where entries expire automatically after a set time. It also had to support:
- thread safety
- enumeration (`IEnumerable<T>`, so LINQ just works)
- point-in-time snapshots
- cleanup that doesn't block readers or writers

`MemoryCache` is keyed and heavier than I wanted. `ConcurrentBag` has no concept of time. Nothing I found covered all four, so I wrote my own and published it as an open source package.

- Source Code: [on github](https://github.com/aarace/TemporalBag)
- Nuget Package: [on nuget](https://www.nuget.org/packages/TemporalBag)

## Usage

```
dotnet add package TemporalBag
```

Create a bag with a default time-to-live, add items, and enumerate. Expired items are never returned.

```csharp
using TemporalBag.Core;

// Items expire after 30 seconds; background cleanup runs every 10 seconds (default).
using var bag = new TemporalBag<string>(maxAge: TimeSpan.FromSeconds(30));

bag.Add("hello");
bag.Add("short-lived", TimeSpan.FromSeconds(5));  // per-item TTL override
bag.Add("long-lived",  TimeSpan.FromHours(1));

foreach (var item in bag)
    Console.WriteLine(item);

var matches = bag.Where(x => x.StartsWith("long")).ToList();
bool exists = bag.Contains("hello");
int live    = bag.Count;
```

`AsLive()` returns each item along with how much time it has left:

```csharp
foreach (var (value, remaining) in bag.AsLive())
    Console.WriteLine($"{value} expires in {remaining.TotalSeconds:F1}s");
```

## How it works

### Each item stores its own expiration

Every value is wrapped with a deadline measured in `Stopwatch` ticks. `Stopwatch` uses a monotonic clock, so changes to the system clock (NTP sync, DST) can't make items expire early or late.

```csharp
private class TtlValue(TValue value, TimeSpan ttl)
{
    public TValue Value { get; } = value;
    private readonly long _tickCountWhenToKill =
        Stopwatch.GetTimestamp() + (long)(ttl.TotalSeconds * Stopwatch.Frequency);

    public bool IsExpired(long currTickCount) => currTickCount > _tickCountWhenToKill;
}
```

### Reads filter out expired items

Enumeration, `Count`, and `Contains` all read the clock once at the start and skip anything past its deadline. Every item in one pass is checked against the same instant, so you get a consistent snapshot. Because of that, reads are always correct whether or not cleanup has run yet.

```csharp
private IEnumerator<TValue> GetEnumeratorCore()
{
    var currTime = Stopwatch.GetTimestamp();
    foreach (var item in _list)
    {
        if (!item.IsExpired(currTime))
            yield return item.Value;
    }
}
```

### Cleanup swaps in a new bag

The items live in a `ConcurrentBag`. A background timer periodically compacts it. Instead of removing items one at a time, it atomically swaps in an empty bag and copies over only the items that are still live. Any `Add()` that happens during cleanup goes straight into the new bag, so nothing is lost and writers never wait on a lock.

```csharp
private void DoCleanUp()
{
    var currTime = Stopwatch.GetTimestamp();
    var oldList = Interlocked.Exchange(ref _list, new ConcurrentBag<TtlValue>());
    foreach (var item in oldList)
    {
        if (!item.IsExpired(currTime))
            _list.Add(item);
    }
}
```

`Clear()` uses the same swap, so emptying the bag is a single atomic operation no matter how many items it holds.

A `SemaphoreSlim` makes sure only one cleanup runs at a time. A manual `CleanUp()` call uses `Wait(0)`, so if a cleanup is already running it returns `false` right away instead of waiting:

```csharp
public bool CleanUp()
{
    if (_cleanupLock.Wait(0))
    {
        try { DoCleanUp(); return true; }
        finally { _cleanupLock.Release(); }
    }
    return false;
}
```

The bag implements `IDisposable` to stop the background timer, so use it in a `using` block or dispose it when you're done.

## Targets

`netstandard2.0`, `net8.0`, and `net10.0`. MIT licensed.
