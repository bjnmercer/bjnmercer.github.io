---
layout: post
title: "Reverse Engineering the QBASIC DRAW Command"
date: 2017-12-09
---

>(This article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

I have been asked a number of times now:

>"Why don't you just use sprites?"

It _probably_ would have been easier to use sprites, but here are my (_probably poor_) reasons for re-implementing the QBASIC DRAW command:
1. I'm lazy (specifically, I would have had to capture images for over 40 different drawings)
2. Vector graphics scale regardless of resolution. (Why use sprites to implement a Vector Graphics game?)
3. I was interested in the technical challenge of re-implementing the DRAW command.

Hence this article. <s>So that's the plan, let's get started</s> With that out of the way, we can begin.

## What is the QBASIC DRAW function?

QBASIC had several functions for doing simple primitives such as lines, circles and pixels.

For more complex shapes, it had the DRAW function, which was essentially a cut down version of [LOGO](https://en.wikipedia.org/wiki/Logo_(programming_language)).

![The DRAW Command help screen](/assets/bgc/draw-help.PNG)

Each command represents a direction to move, and the amount of pixels to move. If it is a "Blind move", then no line is drawn. I think it easier to show you rather than explain. Here is the DRAW commands that make up the eponymous BGC tank:

```basic
tank$ = "BL5 BU5 BR1 R8 F D2 G D2 F D2 G L8 H U2 E U2 H U2 E"
```

And here is the graphical output of those commands.

![The Tank](/assets/bgc/bgctank.PNG)

## DrawCommand

First up, I decided to use a Discriminated Union to represent all the DRAW commands. You can read more about Discriminated Unions at the [F# for Fun and Profit site](https://fsharpforfunandprofit.com/posts/discriminated-unions/).

```fsharp
type DrawCommand =
| B of DrawCommand //"B" (Blind) before a line move designates that the line move will be hidden. Use to offset from a "P" or PAINT border.
| N of DrawCommand //"N" before a line move designates that the drawn line will return to the line start position. Saves moves!
| C of int  //"C n" designates the color attribute or _RGB string numerical color value to be used in the draw statement immediately after.
| M of int * int //M[{+|-}]x%,y%    Moves cursor to point x%,y%. If x% is preceded, by + or -, moves relative to the current point.
| P of int * int //Sets the paint fill and border colors of an object (n1% is the fill-color attribute, n2% is the border-color attribute).
| D of int //"D n" draws a line vertically DOWN n pixels.
| E of int //"E n" draws a diagonal / line going UP and RIGHT n pixels each direction.
| F of int //"F n" draws a diagonal \ line going DOWN and RIGHT n pixels each direction.
| G of int //"G n" draws a diagonal / LINE going DOWN and LEFT n pixels each direction.
| H of int //"H n" draws a diagonal \ LINE going UP and LEFT n pixels each direction.
| L of int //"L n" draws a line horizontally LEFT n pixels.
| R of int //"R n" draws a line horizontally RIGHT n pixels.
| U of int //"U n" draws a line vertically UP n pixels.
| A of int //"A n" can use values of 1 to 3 to rotate up to 3 90 degree(270) angles.
| TA of int //TA n" can use any n angle from -360 to 0 to 360 to rotate a DRAW (Turn Angle). "TA0" resets to normal.
```

Now I could have given the cases more descriptive names (i.e. B -> Blind, G -> DiagDownLeft), but I kept them the same as the DRAW commands for brevity and the ability to easy translate from QBASIC DRAW string to a DU list.

For example, here is the tank from the original QBASIC game:
```basic
tank$ = "BL5 BU5 BR1 R8 F D2 G D2 F D2 G L8 H U2 E U2 H U2 E"
```

This draws the eponymous tank:
![The Tank](/assets/bgc/bgctank.PNG)

And here is the DrawCommand version in F#
```fsharp
//tank$ = " BL5     BU5     BR1     R8   F    D2   G    D2   F    D2   G    L8   H    U2   E    U2   H    U2   E"
let tank = [B(L 5); B(U 5); B(R 1); R 8; F 1; D 2; G 1; D 2; F 1; D 2; G 1; L 8; H 1; U 2; E 1; U 2; H 1; U 2; E 1]
```

Thanks to the succint syntax of F#, expressing a list of DrawCommands is almost identical to the QBASIC DRAW version.


## Building Coordinates

Next up was a function to take the above list of Draw Commands, and map these into a list of pixel co-ordinates. Not all the DrawCommand cases are not handled.

```fsharp
let buildCoords (commands:DrawCommand list) =
    
    let rec nextCoord (b:bool, (px:int, py:int)) (command:DrawCommand) =
        let notImplemented () = failwith "not implemented"
        (match command with
        | B(_) -> false
        | _ -> true),
        match command with
        | D (dx) -> (px, py + dx)
        | E (dx) -> (px + dx, py - dx)
        | F (dx) -> (px + dx, py + dx)
        | G (dx) -> (px - dx, py + dx)
        | H (dx) -> (px - dx, py - dx)
        | L (dx) -> (px - dx, py)
        | R (dx) -> (px + dx, py)
        | U (dx) -> (px, py - dx)        
        | B (c) -> snd (nextCoord (false, (px, py)) c)
        
        //unsupported yet        
        | N (_) 
        | C (_) 
        | M (_, _) 
        | P (_, _) 
        | A (_)
        | TA (_) -> notImplemented()
        
    commands
    |> List.scan nextCoord (false, (0,0)) 
    |> List.where(fun (b,_) -> b) 
    |> List.map snd 
    |> List.map (fun (x,y) -> float(x), float (-1*y)) //invert        
```

So what is going on?

We'll start with the nested function that calculates co-ordinates, `nextCoord`

```fsharp
    let rec nextCoord (b:bool, (px:int, py:int)) (command:DrawCommand) =
        let notImplemented () = failwith "not implemented"
        (match command with
        | B(_) -> false
        | _ -> true),
        match command with
        | D (dx) -> (px, py + dx)
        | E (dx) -> (px + dx, py - dx)
        | F (dx) -> (px + dx, py + dx)
        | G (dx) -> (px - dx, py + dx)
        | H (dx) -> (px - dx, py - dx)
        | L (dx) -> (px - dx, py)
        | R (dx) -> (px + dx, py)
        | U (dx) -> (px, py - dx)        
        | B (c) -> snd (nextCoord (false, (px, py)) c)
        
        //unsupported yet        
        | N (_) 
        | C (_) 
        | M (_, _) 
        | P (_, _) 
        | A (_)
        | TA (_) -> notImplemented()

```

This function takes a (bool * (int * int)) tuple and a DrawCommand, and returns a (bool * (int * int)) tuple. This can be checked using FSI:

```
val nextCoord :
  b:bool * (int * int) -> command:DrawCommand -> bool * (int * int)
```

The basic idea behind `nextCoord` is to take a previous tuple of pixel coordinates, and calculate the next coordinates relative to the previous, depending on the DU case. This is accomplished using a match expression. At the same time, another match expression is used to determine if it is a Blind move. You can read more about [Match expressions at the excellent F# for Fun and Profit site](https://fsharpforfunandprofit.com/posts/match-expression/).

Now I could have made this functional external (i.e. not nested), but FSI allows you to work directly with it by selecting code block and evaluating it in Interactive. It's worth mentioning that this function is specifically designed to work with `List.scan` function, which I will get to later on in this article.

Here is an example of this function in action. If the previous co-ordinates was 10x,10y and we want to move by E5 -- Diagonally Up and Right -- then we should see 10x+5,10y-5

```
> nextCoord (false, (10,10)) (E 5);;
val it : bool * (int * int) = (true, (15, 5))
```

Note that the first value in the tuple is `true`, this is because it is not a blind move.

```
> nextCoord (false, (10,10)) (B(E 5));;
val it : bool * (int * int) = (false, (15, 5))
```

The Blind move is accomplished by storing a `DrawCommand` inside the B DU case. The nextCoord function matches on the B case and extracts the DrawCommand, and recursively calls itself.

This demonstrates a nice feature of Discriminated Unions: the ability to refer to itself. This makes representing tree structures trivial. However, I discovered later that this is not the best way to handle this scenario. I will talk about that in the next article.

Now at this point you may be wondering how a function that only relies on the last input co-ordinates to compute the next set of co-ordinates can produce a list of coordinates. As said earlier, this function is made to be compatable with the `List.scan` function. We'll explore this now.

I'll once again refer you to [F# For Fun and Profit for the indepth treatment of this function](https://fsharpforfunandprofit.com/posts/list-module-functions/#19), but hopefully this example will serve to give an idea of what it does.

List.scan takes a list of values as input, a function for iterating over those values, as well as an _initial starting state_. Your iterator function is fed the last calculation from your function as input (or if it is the first value, the _initial starting state_), as well as the next value in the input list.

Hopefully you will now recognise that nextCoord has that exact function signature:

```fsharp
bool * (int * int) -> command:DrawCommand -> bool * (int * int)
```

This is the output from List.scan when it is fed the tank DrawCommand list. Note how I need to provide (false, (0,0)) as the initial starting state:

```
> tank |> List.scan nextCoord (false, (0,0));;
val it : (bool * (int * int)) list =
  [(false, (0, 0)); (false, (-5, 0)); (false, (-5, -5)); (false, (-4, -5));
   (true, (4, -5)); (true, (5, -4)); (true, (5, -2)); (true, (4, -1));
   (true, (4, 1)); (true, (5, 2)); (true, (5, 4)); (true, (4, 5));
   (true, (-4, 5)); (true, (-5, 4)); (true, (-5, 2)); (true, (-4, 1));
   (true, (-4, -1)); (true, (-5, -2)); (true, (-5, -4)); (true, (-4, -5))]
```

Also note how the initial starting state is the very first item in the list returned. List.scan has taken care of the iteration, and maintaining the state for us. As such it was a good candidate for producing the coordinates as it works very similar to the QBASIC DRAW command, where each command moves relative to the last command.

OK, so now we have a tuple of values, lets see if the output was correct. To do this, I will use the FSharp.Charting library. You can get the binaries at [fslab.org](https://fslab.org/FSharp.Charting/). The documentation on getting started suggests that you can simply #load the FSharp.Charting.fsx script, but it had some issues so I just referenced the DLL directly in my FSX file.

```fsharp
#r "System.Windows.Forms.DataVisualization.dll"
#nowarn "211"
#r @"<insert-path-to->FSharp.Charting.dll"

open FSharp.Charting
module FsiAutoShow = 
    fsi.AddPrinter(fun (ch:FSharp.Charting.ChartTypes.GenericChart) -> ch.ShowChart(); "(Chart)")
```

The Chart.Line function expects a list of two-element tuples -- in this case (int * int). Right now, List.scan is returning a (bool * (int * int)) list, so I feed the output to `List.map snd`. The `snd` function extracts the second element of the tuple:

```
> tank |> List.scan nextCoord (false, (0,0)) |> List.map snd;;
val it : (int * int) list =
  [(0, 0); (-5, 0); (-5, -5); (-4, -5); (4, -5); (5, -4); (5, -2); (4, -1);
   (4, 1); (5, 2); (5, 4); (4, 5); (-4, 5); (-5, 4); (-5, 2); (-4, 1);
   (-4, -1); (-5, -2); (-5, -4); (-4, -5)]
```

Which is what we want. Finally, the output is fed into Chart.Line:

```
> tank |> List.scan nextCoord (false, (0,0)) |> List.map snd |> Chart.Line;;
val it : ChartTypes.GenericChart = (Chart)
```

This pops up a window containing a line chart:

![Line chart depicting the eponymous tank](/assets/bgc/draw1.PNG)

However, the output is upside down. This is because the chart has it's Y-axis starting from the bottom rather than the top.

This is essentially how I arrived at the buildCoords function.

### Bonus round: co-ordinate rotation

I could not resist having a go at implementing a function which will rotate a list of coordinates about their center. Sadly, I could not remember the exact math, so I went and dug up some code from the failed PureBasic rewrite version.

One of the things I found amusing was observing my younger self's apparent aversion to defining local variables and functions. Take this code for example:

```
  rotatey.f = ((unit()\weapon_x[I]) * Cos(((-tankangle * PI.f) / 180))) - ((unit()\weapon_y[I]) * Sin(((-tankangle * PI.f) / 180)))
  rotatex.f = ((unit()\weapon_y[I]) * Cos(((-tankangle * PI.f) / 180))) + ((unit()\weapon_x[I]) * Sin(((-tankangle * PI.f) / 180)))     
```

At no point did it occur to me to store the result of the calculation `(-tankangle * PI.f) / 180)` into a local variable. In fact, it never occurred to me to at least store the result of `Cos(((-tankangle * PI.f) / 180))` into a local variable. Nope, I just calculate the same values over and over.


Thankfully, I have learned a little since those days.

```fsharp
let rotatePoint (angle:float) (x:float, y:float) =     
    let rad = (angle*Math.PI)/180.0
    let cos' = cos(rad)
    let sin' = sin(rad)    
    //you know, I can't remember I came up with this, but it works.
    (y * cos') + (x * sin'), (x * cos') - (y * sin')

```

rotatePoint is actually a standalone function; I figured I may require co-ordinate rotation in other areas as well.

```
val rotatePoint : angle:float -> x:float * y:float -> float * float
```

The function rotatePoints is even simpler:

```fsharp
let rotatePoints (angle:float) (points:(float * float) list) =               
    points |> List.map (rotatePoint angle)
```

Let's see this function in action:

```
tank |> buildCoords |> (rotatePoints 85.0) |> Chart.Line
```

![Line chart depicting the eponymous tank rotated](/assets/bgc/draw2.PNG)

## Unit Testing?

You might be thinking "Where are the unit tests"? I guess the function nextCoord would be a prime candidate for unit testing, given its purely functional nature. But I find that I do far less unit testing when I develop with F# Interative, since I can execute discreet blocks of code, and see the output immediately, even visual output thanks to FSharp.Charting. I will admit that the output from the buildCoords function was not correct straight away. However, I knew what the shape of the output should have been, so it was easier to just display it on the chart directly and iterate until I got the correct output. If I ever feel the need to muck with this function again, I'll build a regression test.

## End of Article

That actually was not too bad. In the next article, I discuss how I use the FParsec library to directly parse QBASIC DRAW strings.