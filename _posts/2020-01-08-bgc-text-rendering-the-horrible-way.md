---
layout: post
title: "Text Rendering The Horrible Way"
date: 2020-01-08
---

>(Notice: this article is part of the [BGC Rewrite Series](/blog/2017/10/03/bgc-rewrite-series))

# Slacker notice

Wow, it's been nearly a year since my last post (*what is this, Project Crown!?* - Editor), and a year and a half since the last BGC post

I'd love to say I've done heaps more on BGC but according to the last modified date, it's been 9 months. Also, I have yet to actually commit any code to github. So time to restart the blog and maybe that'll get me motivated again.

# Where we left off

The last article was about the Line Rendering engine. After that, I made some (alledged) rapid progress, so quick that I was not really able to write articles.

I find it amusing that after creating the line rendering engine, I had decided I had reached a critical point and it was time to start migrating the code from QBASIC to F#.

So of course, the very first lines in the game file are this:

```qbasic
CLS
SCREEN 0
COLOR 30
PRINT "Game Completed!"
COLOR 7
PRINT "Battle Ground Copyright!"; : COLOR 20: PRINT " v1.28"
SLEEP 2
```

Text! Not just text, but _flashing text_ (note `COLOR 30` and `COLOR 20` mean bright yellow (flashing) and bright red (flashing).

![The start screen of the original QBASIC DOS based game](/assets/bgc/fonts2.gif)
*Above: The start screen of the original QBASIC DOS based game*

I'm supposed to faithly re-create this game and already I've hit a fairly massive roadblock...

# Putting Text on the Screen in SDL2

The process is faily straightforward. Assuming you already have a renderer instance:

```fsharp
SDL_ttf.TTF_Init() |> ignore

let font = SDL_ttf.TTF_OpenFont(@"C:\windows\fonts\arial.ttf", 16)

let color = SDL.SDL_Color(r = 255uy, g = 255uy, b = 255uy)

let text = SDL_ttf.TTF_RenderText_Solid(font, "That was easy!", color)

let textSurface = 
    text
    |> NativePtr.ofNativeInt<SDL.SDL_Surface>
    |> NativePtr.read


let texture = SDL.SDL_CreateTextureFromSurface(renderer, text)

let mutable textRect = SDL.SDL_Rect(x = 100, y = 100, w = textSurface.w, h = textSurface.h )

SDL.SDL_RenderCopy(renderer, texture, 0n, &textRect) |> ignore

SDL.SDL_RenderPresent(renderer)
```

And viola!

!["That was easy!" is written to the screen.](/assets/bgc/fonts1.PNG)

*Above: how easy was that?*

Okay okay, I will admit it would be somewhat tedius to have to do that every time you wanted to put text on the screen. That's why F# has functions!

... Oh, you are concerned about performance? Surely it must be an expensive operation to keep recreating the textures over and over again. On the otherhand, maybe it is not? Honestly, I did not do any profiling. Instead, I jumped straight to a font caching and rendering system, based on no evidence what-so-ever.

# Re-implementing the QBASIC PRINT command

Text rendering in SDL2 sure makes you appreciate all that must go on behind the scenes of the humble QBASIC PRINT command.

My approach to the possible performance non-issue was to do the following:
1. Find the perfect looking VGA BIOS font. Well how good is this?
2. For each colour possible in QBASIC SCREEN 0 mode (16 colours), render each character from ASCII 32 to 126, and store into a font cache
3. Calculate the position of each char and render it from the font cache

This turned out to be easier than expected (the code was written on a train ride home*)

**Note: Geelong can be a very long ride home, however :P*

```fsharp
let getSdlColor (r,g,b) = SDL.SDL_Color(r = r, g = g, b = b)

let font = SDL_ttf.TTF_OpenFont(@"C:\windows\fonts\Perfect DOS VGA 437 Win.ttf", 16)

let fontCache =    

    let buildFontChr chr color =                          
        let sdlColor = color |> drawColor |> getSdlColor
        let chr = SDL_ttf.TTF_RenderText_Solid(font, chr, sdlColor)
    
        let chrSurface = 
            chr
            |> NativePtr.ofNativeInt<SDL.SDL_Surface>
            |> NativePtr.read
    
        SDL.SDL_CreateTextureFromSurface(renderer, chr), chrSurface

    let buildFont color = 
        [32..126]
        |> List.map (Convert.ToChar >> string)        
        |> List.map (fun s -> buildFontChr s color)

    [0..15]
    |> List.map buildFont
```

Finally, a function for rendering a text string to the screen. This code assumes a monospaced font, therefore it will not work with regular fonts.

```fsharp
let renderFont (pos:(int*int), color, text:string) =

    //apparently F# is happy to have variables named after the function
    let renderFont = fontCache |> List.item color

    let renderChar i chr =
    
        let chr,chrSurface = renderFont |> List.item (chr-32)

        let posX,posY = pos;

        let newPosX = posX + (chrSurface.w * i)
   
        let mutable chrRect = getRect(Point(newPosX, posY), Size(chrSurface.w, chrSurface.h))

        SDL.SDL_RenderCopy(renderer, chr, 0n, &chrRect) |> ignore
        ()

    text.ToCharArray()
    |> Array.map int
    |> Array.iteri renderChar
```

And of course, a function to emulate the QBASIC PRINT command:

```fsharp
let print ((row,col), color, text:string) = renderFont (((col-1)*9,(row-1)*16), color, text)
```

# No problem, now just make the text flash

This is another thing QBASIC makes deceptively simple. If you want flashing text, you just need to add 16 to the colour of choice and it will flash. Unfortunately, I don't have a video subsystem that automagically takes care of flashing, so the hard way it is.

The obvious choice is to model it after QBASIC. Regardless of when you print flashing text, all flashing text will flash at the same time. This suggests a global timer.

Well I have a game loop, so that shall be the "global timer". What follows is a crude implementation:


```fsharp
    let flashList = ResizeArray<FlashText>()

    flashList.Add({ points = (1, 1); color = 14; text = "Game in creation" })
    flashList.Add({ points = (2, 26); color = 14; text = "v2.0!" })

    let mutable ticks = 0L
    let mutable now = 0L
    let mutable flashTicks = 0L    
    let mutable inFlash = false

    
    //game loop
    while true do
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
            
```

And just like the original game, I now have flashing text

![Yes, all that work for 2 seconds of flashing text](/assets/bgc/fonts2a.gif)
*Above: Yes, all that work for 2 seconds of flashing text*

Now you might be thinking "Shouldn't you be abstracting that code behind some generic Entity update system so that the game loop does not become cluttered?"

To which I answer: "Yes, but this article is getting too long" and also "We're only concerned with putting text on the screen at this point". Abstraction comes later. And honestly, it's not much abstraction. But this is F#, functions are pretty much the only level of abstraction you need. :P

Solving this game loop clutter problem actually led to a more general solution to the problem of updating a list of game entities, which I will talk about in the next article.

## And now for some Q&A

Q: Just how slow is continually creating textures?

> A: Need to test

Q: Was that the only question?

> A: Apparently, yes
