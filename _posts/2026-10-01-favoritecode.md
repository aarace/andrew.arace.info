---
layout: post
title: "Favorite Code"
description: "One of my favorite pieces of code"
tags: [dotnet]
---

I sometimes think back to this little block of code that I am proud of:

```csharp
private Queue<int> DistributeAmountToBuckets(int amount, int buckets)
{
    int initialAmount = amount / buckets;
    int disbursedAmount = initialAmount * buckets;
    int remaining = amount - disbursedAmount;
    var results = new Queue<int>();
    for (int i = 0; i < buckets; i++)
    {
        results.Enqueue(i < remaining ? initialAmount + 1 : initialAmount);
    }
    return results;
}
```

It distributes an `int` amount into an `int` number of buckets. It was used to distribute pennies amongst a number of people's wallets, with overflow spread out in order of the buckets.
For example, if you had 500 pennies ($5) to distribute among 3 people:

1. `int initialAmount` = 500 / 3 = 166
1. `int disbursedAmount` = 166 * 3 = 498
1. `int remaining` = 500 - 498 = 2
1. so the returned `results` queue ends up with: `[167, 167, 166]`, meaning a $5 prize pool would be split up between 3 people as: ($1.67, $1.67, $1.66)

## Why it works

The obvious approach is to divide the money as a decimal and round each share: $5.00 / 3 = $1.666…, which rounds to $1.67 for everyone. But $1.67 × 3 = $5.01, so you've paid out a penny that doesn't exist. Round down instead and you get $4.98, with two pennies left over and no one to give them to.

This code never touches decimals. Integer division gives everyone the same base share, and the leftover pennies are handed out one at a time. The remainder is always smaller than the number of buckets, so:

- the shares always add up to exactly the original amount, with no penny lost or invented
- no one ever gets more than one penny more than anyone else

I know it could be more succinct (it's over 10 years old at this point) (could use `Math.DivRem` to collaspe the 3 variable lines into one), but I'm proud of its simple complexity.