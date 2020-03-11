---
layout: post
title: "Rolling Your Own Circle Rendering"
date: 2020-03-05
---

Alternative title: Going Around In Circles with SDL2

>(Notice: this article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

So, I've got flashing text, and I've got a system for updating game entity stuff... Now I just need <s>explosions</s> to render explosions.

Wait a second...

# Where are the Circles?

As it turns out, you're on your own when it comes to SDL2. SDL2 only provides support for basic rendering operations such as pixels, and lines, but for some reason does not provide circle drawing.

It is unclear why there is no SDL_RenderDrawCircle when there is SDL_RenderDrawLine, and asking google "sdl2 why no render draw circle function" yields no developers asking that question. It may be that there are so many different circle drawing algorithms that perhaps performance conscious folks will roll their own cicle renderer -- who knows.

So I borrowed some code from Stackoverflow.

# Stackoverflow Driven Development

I must admit that in my previous job as a software developer, a lot of folks thought the work I did required some level of genious intellect, when in actual fact it just required one to know what to type into google search. :P

I did read library documentation (especially when it came to Delphi) and experimented in F# Interactive, but I am guilty of just looking at stackoverflow when I need a quick answer to a specific problem.

The algorithm is known as Midpoint Algorithm, and the code can be found here: [https://stackoverflow.com/a/38335842](https://stackoverflow.com/a/38335842)

Just in case Stackoverflow ever goes away (and simultaneously ruins the careers of many developers :P ), the code is included here

```c
void DrawCircle(SDL_Renderer * renderer, int32_t centreX, int32_t centreY, int32_t radius)
{
   const int32_t diameter = (radius * 2);

   int32_t x = (radius - 1);
   int32_t y = 0;
   int32_t tx = 1;
   int32_t ty = 1;
   int32_t error = (tx - diameter);

   while (x >= y)
   {
      //  Each of the following renders an octant of the circle
      SDL_RenderDrawPoint(renderer, centreX + x, centreY - y);
      SDL_RenderDrawPoint(renderer, centreX + x, centreY + y);
      SDL_RenderDrawPoint(renderer, centreX - x, centreY - y);
      SDL_RenderDrawPoint(renderer, centreX - x, centreY + y);
      SDL_RenderDrawPoint(renderer, centreX + y, centreY - x);
      SDL_RenderDrawPoint(renderer, centreX + y, centreY + x);
      SDL_RenderDrawPoint(renderer, centreX - y, centreY - x);
      SDL_RenderDrawPoint(renderer, centreX - y, centreY + x);

      if (error <= 0)
      {
         ++y;
         error += ty;
         ty += 2;
      }

      if (error > 0)
      {
         --x;
         tx += 2;
         error += (tx - diameter);
      }
   }
}
```

I naively translated directly into F# and called it a day

```fsharp
let renderDrawCircle(_x, _y, radius) renderer =
   let mutable x = radius - 1;
   let mutable y = 0;
   let mutable tx = 1;
   let mutable ty = 1;
   let mutable err = tx - (radius * 2); // shifting bits left by 1 effectively      
                                   // doubles the value. == tx - diameter   
   
   while (x >= y) do
   
      //  Each of the following renders an octant of the circle
      SDL.SDL_RenderDrawPoint(renderer, _x + x, _y - y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x + x, _y + y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - x, _y - y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - x, _y + y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x + y, _y - x) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x + y, _y + x) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - y, _y - x) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - y, _y + x) |> ignore

      if (err <= 0) then
         y <- y + 1
         err <- err + ty
         ty <- ty + 2     
      elif (err > 0) then      
         x <- x - 1 
         tx <- tx + 2
         err <- err + tx - (radius * 2)
         
         
```

I bodged up some test explosion rendering code in the main game loop

```fsharp
  let keys = sdlGetKeys()

  for key in keys do
      if key = SDL.SDL_Scancode.SDL_SCANCODE_RETURN then                    
          for i in [1..5] do
              let ex,ey = r.Next(0, 20), r.Next(0, 20)
              timers.Add(animationTimer(5, 100, (fun f ->
                          let explosion = [15;15;14;14;12;12;12;4;4;4;4]                    
                          explosion
                          |> List.iteri (fun i c -> 
                              renderer 
                              |> sdlSetDrawColor (drawColor c) 
                              |> renderDrawCircle((int ex)+400,(int ey)+400, f+i+10)
                              ()
                              )
                          )))
```

In case you are wondering, `animationTimer` is defined as so:

```fsharp
let animationTimer (frames:int, interval:int, callback) =
    let mutable elapsed = 0
    let mutable counter = 0

    let update gs =
        elapsed <- elapsed + gs.elapsed               

        if (elapsed > interval) then            
            elapsed <- 0
            counter <- counter + 1
        
        if counter < frames || frames = -1 then
            callback(counter)
            Continue
        else
            Terminate

    update
```

I have actually found timers to be a fairly versatile concept, but to talk about them here would require their own article, so I'll cover that later. For now, we're talking about explosions... rendering.

Anyway, one shiny new explosion generator, let's give it a run

![Animated GIF: Lots of explosions, so little framerate](/assets/bgc/explosions1.gif)

*Above: Lots of explosions, so little framerate*

To my dismay, the framerate nearly halves with less than 50 explosions per second. In the animation, about 495 circles are being rendered per second, and it's bringing a reasonably modern quad core laptop to it's knees. It was time to go back to the drawing board.

At this point you might reasonably assume I did the sensible thing and use the SDL_gfx library. But as it turns out, <s>I am an unreasonable person</s> I really just needed circles, and knowing from experience how it took about 5 minutes to compile the SDL2 CS project, I decided I would rather spend several hours optimising this algorithm than spend 5 minutes compiling SDL_gfx :) after all, rolling your own is more fun than using something off the shelf. *cough*


Ok ok, it's worth mentioning that there is no .NET wrapper library for SDL2_gfx, but I'm sure it would have taken half an hour to create one using interop. Anyway, onto the problem solving.

# Optimisation

Before we start making chages, we need a means of measuring the speed of the function so that a baseline (un-optimised speed) can be established. This baseline is then used to measure the improvement (if any) on making a change.

Now you might be tempted to do something like

```fsharp
let renderDrawCircle(_x, _y, radius) renderer =

  //insert some debug output
  let sw = System.Diagnostics.Stopwatch()
  sw.Start()
  
  
  let mutable x = radius - 1;
  let mutable y = 0;
  let mutable tx = 1;
  let mutable ty = 1;
  let mutable err = tx - (radius * 2); // shifting bits left by 1 effectively      
                                 // doubles the value. == tx - diameter   
   
  while (x >= y) do
   
      //  Each of the following renders an octant of the circle
      SDL.SDL_RenderDrawPoint(renderer, _x + x, _y - y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x + x, _y + y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - x, _y - y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - x, _y + y) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x + y, _y - x) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x + y, _y + x) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - y, _y - x) |> ignore
      SDL.SDL_RenderDrawPoint(renderer, _x - y, _y + x) |> ignore

      if (err <= 0) then
         y <- y + 1
         err <- err + ty
         ty <- ty + 2     
      elif (err > 0) then      
         x <- x - 1 
         tx <- tx + 2
         err <- err + tx - (radius * 2)
         
  printfn "%d" sw.ElapsedMilliseconds
```
*cough*

But one needs to run the same function multiple times with a range of parameters, so having to check and sum multiple values printed to the screen will get tedius. Instead, why not a general purpose high order function that executes a unit returning function and measures how long it took? Let's call it, I dunno, `measureTime`

```fsharp
let measureTime func = 
    let watch = System.Diagnostics.Stopwatch()
    watch.Start()
    func()
    watch.ElapsedMilliseconds
```

Now we need to isolate the explosion drawing code

```fsharp
let drawExplosion (ex,ey) frame =
  let explosion = [15;15;14;14;12;12;12;4;4;4;4]                    
  explosion
  |> List.iteri (fun i c ->               
        renderer 
        |> sdlSetDrawColor (drawColor c) 
        |> renderDrawCircle((int ex)+250,(int ey)+400, frame+i+10)
      )
```

Great! And now we can call the function and measure how long it took to execute

```fsharp
> measureTime (fun () -> drawExplosion(100,200) 10);;
val it : int64 = 5L

```

5 milliseconds. But re-run it again, and you'll see it's not the true speed

```fsharp
> measureTime (fun () -> drawExplosion(100,200) 10);;
val it : int64 = 1L

> measureTime (fun () -> drawExplosion(100,200) 10);;
val it : int64 = 0L

> measureTime (fun () -> drawExplosion(100,200) 10);;
val it : int64 = 0L

> measureTime (fun () -> drawExplosion(100,200) 10);;
val it : int64 = 0L
```

More than likely that 5 milliseconds was required for Just in Time compilation, initialisation etc. It's important to run something many times when profiling. Not to mention that this function is quick enough to return less than 1 millisecond.

So here you go, something to execute the function many times and measure the total duration

```fsharp
  measureTime (fun () ->
      [1..100] 
      |> List.iter(fun frame -> drawExplosion(100,200) frame))       
  |> printfn "%d"
```

This is drawing 1400 circles gradually increasing in size up to 100 pixels in radius. The times are:

```fsharp
403
402
380 
396
```

OK we're ready to start optimising the function. An obvious starting point was to reduce the number of function calls to `SDL_RenderDrawPoint`, especially as there is an equivilent `SDL_RenderDrawPoints` which, as the name suggests, takes a list of points.

```fsharp
let renderDrawCircle1(_x, _y, radius) renderer = 
   let mutable x = radius - 1;
   let mutable y = 0;
   let mutable tx = 1;
   let mutable ty = 1;
   let mutable err = tx - (radius * 2); // shifting bits left by 1 effectively
      
                                   // doubles the value. == tx - diameter
   
   
   while (x >= y) do
      
      let points =
          [|
              SDL.SDL_Point(x = _x + x, y = _y - y)
              SDL.SDL_Point(x = _x + x, y = _y + y)
              SDL.SDL_Point(x = _x - x, y = _y - y)
              SDL.SDL_Point(x = _x - x, y = _y + y)
              SDL.SDL_Point(x = _x + y, y = _y - x)
              SDL.SDL_Point(x = _x + y, y = _y + x)
              SDL.SDL_Point(x = _x - y, y = _y - x)
              SDL.SDL_Point(x = _x - y, y = _y + x)
          |] //|> Array.map (fun (x,y)-> SDL.SDL_Point(x = x, y = y))
          
      SDL.SDL_RenderDrawPoints(renderer, points, 8) |> ignore  
         
      if (err <= 0) then
         y <- y + 1
         err <- err + ty
         ty <- ty + 2     
      elif (err > 0) then      
         x <- x - 1 
         tx <- tx + 2
         err <- err + tx - (radius * 2)
```

The renderDrawCircle1 function takes a quarter of the time, by virtue of calling the `SDL_RenderDrawPoint` function 7 times less often. Below are the times:

```fsharp
99
96
93
93
```

The next optimisation that occurred to me is that there was no reason all the points of the circle couldn't be computed all at once

```fsharp
let renderDrawCircle2(_x, _y, radius) renderer =
   let mutable x = radius - 1;
   let mutable y = 0;
   let mutable tx = 1;
   let mutable ty = 1;
   let mutable err = tx - (radius * 2); // shifting bits left by 1 effectively
      
                                   // doubles the value. == tx - diameter
   
   let points = [|
       while (x >= y) do
      
          let points =
              [|
                  SDL.SDL_Point(x = _x + x, y = _y - y)
                  SDL.SDL_Point(x = _x + x, y = _y + y)
                  SDL.SDL_Point(x = _x - x, y = _y - y)
                  SDL.SDL_Point(x = _x - x, y = _y + y)
                  SDL.SDL_Point(x = _x + y, y = _y - x)
                  SDL.SDL_Point(x = _x + y, y = _y + x)
                  SDL.SDL_Point(x = _x - y, y = _y - x)
                  SDL.SDL_Point(x = _x - y, y = _y + x)
              |]
                 
          yield! points 

          if (err <= 0) then
             y <- y + 1
             err <- err + ty
             ty <- ty + 2     
          elif (err > 0) then      
             x <- x - 1 
             tx <- tx + 2
             err <- err + tx - (radius * 2)
        |] 

   SDL.SDL_RenderDrawPoints(renderer, points, points.Length) |> ignore  

```

This function makes use of the `seq` capability to generate an as required number of points. Surprisingly, it's not much quicker:

```fsharp
74
92
76
69
71
```

For the 200 pixel wide circles there are 120 calls to the `SDL_RenderDrawPoints`, so you would think 120 calls less would make a difference. I suspect it's probably the use of `seq`; faster times might be realised switching to unfold or the like, but at this point I ... hang on, wait a minute

# Profiling needs to be done right

I got through most of this article before I realised I had potentially been profiling the wrong part of the code.

Originally I had this code for measuring the total time taken to run a particular version of the explosion renderer

```fsharp
        [1..100] 
        |> List.map(fun frame -> measureTime (fun () -> drawExplosion(100,200) frame))       
        |> List.sum
        |> printfn "%d"
```

What it was doing was measuring each individual time taken to draw an explosion frame, and then summing the results. In my boneheadedness, I had surmised that since a call to drawExplosion was regularly executing in less than a millisecond, that I would often be summing zeros, making it seem like the time taken to iterate 100 times was less than it actually was.

So I switched to the following code:

```fsharp
  measureTime (fun () ->
      [1..100] 
      |> List.iter(fun frame -> drawExplosion(100,200) frame))       
  |> printfn "%d"
```

It seemed logical at the time: measure the total time taken to draw 100 explosion frames. But keep in mind you are now also measuring the Interation code. As it turns out, it adds about 2400 ticks to the total (abour 0.24 milliseconds)

```fsharp
  measureTime (fun () ->
    [1..100] 
    |> List.iter(fun _ -> ()))       
  |> printfn "%d"
```

```fsharp
2228
2654
2245
2604
```

In this case, 0.24 milliseconds is not much, but it's something to keep in mind. In this case, the answer is not to change the way you measure, but to increase the resolution of the timer. In this case, ElapsedTicks provides a higher resolution.

```fsharp
        let measureTime func = 
            let watch = System.Diagnostics.Stopwatch()
            watch.Start()
            func()
            watch.ElapsedTicks
```


It's also important to average out the values to remove any runtime quirks. Now that we are measuring ElapsedTicks, consider the following which is executed 8 times



```fsharp
measureTime (fun () -> renderDrawCircle2(100,200,200) renderer);;

results:
5619
1812
1669
15940 <<< WHAT THE HECK 10 TIMES SLOWER!?
2403
1700
1890
1693
```

For seemingly no reason, the 4th time took an order of magnitude longer than the rest. But if you average a thousand measurements you will effectively filter out those peaks (unless there are lots of them!).

And as a final point, you need to iterate enough times such that re-running the test has little variance compared to the previous. To ensure this horse is properly dead, consider the following is executed 5 times:



```fsharp
[1..10]
|> List.map(fun frame -> measureTime (fun () -> renderDrawCircle(100,200,frame+10) renderer))       
|> List.sum
|> printfn "%d"

results:
5514
4861
8408
14179
8743
```

But this almost always gives the same answer, because it's running the circle drawing code ten thousand times!

```fsharp
[1..100] |> List.map (fun i -> [10..110] |> List.map id) |> List.concat 
|> List.map(fun frame -> measureTime (fun () -> renderDrawCircle1(100,200,frame) renderer))       
|> List.sum
|> printfn "%d"
```

Even then, you still need to run a few times to make sure the total time stabilises

# Get to the point

Hey, profiling is important to me. It's not enough to simply think something is faster, actual metrics are required.

Anyway, back to the original article, which was explosions (rendering). I define a list of each version of renderDrawCircle to be called in succession, hopefully ensuring that any .NET weirdness in measuring time at least affects all three versions equally.

```fsharp
let funcs = [
            renderDrawCircle
            renderDrawCircle1
            renderDrawCircle2
            ]

funcs |> List.iter (fun func ->
[1..100] |> List.map (fun i -> [10..110]) |> List.concat 
|> List.map(fun frame -> measureTime (fun () -> func(100,200,frame) renderer))       
|> List.sum
|> printfn "%d")
```

This is run a few times, including after waiting a long time between run 4 and run 5, in order to see stable results.

|version|run 1|run 2|run 3|run 4|run 5*|run 6|average|% faster than v1|
|---|---|---|---|---|---|---|---|---|
|1|31758916|31235178|31142280|31205663|34267728|33657498|32211210.5|N/A|
|2|7576802|6936058|7112976|7061530|7367735|7318996|7229016.167|77.5|
|3|6398260|6230180|6212428|6025629|6930623|6114915|6318672.5|80.4|

We're iterating enough times that each version consistently performs faster than the last, so we have eliminated (for the most part) those runtime quirks.

Interestingly, there's not a lot of performance difference between v2 and v3. It may be 10% faster than v2, but this only translates to about 3% faster overall. Perhaps with a different iteration strategy it may be able to get faster.

# The Bottom Line

Battle Ground Copyright is now one step closer to ripping off Liero!

![Animated GIF: now with faster explosions!](/assets/bgc/explosions2.gif)

*Above: now with faster explosions!*

The new explosion rendering code now renders explosions without an appreciable drop in framerate. In fact, I am sure that if I did better framerate limiting than simply Thread.Sleep(40) it might even not drop the framerate.
