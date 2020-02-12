---
layout: post
title: "Game Entity Management"
date: 2020-02-12
---

>(Notice: this article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

In the previous article, I mentioned how solving the issue of flashing text led to a more generalised solution of game entity management. In this article, I discuss a simple solution to the problem of game loop clutter by introducing a *Poor Man's Game Engine*.

So the game loop was starting to look like this

```fsharp

  let run() =

    //**NOTE ... bunch of setup stuff omitted for clarity
    
    let mutable angle = 0.0

    
    //**NOTE: I hope you can infer the structure of FlashText from the code below
    let flashList = ResizeArray<FlashText>()

    flashList.Add({ points = (1, 1); color = 14; text = "Game in creation" })
    flashList.Add({ points = (2, 26); color = 14; text = "v2.0!" })

    let mutable ticks = 0L
    let mutable now = 0L
    let mutable flashTicks = 0L    
    let mutable inFlash = false

    while true do
        renderer |> clearScreen        

        //needed even if not using keys, otherwise game eventually halts
        SDL.SDL_PumpEvents()        

        print ((2,1), 7, "Battle Ground Copyright!")

        
        //**NOTE: for some reason I wanted the resolution of ticks. Yeah I know...
        flashTicks <- flashTicks + 400000L
            
        let flashDuration = if inFlash then
                                2000000L
                            else
                                2000000L
        if (flashTicks > flashDuration) then
            flashTicks <- 0L
            inFlash <- not inFlash

        for flash in flashList do
            print (flash.points, (if inFlash then flash.color else 0), flash.text)
        
        
        //**NOTE: this bit of code here is rendering the health box and rotating it 10 degrees every frame        
        angle <- angle + 10.0;

        if angle > 360.0 then angle <- 0.0

        renderer |> setDrawColor (255uy,0uy,0uy)    
    
        lines         
        |> transformLines (rotatePoint (angle) >> (translatePoint (200.0, 200.0)))
        |> List.iter (fun (v1,v2) ->             
             renderer 
                |> setDrawColor (v2.color |> drawColor)
                |> renderDrawLine (v1,v2)
                |> ignore
             )        
        
        renderer |> renderPresent

        counter <- counter + 1

        //**NOTE: bad bad way of ensuring 25 frames per second. 
        //If this sort of thing makes you cringe, consider using a game engine instead :P
        System.Threading.Thread.Sleep(40)
        
        now <- DateTime.Now.Ticks
        
        //System.Threading.Thread.Sleep(1)

    //let r = SDL.SDL_RenderDrawLine(v, 10, 10, 50, 50)

    ()

```

As you can probably see, it's starting to get a bit cluttered in here.

And maybe that's OK! Your game could be very simple. Or, perhaps if you don't value abstraction much like I did, you can put the entire game logic into the game loop. You just may have trouble finding things later on :)

# Update Lists

In the past couple of attempts at writing games in C#, I would often do something like this:

```csharp

public interface IEntity
{
  public void Update();
}

```

Any class implementing this inteface could then be stored generically in a List<IEntity>, and iterated over each game loop.

In this case it's the same, except I will just have a list of functions to be called:

```fsharp
type UpdateFunc = int -> unit

let updateList = ResizeArray<UpdateFunc>()
```

The type declation is not really required. You can of course just write

```fsharp
let updateList = ResizeArray<int -> unit>()
```

However, as I found out later this was a good move as it allowed me to change the function signature in just one spot, although you still have to go update all code that references the UpdateFunc type.

The other thing I considered was whether UpdateFunc should actually be an `interface` i.e. `IEntity` like before. Maybe there are a set of related things that all entities should be capable of, like setting and getting position etc.

I decided against that; I wanted to see how far I could get with just functions. Ultimately, that turned out to be pretty much all the way, as you will see.

So the next step was to make a function whose sole job was to update all the text registered for flashing

```fsharp

let flasher (durationOn, durationOff, callback) =
    let mutable ticks = 0L
    let mutable now = 0L
    let mutable flashTicks = 0    
    let mutable inFlash = false

    let update ms =        
        flashTicks <- flashTicks + ms
            
        let flashDuration = if inFlash then
                                durationOn
                            else
                                durationOff
        if (flashTicks > flashDuration) then
            flashTicks <- 0
            inFlash <- not inFlash
        
        callback(inFlash)

    update

```

Well actually, a function that builds the function :) this is making use of closures to emulate objects.

... OK, if you are still here, you have not left in disgust. Perhaps you are here to witness the train wreck?

So anyway, this function can be used for any duty requiring a constant toggle, not just flashing text as you will see, because it takes a callback. Maybe I should actually call it a toggle.

```fsharp
    let flashList = ResizeArray<FlashText>()
    flashList.Add({ points = (1, 1); color = 14; text = "Game in creation" })
    flashList.Add({ points = (2, 26); color = 14; text = "v2.0!" })

    let callback inFlash =
        if inFlash then
            for flash in flashList do
                print (flash.points, flash.color, flash.text)            


    updateList.Add(flasher(200, 200, callback))
       
```

In hindsight, FlashList could also be just a collection of `print` functions to be called.

And now in the main game loop:

```fsharp
    while true do                       

      renderer |> clearScreen

      SDL.SDL_PumpEvents()                            

      let elapsedMS = stopWatch.Elapsed.Milliseconds
      stopWatch.Restart()

      
      //**NOTE: one bit of code eliminated
      updateList |> Seq.iter (fun f -> f(elapsedMS))

      print ((2,1), 7, "Battle Ground Copyright!")
      
      angle <- angle + 10.0;

      if angle > 360.0 then angle <- 0.0

      renderer |> setDrawColor (255uy,0uy,0uy)    
  
      lines         
      |> transformLines (rotatePoint (angle) >> (translatePoint (200.0, 200.0)))
      |> List.iter (fun (v1,v2) ->             
           renderer 
              |> setDrawColor (v2.color |> drawColor)
              |> renderDrawLine (v1,v2)
              |> ignore
           )        
      
      renderer |> renderPresent

      counter <- counter + 1

      System.Threading.Thread.Sleep(40)                            

    ()

```

You can also see how I switched to milliseconds, because Ticks ultimately just required more zeroes :)

And thereafter, pretty much every aspect of the game could be solved the same way, however I found I needed to introduce a `GameState` to allow more information to be passed to the update functions.

```fsharp
type GameState = {
    elapsed: int
    keys: SDL.SDL_Scancode[]
}
```

So far, to accomplish the following:
- player tank control
- ammo refills
- the "surprise bonus crate"
- nuclear missiles
- bullets
- particle effects

All I have required are just the elapsed milliseconds and the current key state being passed to a function with signature `type GameState -> unit`

To avoid this article getting too long, I can go into the grittier details in another article, but here for example is the particle generator:

```fsharp
type Particle = {
    mutable pos: float*float
    mutable vector: float*float    
    mutable life: int
    mutable color: int
}

let particleList = ResizeArray<Particle>()

let particleListUpdater renderer =

    let update gs =
        for particle in particleList.ToArray() do
            if particle.life > 0 then
                particle.life <- particle.life - gs.elapsed
                let dx,dy = translatePoint particle.vector particle.pos 
                renderer |> sdlSetDrawColor (drawColor (particle.color)) |> ignore
                SDL.SDL_RenderDrawPoint(renderer, int dx, int dy) |> ignore            
                particle.pos <- (dx,dy)
            else
                particleList.Remove(particle) |> ignore
    update

```

As `particleList` is global, adding a particle is a matter of adding a particle to the `particalList` array. A smoke generating function makes use of the particle system like so:


```fsharp
let smokeGenerator =
    
    let r = Random()

    let update currentState =
        let (px,py),angle = currentState        
        for i in 1..10 do            
            particleList.Add(
                { 
                    color = (if i % 2 = 0 then 7 else 15); 
                    pos = (px,py); 
                    vector = getVector(float(r.Next(1,5)), angle + float(r.Next(-15, 15))); 
                    life = 400
                })                

        ()
    
    update
```

One might argue what is the difference between a bullet and a particle... To which I say, watch out for the article where some massive refactoring occurs :)