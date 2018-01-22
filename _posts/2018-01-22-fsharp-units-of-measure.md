---
layout: post
title: "F# Units of Measure - and how they can help"
date: 2018-01-22
---

I have this car. It has a four speed manual transmission. At highway speeds -- typically 100km/h here in Australia -- I noticed that the engine seemed to be revving quite a lot. As a result, I was curious to determine the _Engine Revolutions Per Minute_ (RPM).

I have driven other cars of the same model with the same engine, but with a three speed automatic with overdrive. The third gear was a 1:1 ratio, and I remember that the engine would sit above 3000rpm at 100km/h if overdrive was disabled.

This car, according to online sources, indicates that the 4th gear is also 1:1. And to my ear, it seemed like the engine was revving about 3000rpm in 4th gear.

Well. I could install a tachometer... Or I can use some mathematics to approximate the RPM based on what I know. And of course, I used F# for the calculations.

Known facts:

* According to the compliance plate, it is equipped with an 3.909:1 ratio diff.
* The size of the tyres are 195/75 on 14 inch rims
* 4th gear has a ratio of 1:1
* The velocity of the car when doing 100km/h is about 27.77 meters per second

From this, the engine RPM can be calculated:
1. The wheel diameter is calculated by multiplying the tyre sidewall size by 2 and adding that to 14 inches. The two numbers 195/75 are tyre width and aspect ratio -- which means the tyre is 195mm wide, and the sidewall is 75% of that figure.
2. The wheel revolutions per second (RPS) can be calculated using the velocity divided by the circumference of the tyre.
3. The tailshaft RPS is calculated by multiplying wheel RPS by the diff ratio (3.909)
4. The engine RPS is the same as the tailshaft RPS
5. Finally, the RPM is obtained by multiplying by 60

The above is expressed as a series of calculations:

```fsharp
let wheelDiam = 7.0*0.0254+(0.195*0.75)

let wheelCirc = Math.PI * wheelDiam

let v = 100.0*1000.0/3600.0

let wheelRps = v / wheelCirc

let gearBoxRatio = 1.0

let engineRps = tailShaftRps * gearBoxRatio

let engineRpm = engineRps * 60.0

```

And now for the result:

```
val wheelDiam : float = 0.32405
val wheelCirc : float = 1.018033099
val v : float = 27.77777778
val wheelRps : float = 27.28573147
val tailShaftRps : float = 106.6599243
val gearBoxRatio : float = 1.0
val engineRps : float = 106.6599243
val engineRpm : float = 6399.595459
```

WHAT? Hang on a second, that wheel diameter seems suspiciously small! I know from the website [www.willtheyfit.com](www.willtheyfit.com) that a 195/75r14 tyre is about 648mm in diameter.

And of course, I have hit my first problem: the wheel circumference is expecting a diameter, but I accidently gave it a radius.

It should have been obvious given the variable name is "wheelDiam" that I was missing the required multiplier, but somehow I missed it.

## F# Units of Measure

Units of Measure have been around since at least F# 2.0. They introduce some extra metadata to numerical calculations, and hence also an extra type check that the compiler can enforce by way of compilation errors.

The most basic example is this:

```fsharp
[<Measure>] type mm
[<Measure>] type inch

let diameter = 10.0<inch>

let diameterInMM = diameter * 25.4<mm/inch>

val diameter : float<inch> = 10.0
val diameterInMM : float<mm> = 254.0
```

The unit "inch" is associated with the numeric literal "10.0". In effect, it will be a compile error if you attempt to feed this value to any function or calculation expecting meters.

```
> calcGravitationalPotentialEnergy 100.0<kg> 10.0<inch>;;

  calcGravitationalPotentialEnergy 100.0<kg> 10.0<inch>;;
  -------------------------------------------^^^^^^^^^^

error FS0001: Type mismatch. Expecting a
    float<m>    
but given a
    float<inch>    
The unit of measure 'm' does not match the unit of measure 'inch'

```

So, would have F#'s Units of Measure helped in the above engine RPM calculation? In this case, probably not.

You might be tempted to do something like this:

```fsharp
[<Measure>] type Radius
[<Measure>] type Diameter

let wheel = ((7.0*0.0254)+(0.195*0.75))*1.0<Radius>

let calcCircumference (wheelDiam:float<Diameter>) = Math.PI * wheelDiam
```

Which then you get a compile error when attempting to feed the wheel "Radius" to the function which expects a "Diameter":

![Image of what happens when you attempt to use the wrong units](/assets/fsunits1.PNG)

But this isn't as useful as you think. For example:

```fsharp
//wrong! actually a diameter
let wheel = ((14.0*0.0254)+(0.195*0.75*2.0))*1.0<Radius>
```

or

```fsharp
//also wrong! actually a radius
let wheel = ((7.0*0.0254)+(0.195*0.75))*1.0<Diameter>
```

Well, perhaps the extra type info might jog your brain into remembering that you are dealing with a diameter rather than a radius, but that did not help when I had explicitly called the variable `wheelDiam` earlier, either. The Units of Measure cannot catch `ID10T` calculation errors like this.

## Where Units of Measure help

Units of Measure help when you are dealing with conversions from unit to another, or calculations that involve different units.

Take that calculation of a wheel diameter for example.

```fsharp
let wheelDiam = (14.0*0.0254+(0.195*0.75)*2.0)
```

This has the potential for a classic problem: conversion between units. It's the error that [crashed a 150 million space probe into the atmosphere of Mars](https://en.wikipedia.org/wiki/Mars_Climate_Orbiter).

Due to historical reasons the rim diameter is specified in inches, but the tyre width is in millimeters. At least here in Australia. Thus, the calculation for wheel diameter is mixing two different units.

To add further to the mess, my calculations deal in meters per second, so the ultimate circumference should also be in meters.

Let's see how Unit of Measures help.

First, I start with some measurements not provided by `Microsoft.FSharp.Data.UnitSystems.SI`

```fsharp
[<Measure>] type mm
[<Measure>] type inch

```

Here is what my function looks like. 

```fsharp
let calcWheelDiam (rimDiam:float<inch>) (tyreWidth:float<mm>) (aspectRatio:float):float<m> =
    (rimDiam*0.0254<m/inch>)+((tyreWidth*0.001<m/mm>)*aspectRatio * 2.0)    
```

The UoM annotations clearly spell out the units of the inputs and outputs. Of course, you are free to put `195.0<inch>` and `14.0<mm>` into this function, but again Units of Measure cannot guard against errors. Although, it does make such errors more *obvious*. (195 inch rims? Really?)

Also note in the calculation itself, how you must specify the units in the conversion factors.

```
> calcWheelDiam 14.0<inch> 195.0<mm> 0.75;;
val it : float<m> = 0.6481
```

This is the correct answer.

One other nice feature of Units of Measure is that you can define safe conversion factors trivially. For example:

```fsharp
let inchesToMeters = 0.0254<m/inch>
let mmToMeters = 0.001<m/mm>
```

If you try to multiply against the wrong conversion factor, you will end up with a weird unit thanks to the compiler's type inference.

```
> 7.0<inch>*mmToMeters;;
val it : float<inch m/mm> = 0.007
```

...What is an inch-m/mm ? This is why it's a good idea to at least type-annotate your variable with their expected units like so:

```
> let m:float<m> = 7.0<inch>*mmToMeters;;

  let m:float<m> = 7.0<inch>*mmToMeters;;
  ---------------------------^^^^^^^^^^
 
error FS0001: The unit of measure 'm' does not match the unit of measure 'inch m/mm'

```

Or define functions which make the units explicit:

```
let convertInchesToMeters (inp:float<inch>) =
    inp * inchesToMeters;


let fail = convertInchesToMeters 100.0<mm>

>   let fail = convertInchesToMeters 100.0<mm>;;
  ---------------------------------^^^^^^^^^

error FS0001: Type mismatch. Expecting a
    float<inch>    
but given a
    float<mm>    
The unit of measure 'inch' does not match the unit of measure 'mm'
```


So now my wheel diameter calculation function looks like so:

```fsharp
let calcWheelDiam (rimDiam:float<inch>) (tyreWidth:float<mm>) (aspectRatio:float):float<m> =
    (rimDiam*inchesToMeters)+((tyreWidth*mmToMeters)*aspectRatio * 2.0)
```

How about the rest of the calculations? Lets try the circumference

```fsharp
let wheelDiam = calcWheelDiam 14.0<inch> 195.0<mm> 0.75

let wheelCirc = Math.PI * wheelDiam
```

```fsharp
val wheelDiam : float<m> = 0.6481

val wheelCirc : float<m> = 2.036066199
```

The circumference is correctly inferred to be in meters.

Now for the velocity. We know the velocity as 100km/h, but we want meters per second. Two more measures are required:

```fsharp
[<Measure>] type km
[<Measure>] type h

let v = 100.0<km/h>*1000.0<m/km>/3600.0<s/h>

val v : float<m/s> = 27.77777778
```

And the correct unit of meters per second are inferred. This gives me confidence that I at least have a sensible answer. Once again, those conversion factors can be factored away:

```fsharp
let kmToMeters = 1000.0<m/km>
let hourToSeconds = 3600.0<s/h>
let kmPerHourToMetersPerSecond = kmToMeters/hourToSeconds

let v = 100.0<km/h>*kmPerHourToMetersPerSecond

val kmToMeters : float<m/km> = 1000.0
val hourToSeconds : float<s/h> = 3600.0
val kmPerHourToMetersPerSecond : float<h m/(km s)> = 0.2777777778
val v : float<m/s> = 27.77777778

```

Of course, this probably seems contrived in this simple throw-away calculation problem, but you can see how a library of units and conversion factors might be built up.

How about wheel revolutions per second?

```fsharp
let wheelRps = v / wheelCirc

val wheelRps : float</s> = 13.64286574
```

Hrrrm. The m unit is gone as expected, leaving us with the odd `</s>` unit. This might be a clue that we are missing another measure

```fsharp
[<Measure>] type rev

let wheelRps = 1.0<rev> * v / wheelCirc

val wheelRps : float<rev/s> = 13.64286574
```

This is the expected unit. The rest of the calculations follow:

```fsharp
let tailShaftRps = wheelRps * 3.909

let gearBoxRatio = 1.0

let engineRps = tailShaftRps * gearBoxRatio

let engineRpm = engineRps * 60.0

val tailShaftRps : float<rev/s> = 53.32996216
val gearBoxRatio : float = 1.0
val engineRps : float<rev/s> = 53.32996216
val engineRpm : float<rev/s> = 3199.797729

```

All looks good... except for engineRpm. We still need one more conversion factor

```fsharp
[<Measure>] type min

let engineRpm = engineRps * 60.0<s/min>

val engineRpm : float<rev/min> = 3199.797729
```

## Final thoughts

Many years ago when I was studying to be a mechanical engineer, I partook in a class called _Computational Fluid Dynamics_. In these classes, we wrote programs that used the _Finite Element Method_ to numerically solve the Navier-Stokes equations (simplified as a 2 dimensional fluid flow problem). At said university, MATLAB was the language of choice. I chose C (_just_ plain C) to implement my program for two reasons:
1. I disliked MATLAB; it was an interpreted language, that has been implemented in an interpreted language (Java).
2. MATLAB would not run on OS/2

In hindsight, I see that F# would have been a million times better choice. In fact, given the above Units of Measure and F#'s support for Matrices, means F# is simply a better choice compared to MATLAB; although that's my opinion only. MATLAB has the advantage of there being a library for any problem you can think of -- [the price however is dealing with a badly designed language ;)](https://www.rath.org/matlab-is-a-terrible-programming-language.html)

That said, I went through this period where I disliked interpreted languages for some reason, therefore I would probably have not considered F# an option at the time. Silly me.

## End of Article

