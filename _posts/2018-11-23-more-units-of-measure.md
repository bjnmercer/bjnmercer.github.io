---
layout: post
title: "More Fun With F# Units of Measure"
date: 2018-11-23
---

Wow, 6 months have passed since the last post! I have actually been working on my projects since, just have not found time to actually blog about it. I figure I would get back into the swing of things with a quick post.

This article focuses on F# Units of Measure; perhaps *the* killer language feature.

# Calculating the cost of a mug of coffee

The problem was rather simple: I wanted to calculate how much each mug of coffee was costing me. I use an electric drip filter machine, which brews about 1.3 litres from 68 grams of coffee beans.

For fun, I thought I would attach the appropriate units. Starting off

```fsharp
[<Measure>] type gram
[<Measure>] type dollar
```

I think F# already comes with certain measures already built in, but I might as well just define them here.

Next, my coffee beans cost $9 and mass 250grams.

```fsharp
let cost = 9.0<dollar>
let mass = 250.0<gram>;;
```

So now it's a trivial calculation to get the dollars per gram

```fsharp
let dollarsPerGram = mass/cost;;
```

...Except, for some reason I always manage to stuff this calculation up. The name of the variable is literally dollarsPerGram yet I've got gramsPerDollar, so to speak.

Thanks to F#'s type inference which extends to inferring new units of measure from existing ones, I saw my mistake immediately

```fsharp
val dollarsPerGram : float<gram/dollar> = 27.77777778
```

The inferred unit of gram/dollar shows I've once again mucked up the calculation.

```fsharp
let dollarsPerGram = cost/mass;;
val dollarsPerGram : float<dollar/gram> = 0.036
```

You can use descriptive variable names all you like, but only the F# compiler will save you from ID10T errors!

# End of article

And in case you were wondering, it's costing me about 50 cents per mug.